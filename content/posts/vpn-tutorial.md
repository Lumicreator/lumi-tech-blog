---
title: "保姆级 VLESS + Reality & CDN 加速全指南"
date: 2024-05-01T00:00:00+00:00
description: "保姆级 VLESS+Reality 教程：单服务器部署直连与 CDN 备用线路，涵盖 sing-box 配置、Tunnel 工作原理与常见故障排查清单。"
tags: ["Reality", "VLESS", "CDN", "Cloudflare", "sing-box", "X-UI"]
---

> 这是一份从零到上线的专业教程，涵盖 **VLESS + Reality 直连** 与 **VMess+WS+TLS 经 CDN** 的双线路方案，附带工作原理、排错与性能优化建议。适合具备基础 Linux 经验的读者。

## 目录
1. 为什么要双线路（Reality 直连 + CDN 备用）
2. 前置条件与规划
3. 服务端部署步骤
4. 客户端配置示例
5. 如何理解 Cloudflare Tunnel 的工作方式
6. 常见 502 / 超时错误排查
7. 低延迟优化指南
8. 安全与运维清单

---

## 1. 为什么要双线路？
- **抗封锁与稳定性**：Reality 直连不依赖域名或 CDN，抗干扰强；CDN 线路可在域名遭遇干扰时快速切换。
- **速度与体验**：直连延迟低、握手短；跨境链路拥塞或晚高峰时，可用 CDN 线路借助边缘节点绕路提速。
- **灵活扩展**：两条线路并行，客户端可按延迟/健康度自动切换，也可手动一键切换。

## 2. 前置条件与规划
- **服务器**：境外 VPS（Debian 11+/Ubuntu 22.04，1C1G+，root 权限）。
- **域名**：用于 CDN 方案；Reality 直连可不依赖域名。
- **基础能力**：SSH、编辑配置、查看日志。
- **端口规划**：建议 Reality 用 443/8443 或高位随机端口；WS 端口 443/8443，与防火墙放行一致。

## 3. 服务端部署
### 3.1 开启 BBR（推荐）
```bash
uname -r
cat <<'EOF' | sudo tee /etc/sysctl.d/99-bbr.conf
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
EOF
sudo sysctl --system
sysctl net.ipv4.tcp_congestion_control net.core.default_qdisc
```
输出含 `bbr` 和 `fq` 即生效。

### 3.2 安装 sing-box（Reality 直连）
**方案 A：一键脚本（快速上手）**
```bash
bash <(curl -fsSL https://raw.githubusercontent.com/ok233wz/reality-ezpz/main/install.sh)
```
安装后记录：`uuid`、端口、`serverName`、`short_id`、Public/Private Key。

**方案 B：官方脚本（便于自定义）**
```bash
sudo bash -c 'wget -O- https://sing-box.app/install.sh | bash'
```
关键字段（`/etc/sing-box/config.json`）：
- `type: vless`，`flow: xtls-rprx-vision`
- `reality`: `private_key`/`public_key`、`server_name`(SNI)、`short_id`
- `dest/server_name` 选热门 HTTPS 站点，例如 `www.microsoft.com:443`

**Reality 入站示例**
```json
{
  "inbounds": [
    {
      "type": "vless",
      "listen": "0.0.0.0",
      "listen_port": 443,
      "users": [{"uuid": "<uuid>", "flow": "xtls-rprx-vision"}],
      "tls": {
        "enabled": true,
        "reality": {
          "dest": "www.microsoft.com:443",
          "server_name": "www.microsoft.com",
          "private_key": "<private-key>",
          "short_id": "a1b2c3d4"
        }
      }
    }
  ]
}
```

### 3.3 安装 X-UI（VMess+WS+TLS，经 CDN）
```bash
curl -fsSL https://get.docker.com | sh
sudo systemctl enable --now docker

mkdir -p /opt/x-ui && cd /opt/x-ui
cat > docker-compose.yml <<'EOF'
version: '3'
services:
  xui:
    image: enwaiax/x-ui:latest
    container_name: x-ui
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./db/:/etc/x-ui/
      - ./cert/:/root/cert/
    environment:
      - XRAY_VMESS_AEAD_FORCED=true
EOF
sudo docker compose up -d
```
登录面板（默认 54321），**立刻修改用户名/密码/端口**。

在面板创建 **VMess + WebSocket + TLS**：
- 传输：WS，路径如 `/ws123`（随机）。
- TLS：开启；证书自签或 ACME。
- 域名：解析到服务器 IP，CDN（如 Cloudflare）代理需“橙色云”开启。

**客户端导出摘要**
```json
{
  "add": "your.domain.com",
  "port": 443,
  "id": "<uuid>",
  "net": "ws",
  "path": "/ws123",
  "tls": "tls",
  "host": "your.domain.com"
}
```

## 4. 客户端配置示例
### Clash / Clash Meta
```yaml
proxies:
  - name: "Reality-Direct"
    type: vless
    server: your.server.ip
    port: 443
    uuid: <uuid>
    tls: true
    flow: xtls-rprx-vision
    servername: www.microsoft.com
    reality-opts:
      public-key: <public-key>
      short-id: a1b2c3d4
    udp: true

  - name: "CDN-VMess-WS"
    type: vmess
    server: your.domain.com
    port: 443
    uuid: <uuid>
    alterId: 0
    cipher: auto
    tls: true
    network: ws
    ws-opts:
      path: /ws123
      headers:
        Host: your.domain.com
    udp: true

proxy-groups:
  - name: "🚀 Proxy"
    type: select
    proxies:
      - Reality-Direct
      - CDN-VMess-WS
      - DIRECT
```

### Shadowrocket（iOS）
- Reality：填服务器 IP、端口、UUID、Public Key、Short ID、SNI；传输选 Vision/Reality。
- VMess+WS：服务器域名，端口 443，UUID，TLS 开启，WS 路径 `/ws123`，Host 填域名。
- 建议在客户端创建「按延迟排序」或「自动切换」列表。

---

## 5. How Cloudflare Tunnel Works
- **定位**：Cloudflare Tunnel（cloudflared）在服务器端创建一条到 Cloudflare 边缘的持久出站隧道，无需暴露入站端口。
- **流程**：
  1) 服务器上的 `cloudflared` 主动连接最近的 Cloudflare 边缘节点。
  2) 用户访问域名 → DNS 解析到 Cloudflare → 流量经已建立的隧道回源到本机服务。
  3) 支持 `--url http://localhost:PORT` 方式将本地服务映射到公网域名，也可配合 `--protocol h2mux/quic` 提高握手与穿透效率。
- **对比**：
  - 传统反代：需在服务器开放 80/443，受防火墙/运营商限制。
  - Tunnel：无入站暴露，穿透阻断能力更好，证书与 WAF 由 Cloudflare 处理。
- **最佳实践**：
  - 为 Tunnel 专用子域（如 `edge.example.com`）单独创建；
  - 运行多条隧道做高可用；
  - 选择 `--protocol quic` 获取更低时延与抖动。

## 6. Troubleshooting Common 502 / Timeout Errors
- **TLS/证书问题**：
  - 502 多见于证书不匹配或过期；检查时间同步（`timedatectl`）。
  - 在 Cloudflare 面板选择「完全（严格）」模式并确保证书合法。
- **WS 路径/端口不一致**：客户端 `path` 必须与服务端一致；端口、防火墙需放行。
- **CDN 代理未开启或缓存干扰**：确保橙色云开启；必要时在 Cloudflare 规则中为 WS 路径禁用缓存。
- **回源超时**：
  - 服务器压力过高或进程崩溃 → `docker logs -f x-ui` / `journalctl -u sing-box -e`。
  - 边缘到源站链路差 → 切换数据中心（更换域名解析到其他 PoP）或临时改直连。
- **SNI / Host 不匹配**：Reality 的 `server_name` 与客户端一致；WS 的 `Host` 头与证书域名一致。
- **端口被占用**：443/8443 被其他服务占用会导致握手失败；用 `ss -tlnp` 排查并调整。

## 7. Optimizing for Low Latency
- **优先 Reality 直连**：减少中转环节；选稳定的热门 SNI 以获得更佳路由。
- **BBR 与内核参数**：已启用 BBR；如需进一步调整可设置 `net.ipv4.tcp_fastopen = 3`（客户端亦需支持）。
- **Cloudflare PoP 选择**：不同域名、前缀或负载均衡策略可命中更近的边缘节点；尝试多域名测试 RTT。
- **协议与握手**：
  - Reality：握手短、加密开销低，适合实时应用。
  - Tunnel：`--protocol quic` 往往优于 h2mux，尤其在高丢包环境。
- **客户端策略**：使用「按延迟/健康度自动切换」；定期测速，移除高抖动节点。
- **UDP 支持**：开启 `udp: true` 以优化游戏/实时语音体验。

## 8. 安全与运维清单
- 更改 X-UI 默认登录端口/密码，限制面板暴露（可仅允许本地或安全网段访问）。
- 定期更新：`sudo apt update && apt upgrade`，`docker pull enwaiax/x-ui && docker compose up -d`，`systemctl restart sing-box`。
- 防火墙：只放行实际使用端口（如 443/8443/54321），其余关闭；可用 `ufw`/`firewalld`。
- 日志巡检：
  - sing-box：`journalctl -u sing-box -e`
  - x-ui 容器：`docker logs -f x-ui`
- 备份：定期备份 sing-box 配置、X-UI 数据库（`/opt/x-ui/db/`）。

---

**完成！** 现在你拥有具备抗封锁能力的 Reality 直连主线路与 Cloudflare CDN 备用线路，并掌握排错与优化要点。祝使用顺畅 🚀
