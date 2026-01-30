# 保姆级教程：从零搭建 VLESS+Reality 与 CDN 加速梯子

> 适合有基础 Linux 经验的新手，跟着做即可。全程采用双线路：直连 VLESS+Reality（抗封锁、低延迟）+ CDN VMess+WS（加速、伪装）。

## 1. 为什么这样搭？
- **稳定性**：Reality 直连不依赖域名或 CDN，抗干扰强；CDN 线路可作为备用，域名被墙还能换解析。 
- **速度**：本地直连走 Reality，延迟低；遇到跨境网络差时，切换到 CDN 线路利用边缘节点提速。 
- **抗封锁**：双模并行，端口、传输、域名多样化，降低被针对的概率。🎯

## 2. 前置条件
- 一台境外 VPS（常见：Debian 11+/Ubuntu 22.04，1C1G 以上更佳，需 root）。
- 基础 SSH/命令行操作经验，懂得修改配置文件、查看日志。 
- 一个可用的域名（用于 CDN 方案），支持配置 DNS 记录。

## 3. 服务端部署
### 3.1 开启 BBR
```bash
# 内核与 bbr 检查（Debian/Ubuntu）
uname -r
# 启用 BBR
cat <<'EOF' | sudo tee /etc/sysctl.d/99-bbr.conf
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
EOF
sudo sysctl --system
sysctl net.ipv4.tcp_congestion_control net.core.default_qdisc
```
提示：输出含 `bbr` 和 `fq` 即生效。⚡

### 3.2 安装 sing-box（Reality）
两种方式任选其一：
1) **一键脚本 reality-ezpz**（快速上手）
```bash
bash <(curl -fsSL https://raw.githubusercontent.com/ok233wz/reality-ezpz/main/install.sh)
```
脚本会生成 Reality 配置（含公私钥、短指纹），安装后按提示记录 `uuid/端口/serverName`。

2) **手动安装 sing-box**（自定义更强）
```bash
# 以 Debian/Ubuntu 为例
sudo bash -c 'wget -O- https://sing-box.app/install.sh | bash'
# 配置文件路径通常在 /etc/sing-box/config.json
```
Reality 核心字段：
- `type: vless` + `flow: xtls-rprx-vision`
- `reality`: `private_key`/`public_key`、`server_name`（SNI）、`short_id`
- 建议端口：443/8443/随机高位端口。

### 3.3 安装 X-UI（VMess+WS+TLS+CDN）
使用 docker-compose 方便升级回滚：
```bash
# 安装 docker & compose
curl -fsSL https://get.docker.com | sh
sudo systemctl enable --now docker

# 创建目录
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

# 启动
sudo docker compose up -d
# 首次登录面板（默认 54321 端口）后请立即修改用户名/密码/端口！
```
在面板中添加 **VMess + WebSocket + TLS** 节点：
- 传输：WS，路径如 `/ws123`（随机字符串）。
- TLS：勾选，自备证书或使用 ACME 获取；CDN 场景可用灵活证书。
- 端口：443/8443/随机高位（配合防火墙放行）。
- 域名：指向 CDN（如 Cloudflare）后端解析到服务器 IP，WS 路径需与面板一致。

## 4. 双模策略：VLESS+Reality（直连）& VMess+WS（CDN）
- **直连 VLESS+Reality**：无需域名，抗封锁，延迟低。适合日常主线路。 
- **CDN VMess+WS**：走 443/80 伪装 HTTP，前置 CDN（Cloudflare/百度云加速等）。适合晚高峰或直连不稳时切换。 
- 建议在客户端同时导入两条线路，使用「按延迟/按自动切换」策略或手动一键切换。

### Reality 配置示例（/etc/sing-box/config.json 摘要）
```json
{
  "inbounds": [
    {
      "type": "vless",
      "listen": "0.0.0.0",
      "listen_port": 443,
      "users": [{"uuid": "<your-uuid>", "flow": "xtls-rprx-vision"}],
      "tls": {
        "enabled": true,
        "reality": {
          "show": false,
          "dest": "www.microsoft.com:443",
          "xver": 0,
          "server_name": "www.microsoft.com",
          "private_key": "<your-private-key>",
          "short_id": "a1b2c3d4"
        }
      }
    }
  ]
}
```
- `dest/server_name` 选热门 HTTPS 站点（更像正常流量）。
- `short_id` 4~8 hex，客户端需一致。

### VMess+WS+TLS（X-UI 端导出的 JSON 摘要）
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
- CDN 里将 `your.domain.com` 解析到服务器 IP，开启「代理」橙色云；WS 路径和端口保持一致。

## 5. 客户端配置
### Clash 系列（Clash Meta/Meta for Windows/Mac/Linux）
在配置文件 `config.yaml` 中新增两个节点，示例：
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
在客户端选择 `🚀 Proxy` 后可随时切换两条线路。

### Shadowrocket（iOS）
1. 扫码或导入两条订阅/节点（Reality 与 VMess+WS）。
2. Reality 填写：服务器 IP、端口、UUID、Public Key、Short ID、SNI；传输选 Vision/Reality。 
3. VMess+WS：服务器为域名，端口 443，UUID，TLS 开启，WS 路径 `/ws123`，Host 同域名。 
4. 建议创建「按延迟排序」列表，方便快速切换。📱

## 6. 安全与排错 Tips
- 🔒 **改默认信息**：X-UI 登录端口、账号密码务必修改；定期更新容器镜像与系统。 
- 🧱 **防火墙**：只放行实际使用端口（如 443/8443/54321 面板），其他端口关闭；可用 `ufw`/`firewalld`。 
- 🎭 **伪装多样化**：Reality `server_name/dest` 可用常见大站；WS 路径/端口随机；如需再增强可加 WAF/CDN 自选端口。 
- 🔄 **日志查看**：
  - sing-box：`journalctl -u sing-box -e`
  - docker x-ui：`docker logs -f x-ui`
- 🧪 **常见问题**：
  - 443 被占用 → 换端口或释放占用服务。
  - Reality 无法连接 → 核对 Public Key、Short ID、SNI、端口防火墙。
  - CDN 走不通 → 检查 WS 路径、证书、CDN 代理是否开启（橙色云）。
  - TLS 失败 → 确认证书有效、时间同步（`timedatectl`）。

---
收工！现在你拥有直连抗封锁 + CDN 加速的双模梯子，按需切换，稳速兼得。🚀