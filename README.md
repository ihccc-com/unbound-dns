# unbound-dns

**[English](#quick-deployment-guide) | [中文](#快速部署指南)**

---

## Quick Deployment Guide

### Overview

Enterprise-grade **Unbound** recursive DNS server installation script for **Debian 13 (Trixie)** on **Azure Standard_B2ats_v2** (2 vCPU Arm64, 1 GiB RAM) virtual machines. Provides public DNS service with DNSSEC, rate limiting, UFW firewall, Fail2Ban, and systemd sandboxing.

> For full documentation see [README-en.md](README-en.md).

### Requirements

| Item | Requirement |
|------|-------------|
| OS | Debian 13 (Trixie) |
| VM | Azure Standard_B2ats_v2 (2 vCPU Arm64, 1 GiB RAM) or equivalent |
| Network | Public IP, port 53 open (ports 443/853 if DoH/DoT via NGINX) |
| Access | Root / sudo |

### Step 1 — Clone the repository

```bash
git clone https://github.com/ihccc-com/unbound-dns.git
cd unbound-dns
```

### Step 2 — (Optional) Dry-run preview

Review every action the script will take without changing anything:

```bash
sudo bash install_unbound.sh --dry-run
```

### Step 3 — Install

```bash
sudo bash install_unbound.sh install
```

The script will:
- Install and configure Unbound with DNSSEC, caching, and rate limiting
- Apply UFW firewall rules (default-deny, port 53 UDP/TCP + port 853 DoT)
- Configure Fail2Ban for DNS abuse protection
- Enable systemd sandboxing and set up log rotation (365-day retention)
- Install health-check and statistics helper scripts

### Step 4 — Verify

```bash
# Check service status
sudo systemctl status unbound

# Run health check
sudo /usr/local/bin/unbound-health-check -v

# Test DNS resolution (replace <server-ip> with your public IP)
dig @<server-ip> example.com A

# Test DNSSEC validation
dig @<server-ip> +dnssec example.com A

# DNSSEC rejection test (should return SERVFAIL)
dig @<server-ip> dnssec-failed.org A
```

### Other Commands

```bash
# Update packages, root hints, and trust anchor
sudo bash install_unbound.sh update

# Uninstall and clean up all configurations
sudo bash install_unbound.sh uninstall

# Show help
sudo bash install_unbound.sh --help
```

### Azure NSG — Required Inbound Rules

| Priority | Port | Protocol | Description |
|----------|------|----------|-------------|
| 100 | 53 | UDP | DNS queries |
| 110 | 53 | TCP | DNS queries (TCP) |
| 120 | 853 | TCP | DNS-over-TLS (DoT) |
| 130 | 443 | TCP | DNS-over-HTTPS (DoH) |
| 140 | 22 | TCP | SSH management (your IP only) |

---

## DNS-over-TLS (DoT) & DNS-over-HTTPS (DoH) Deployment

DoT (port 853) and DoH (port 443) are served by **NGINX** as a reverse proxy in front of Unbound. Complete Steps 1–4 and verify port-53 DNS works before proceeding.

### Prerequisites

1. A **domain name** (e.g. `dns.example.com`) with an **A record** pointing to your server's public IP.
2. Verify DNS propagation: `dig dns.example.com`

### Step 5 — Install NGINX & obtain a TLS certificate

```bash
# Install NGINX (nginx-full includes the stream module for DoT)
sudo apt-get update
sudo apt-get install -y nginx certbot python3-certbot-nginx

# Verify the stream module is present
nginx -V 2>&1 | grep -o with-stream   # should print: with-stream

# Stop NGINX temporarily so certbot can use port 80
sudo systemctl stop nginx

# Obtain a Let's Encrypt certificate (replace dns.example.com and admin@example.com)
sudo certbot certonly --standalone \
    -d dns.example.com \
    --agree-tos \
    --email admin@example.com \
    --non-interactive

# Verify certificate files exist
ls /etc/letsencrypt/live/dns.example.com/
# fullchain.pem  privkey.pem  cert.pem  chain.pem

# Enable automatic renewal and create a post-renewal hook to reload NGINX
sudo systemctl enable --now certbot.timer
sudo mkdir -p /etc/letsencrypt/renewal-hooks/post
cat <<'HOOK' | sudo tee /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
#!/bin/bash
systemctl reload nginx 2>/dev/null || true
HOOK
sudo chmod 755 /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
```

### Step 6 — Configure DoT (DNS-over-TLS)

NGINX's `stream` module terminates TLS on port 853 and proxies raw TCP DNS to Unbound on port 53.

**6.1 Add the stream include to `nginx.conf`**

Edit `/etc/nginx/nginx.conf` and add the following line **before** the `http {` block:

```nginx
# DoT stream configuration — must be outside the http {} block
include /etc/nginx/stream.conf.d/*.conf;

http {
    # ... existing content ...
}
```

**6.2 Create the DoT stream configuration**

```bash
sudo mkdir -p /etc/nginx/stream.conf.d

# Replace YOUR_DOMAIN with your actual domain (e.g. dns.example.com)
DOMAIN="YOUR_DOMAIN"

cat <<DOTCONF | sudo tee /etc/nginx/stream.conf.d/dns-over-tls.conf
stream {
    upstream unbound_tcp {
        server 127.0.0.1:53;
    }

    server {
        listen 853 ssl;
        listen [::]:853 ssl;

        ssl_certificate     /etc/letsencrypt/live/${DOMAIN}/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/${DOMAIN}/privkey.pem;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:DoT_SSL:10m;
        ssl_session_timeout 1h;
        ssl_session_tickets off;

        proxy_pass unbound_tcp;
        proxy_timeout 10s;
        proxy_connect_timeout 5s;
    }
}
DOTCONF
```

### Step 7 — Configure DoH (DNS-over-HTTPS)

Unbound exposes a local HTTP/2 DoH endpoint on `127.0.0.1:8443`. NGINX terminates public TLS on port 443 and forwards requests to that endpoint.

**7.1 Enable Unbound's DoH backend**

```bash
cat <<'DOHCONF' | sudo tee /etc/unbound/unbound.conf.d/03-doh.conf
server:
    interface: 127.0.0.1@8443
    https-port: 8443
    http-endpoint: "/dns-query"
    http-notls-downstream: yes
DOHCONF

# Validate and reload Unbound
sudo unbound-checkconf
sudo unbound-control reload

# Verify Unbound is listening on 8443
sudo ss -tlnp | grep 8443
```

> **Note**: `http-notls-downstream: yes` requires Unbound 1.17+. Debian 13 ships Unbound 1.19+.

**7.2 Create the NGINX DoH site**

```bash
# Open port 443 in the firewall (port 853 was already opened by the install script)
sudo ufw allow 443/tcp

# Replace YOUR_DOMAIN with your actual domain (e.g. dns.example.com)
DOMAIN="YOUR_DOMAIN"

cat <<DOHSITE | sudo tee /etc/nginx/sites-available/dns-over-https
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name ${DOMAIN};

    ssl_certificate     /etc/letsencrypt/live/${DOMAIN}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/${DOMAIN}/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:DoH_SSL:10m;
    ssl_session_timeout 1h;
    ssl_session_tickets off;

    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;

    # DoH endpoint (RFC 8484)
    # grpc_pass is required because Unbound's DoH module (libnghttp2)
    # strictly requires HTTP/2 and rejects HTTP/1.1 connections.
    location /dns-query {
        grpc_pass grpc://127.0.0.1:8443;
        grpc_set_header Content-Type \$content_type;
        grpc_set_header Host \$host;
        grpc_set_header X-Real-IP \$remote_addr;
        grpc_connect_timeout 5s;
        grpc_send_timeout 10s;
        grpc_read_timeout 10s;
        client_max_body_size 4k;
    }

    location /health {
        access_log off;
        return 200 "OK\n";
        add_header Content-Type text/plain;
    }

    location / {
        return 404;
    }
}

server {
    listen 80;
    listen [::]:80;
    server_name ${DOMAIN};
    return 301 https://\$host\$request_uri;
}
DOHSITE

# Enable site and remove default
sudo ln -sf /etc/nginx/sites-available/dns-over-https /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default

# Test configuration syntax and start NGINX
sudo nginx -t
sudo systemctl enable --now nginx

# Verify NGINX is listening on 443 and 853
sudo ss -tlnp | grep -E ':(443|853)'
```

### Step 8 — Verify DoT and DoH

**Test DoT:**

```bash
# Install kdig (knot-dnsutils) for DoT testing
sudo apt-get install -y knot-dnsutils

# Test from the server itself
kdig @127.0.0.1 +tls -p 853 example.com A

# Test from an external machine (replace with your domain)
kdig @dns.example.com +tls example.com A

# Check the TLS certificate on port 853
echo | openssl s_client -connect dns.example.com:853 -servername dns.example.com 2>/dev/null \
    | openssl x509 -noout -subject -dates
```

**Test DoH:**

```bash
# POST method (RFC 8484 wire format) — queries example.com A
echo -n 'q80BAAABAAAAAAAAB2V4YW1wbGUDY29tAAABAAE=' | base64 -d | \
    curl -sSf -H 'content-type: application/dns-message' \
    --data-binary @- \
    'https://dns.example.com/dns-query' | od -A x -t x1

# GET method (base64url-encoded dns parameter)
curl -sSf 'https://dns.example.com/dns-query?dns=q80BAAABAAAAAAAAB2V4YW1wbGUDY29tAAABAAE' \
    | od -A x -t x1

# Health check endpoint
curl -sSf https://dns.example.com/health   # should return: OK
```

> **Note**: Unbound's DoH only supports RFC 8484 wire format (`application/dns-message`). JSON (`application/dns-json`) is **not** supported.

### Step 9 — Configure Clients

| Platform | Protocol | Configuration |
|----------|----------|---------------|
| Android 9+ | DoT | Settings → Network → Private DNS → enter `dns.example.com` |
| Windows 11 | DoH | Settings → Network → DNS → Manual, enable "DNS over HTTPS", template: `https://dns.example.com/dns-query` |
| Firefox | DoH | Settings → Privacy & Security → DNS over HTTPS → Custom: `https://dns.example.com/dns-query` |
| iOS 14+ / macOS | DoH | Install a `.mobileconfig` profile (see [README-en.md](README-en.md) for the full profile XML) |

### DoT/DoH Troubleshooting

```bash
# NGINX error logs
sudo tail -20 /var/log/nginx/error.log

# Check NGINX is listening on 853 and 443
sudo ss -tlnp | grep -E ':(443|853)'

# Check Unbound DoH backend is listening on 8443
sudo ss -tlnp | grep ':8443'

# Test Unbound DoH backend directly (HTTP/2 required)
curl --http2-prior-knowledge -sSf \
    'http://127.0.0.1:8443/dns-query?dns=q80BAAABAAAAAAAAB2V4YW1wbGUDY29tAAABAAE' \
    | od -A x -t x1

# Check TLS certificates
sudo certbot certificates

# Test certificate renewal
sudo certbot renew --dry-run

# Validate all configs
sudo nginx -t && sudo unbound-checkconf
```

**Common issues:**
- **Port 853 "Connection refused"**: Check that `include /etc/nginx/stream.conf.d/*.conf;` is **outside** the `http {}` block in `nginx.conf`.
- **DoH "502 Bad Gateway"**: Ensure Unbound listens on 8443 (`ss -tlnp | grep 8443`). Check `unbound-checkconf` for `https-port` errors.
- **DoH direct test "Recv failure"**: You must use `--http2-prior-knowledge` when bypassing NGINX because Unbound requires HTTP/2.
- **Certificate errors**: Run `sudo certbot certificates` and verify the domain matches.

---

## 快速部署指南

### 项目简介

面向 **Debian 13 (Trixie)** 和 **Azure Standard_B2ats_v2**（2 vCPU Arm64，1 GiB RAM）的企业级 **Unbound** 递归 DNS 服务器安装脚本。提供带 DNSSEC、速率限制、UFW 防火墙、Fail2Ban 和 systemd 沙箱的公共 DNS 服务。

> 完整文档请参阅 [README.zh-CN.md](README.zh-CN.md)。

### 系统要求

| 项目 | 要求 |
|------|------|
| 操作系统 | Debian 13 (Trixie) |
| 虚拟机 | Azure Standard_B2ats_v2（2 vCPU Arm64，1 GiB RAM）或同等配置 |
| 网络 | 公网 IP，端口 53 已开放（如需 DoH/DoT 还需开放 443/853） |
| 权限 | Root / sudo |

### 第一步 — 克隆仓库

```bash
git clone https://github.com/ihccc-com/unbound-dns.git
cd unbound-dns
```

### 第二步 — （可选）试运行预览

在不做任何更改的情况下，预览脚本将执行的所有操作：

```bash
sudo bash install_unbound.sh --dry-run
```

### 第三步 — 安装

```bash
sudo bash install_unbound.sh install
```

脚本将自动完成以下操作：
- 安装并配置 Unbound，启用 DNSSEC、缓存和速率限制
- 应用 UFW 防火墙规则（默认拒绝，开放端口 53 UDP/TCP 和 853 DoT）
- 配置 Fail2Ban，防御 DNS 滥用
- 启用 systemd 沙箱并设置日志轮转（保留 365 天）
- 安装健康检查和统计信息辅助脚本

### 第四步 — 验证

```bash
# 检查服务状态
sudo systemctl status unbound

# 运行健康检查
sudo /usr/local/bin/unbound-health-check -v

# 测试 DNS 解析（将 <server-ip> 替换为您的公网 IP）
dig @<server-ip> example.com A

# 测试 DNSSEC 验证
dig @<server-ip> +dnssec example.com A

# DNSSEC 拒绝测试（应返回 SERVFAIL）
dig @<server-ip> dnssec-failed.org A
```

### 其他命令

```bash
# 更新软件包、根提示文件和信任锚
sudo bash install_unbound.sh update

# 卸载并清理所有配置
sudo bash install_unbound.sh uninstall

# 显示帮助信息
sudo bash install_unbound.sh --help
```

### Azure NSG — 所需入站规则

| 优先级 | 端口 | 协议 | 说明 |
|--------|------|------|------|
| 100 | 53 | UDP | DNS 查询 |
| 110 | 53 | TCP | DNS 查询（TCP） |
| 120 | 853 | TCP | DNS-over-TLS（DoT） |
| 130 | 443 | TCP | DNS-over-HTTPS（DoH） |
| 140 | 22 | TCP | SSH 管理（仅限您的 IP） |

---

## DNS-over-TLS (DoT) 和 DNS-over-HTTPS (DoH) 部署教程

DoT（端口 853）和 DoH（端口 443）由 **NGINX** 作为 Unbound 前端的反向代理提供。请先完成第一步至第四步，确认端口 53 的 DNS 解析正常后再继续。

### 前置条件

1. 准备一个**域名**（如 `dns.example.com`），通过 **A 记录**指向服务器公网 IP。
2. 验证 DNS 传播：`dig dns.example.com`

### 第五步 — 安装 NGINX 并获取 TLS 证书

```bash
# 安装 NGINX（nginx-full 包含 DoT 所需的 stream 模块）
sudo apt-get update
sudo apt-get install -y nginx certbot python3-certbot-nginx

# 验证 stream 模块可用
nginx -V 2>&1 | grep -o with-stream   # 应输出：with-stream

# 临时停止 NGINX 以便 certbot 使用端口 80
sudo systemctl stop nginx

# 获取 Let's Encrypt 证书（将 dns.example.com 和 admin@example.com 替换为您的实际值）
sudo certbot certonly --standalone \
    -d dns.example.com \
    --agree-tos \
    --email admin@example.com \
    --non-interactive

# 验证证书文件存在
ls /etc/letsencrypt/live/dns.example.com/
# fullchain.pem  privkey.pem  cert.pem  chain.pem

# 启用自动续期，并创建续期后重载 NGINX 的钩子脚本
sudo systemctl enable --now certbot.timer
sudo mkdir -p /etc/letsencrypt/renewal-hooks/post
cat <<'HOOK' | sudo tee /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
#!/bin/bash
systemctl reload nginx 2>/dev/null || true
HOOK
sudo chmod 755 /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
```

### 第六步 — 配置 DoT（DNS-over-TLS）

NGINX 的 `stream` 模块在端口 853 终止 TLS，并将原始 TCP DNS 流量代理到 Unbound 的端口 53。

**6.1 在 `nginx.conf` 中添加 stream 包含指令**

编辑 `/etc/nginx/nginx.conf`，在 `http {` 块**之前**添加以下行：

```nginx
# DoT stream 配置 — 必须在 http {} 块外部
include /etc/nginx/stream.conf.d/*.conf;

http {
    # ... 现有内容 ...
}
```

**6.2 创建 DoT stream 配置文件**

```bash
sudo mkdir -p /etc/nginx/stream.conf.d

# 将 YOUR_DOMAIN 替换为您的实际域名（如 dns.example.com）
DOMAIN="YOUR_DOMAIN"

cat <<DOTCONF | sudo tee /etc/nginx/stream.conf.d/dns-over-tls.conf
stream {
    upstream unbound_tcp {
        server 127.0.0.1:53;
    }

    server {
        listen 853 ssl;
        listen [::]:853 ssl;

        ssl_certificate     /etc/letsencrypt/live/${DOMAIN}/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/${DOMAIN}/privkey.pem;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:DoT_SSL:10m;
        ssl_session_timeout 1h;
        ssl_session_tickets off;

        proxy_pass unbound_tcp;
        proxy_timeout 10s;
        proxy_connect_timeout 5s;
    }
}
DOTCONF
```

### 第七步 — 配置 DoH（DNS-over-HTTPS）

Unbound 在 `127.0.0.1:8443` 暴露本地 HTTP/2 DoH 端点，NGINX 在公网端口 443 终止 TLS 并将请求转发到该端点。

**7.1 启用 Unbound 的 DoH 后端**

```bash
cat <<'DOHCONF' | sudo tee /etc/unbound/unbound.conf.d/03-doh.conf
server:
    interface: 127.0.0.1@8443
    https-port: 8443
    http-endpoint: "/dns-query"
    http-notls-downstream: yes
DOHCONF

# 验证配置并重载 Unbound
sudo unbound-checkconf
sudo unbound-control reload

# 验证 Unbound 正在监听 8443 端口
sudo ss -tlnp | grep 8443
```

> **注意**：`http-notls-downstream: yes` 需要 Unbound 1.17+。Debian 13 附带 Unbound 1.19+。

**7.2 创建 NGINX DoH 站点配置**

```bash
# 放通端口 443（安装脚本已自动放通 853）
sudo ufw allow 443/tcp

# 将 YOUR_DOMAIN 替换为您的实际域名（如 dns.example.com）
DOMAIN="YOUR_DOMAIN"

cat <<DOHSITE | sudo tee /etc/nginx/sites-available/dns-over-https
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name ${DOMAIN};

    ssl_certificate     /etc/letsencrypt/live/${DOMAIN}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/${DOMAIN}/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:DoH_SSL:10m;
    ssl_session_timeout 1h;
    ssl_session_tickets off;

    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;

    # DoH 端点（RFC 8484）
    # 必须使用 grpc_pass，因为 Unbound DoH 模块（libnghttp2）严格要求 HTTP/2，
    # 会拒绝 HTTP/1.1 连接。NGINX 的 proxy_pass 仅支持 HTTP/1.x。
    location /dns-query {
        grpc_pass grpc://127.0.0.1:8443;
        grpc_set_header Content-Type \$content_type;
        grpc_set_header Host \$host;
        grpc_set_header X-Real-IP \$remote_addr;
        grpc_connect_timeout 5s;
        grpc_send_timeout 10s;
        grpc_read_timeout 10s;
        client_max_body_size 4k;
    }

    location /health {
        access_log off;
        return 200 "OK\n";
        add_header Content-Type text/plain;
    }

    location / {
        return 404;
    }
}

server {
    listen 80;
    listen [::]:80;
    server_name ${DOMAIN};
    return 301 https://\$host\$request_uri;
}
DOHSITE

# 启用站点，删除默认站点
sudo ln -sf /etc/nginx/sites-available/dns-over-https /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default

# 测试配置语法并启动 NGINX
sudo nginx -t
sudo systemctl enable --now nginx

# 验证 NGINX 正在监听 443 和 853
sudo ss -tlnp | grep -E ':(443|853)'
```

### 第八步 — 验证 DoT 和 DoH

**测试 DoT：**

```bash
# 安装 kdig（knot-dnsutils）用于 DoT 测试
sudo apt-get install -y knot-dnsutils

# 从服务器本机测试
kdig @127.0.0.1 +tls -p 853 example.com A

# 从外部机器测试（替换为您的域名）
kdig @dns.example.com +tls example.com A

# 检查端口 853 的 TLS 证书
echo | openssl s_client -connect dns.example.com:853 -servername dns.example.com 2>/dev/null \
    | openssl x509 -noout -subject -dates
```

**测试 DoH：**

```bash
# POST 方法（RFC 8484 线格式）— 查询 example.com A 记录
echo -n 'q80BAAABAAAAAAAAB2V4YW1wbGUDY29tAAABAAE=' | base64 -d | \
    curl -sSf -H 'content-type: application/dns-message' \
    --data-binary @- \
    'https://dns.example.com/dns-query' | od -A x -t x1

# GET 方法（base64url 编码的 dns 参数）
curl -sSf 'https://dns.example.com/dns-query?dns=q80BAAABAAAAAAAAB2V4YW1wbGUDY29tAAABAAE' \
    | od -A x -t x1

# 健康检查端点
curl -sSf https://dns.example.com/health   # 应返回：OK
```

> **注意**：Unbound 的 DoH 仅支持 RFC 8484 线格式（`application/dns-message`），**不**支持 JSON 格式（`application/dns-json`）。

### 第九步 — 客户端配置

| 平台 | 协议 | 配置方法 |
|------|------|----------|
| Android 9+ | DoT | 设置 → 网络 → 私人 DNS → 输入 `dns.example.com` |
| Windows 11 | DoH | 设置 → 网络 → DNS → 手动，启用"DNS over HTTPS"，模板：`https://dns.example.com/dns-query` |
| Firefox | DoH | 设置 → 隐私与安全 → DNS over HTTPS → 自定义：`https://dns.example.com/dns-query` |
| iOS 14+ / macOS | DoH | 安装 `.mobileconfig` 描述文件（完整 XML 见 [README.zh-CN.md](README.zh-CN.md)） |

### DoT/DoH 故障排查

```bash
# 查看 NGINX 错误日志
sudo tail -20 /var/log/nginx/error.log

# 检查 NGINX 是否在 853 和 443 端口监听
sudo ss -tlnp | grep -E ':(443|853)'

# 检查 Unbound DoH 后端是否在 8443 端口监听
sudo ss -tlnp | grep ':8443'

# 直接测试 Unbound DoH 后端（必须使用 HTTP/2）
curl --http2-prior-knowledge -sSf \
    'http://127.0.0.1:8443/dns-query?dns=q80BAAABAAAAAAAAB2V4YW1wbGUDY29tAAABAAE' \
    | od -A x -t x1

# 检查 TLS 证书状态
sudo certbot certificates

# 测试证书续期（模拟）
sudo certbot renew --dry-run

# 验证所有配置
sudo nginx -t && sudo unbound-checkconf
```

**常见问题：**
- **端口 853 "Connection refused"**：检查 `include /etc/nginx/stream.conf.d/*.conf;` 是否在 `nginx.conf` 的 `http {}` 块**外部**。
- **DoH "502 Bad Gateway"**：确认 Unbound 正在监听 8443（`ss -tlnp | grep 8443`），并运行 `unbound-checkconf` 排查 `https-port` 错误。
- **DoH 直接测试"Recv failure"**：绕过 NGINX 直接测试时必须使用 `--http2-prior-knowledge`，因为 Unbound 仅接受 HTTP/2。
- **证书错误**：运行 `sudo certbot certificates` 检查证书状态，确认域名匹配。