---
title: "OpenClaw 在 VPS 上使用有头浏览器完全指南"
date: 2026-02-04T00:00:00+08:00
description: "教你如何让 OpenClaw 在 VPS 上控制有头浏览器，绕过反爬检测，实现真实的网页自动化操作。"
tags: ["OpenClaw", "Browser", "VPS", "Automation", "VNC"]
---

> 本教程教你如何在 VPS 上配置有头浏览器，让 OpenClaw 像真人一样操作网页，绕过各种反爬和机器人检测。

## 目录
1. 为什么需要有头浏览器
2. 环境准备
3. 配置 OpenClaw Browser Profile
4. 使用 Browser 工具
5. 持久化登录状态
6. 身份伪装技巧

---

## 1. 为什么需要有头浏览器？

**Headless 浏览器的问题**：
- 容易被网站检测为机器人
- 很多网站（如 Twitter/X、Google）会直接拒绝服务
- 验证码更难通过

**有头浏览器的优势**：
- 行为特征与真实用户一致
- 可以通过 VNC 手动介入处理验证码
- 登录状态可持久保存

---

## 2. 环境准备

### 2.1 安装 VNC 服务

```bash
# 安装 TigerVNC
apt update && apt install -y tigervnc-standalone-server

# 设置 VNC 密码
vncpasswd

# 启动 VNC（分辨率 1280x800）
vncserver :1 -geometry 1280x800 -depth 24
```

### 2.2 安装桌面环境（可选，轻量）

```bash
# 安装 fluxbox（轻量级窗口管理器）
apt install -y fluxbox

# 或者安装 xfce4（功能更全）
apt install -y xfce4 xfce4-goodies
```

### 2.3 安装 Chromium

```bash
apt install -y chromium
```

### 2.4 启动有头浏览器

```bash
# 在 VNC 显示器上启动 Chromium，开启远程调试端口
nohup bash -c 'DISPLAY=:1 chromium \
  --no-sandbox \
  --no-first-run \
  --remote-debugging-port=9222 \
  --user-data-dir=/root/chrome-data \
  https://google.com' > /tmp/chrome.log 2>&1 &
```

**参数说明**：
- `DISPLAY=:1`：指定 VNC 显示器
- `--remote-debugging-port=9222`：开启 CDP 调试端口
- `--user-data-dir=/root/chrome-data`：持久化用户数据（cookies、登录状态）
- `--no-sandbox`：VPS 上需要此参数

---

## 3. 配置 OpenClaw Browser Profile

编辑 OpenClaw 配置文件 `~/.openclaw/openclaw.json`：

```json
{
  "browser": {
    "headless": false,
    "defaultProfile": "vnc",
    "profiles": {
      "vnc": {
        "cdpUrl": "http://127.0.0.1:9222",
        "color": "#00AA00"
      }
    }
  }
}
```

**配置说明**：
- `cdpUrl`：指向 Chromium 的远程调试端口
- `defaultProfile`：设为 `vnc`，默认使用有头浏览器
- `headless: false`：确保不使用无头模式

配置完成后重启 OpenClaw Gateway：
```bash
openclaw gateway restart
```

---

## 4. 使用 Browser 工具

### 4.1 查看当前标签页

```
browser action=tabs profile=vnc
```

### 4.2 获取页面快照（DOM 结构）

```
browser action=snapshot profile=vnc targetId=<tab_id>
```

### 4.3 执行操作

**点击元素**：
```
browser action=act profile=vnc request={"kind":"click","ref":"e123"}
```

**输入文本**：
```
browser action=act profile=vnc request={"kind":"type","ref":"e123","text":"Hello"}
```

**导航到 URL**：
```
browser action=navigate profile=vnc targetUrl="https://example.com"
```

### 4.4 截图

```
browser action=screenshot profile=vnc targetId=<tab_id>
```

---

## 5. 持久化登录状态

### 5.1 使用持久化目录

启动浏览器时指定 `--user-data-dir` 到非临时目录：

```bash
# ❌ 错误：/tmp 重启后会丢失
--user-data-dir=/tmp/chrome-debug

# ✅ 正确：持久化目录
--user-data-dir=/root/chrome-data
```

### 5.2 迁移已有数据

如果已经在临时目录登录了账号：

```bash
# 关闭浏览器
pkill -f "chromium.*remote-debugging-port"

# 复制数据到持久目录
cp -r /tmp/chrome-debug /root/chrome-data

# 用新目录重启浏览器
nohup bash -c 'DISPLAY=:1 chromium \
  --no-sandbox \
  --no-first-run \
  --remote-debugging-port=9222 \
  --user-data-dir=/root/chrome-data' > /tmp/chrome.log 2>&1 &
```

---

## 6. 身份伪装技巧

### 6.1 心态设定

把这个浏览器当作**你自己的个人电脑**：
- 这是你的浏览器，不是远程控制的工具
- 登录的账号是你的账号
- 你是一个真实的人在操作

### 6.2 行为模式

像真人一样操作：
- 不要机械地快速点击
- 有自然的停顿和阅读时间
- 偶尔滚动页面浏览内容

### 6.3 账号管理

建议为 AI 助手创建专属账号：
- Google 账号（用于各种 OAuth 登录）
- 社交媒体账号（Twitter/X、Discord 等）
- 保持账号活跃，定期登录使用

---

## 7. 常见问题

### Q: 浏览器启动失败？

检查 VNC 是否运行：
```bash
vncserver -list
```

检查端口是否被占用：
```bash
lsof -i :9222
```

### Q: OpenClaw 连接不上浏览器？

确认 CDP 端口可访问：
```bash
curl http://127.0.0.1:9222/json/version
```

### Q: 遇到验证码怎么办？

通过 VNC 客户端连接到 VPS，手动完成验证码：
```
VNC 地址：your-vps-ip:5901
```

---

## 8. 完整启动脚本

创建 `/root/start-browser.sh`：

```bash
#!/bin/bash

# 确保 VNC 运行
vncserver -list | grep -q ":1" || vncserver :1 -geometry 1280x800 -depth 24

# 关闭旧的浏览器实例
pkill -f "chromium.*remote-debugging-port=9222" 2>/dev/null

sleep 1

# 启动浏览器
nohup bash -c 'DISPLAY=:1 chromium \
  --no-sandbox \
  --no-first-run \
  --remote-debugging-port=9222 \
  --user-data-dir=/root/chrome-data \
  --disable-gpu \
  --disable-software-rasterizer' > /tmp/chrome.log 2>&1 &

echo "Browser started. Check with: curl http://127.0.0.1:9222/json/version"
```

设置开机自启：
```bash
chmod +x /root/start-browser.sh
echo "@reboot /root/start-browser.sh" | crontab -
```

---

## 总结

通过本教程，你学会了：
1. 在 VPS 上配置 VNC + 有头浏览器
2. 配置 OpenClaw 连接到有头浏览器
3. 使用 browser 工具进行网页自动化
4. 持久化登录状态
5. 身份伪装技巧

现在你的 AI 助手可以像真人一样使用浏览器了！🎉
