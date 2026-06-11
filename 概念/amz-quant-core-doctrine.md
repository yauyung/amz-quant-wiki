---
title: AMZ Quant 核心 Doctrine
created: 2026-06-11
updated: 2026-06-11
type: concept
tags: [amz-quant, strategy, factor]
sources: [原始资料/文章/amz_quant_美股多因子最终路线_20260531.md]
confidence: high
---

# AMZ Quant 核心 Doctrine

> **信号先量化，Agent 后解释；先建立"不撒谎"的数据与验证系统，再谈策略增强和自动化。**

AMZ Quant 美股多因子的核心教义，由 OpenAI 和 Claude/Anthropic 两份独立研究报告交叉验证后合并确定。

## 最终原则

**OpenAI 定边界，Claude 定步骤。**

- OpenAI 提供战略克制：不让 LLM 炒股、不做高频、不急做实盘 long-short
- Claude 提供执行规程：IC/分层/成本/样本外验证、Parquet 工程栈、survivorship 近似方案

## 硬规则

1. **信号先量化** — 统计/量化信号是主线，Agent 层是 process alpha（解释/总结），不是 return alpha（买卖决策）
2. **Agent 禁止输出买卖** — 可解释、可草拟 thesis、可读财报、可总结回测，但绝不给买卖信号/仓位/下单
3. **原始数据落盘** — EDGAR JSON、OHLCV、universe 快照，保留 provenance
4. **用 filed 不用 end** — 财务数据按实际可得时间对齐，不用财期结束日
5. **当前 S&P 500 ≠ 历史 universe** — survivorship bias 是真实的，需诚实近似
6. **复权价用于价格因子** — 动量/低波用 adjusted close
7. **不过关就放弃** — 因子在成本后/样本外/分层/IC 上不过关，不加规则救活

## 相关

- [[美股多因子路线图]] — 完整实施路线
- [[9级验证阶梯]] — 因子验证纪律
- [[Agent 边界]] — LLM/Agent 在量化系统中的角色
