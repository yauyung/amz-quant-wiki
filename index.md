# Wiki Index

> AMZ Quant 知识库目录。每个 Wiki 页面按类型列出，附一句话摘要。
> 查询时先读此文件定位相关页面，再深入阅读。
> 最后更新：2026-06-11 | 总页数：14

## AI · 人工智能

### Agents & Multi-Agent（智能体）

- [[多 Agent 量化体系（四代架构）|multi-agent-quant-system]] — 二代→四代的量化体系演进，Agent 分工模型

---

## Quant · 量化

### Factors & Strategies（因子与策略）

- [[多因子选股核心因子|core-factors]] — 4 个核心因子：Momentum、Low Vol、GP、EY/BM
- [[美股多因子路线图|equity-factor-roadmap]] — 12 周分阶段实施路线

### Backtest & Validation（回测与验证）

- [[9 级验证阶梯|validation-ladder]] — Rank IC → OOS 的因子验证纪律与过关线

---

## Concepts · 概念

- [[AMZ Quant 核心 Doctrine|amz-quant-core-doctrine]] — "信号先量化，Agent 后解释"
- [[Agent 边界|agent-boundary]] — Process alpha vs Return alpha，LLM 不允许的操作
- [[量纲混用|duration-mixing]] — 季度/年度值混合的陷阱与 TTM 修复
- [[Survivorship 泄露|survivorship-bias]] — 当前成分股回测历史的 look-ahead bias
- [[市值扭曲]] — 用 adj_close 算市值的分红偏差

## Entities · 实体

- [[quant-equity]] — AMZ Quant 美股多因子研究项目仓库
- [[TradingAgents|tradingagents]] — 12 Agent LangGraph 辩论式交易框架
- [[Freqtrade|freqtrade]] — 开源加密交易机器人，amz quant 主引擎

## Cross-cutting · 交叉

- [[LLM 交易 Agent 论文精读|llm-trading-agent-papers]] — KTD-FIN / News RL / TradeArena 三篇精读

## Raw Sources · 原始资料

| 来源 | 路径 |
|------|------|
| 美股多因子最终路线 | `原始资料/文章/amz_quant_美股多因子最终路线_20260531.md` |
| Claude Code 审计报告 | `原始资料/文章/claude_code_audit_20260611.md` |
| TradingAgents 架构分析 | `原始资料/文章/tradingagents_arch.md` |
| LLM 交易 Agent 论文精读 | `原始资料/论文/论文精读_LLM交易Agent_20260529.md` |
