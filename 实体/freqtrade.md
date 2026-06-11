---
title: Freqtrade
created: 2026-06-11
updated: 2026-06-11
type: entity
tags: [tool, strategy, backtest]
sources: [原始资料/论文/论文精读_LLM交易Agent_20260529.md, 原始资料/文章/tradingagents_arch.md]
confidence: high
---

# Freqtrade

开源加密交易机器人框架（35k+ ⭐），amz quant 的主回测/执行引擎。

## 当前配置

- 版本：2026.4
- 路径：`/root/quant/freqtrade/`
- 数据：Binance 5m K 线 BTC/ETH/SOL

## 策略

| 策略 | 类型 | 状态 |
|------|------|------|
| SmaCross | 教学/基线 | ✅ |
| TimeSeriesMomentum | Moskowitz 2012 学术策略 | ✅ |
| TradingAgentsStrategy | LLM 多 Agent 驱动（双模式） | ✅ 回测跑通 |

## 与 TradingAgents 集成

`TradingAgentsStrategy.py` 支持双模式：
- `use_llm=True` — 调用 TradingAgents 多 Agent 管道
- `use_llm=False` — 纯规则模式，回测安全

关键限制：MiMo 是推理模型，每决策 ~9 次 LLM 调用，不能每根 K 线都调。策略须"低频调用 + 缓存 + 可关闭 LLM"。

## 相关

- [[TradingAgents]]
- [[LLM 交易 Agent 论文精读]]
- [[多 Agent 量化体系]]
