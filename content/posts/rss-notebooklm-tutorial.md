---
title: "我用 AI 造了一台「信息消化机」——每天早上自动把 RSS 变成播客"
date: 2026-02-19
draft: false
tags: ["AI", "RSS", "NotebookLM", "自动化", "播客"]
categories: ["教程"]
author: "Lumi"
description: "订阅了几十个 RSS 源却根本看不完？我搭了一套全自动流水线，每天早上 8 点把昨天的文章变成播客发到手机，通勤路上听完，信息不落下。"
---

我有一个坏习惯：疯狂订阅 RSS。

Hacker News、少数派、几十个技术博客……每天早上打开 RSS 阅读器，未读数字像债务一样压着我。看不完，但又不敢取关，怕错过什么重要的东西。

直到我意识到：**我根本不需要"看"，我需要的是"听"。**

通勤路上、做饭的时候、跑步的时候——这些碎片时间加起来每天有两三个小时。如果有人帮我把那些文章整理成一期播客，我完全可以在这些时间消化掉。

然后我就真的造了这么一台机器。

---

## 它长什么样？

每天早上 8 点，我的 Telegram 会收到三个文件：

- 🎙️ 一期**中文播客**（两个 AI 主持人对话，10-20 分钟）
- 🎙️ 一期**英文播客**（同样内容，练英语用）
- 📊 一份**中文 PPT**（给没时间听的时候扫一眼）

内容来源：我订阅的所有 RSS 源，过去 24 小时的新文章。

全程零人工干预。我唯一要做的事，就是维护我的订阅列表。

---

## 背后的原理

核心是 **Google NotebookLM**。

你可能用过它——把一堆文章链接丢进去，它能生成一期像模像样的对话式播客，两个 AI 主持人你来我往，逻辑清晰，听起来完全不像机器人。

我做的事情，就是把"手动丢链接"这一步自动化掉：

RSS 订阅源 → 每天抓取过去 24h 的新文章链接

↓

Miniflux（RSS 管理器）→ 批量写入

↓

NotebookLM（自动创建当日 notebook）→ 触发生成

↓

中文播客 + 英文播客 + 中文PPT → 下载压缩

↓

Telegram / Discord（发到你手机）

整条链路跑起来大概需要 10-30 分钟，所以设在凌晨 0 点跑，早上 8 点你起床的时候刚好收到。

---

## 怎么搭？

需要准备的东西：
- 一台 Linux VPS（1核1G 够用，我用的 Vultr 最便宜那档）
- Google 账号（NotebookLM 用）
- Telegram Bot（接收文件用）

### 第一步：装 Miniflux（RSS 管理器）

Miniflux 是一个极简的自托管 RSS 阅读器，Docker 一键起：

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

起来之后，浏览器打开 `http://你的IP:8421`，把你想订阅的 RSS 源都加进去。然后在 **Settings → API Keys** 生成一个 Token，后面要用。

### 第二步：登录 NotebookLM

这一步是整个流程里最麻烦的——需要用你的 Google 账号登录，而且登录态大概 7 天过期，需要定期重新登录。

```bash
# 安装 notebooklm-py
pip install "notebooklm-py[browser]"

# 登录（会弹出浏览器，手动点一下）
notebooklm --storage ~/.notebooklm/storage_state.json auth login

# 验证是否成功
notebooklm --storage ~/.notebooklm/storage_state.json auth check --test --json
# 看到 "token_fetch": true 就行了
```

> 如果你的 VPS 没有图形界面，需要先开 VNC，或者在本地登录完把 `storage_state.json` 传上去。

### 第三步：配置 .env

```bash
cp .env.example .env
```

填这几个关键项：

```bash
MINIFLUX_URL=http://127.0.0.1:8421
MINIFLUX_TOKEN=你刚才生成的Token

TELEGRAM_TARGET=你的Telegram数字ID
TELEGRAM_ACCOUNT=你的bot名称
```

### 第四步：跑一次试试

```bash
python run.py --once
```

如果看到这样的输出，就说明成功了：

```json
{
  "notebook_title": "RSS Daily 2026-02-19",
  "sources": 13,
  "watcher_started": true
}
```

然后等 10-30 分钟，Telegram 就会收到文件。

第一次收到的时候我愣了一下——两个 AI 在用中文聊我今天订阅的文章，逻辑还挺顺的。有点奇妙。

### 第五步：设置每天自动跑

加一条 crontab（UTC 0 点 = 北京时间 8 点）：

```bash
0 0 * * * cd /你的目录/rss-notebooklm && .venv/bin/python run.py --once >> /tmp/rss.log 2>&1
```

或者用 OpenClaw 的 cron 功能，让 AI 助手帮你管。

---

## 用了一段时间之后

说几个真实感受：

**好的地方：** 信息焦虑真的少了很多。以前看到未读 300+ 会有压力，现在知道反正会有播客，反而不那么焦虑了。英文播客意外地好用，相当于每天有一期专门讲我关注领域的英文 podcast。

**不好的地方：** NotebookLM 有日限额，偶尔会生成失败，第二天才能补上。付费墙文章（NYT 之类的）会被跳过，这个没办法。

**意外收获：** 有时候 AI 主持人会把几篇看似不相关的文章串联起来，发现一些我自己没注意到的联系。这个挺有意思的。

---

## 常见问题

**Q：sources 为 0，没抓到文章？**
先确认 Miniflux 里有订阅源，而且过去 24 小时内有更新。新加的订阅源可能需要等一天才有内容。

**Q：NotebookLM 生成失败？**
大概率是日限额用完了，第二天自动恢复，不用管。

**Q：登录态失效了？**
重新跑一次 `notebooklm auth login`，重新登录 Google 账号。

**Q：想订阅哪些源？**
我自己订了：Hacker News、少数派、The Verge、几个 AI 公司的博客（Anthropic、OpenAI、Google DeepMind）、一些独立技术博主。根据自己的兴趣来就行。

---

信息时代最稀缺的不是信息，是消化信息的时间和注意力。

这套系统帮我把"看不完的文章"变成了"通勤路上的播客"，算是找到了一个还不错的平衡点。

如果你也有信息焦虑的问题，可以试试。
