---
title: "在 VPS 上部署 nginx"
summary: "介绍 nginx 在 VPS 上的配置结构、location 语法、常用命令、静态站点与反向代理、TLS 证书的挂载"
date: 2025-10-17T18:20:00+08:00
categories: [Infra, Linux]
tags: [nginx, vps, reverse-proxy, tls]
---

## 概览

- 参考文档：[nginx documentation](https://nginx.org/en/docs/)
- `nginx` 是 HTTP 服务器和反向代理服务器，也提供内容缓存和负载均衡
- `nginx` 统一承担以下职责：
  - 静态文件直接返回
  - 其余请求转发给 `127.0.0.1` 上的后端服务
  - 对外只暴露 `80`/`443`
  - TLS、压缩、限流、日志集中处理
  - 后端只监听 `127.0.0.1`，不需要各自处理证书

  ```mermaid
  flowchart LR
      C["客户端"] -->|"80/443"| N["nginx"]
      N -->|"静态文件"| S["/var/www"]
      N -->|"proxy_pass"| A["应用(127.0.0.1:8080)"]
      N -->|"proxy_pass"| B["另一个服务(127.0.0.1:9090)"]
  ```

## 快速开始

- 主配置 `/etc/nginx/nginx.conf`，`include` 决定站点配置从哪里加载：

  ```nginx
  user  nginx;                                      # 运行用户
  worker_processes  auto;                           # 工作进程数，auto 跟随 CPU 核数
  error_log  /var/log/nginx/error.log warn;         # 错误日志与级别
  pid        /var/run/nginx.pid;                    # 主进程 PID 文件

  events {
      worker_connections  1024;                     # 单个 worker 的最大连接数
  }

  http {
      include       /etc/nginx/mime.types;          # MIME 类型映射
      default_type  application/octet-stream;       # 未识别类型时的默认值

      sendfile        on;                           # 零拷贝发送文件
      keepalive_timeout  65;                        # 长连接空闲超时

      include /etc/nginx/conf.d/*.conf;             # 站点配置入口
  }
  ```

  - Debian/Ubuntu：`/etc/nginx/sites-enabled/*`
  - RHEL 系 / Arch Linux：`/etc/nginx/conf.d/*.conf`
  - Alpine Linux：`/etc/nginx/http.d/*.conf`

- 一个反向代理站点，写到 `/etc/nginx/conf.d/app.example.com.conf`：

  ```nginx
  server {
      listen 80;                                    # 对外监听端口
      server_name app.example.com;                  # 匹配的域名

      location / {
          proxy_pass http://127.0.0.1:8080;         # 转发目标
          proxy_set_header Host $host;              # 保留原始 Host
          proxy_set_header X-Real-IP $remote_addr;  # 传递真实客户端 IP
      }
  }
  ```

- 重载配置：

  ```bash
  sudo nginx -t && sudo systemctl reload nginx    # 先检查语法，通过后热重载
  ```

- 静态站点，`Hugo` 构建产物在 `/var/www/blog`：

  ```nginx
  server {
      listen 80;
      server_name example.com;
      root /var/www/blog;                           # 构建产物目录
      index index.html;                             # 默认首页

      location / {
          try_files $uri $uri/ =404;                # 依次尝试文件、目录，最后回退 404
      }

      location ~* \.(css|js|png|jpg|woff2)$ {
          expires 30d;                              # 带文件名指纹的资源加长缓存
      }
  }
  ```

- 反向代理如果需要支持长连接，需要添加 `map` 块将普通请求与长连接请求区分开：

  ```nginx
  map $http_upgrade $connection_upgrade {
      default upgrade;                              # 有 Upgrade 头时透传 upgrade
      ''      close;                                # 普通请求关闭连接升级
  }

  server {
      listen 80;
      server_name app.example.com;

      location / {
          proxy_pass http://127.0.0.1:8080;
          proxy_http_version 1.1;                   # 长连接需要 1.1
          proxy_set_header Upgrade $http_upgrade;   # 透传 WebSocket 升级头
          proxy_set_header Connection $connection_upgrade;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_read_timeout 1h;                    # 默认 60s 会掐断空闲连接
          client_max_body_size 100m;                # 默认 1m，上传大文件会 413
      }
  }
  ```

- 挂证书，`HTTP` 重定向到 `HTTPS`：

  ```nginx
  server {
      listen 80;
      server_name example.com;
      return 301 https://$host$request_uri;         # 明文请求重定向到 HTTPS
  }

  server {
      listen 443 ssl;
      server_name example.com;

      ssl_certificate     /etc/nginx/certs/example.com.pem;    # 证书链
      ssl_certificate_key /etc/nginx/certs/example.com.key;    # 私钥

      location / {
          proxy_pass http://127.0.0.1:8080;
      }
  }
  ```

## 配置结构

- 站点配置子目录随发行版不同：
  - Debian/Ubuntu：`sites-available/` 存配置，`sites-enabled/` 放软链接启用
  - RHEL 系：`conf.d/` 下放 `.conf` 文件，无需软链接
  - Arch Linux：站点配置放在 `conf.d/`
  - Alpine Linux：`http.d/` 放虚拟主机配置，`conf.d/` 仅用于根上下文片段
- 配置字段：
  - `http`：全局 `HTTP` 层参数
  - `http.server`：一个 `listen` + `server_name` 组合对应一个站点
  - `http.server.location`：站点内的路径规则，决定请求交给谁
  - `http.map`：根据一个变量的值生成另一个变量，只能放在 `http` 层，基本形式为 `map $源变量 $目标变量 { 值 结果; ... }`
    - 第一个参数是来源，比如 `$http_upgrade`
    - 第二个参数是生成的新变量名，比如 `$connection_upgrade`
    - 块内每行是"匹配值 结果"，`default` 匹配其余情况，空字符串用 `''` 表示

### location 语法

```nginx
location = /exact { }        # 精确匹配
location ^~ /static/ { }     # 前缀匹配，命中后不再检查正则
location ~ \.php$ { }        # 区分大小写的正则匹配
location ~* \.(jpg|png)$ { } # 不区分大小写的正则匹配
location /api/ { }           # 普通前缀匹配
```

- 匹配优先级从高到低：精确`=`、`^~`前缀、正则`~`/`~*`、普通前缀；正则会按书写顺序取第一个匹配项
- 常用内部指令：
  - `root`：把 URI 直接拼到指定目录后面
  - `alias`：用指定目录替换 `location` 匹配到的前缀
  - `try_files`：按顺序尝试文件，全部失败时走最后的回退，比如 `=404` 或转发到后端
  - `proxy_pass`：把请求转给上游，配合 `proxy_set_header` 传递真实 IP 与协议

## 常用命令

```bash
nginx -t                      # 检查配置语法
nginx -T                      # 打印合并后的完整配置
nginx -s reload               # 向主进程发送重载信号，等价于 systemctl reload nginx
nginx -s quit                 # 优雅退出
```

## 静态站点

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/blog;              # 构建产物目录
    index index.html;                # 默认首页

    location / {
        try_files $uri $uri/ =404;   # 依次尝试文件、目录，最后回退 404
    }

    location ~* \.(css|js|png|jpg|woff2)$ {
        expires 30d;                 # 带文件名指纹的资源加长缓存
    }
}
```

## 反向代理

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;                 # 有 Upgrade 头时透传 upgrade
    ''      close;                   # 普通请求关闭连接升级
}

server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;                    # 长连接需要 1.1
        proxy_set_header Upgrade $http_upgrade;    # 透传 WebSocket 升级头
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 1h;                     # 默认 60s 会掐断空闲连接
        client_max_body_size 100m;                 # 默认 1m，上传大文件会 413
    }
}
```

## TLS

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;    # 明文请求重定向到 HTTPS
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/nginx/certs/example.com.pem;    # 证书链
    ssl_certificate_key /etc/nginx/certs/example.com.key;    # 私钥

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

- 证书来源有两种：
  - `Let's Encrypt`：`certbot --nginx -d example.com` 自动签发并接管续期
  - `Cloudflare Origin CA`：`15` 年有效期，适合源站只接受 `Cloudflare` 回源的场景，生成方法见[用 Cloudflare 配置 HTTPS](/2025/10/17/cloudflare-https/)
