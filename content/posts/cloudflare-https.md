---
title: "用 Cloudflare 配置 HTTPS"
summary: "介绍 Cloudflare 代理下的两段加密模型、边缘证书、SSL/TLS 加密模式与回源证书的选择"
date: 2025-10-17T18:20:00+08:00
categories: [Tools & Utilities, 3P Tools, Cloudflare]
tags: [cloudflare, https, tls]
---

## 概览

- 参考文档：[Cloudflare SSL/TLS](https://developers.cloudflare.com/ssl/)
- 前提：域名的 `DNS` 托管在 `Cloudflare`，并且解析记录开启了代理(橙色云朵)，证书与防护都只在代理流量上生效
- `Cloudflare` 的 `HTTPS` 是**两段加密**，两段独立配置、独立证书：

  ```mermaid
  flowchart LR
      V["访客浏览器"] -->|"HTTPS(边缘证书)"| C["Cloudflare 边缘节点"]
      C -->|"回源(SSL/TLS 模式)"| O["源站服务器"]
  ```

  - 前一段：访客到 `Cloudflare`，由**边缘证书**加密，`Cloudflare` 自动签发与续期
  - 后一段：`Cloudflare` 回到源站，由 `SSL/TLS` 加密模式决定走不走加密、验不验证证书

## 边缘证书

- `Universal SSL` 对 apex 域名与一级子域名(`*.example.com`)免费签发，激活代理后自动生效，无需手工续期
- 深一级的子域名(`a.b.example.com`)不在 `Universal SSL` 覆盖范围内，需要购买高级证书或改用 `ACME` 自签
- 配套开关在 `SSL/TLS` -> `Edge Certificates`：
  - `Always Use HTTPS`：把 `HTTP` 请求`301`重定向到 `HTTPS`，建议开启
  - `HSTS`：强制浏览器后续只走 `HTTPS`，**开启前确认全站资源都是 `HTTPS`**，否则浏览器会直接拒载
  - `Minimum TLS Version`：建议 `TLS 1.2` 起

## SSL/TLS 加密模式

- 模式决定回源那段怎么走，在 `SSL/TLS` -> `Overview` 里切换：
  - `Flexible`：回源走明文 `HTTP`，源站 `IP` 与内容在公网裸奔，只适合临时过渡
  - `Full`：回源走 `HTTPS`，但不验证源站证书，自签证书也能过，能被中间人冒充源站
  - **`Full (strict)`**：回源走 `HTTPS` 并验证证书，常规选择
- 一句话：**永远用 `Full (strict)`**，剩下的差别只是源站证书从哪来

## 回源证书

- 源站证书三种来源，按场景选择：
  - `Cloudflare Origin CA`：在 `SSL/TLS` -> `Origin Server` 生成，有效期最长 `15` 年，只被 `Cloudflare` 信任，浏览器直接访问会报错，适合源站**只接受 `Cloudflare` 回源**的服务
  - `Let's Encrypt`：公开可信，配合 `certbot` 自动续期，源站还需要被浏览器直连(灰云)时用它
  - 商业证书：有现成的就用，效果同 `Let's Encrypt`
- 用 `Origin CA` 证书时把生成的 `pem` 与私钥装到源站 `Web` 服务器，`nginx` 的挂法见[在 VPS 上部署 nginx](/2025/10/17/nginx-vps/)
- 证书装好后，把模式切到 `Full (strict)`，用浏览器访问验证证书链正常

## DNS 与代理

- 解析记录照常指向 `VPS` 的 `IP`，橙色云朵开启即享受证书与防护
- 灰色云朵是纯 `DNS`，流量不经过 `Cloudflare`，边缘证书与加密模式都不参与，源站要自己扛 `TLS`(如裸 `IP` 场景，参考[自建 ntfy 节点把通知推到手机](/2026/10/17/ntfy/))
- 源站 `IP` 暴露在解析记录里，配合防火墙只放行 `Cloudflare` 回源网段，可以挡掉直连源站的扫描
