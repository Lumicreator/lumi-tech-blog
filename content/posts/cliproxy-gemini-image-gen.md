---
title: "用 CLIProxyAPI 反代 Gemini，让任何工具都能 AI 生图"
date: 2026-02-27T00:00:00+08:00
description: "通过 CLIProxyAPI 将 Gemini CLI 的 OAuth 登录转成标准 OpenAI API，零成本获得 AI 生图能力，接入任何工具链。"
categories: ["教程"]
tags: ["CLIProxyAPI", "Gemini", "AI生图", "反向代理", "OpenAI API"]
---

> 你有 Gemini CLI 的免费账号吗？那你就已经有了 AI 生图的 API。CLIProxyAPI 把 CLI 的 OAuth 登录态转成标准 OpenAI 格式接口，你的 Agent、脚本、任何工具都能直接调用。

## 核心思路

Gemini CLI、Claude Code、OpenAI Codex 这些工具都是通过 OAuth 登录的，登录后你就有了对应模型的访问权限。但问题是——这些权限被锁在 CLI 里，你的其他工具用不了。

**CLIProxyAPI** 做的事：把 CLI 的 OAuth 登录态"解锁"出来，转成标准的 `/v1/chat/completions` 接口。

这意味着：
- 不需要付费 API Key
- 用 CLI 的免费额度就能调 API
- 所有支持 OpenAI 格式的工具直接接入

## 架构

![CLIProxyAPI 架构图](/images/cliproxy-architecture.png)

```
Gemini CLI  ─┐
Claude Code ─┤── OAuth 登录 ──→ CLIProxyAPI ──→ /v1/chat/completions ──→ 你的工具
Codex       ─┘                  (反代层)                                  Agent/脚本/App
```

CLIProxyAPI 在中间做了两件事：
1. **管理 OAuth 登录态**：帮你维护多个 CLI 账号的登录，支持轮询负载均衡
2. **协议翻译**：把各家 CLI 的私有协议统一翻译成 OpenAI 兼容格式

## 部署

### Docker 一键启动

```bash
git clone https://github.com/eceasy/cli-proxy-api.git
cd cli-proxy-api
cp config.example.yaml config.yaml
docker compose up -d
```

### 配置文件

```yaml
# config.yaml
host: ""
port: 8317

# 客户端鉴权用的 Key
api-keys:
  - "my-client-key"

# Gemini CLI OAuth 登录（核心）
# 启动后访问管理面板完成 OAuth 登录
# 登录态会自动保存，支持多账号轮询

# 也可以配 API Key 作为备选
gemini-api-key:
  - api-key: "可选，有OAuth就不需要"
```

启动后访问管理面板（默认端口的 `/panel`），按引导完成 Gemini CLI 的 OAuth 登录。登录一次，后续自动续期。

### 多账号负载均衡

CLIProxyAPI 支持同时登录多个 Gemini 账号，请求会自动轮询分配：

```
账号A (免费额度) ─┐
账号B (免费额度) ─┤── 轮询 ──→ 对外统一接口
账号C (免费额度) ─┘
```

单个免费账号额度不够？多登几个就行。

## AI 生图实战

Gemini 3 系列模型原生支持图片生成。通过 CLIProxyAPI，你可以用标准 OpenAI 格式调用这个能力。

![生图流程](/images/cliproxy-image-gen-flow.png)

### 基础调用

```python
import requests, base64
from pathlib import Path

resp = requests.post("http://127.0.0.1:9417/v1/chat/completions",
    headers={"Authorization": "Bearer my-client-key"},
    json={
        "model": "gemini-3.1-flash-image",
        "messages": [{
            "role": "user",
            "content": "画一只橘猫在沙发上跳跃，简笔画风格"
        }],
        "max_tokens": 4096
    }, timeout=120)

# 图片在 images 字段（不是 content）
images = resp.json()["choices"][0]["message"]["images"]
b64 = images[0]["image_url"]["url"].split(",", 1)[1]
Path("cat.jpg").write_bytes(base64.b64decode(b64))
```

> ⚠️ 注意：图片返回在 `choices[0].message.images[]` 字段，不是 `content`。格式是 `data:image/jpeg;base64,...`。

### 批量生图

实际场景：我每天用这套给小红书账号批量生成科普配图。

```python
import requests, base64, time
from pathlib import Path

ENDPOINT = "http://127.0.0.1:9417/v1/chat/completions"
OUT_DIR = Path("./images")
OUT_DIR.mkdir(exist_ok=True)

prompts = [
    "简笔画风格，一只猫凌晨3点从沙发上起飞，主人被吵醒...",
    "简笔画风格，猫上完厕所后疯狂冲刺，灰尘飞扬...",
    # ...
]

for i, prompt in enumerate(prompts, 1):
    resp = requests.post(ENDPOINT, json={
        "model": "gemini-3.1-flash-image",
        "messages": [{"role": "user", "content": prompt}],
        "max_tokens": 4096
    }, headers={"Authorization": "Bearer my-client-key"}, timeout=120)

    images = resp.json()["choices"][0]["message"]["images"]
    b64 = images[0]["image_url"]["url"].split(",", 1)[1]
    Path(f"images/img_{i}.jpg").write_bytes(base64.b64decode(b64))
    print(f"[{i}] ✅")
    time.sleep(2)  # 避免限流
```

### 模型怎么选

| 模型 | 速度 | 质量 | 配额 | 建议 |
|------|------|------|------|------|
| `gemini-3-pro-image-preview` | 慢 | 最高 | 紧 | 重要封面图 |
| `gemini-3.1-flash-image` | 快 | 够用 | 宽松 | 日常批量生图 |

日常用 flash，配额不容易爆。pro 留给关键场景。

## 不只是生图

CLIProxyAPI 统一的是整个 API 层，不只是生图。你还可以：

- **Claude Code OAuth → API**：用 Claude Code 的免费/Pro 额度调 Claude 模型
- **Codex OAuth → API**：用 ChatGPT Plus 的额度调 GPT 模型
- **混合路由**：不同模型走不同后端，对外一个 endpoint

```yaml
# 同时接入多家
openai-compatibility:
  - name: 阿里百炼
    base-url: https://coding.dashscope.aliyuncs.com/v1
    api-key-entries:
      - api-key: "sk-xxx"
    models:
      - name: kimi-k2.5
```

一个 endpoint 管所有模型，后面接什么随时切。

## 踩坑记录

1. **图片在 `images` 不在 `content`**：CLIProxyAPI 返回图片的位置和标准 OpenAI 不一样，别找错了

2. **竖版图片**：prompt 里写 `768x1024` 就行，实际输出 `896x1200`

3. **配额共享**：如果你同时用 Gemini API Key 做 embedding 和生图，配额是共享的。生图一顿造，embedding 也会 429。建议分开

4. **超时要长**：生图比文本慢很多，timeout 至少 120 秒

5. **OAuth 续期**：CLI 的 OAuth token 会过期，CLIProxyAPI 会自动刷新，但偶尔需要重新登录一次

## 总结

CLIProxyAPI 的核心价值：

- **把 CLI 的免费额度变成 API**——OAuth 登录一次，所有工具都能用
- **统一协议**——不管后面是 Gemini、Claude 还是 GPT，对外都是 OpenAI 格式
- **多账号负载均衡**——单账号额度不够就多登几个
- **零成本生图**——Gemini 免费额度 + CLIProxyAPI = 任何工具都能 AI 生图

如果你在用 AI Agent 做内容生产，这套方案基本是零成本的。

---

*相关链接：*
- [CLIProxyAPI GitHub](https://github.com/eceasy/cli-proxy-api)
- [CLIProxyAPI 文档](https://help.router-for.me/)
- [Google AI Studio](https://aistudio.google.com/apikey)
- [Gemini 生图文档](https://ai.google.dev/gemini-api/docs/image-generation)
