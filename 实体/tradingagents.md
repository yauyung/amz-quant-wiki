---
title: TradingAgents 多 Agent 交易框架
created: 2026-06-11
updated: 2026-06-11
type: entity
tags: [agent, tool, strategy]
sources: [原始资料/文章/tradingagents_arch.md]
confidence: high
---

# TradingAgents

arXiv 2412.20138 · 基于 LangGraph 的多 Agent LLM 金融交易框架。

## 架构

用 LangGraph 把 ~12 个 LLM Agent 串成有向图，模拟一家交易公司的完整决策链：

```
分析师团队（4个）→ 多空辩论 → 研究经理 → 交易员 → 风控辩论 → 投资组合经理 → 决策
```

## Agent 角色

| 团队 | Agent | 模型等级 |
|------|-------|---------|
| 分析师 | Market（技术）、Social（情绪）、News（宏观）、Fundamentals（基本面） | quick |
| 研究员 | Bull、Bear（多空辩论） | quick |
| 经理 | Research Manager、Trader、Portfolio Manager | deep |
| 风控 | Aggressive、Conservative、Neutral（三方辩论） | quick |

## 技术要点

- **数据路由**：`dataflows/interface.py` 的 `route_to_vendor()` — 接入 ccxt 的唯一切入点
- **加密支持**：已有 `asset_type="crypto"` 参数，但无 ccxt 实现
- **LLM**：本项目配置 `openrouter` 通道 + 小米 MiMo（`mimo-v2.5-pro`，推理模型）
- **决策输出**：5 档评级 Buy / Overweight / Hold / Underweight / Sell

## 与 amz quant 的关系

已复现 v0.2.5，ccxt 适配完成，Freqtrade 策略集成完成。但每决策 ~9 次 LLM 调用，不适合高频。

## 相关

- [[Freqtrade]]
- [[多 Agent 量化体系]]
- [[LLM 交易 Agent 论文精读]]
