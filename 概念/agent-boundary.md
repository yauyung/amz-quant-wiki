---
title: Agent 边界（Process Alpha vs Return Alpha）
created: 2026-06-11
updated: 2026-06-11
type: concept
tags: [agent, strategy, amz-quant]
sources: [原始资料/文章/amz_quant_美股多因子最终路线_20260531.md, 原始资料/论文/论文精读_LLM交易Agent_20260529.md]
confidence: high
---

# Agent 边界

Agent/LLM 在量化系统中的角色边界。核心原则：**Agent 是 process alpha，不是 return alpha。**

## 允许做的事

- 候选股解释
- 财报 / 10-K / 10-Q / 8-K 阅读
- bull / bear thesis 草稿
- 交易前 checklist
- 数据异常解释
- 回测结果总结
- 交易后复盘
- 研究日志整理

## 禁止做的事

- 🚫 直接给买卖信号
- 🚫 直接决定仓位
- 🚫 直接下单
- 🚫 在因子打分上游替代统计信号
- 🚫 用叙事结果反向选择因子

## 学术证据

KTD-FIN（2026）实验表明：LLM agent 收益 ≈ 市场 β + 风格因子暴露，选股 alpha 接近零甚至为负。这验证了"Agent 不应替代统计信号"的边界设计是正确的。

## 相关

- [[AMZ Quant 核心 Doctrine]]
- [[LLM 交易 Agent 论文精读]]
