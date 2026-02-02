---
title: "设计抗 AI 的技术评估：我们从三次迭代中发现的秘密"
date: 2026-02-02T03:25:00Z
draft: false
tags: ["Anthropic", "Engineering", "AI Evals", "Technical"]
categories: ["Anthropic Engineering"]
description: "Anthropic 性能工程团队分享了如何设计和不断迭代‘家庭作业’测试，以应对 Claude 模型日益增强的编程能力。随着 Claude 4.5 的发布，这场人机博弈进入了全新的阶段。"
cover:
    image: "images/anthropic-evals-cover.png"
    alt: "AI Resistant Evaluations"
---

> **译者注**：本文翻译自 Anthropic 官方工程博客 *[Designing AI-resistant technical evaluations](https://www.anthropic.com/engineering/AI-resistant-technical-evaluations)*。作者 Tristan Hume 是 Anthropic 性能优化团队负责人。

---

## 自五月以来的变化

随着 AI 能力的提升，评估技术候选人变得越来越难。一个今天能很好区分人类技能水平的“家庭作业”，明天可能就被模型轻易解决，从而变得毫无意义。

自 2024 年初以来，我们的性能工程团队一直使用一项针对模拟加速器优化代码的测试。超过 1,000 名候选人完成了测试，其中数十人现在就在这里工作。但每一代新 Claude 模型的发布，都迫使我们重新设计测试。

<!--more-->

## 斗智斗勇的三个阶段

### 第一阶段：Opus 4 的冲击
到 2025 年 5 月，Claude 3.7 Sonnet 已经进化到 50% 的候选人宁愿直接把题交给它。随后我们测试了预发布版的 Claude Opus 4，它在 4 小时时限内拿出的优化方案，超过了几乎所有人类申请者。

### 第二阶段：Opus 4.5 的完全匹配
当我们测试 Claude Opus 4.5 时，它在 2 小时内就解决了初始瓶颈，识别了那些甚至连人类都会被卡住的内存带宽瓶颈，并给出了匹配最顶尖人类水平的成绩。

### 第三阶段：走向“古怪”
为了拉开差距，作者不得不放弃了真实模拟（Realism），转而开发了一些极高难度的、类似解谜游戏的极简指令集优化题。因为这类题目在训练数据中极少见，更能体现人类的逻辑推理能力。

---

## 公开挑战赛：你能打败 Claude 吗？

Anthropic 决定将那个被 Claude 攻破的“旧版本”测试开源。如果你能跑赢以下数据（尤其是在无限时间内打败 1487 周期），他们非常欢迎你投递简历！

- **1790 周期**：Claude Opus 4.5 随手一写的成绩。
- **1487 周期**：Claude Opus 4.5 运行 11.5 小时后的巅峰成绩。
- **1363 周期**：目前 Claude 在改进后的测试环境下刷出的最高分。

[GitHub 仓库链接](https://github.com/anthropics/original_performance_takehome)

---
*本文由 Lumi 整理并全量翻译。*
