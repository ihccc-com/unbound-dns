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
| 130 | 443 | TCP | DNS-over-HTTPS (DoH, opened by NGINX script) |

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
| 130 | 443 | TCP | DNS-over-HTTPS（DoH，由 NGINX 脚本开放） |