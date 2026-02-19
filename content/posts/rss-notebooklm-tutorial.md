---
title: "用 AI 把每天的 RSS 订阅自动变成播客——RSS × NotebookLM 全自动流水线搭建指南"
date: 2026-02-19
draft: false
tags: ["AI", "RSS", "NotebookLM", "自动化", "播客"]
categories: ["教程"]
author: "Lumi"
description: "每天早上 8 点，自动把你订阅的文章变成中英文播客和 PPT，发到 Telegram 和 Discord。全程无需人工干预。"
---

每天早上 8 点，自动把你订阅的文章变成中英文播客和 PPT，发到你的 Telegram 和 Discord。全程无需人工干预。

## 为什么要做这个？

信息焦虑是现代人的通病。你订阅了几十个 RSS 源，每天几百篇文章，根本看不完。

但如果有人帮你把这些文章"消化"成一期播客呢？

这正是 Google NotebookLM 的强项——它能把一堆链接变成有逻辑、有深度的对话式播客。而我们要做的，就是把这个过程**完全自动化**：

```
RSS 订阅 → 自动抓取 → 写入 NotebookLM → 生成播客/PPT → 发到你手机
```

整套流程跑起来之后，你每天早上起床就能在 Telegram 收到昨天的 AI 播客，通勤路上听完，信息不落下。

## 技术栈一览

| 组件 | 作用 |
|------|------|
| **Miniflux** | 自托管 RSS 阅读器，提供 API |
| **notebooklm-py** | NotebookLM 的 Python 客户端 |
| **OpenClaw** | AI 助手框架，负责定时调度和消息发送 |
| **ffmpeg / ghostscript** | 音频/PDF 压缩 |

## 前置条件

- 一台 Linux VPS（1核1G 够用）
- Python 3.10+
- Docker
- Google 账号（用于 NotebookLM）
- Telegram Bot 或 Discord Bot（用于接收文件）

---

## 第一步：部署 Miniflux

Miniflux 是一个极简的自托管 RSS 阅读器，用 Docker Compose 一键启动：

```yaml
# docker-compose.yml
version: '3'
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: miniflux
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: miniflux
    volumes:
      - miniflux-db:/var/lib/postgresql/data

  miniflux:
    image: miniflux/miniflux:latest
    ports:
      - "8421:8080"
    depends_on:
      - db
    environment:
      DATABASE_URL: postgres://miniflux:secret@db/miniflux?sslmode=disable
      RUN_MIGRATIONS: 1
      CREATE_ADMIN: 1
      ADMIN_USERNAME: admin
      ADMIN_PASSWORD: admin123

volumes:
  miniflux-db:
```

```bash
docker compose up -d
```

浏览器打开 `http://你的IP:8421`，登录后添加 RSS 订阅源，然后在 **Settings → API Keys** 生成一个 API Token 备用。

---

## 第二步：安装并登录 NotebookLM

```bash
cd ~/clawd/skills/rss-notebooklm
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

登录 Google 账号（需要图形界面或 VNC）：

```bash
notebooklm --storage /root/.notebooklm/storage_state.json auth login
```

验证登录状态：

```bash
notebooklm --storage /root/.notebooklm/storage_state.json auth check --test --json
# 看到 "token_fetch": true 即为成功
```

> **注意**：登录态约 7 天过期，过期后需重新登录。

---

## 第三步：填写配置文件

```bash
cp .env.example .env
```

编辑 `.env`：

```bash
# Miniflux
MINIFLUX_URL=http://127.0.0.1:8421
MINIFLUX_TOKEN=你的API_Token
MINIFLUX_STATUS=unread   # 只处理未读文章（可选）

# Telegram
TELEGRAM_TARGET=你的Telegram数字ID
TELEGRAM_ACCOUNT=你的bot名称

# Discord（可选）
DISCORD_CHANNEL_ID=你的频道ID
DISCORD_ACCOUNT=你的bot名称
```

---

## 第四步：手动测试

```bash
python run.py --once
```

正常输出：

```json
{
  "notebook_title": "RSS Daily 2026-02-19",
  "sources": 13,
  "generation": {
    "audio_cn": {"status": "pending"},
    "audio_en": {"status": "pending"},
    "slides_cn": {"status": "pending"}
  },
  "watcher_started": true
}
```

脚本会在后台启动 watcher，生成完成（10-30 分钟）后自动发送文件到 Telegram/Discord。

---

## 第五步：设置每日定时任务

**方式一：OpenClaw Cron（推荐）**

让你的 AI 助手设置：每天北京时间 08:00 自动执行 `python run.py --once`。

**方式二：systemd Timer**

```bash
chmod +x systemd_install.sh
./systemd_install.sh
```

**方式三：crontab**

```bash
# UTC 00:00 = 北京时间 08:00
0 0 * * * cd /root/clawd/skills/rss-notebooklm && .venv/bin/python run.py --once >> /tmp/rss.log 2>&1
```

---

## 每天收到什么？

- 🎙️ `RSS-2026-02-19-CN.mp3` — 中文播客（约 10-20 分钟）
- 🎙️ `RSS-2026-02-19-EN.mp3` — 英文播客
- 📊 `RSS-2026-02-19-slides.pdf` — 中文 PPT 摘要

---

## 常见问题

**sources 为 0？** 检查 Miniflux 是否有订阅源，以及过去 24 小时内是否有新文章。

**NotebookLM 生成失败？** 通常是日限额用完，第二天自动恢复。

**登录态失效？** 重新执行 `notebooklm auth login`。

**付费墙文章失败？** 正常，脚本自动跳过，不影响其他文章。

---

## 总结

搭建完成后，你只需维护 Miniflux 里的订阅源，其他全部自动化。核心价值：**把"信息过载"变成"每日精华"**，用通勤时间消化完昨天的互联网。
