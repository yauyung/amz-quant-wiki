---
title: 多 Agent 量化体系（四代架构）
created: 2026-06-11
updated: 2026-06-11
type: concept
tags: [agent, strategy, amz-quant]
sources: [原始资料/文章/tradingagents_arch.md, 原始资料/文章/amz_quant_美股多因子最终路线_20260531.md]
confidence: medium
---

# 多 Agent 量化体系

amz quant 的四代量化体系演进路线。

## 四代定义

| 代 | 特征 | 框架 | 状态 |
|---:|------|------|------|
| 二代 | 系统化交易（规则驱动，回测+自动化） | Freqtrade/vnpy/QuantConnect | ✅ |
| 三代 | ML + 高频（模型驱动） | QLib/FinRL/HFTBacktest | 🔜 |
| 四代 | Multi-Agent 协作（研究/数据/策略/执行/风控分工） | amz quant 自研 | 🔜 |

## 四代分工

```
研究 Agent → 数据 Agent → 策略 Agent → 执行 Agent → 风控 Agent
     │            │           │           │            │
     └────────────┴───────────┴───────────┴────────────┘
                        共享知识库
```

## 当前进展

- 二代已就绪：Freqtrade + 多策略
- TradingAgents 探索完成（12 Agent 辩论式架构）
- 三代 Qlib/ML 路线待启动
- 四代为远期目标

## 关键教训

- KTD-FIN 论文：LLM agent 选股 alpha ≈ 零，不应替代统计信号
- Claude Code 审计：数据质量比 Agent 架构更重要
- 不要过早平台化——先验证信号，再谈自动化

## 相关

- [[TradingAgents]]
- [[Freqtrade]]
- [[Agent 边界]]
