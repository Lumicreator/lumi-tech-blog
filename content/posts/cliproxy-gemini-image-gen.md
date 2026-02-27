---
title: "用 CLIProxyAPI 反代 Gemini，让任何工具都能 AI 生图"
date: 2026-02-27T00:00:00+08:00
description: "通过 CLIProxyAPI 将 Gemini 的原生生图能力包装成 OpenAI 兼容 API，一行代码调用 AI 生图，零门槛接入任何工具链。"
categories: ["教程"]
tags: ["CLIProxyAPI", "Gemini", "AI生图", "反向代理", "OpenAI API"]
---

> Gemini 原生支持图片生成，但它的 API 格式和 OpenAI 不兼容。CLIProxyAPI 做的事很简单：把 Gemini 包装成 OpenAI 格式，这样你现有的所有工具、脚本、Agent 都能直接调用生图，不用改一行代码。

## 背景：为什么需要反代？

Gemini 3 系列模型（`gemini-3-pro-image-preview`、`gemini-3.1-flash-image`）原生支持图片生成，而且效果相当不错。但问题是：

- Gemini API 的请求/响应格式和 OpenAI 不一样
- 大部分 AI 工具链（LangChain、OpenClaw、各种 Agent 框架）都是按 OpenAI API 格式设计的
- 你不想为了用个生图功能就重写所有调用逻辑

**CLIProxyAPI** 就是解决这个问题的——它是一个轻量级反向代理，把各家 AI 服务统一翻译成 OpenAI 兼容的 `/v1/chat/completions` 格式。

## 架构一览

```
你的工具/脚本/Agent
    ↓ OpenAI 格式请求
CLIProxyAPI (反代层)
    ↓ 翻译成 Gemini 原生格式
Google Gemini API
    ↓ 返回图片
CLIProxyAPI
    ↓ 包装成 OpenAI 格式响应
你的工具/脚本/Agent
```

整个过程对调用方完全透明，你只需要把 `base_url` 指向 CLIProxyAPI 就行。

## 部署 CLIProxyAPI

### Docker 一键部署

```bash
# 克隆仓库
git clone https://github.com/eceasy/cli-proxy-api.git
cd cli-proxy-api

# 编辑配置
cp config.example.yaml config.yaml
vim config.yaml

# 启动
docker compose up -d
```

### 核心配置

```yaml
# config.yaml
host: ""
port: 8317

# 你的 API Key（用于客户端鉴权）
api-keys:
  - "your-client-key"

# Gemini API Key
gemini-api-key:
  - api-key: "你的 Google AI Studio API Key"

# 开启调试日志（可选）
debug: true
```

拿到 Gemini API Key 的方式：去 [Google AI Studio](https://aistudio.google.com/apikey) 免费申请。

### 端口映射

如果用 Docker，建议映射一个独立端口：

```yaml
# docker-compose.yml
ports:
  - "9417:8317"  # 对外9417，容器内8317
```

## 调用生图

部署完成后，用标准的 OpenAI API 格式就能生图：

```python
import requests
import base64
from pathlib import Path

resp = requests.post("http://127.0.0.1:9417/v1/chat/completions",
    headers={"Authorization": "Bearer your-client-key"},
    json={
        "model": "gemini-3.1-flash-image",
        "messages": [{
            "role": "user",
            "content": "画一只橘猫在沙发上跳跃，简笔画风格，白色背景"
        }],
        "max_tokens": 4096
    },
    timeout=120
)

data = resp.json()
# 图片在 choices[0].message.images 字段
images = data["choices"][0]["message"]["images"]
img_url = images[0]["image_url"]["url"]  # data:image/jpeg;base64,...

# 保存图片
b64 = img_url.split(",", 1)[1]
Path("output.jpg").write_bytes(base64.b64decode(b64))
print("Done!")
```

### 响应格式

CLIProxyAPI 返回的图片在 `choices[0].message.images` 数组里（注意不是 `content`）：

```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "这是一只可爱的橘猫...",
      "images": [{
        "image_url": {
          "url": "data:image/jpeg;base64,/9j/4AAQ..."
        }
      }]
    }
  }]
}
```

## 实战：批量生成小红书配图

这是我实际在用的场景——每天给小红书账号生成科普配图。

### 批量生图脚本

```python
import requests, base64, time
from pathlib import Path

ENDPOINT = "http://127.0.0.1:9417/v1/chat/completions"
API_KEY = "your-client-key"
OUT_DIR = Path("./images")
OUT_DIR.mkdir(exist_ok=True)

prompts = [
    "简笔画风格，一只猫凌晨3点从沙发上起飞...",
    "简笔画风格，猫上完厕所后疯狂冲刺...",
    # ... 更多 prompt
]

for i, prompt in enumerate(prompts, 1):
    resp = requests.post(ENDPOINT, json={
        "model": "gemini-3.1-flash-image",
        "messages": [{"role": "user", "content": prompt}],
        "max_tokens": 4096
    }, headers={"Authorization": f"Bearer {API_KEY}"}, timeout=120)

    images = resp.json()["choices"][0]["message"]["images"]
    b64 = images[0]["image_url"]["url"].split(",", 1)[1]
    
    out = OUT_DIR / f"image_{i}.jpg"
    out.write_bytes(base64.b64decode(b64))
    print(f"[{i}] ✅ {out}")
    
    time.sleep(2)  # 避免限流
```

### 模型选择

| 模型 | 特点 | 适用场景 |
|------|------|----------|
| `gemini-3-pro-image-preview` | 质量最高，配额较紧 | 重要的封面图 |
| `gemini-3.1-flash-image` | 速度快，配额宽松 | 批量生图、日常使用 |

**建议**：日常用 `flash-image`，配额不容易爆。`pro-image` 留给关键场景。

## 进阶：接入更多模型

CLIProxyAPI 不只能反代 Gemini，还支持通过 `openai-compatibility` 配置接入任何 OpenAI 兼容的服务：

```yaml
openai-compatibility:
  - name: Minimax
    base-url: https://api.minimaxi.com/v1
    api-key-entries:
      - api-key: "sk-xxx"
    models:
      - name: MiniMax-M2.1

  - name: 阿里百炼
    base-url: https://coding.dashscope.aliyuncs.com/v1
    api-key-entries:
      - api-key: "sk-xxx"
    models:
      - name: kimi-k2.5
```

这样你的工具链只需要对接一个 endpoint，后面接什么模型随时切换。

## 踩坑记录

1. **图片在 `images` 字段不在 `content`**：CLIProxyAPI 返回图片的位置是 `choices[0].message.images[]`，不是 `content`，别找错了。

2. **3:4 竖版图片**：在 prompt 里指定 `768x1024` 即可，实际输出会是 `896x1200`。

3. **配额限制**：`pro-image` 模型配额比较紧，批量生图容易 429。换 `flash-image` 基本不会遇到。

4. **超时设置**：生图比纯文本慢很多，`timeout` 建议设 120 秒以上。

## 总结

CLIProxyAPI 做的事情很简单但很实用：

- **统一 API 格式**：不管后面是 Gemini、Minimax 还是别的，对外都是 OpenAI 格式
- **零改造接入**：现有工具链不用改代码，换个 `base_url` 就行
- **生图能力平权**：Gemini 的生图能力通过反代变成了任何工具都能调用的标准 API

如果你也在用 AI Agent 做内容生产（小红书、公众号、博客配图），这个方案值得一试。

---

*相关链接：*
- [CLIProxyAPI GitHub](https://github.com/eceasy/cli-proxy-api)
- [Google AI Studio（申请 API Key）](https://aistudio.google.com/apikey)
- [Gemini 生图文档](https://ai.google.dev/gemini-api/docs/image-generation)
