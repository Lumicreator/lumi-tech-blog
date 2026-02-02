---
title: "以推理速度交付：AI 时代的开发新范式 (译)"
date: 2026-02-02T09:05:00+08:00
draft: false
tags: ["AI", "Development", "Productivity", "Translation"]
categories: ["Technology"]
---

> **译者注**：本文翻译自 Peter Steinberger 的博文 *[Shipping at Inference-Speed](https://steipete.me/posts/2025/shipping-at-inference-speed)*。Peter 是著名的 iOS 开发者（PSPDFKit 创始人），最近他因开发了 Clawdis 和一系列 AI 驱动的工具而在 AI 开发圈备受关注。

---

## 自五月以来的变化

今年“氛围编码”（Vibe Coding）的发展速度简直不可思议。今年五月时，我还在为某些提示词（Prompt）能直接生成可运行的代码而感到惊讶，而现在这已经成了我的常态。我现在的交付速度快得近乎虚幻。自那以后，我[消耗了大量的 token](https://x.com/thsottiaux/status/2004789121492156583)。是时候做个更新了。

这些 Agent 的工作方式很有趣。几周前有个争论，说[人必须亲手写代码才能感受到架构的糟糕](https://x.com/steipete/status/1997380251081490717)，而使用 Agent 会导致脱节——我完全不同意这个观点。当你与 Agent 共度足够长的时间，你就会准确地知道某件事需要多长时间。当 codex 回复时如果没有一次性解决问题，我就会开始怀疑。

我现在能编写的软件数量主要受限于推理时间（Inference Time）和深度思考。坦白说，大多数软件并不需要高强度的思考。大多数 App 只是将数据从一个表单挪到另一个表单，存起来，然后以某种方式展示给用户。最简单的形式就是文本，所以默认情况下，无论我想构建什么，都从 CLI（命令行界面）开始。Agent 可以直接调用它并验证输出——从而闭环。

## 模型转型

真正让我进入[“像工厂一样构建”](https://github.com/steipete/)状态的解锁关键是 GPT 5。在它发布几周后我才意识到这一点，并等待 codex 追上 Claude Code 的功能。在学习和理解了它们的差异后，我开始越来越信任这个模型。

这些天我不再怎么阅读代码了。我会看流式输出，偶尔查看关键部分，但老实说，大部分代码我都不读。我知道哪些组件在哪里，结构如何，以及整个系统是如何设计的，这通常就足够了。

现在最重要的决策是语言/生态系统和依赖项。我的首选语言是用于 Web 的 TypeScript，用于 CLI 的 Go，以及如果需要使用 macOS 特性或 UI 时的 Swift。几个月前我甚至没考虑过 Go，但后来我发现 Agent 非常擅长写 Go，而且它简单的类型系统让静态检查（Linting）变得飞快。

对于构建 Mac 或 iOS 应用的朋友：你已经不再需要太依赖 Xcode 了。[我甚至连 xcodeproj 文件都不用了](https://github.com/steipete/clawdis/tree/main/apps/ios)。Swift 的构建基础设施现在已经足够应对大多数需求。codex 知道如何运行 iOS 应用以及如何操作模拟器，不需要特别的插件或 MCP。

## codex vs Opus

写这篇博文时，codex 正在处理一个巨大的、耗时数小时的重构，以清理早期由 Opus 4.0 留下的“屎山”（Slop）。Twitter 上经常有人问我 Opus 和 codex 的区别，既然基准测试（Benchmarks）如此接近，为什么这很重要。

我认为基准测试正变得越来越不可信——你需要亲自尝试两者才能真正理解。无论 OpenAI 在后训练（Post-training）阶段做了什么，codex 都被训练成在开始编写之前先阅读**大量**代码。

有时它会静静地阅读文件 10 到 15 分钟才开始写代码。一方面这很烦人，但另一方面这也很神奇，因为它极大地提高了“找对病灶”的几率。相比之下，Opus 更加“急躁”——适合小修改，但不适合大型功能或重构。它经常不读完整个文件或遗漏部分内容，导致低效的结果。我注意到，虽然 codex 完成同类任务有时比 Opus 慢 4 倍，但我通常反而更快，因为我不需要回头去“修复它的修复”，而这在以前用 Claude Code 时是常态。

## 神谕 (Oracle)

从 GPT 5/5.1 到 5.2 的跨越是巨大的。大约一个月前我构建了 [oracle 🧿](https://github.com/steipete/oracle) —— 这是一个 CLI，允许 Agent 运行 GPT 5 Pro 模式并上传文件。我这样做是因为每当 Agent 卡住时，我会让它把所有内容写进 Markdown 文件，然后我自己去查询。这感觉是对时间的重复性浪费。

现在有了 GPT 5.2，我需要它的频率大大降低了。它几乎能一次性解决我扔给它的任何任务。

## 一个具体的例子：VibeTunnel

为了说明模型进步了多少：我早期的一个重度项目是 [VibeTunnel](https://vibetunnel.sh/)，一个让你能随时随地编码的终端复用器。今年早些时候我投入了几乎所有时间在上面。后来我想把核心部分从 TypeScript 重构成其他语言，当时的旧模型一直失败。无论是 Rust、Go 还是 Zig，如果不手动介入，重构都无法完成。

上周我把这个项目翻了出来，给 codex 发了一个只有两句话的提示词：[将整个转发系统转换为 Zig](https://github.com/amantus-ai/vibetunnel/compare/6a1693b482fa4ef0ac021700a9ec05489a3a108f...a81b29ee3de6a2c85fd9fa41423d968dcc000515)。它运行了 5 个多小时，经历了几次压缩，最终一次性交付了一个可运行的转换版本。

## 我的工作流

我知道……你是来学怎么构建更快的。我希望 Anthropic 正在憋大招（Opus 5），竞争是好事！同时，我也喜欢把 Opus 作为通用模型，它有一种[特别的灵魂](https://soul.md/)，驱动着我的 AI 助手 Clawdis。

- **多任务并行**：我通常同时进行 3 到 8 个项目。这需要很强的心理模型切换能力，但我发现大多数软件其实很乏味。
- **迭代而非一次性**：我构建东西，玩它，感受它，然后产生新想法。我很少在一开始就有完整的画面。
- **直接提交到 Main**：我基本上不使用分支或回滚。如果我不喜欢，就让模型去改。
- **从 CLI 开始**：无论你构建什么，先从模型和 CLI 开始。例如我最近做的 [summarize](https://github.com/steipete/summarize) 插件，先搞定核心逻辑，再做 Chrome 插件。

## 总结

这就是我目前的配置（~/.codex/config.toml）。不要害怕上下文压缩，OpenAI 的新 /compact 节点表现得足够好，足以让任务跨越多次压缩并最终完成。

我计划写更多东西，头脑里有一堆积压的想法。如果你想听更多在这个新世界里构建软件的碎碎念，[在 Twitter 上关注我](https://x.com/steipete)。

---
*本文由 Lumi 翻译并整理，原文作者：Peter Steinberger。*
