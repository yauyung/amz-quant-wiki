---
title: LLM 交易 Agent 论文精读
created: 2026-06-11
updated: 2026-06-11
type: comparison
tags: [paper, agent, strategy, comparison]
sources: [原始资料/论文/论文精读_LLM交易Agent_20260529.md]
confidence: high
---

# LLM 交易 Agent 论文精读

Judy 精读的 3 篇关键论文。

## KTD-FIN（2026.05）

**京东 · 清华 | arXiv:2605.28359**

LLM agent 的"能力幻觉"——看似赚钱，实则靠因子暴露，选股 alpha ≈ 零。

- 四层掩码协议逐步匿名化，杜绝 LLM 预训练记忆替代推理
- Barra 风格归因拆解收益：β + 风格因子 + 选股 alpha
- 框架：[[Qlib]]

## News-Aware Direct RL（2025.10）

**中国科学院大学 | arXiv:2510.19173**

LLM 读新闻 → Gemini 情感分 → LSTM/Transformer 编码 → DDQN/GRPO RL 决策。

- 标的：BTC/USDT 1 分钟 K 线
- 可适配 [[Freqtrade]]（加密 + OHLCV + 相同技术栈）

## TradeArena（2026.05）

**Virginia Tech | arXiv:2605.28850**

LLM agent 在亏损前内部表征已开始退化——可提前预警。

- 表征监控：plan embedding 有效秩收缩检测
- 检测精度：0.807
- 长期价值：作为风控诊断层，监视 agent 推理质量

## 实施优先级

| 优先级 | 任务 | 产出 |
|--------|------|------|
| 1 | KTD-FIN Barra 归因 | 策略收益拆解 |
| 2 | News RL DDQN+LSTM | 可运行 RL 策略 |
| 3 | TradeArena 表征监控 | 风控诊断层 |

## 相关

- [[Agent 边界]]
- [[TradingAgents]]
- [[Freqtrade]]
