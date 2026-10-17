---
title: "自建 ntfy 节点把通知推到手机"
summary: "自建 ntfy 节点将通知推送到 Android/iPhone，覆盖域名 HTTPS 与裸 IP 两种部署、访问控制，并以 Claude Code Stop hook 与 CI 为例接入日常工具"
date: 2026-10-17T15:14:03+08:00
categories: [Tools & Utilities, 3P Tools, ntfy]
tags: [ntfy, push, claude-code]
---
## 概述

- 参考文档：[ntfy Documentation](https://docs.ntfy.sh/)
- 一条通知从发布到手机的链路：

  ```mermaid
  flowchart TD
      P["发布方(curl / 脚本 / CI / Claude Code)"] -->|"POST https://server/topic"| N["自建 ntfy 节点"]
      N -->|"常驻 WebSocket"| A["Android App"]
      N -->|"poll request 经 ntfy.sh 中继"| I["APNS"]
      I --> E["iPhone App"]
  ```

- `ntfy` 是基于 `HTTP` 的发布/订阅通知服务，官方提供 `ntfy.sh` 与各平台客户端，节点也可以自建
- 自建节点的好处是消息不经过第三方、路径完全可控，还可以只对内网/特定网段开放，适合配合自建 CI、监控等内部系统

## `ntfy` 是什么

- `ntfy` 的模型极简：
  - 收发双方只约定一个 **topic**(主题名)，发布方 `POST` 到 `https://<server>/<topic>`，请求正文就是通知内容
  - 订阅方在 App 或网页里订阅同一个 topic 即可收到，无需注册账号
  - 附加信息走 `HTTP` 头：`Title`(标题)、`Priority`(优先级)、`Tags`(展示为 emoji)、`Click`(点击跳转的 URL)
- **topic 名即凭证**：任何知道 topic 名的人都能发布与订阅，公网节点要配访问控制(见后文)

## 自建节点

- 服务端是一个单二进制(`ntfy serve`)，配置文件为 `/etc/ntfy/server.yml`，核心字段：

  ```yaml
  base-url: "https://ntfy.ishian.me"
  listen-http: ":80"
  cache-file: "/var/lib/ntfy/cache.db"
  ```

- `Docker` 部署：

  ```bash
  docker run -d --name ntfy \
    -p 80:80 \
    -v /var/lib/ntfy:/var/lib/ntfy \
    -v /etc/ntfy/server.yml:/etc/ntfy/server.yml \
    binwiederhier/ntfy serve
  ```

- `systemd` 直跑二进制(不依赖 `Docker`)，unit 写到 `/etc/systemd/system/ntfy.service`：

  ```ini
  [Unit]
  Description=ntfy server
  After=network.target

  [Service]
  User=ntfy
  ExecStart=/usr/bin/ntfy serve
  Restart=on-failure
  StateDirectory=ntfy

  [Install]
  WantedBy=multi-user.target
  ```

  ```bash
  systemctl enable --now ntfy
  ```

  - `StateDirectory=ntfy` 会自动创建并授权 `/var/lib/ntfy`，对应 `server.yml` 里的 `cache-file` 与 `auth-file` 路径

### 域名 + `HTTPS`

- 手机在蜂窝网络下访问，`TLS` 是基本要求，标准做法是域名 + 反向代理(`Caddy`/`nginx`)终结 `TLS`，再转发到 `ntfy` 的 `listen-http`
- 转发配置以`nginx`为例，`WebSocket` 升级头必须带上，否则 App 的常驻连接建立不起来：

  ```nginx
  server {
      listen 443 ssl;
      server_name ntfy.ishian.me;
      ssl_certificate /etc/nginx/certs/ntfy.ishian.me.pem;
      ssl_certificate_key /etc/nginx/certs/ntfy.ishian.me.key;
      location / {
          proxy_pass http://127.0.0.1:8080;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
          proxy_set_header Host $host;
          proxy_read_timeout 1h;
      }
  }
  ```

  - `server.yml` 的 `listen-http` 相应改成 `"127.0.0.1:8080"`，让 `ntfy` 只在本机监听，`443` 的流量全部走代理
  - `nginx` 的完整用法见[在 VPS 上部署 nginx](/2025/10/17/nginx-vps/)，证书来源见[用 Cloudflare 配置 HTTPS](/2025/10/17/cloudflare-https/)
- `base-url` 必须与 App 里填写的默认服务器地址**完全一致**，否则订阅看起来正常但推送时灵时不灵，这是最常见的一类配置错误
- **`iPhone` 即时推送的特殊处理**：
  - `iOS` 不允许 App 常驻后台连接，即时推送必须走 `APNS`，而 `APNS` 通道只有官方 `ntfy.sh` 接入
  - 自建节点在 `server.yml` 中配置 `upstream-base-url: "https://ntfy.sh"` 后，消息到达时会向 `ntfy.sh` 转发一条轻量的 poll request，由它经 `APNS` 唤醒手机上的 App，App 再回到**自己的节点**拉取完整内容
  - 也就是说消息正文只落自己的节点，`ntfy.sh` 只充当"敲门的门铃"
- 不配 `upstream-base-url` 时 iOS 端退化为后台轮询，延迟以分钟计

### 裸 `IP`

- 没有域名时直接用 `http://<IP>:<port>`：
  - **`Android`**：App 允许明文 `HTTP`(需要在订阅设置里逐条允许)，即时送达靠 App 与节点之间的常驻 `WebSocket` 连接，官方实测约 `17` 小时耗电 `0-1%`，注意把 App 从电池优化中排除，否则连接会被系统杀掉
  - **`iPhone`**：App 强制要求 `HTTPS`，裸 `IP` 拿不到受信证书，基本不可用，想要 iOS 端就回到域名方案
- `WebSocket` 即时送达不依赖任何 `Google` 服务，`F-Droid` 构建的 App 同样可用

### 访问控制

- topic 名即凭证，公网节点建议关掉匿名访问：

  ```yaml
  auth-file: "/var/lib/ntfy/user.db"
  auth-default-access: deny-all
  ```

  ```bash
  ntfy user add ishian
  ntfy access ishian claude-code rw
  ntfy token add ishian
  ```

- 之后发布与订阅都要带凭证：`curl` 加 `-H "Authorization: Bearer tk_xxx"`，App 在服务器设置里填同一个 token

## 发布与订阅

- 最小验证只需要一条 `curl`：

  ```bash
  curl -H "Title: hello" -H "Tags: wave" \
    -d "first push" \
    https://ntfy.ishian.me/claude-code
  ```

- `Android`：
  - App 中添加订阅，服务器填自己的节点地址，topic 填 `claude-code`
  - Play 版走 `FCM`，但自建节点默认没有配 `FCM` 凭证，实际用的是 `WebSocket` 即时连接，务必关闭对该 App 的电池优化
  - `F-Droid` 版不含 `Google` 依赖，同样走 `WebSocket`
- `iPhone`：
  - App 的默认服务器改成 `https://ntfy.ishian.me`(与 `base-url` 严格一致)，配合节点侧的 `upstream-base-url` 即可获得 `APNS` 即时推送
  - 节点没配 `upstream` 时，只能打开 App 时手动拉取
- 排查顺序：先用 `curl` 验证"手机能收到"，再接入具体工具，两端分开定位

## 接入示例：`Claude Code` 的 `Stop` hook

- `Claude Code` 在 `settings.json` 中配置 hooks，按事件名触发外部命令，适合"跑长任务时不盯终端"的场景，用到两个事件：
  - **`Stop`**：主回合结束(`Claude` 完成响应)时触发，本例的主角
  - **`Notification`**：权限确认等待、空闲提示等场景触发，`stdin` 中带 `message` 字段，适合再挂一条"需要人工介入"的通知
- hook 命令从 `stdin` 收到一段 `JSON`，`Stop` 事件的常用字段：
  - `last_assistant_message`：`Claude` 最后一条回复的正文，直接用它做通知内容，不必解析 `transcript_path`——transcript 是异步写入的 `JSONL`，Stop 时刻未必包含最后一条消息
  - `cwd`、`session_id`、`permission_mode` 等上下文字段，可拼进通知标题
  - `stop_hook_active`：为 `true` 表示本次 Stop 由某个 Stop hook 的反馈引起，脚本里要检查并直接退出，否则可能形成循环
- 退出码语义(对 `Stop` 尤其重要)：
  - `exit 0`：正常结束，纯通知类 hook 的唯一正确出口
  - `exit 2`：**阻止停止**，把 `Claude` 拉回来继续干活，通知脚本绝不能走到这个分支
  - 其他非零：非阻塞错误，界面上提示 hook error

### 配置

- 用户级配置文件为 `~/.claude/settings.json`(设置了 `CLAUDE_CONFIG_DIR` 时在对应目录下)，`Stop` 事件不需要 `matcher`：

  ```json
  {
    "hooks": {
      "Stop": [
        {
          "hooks": [
            {
              "type": "command",
              "command": "~/.config/claude/hooks/ntfy-stop.sh",
              "timeout": 10,
              "async": true
            }
          ]
        }
      ]
    }
  }
  ```

- `~/.config/claude/hooks/ntfy-stop.sh`：

  ```bash
  #!/usr/bin/env bash
  set -euo pipefail
  input=$(cat)
  if [[ $(jq -r '.stop_hook_active' <<<"$input") == "true" ]]; then
    exit 0
  fi
  curl -fsS \
    -H "Title: Claude Code $(jq -r '.cwd' <<<"$input")" \
    -H "Tags: robot" \
    -d "$(jq -r '.last_assistant_message // "task finished"' <<<"$input" | head -c 180)" \
    https://ntfy.ishian.me/claude-code
  ```

- `head -c 180` 截断长回复，通知栏只放得下一小段；`last_assistant_message` 可能缺失(如空回合)，用 `// "task finished"` 兜底
- hooks 配置在会话启动时读取，改完需要开新会话，或用 `/hooks` 查看生效状态
- 同样的套路可以接到任何能发 `HTTP` 请求的地方：`CI` 流水线结束、`cron` 任务失败、监控告警脚本，往 topic 里 `POST` 一条即可
