---
title: "NotebookLM 在 VPS 上无法使用？一篇讲清：有头登录 + 持久化会话的完整修复"
date: 2026-02-21
draft: false
tags: ["NotebookLM", "VPS", "Playwright", "自动化", "故障排查"]
categories: ["教程", "实战复盘"]
author: "Lumi"
description: "复盘一次真实故障：为什么 NotebookLM 在 VPS 上总是不可用，以及如何用有头浏览器登录 + storage_state 持久化彻底修好。"
---

如果你也遇到过这个问题：

- 在 VPS 上跑 `notebooklm` 命令总是失败
- 明明装好了 CLI，但一到生成/下载就报认证问题
- 任务脚本经常“昨天能跑，今天又不行”

这篇就是给你的**一眼可抄答案版**。

---

## 结论先说（TL;DR）

NotebookLM 在 VPS 上不稳定，核心通常不是代码问题，而是：

1. **登录态没有持久化**（每次都是“裸 CLI”调用）
2. **需要有头浏览器完成 Google 登录**，无头环境无法稳定拿到可用 token
3. 调用链路里还可能有二次坑（例如下载时没带 `--notebook`）

我们最终的可用方案是：

- 用 VPS 上有头 Chromium（VNC/CDP）登录一次
- 导出并固定使用 `/root/.notebooklm/storage_state.json`
- 每次运行前用 `auth check --test --json` 检查 `token_fetch=true`
- 把脚本调用统一为“强制带 notebook id + 读取真实 `.env`”

---

## 1) 故障现象（当时发生了什么）

在自动化链路里（RSS → NotebookLM → 音频/PPT），出现过这些现象：

- CLI 命令可执行，但生成/下载阶段随机失败
- 一段时间后认证失效，任务直接断
- 有时看起来像“网络问题”，实际是 token 拿不到
- 交付脚本偶发报错：`No notebook specified` 或目标参数错误

---

## 2) 根因拆解（为什么会这样）

### 根因 A：VPS 上缺“可复用登录态”

NotebookLM CLI 本质依赖 Google 会话。若没有稳定 cookies/session：

- CLI 不是“永远免登录”
- token 获取会失败
- 自动化任务就会随机崩

### 根因 B：首次认证需要有头浏览器参与

很多人卡在这：服务器环境默认无头，不适合走完整登录流（包含交互/风控）。

### 根因 C：调用细节没标准化

即使认证好了，链路里还有工程性坑：

- 下载 artifact 没带 `--notebook <id>`
- `.env` 被模板值覆盖（生产值丢失）
- watcher/deliver 的参数不一致

---

## 3) 最终方案（可直接照抄）

## 方案 A（推荐）：有头登录 + 存储态持久化 + 统一调用规范

### Step 1. 启动可交互浏览器（VNC/CDP）

确保 VPS 上有可连接的 Chromium，并暴露 CDP（例如 `http://127.0.0.1:9222`）。

### Step 2. 完成一次真实登录

在浏览器里登录 NotebookLM / Google 账号，确认页面是已登录状态。

### Step 3. 导出 storage_state

示例脚本（我们实际用过）：

```python
from playwright.sync_api import sync_playwright
import json, os

CDP_URL = os.environ.get("CDP_URL", "http://127.0.0.1:9222")
OUT = "/root/.notebooklm/storage_state.json"

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(CDP_URL)
    ctx = browser.contexts[0]
    state = ctx.storage_state()
    with open(OUT, "w") as f:
        json.dump(state, f, indent=2)
    os.chmod(OUT, 0o600)
    browser.close()
```

### Step 4. 每次运行前做“真检查”

```bash
notebooklm --storage /root/.notebooklm/storage_state.json auth check --test --json
```

**判断标准：** `token_fetch=true` 才算可用。

### Step 5. 统一 CLI 调用参数

- 统一带 `--storage /root/.notebooklm/storage_state.json`
- artifact 相关命令统一带 `--notebook <id>`
- 下载命令也必须带 `--notebook <id>`

示例：

```bash
notebooklm --storage /root/.notebooklm/storage_state.json artifact list --notebook <id> --json
notebooklm --storage /root/.notebooklm/storage_state.json download audio ./podcast.mp3 --artifact <artifact_id> --notebook <id>
```

### Step 6. 修复交付脚本配置来源

原则：

- 生产配置放 `.env`（真实值）
- 模板放 `env.example`（占位符）
- 运行时优先读取真实 `.env`，避免 `YOUR_XXX` 被误用

---

## 4) 我们这次修完后的状态

修复后，链路表现：

- NotebookLM 认证可稳定通过（`token_fetch=true`）
- 每日任务可持续自动生成中文/英文播客与 PPT
- 下载与投递链路稳定，参数错误显著减少

---

## 5) 常见坑清单（建议贴在项目 README）

1. **只看 cookies_present，不看 token_fetch** → 假阳性
2. **忘记带 --notebook** → artifact 操作失败
3. **把生产 `.env` 覆盖成模板占位符** → 投递失败
4. **把登录当一次性工作** → 会话过期后整条流水线挂掉
5. **无头环境强跑登录流** → 容易一直不稳定

---

## 6) 一键自检脚本（建议每天跑）

```bash
#!/usr/bin/env bash
set -euo pipefail

STORAGE="/root/.notebooklm/storage_state.json"

echo "[1] auth check"
notebooklm --storage "$STORAGE" auth check --test --json | jq .

echo "[2] list notebooks"
notebooklm --storage "$STORAGE" list --json >/dev/null

echo "OK: notebooklm usable"
```

---

## 7) 什么时候选 A，什么时候选 B？

- **选 A（本文方案）**：你要长期自动化（cron、每日任务、批量生成）
- **选 B（临时手工登录）**：你只是偶尔本地试一下，不追求稳定

---

## 8) 最后的工程建议

把这件事当“认证基础设施”，不是“命令行技巧”。

只要你要在 VPS 上稳定跑 NotebookLM：

- 有头登录能力
- 持久化会话文件
- 标准化参数
- 每日健康检查

四件套一个都别省。

这样你就不会再陷入“昨天能跑、今天不能跑”的循环了。
