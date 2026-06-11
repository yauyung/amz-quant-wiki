---
title: quant-equity 项目架构
created: 2026-06-11
updated: 2026-06-11
type: entity
tags: [amz-quant, factor, backtest, tool]
sources: [原始资料/文章/amz_quant_美股多因子最终路线_20260531.md, 原始资料/文章/claude_code_audit_20260611.md]
confidence: high
---

# quant-equity

AMZ Quant 美股多因子研究项目。仓库：`yauyung/quant-equity`。

## 数据栈

- 价格：yfinance OHLCV + adjusted close
- 基本面：SEC EDGAR CompanyFacts（保留 `filed/accn/start/end` 等 provenance）
- 因子：Ken French FF5/Momentum
- 存储：Parquet + pandas/pyarrow（DuckDB 预留）

## 因子模块

| 因子 | 模块 | 状态 |
|------|------|------|
| 12-1 Momentum | `src/factors/momentum.py` | ✅ |
| Low Volatility | `src/factors/low_volatility.py` | ✅ |
| Gross Profitability | `src/factors/gross_profitability.py` | ✅ TTM |
| Earnings Yield | `src/factors/valuation.py` | ✅ |
| Book-to-Market | `src/factors/valuation.py` | ✅ |

## P0 缺陷（已修复）

1. [[量纲混用]] — TTM 聚合
2. [[Survivorship 泄露]] — historical-approx universe 接入
3. [[市值扭曲]] — raw close 替代 adj_close

## Agent 层

`src/agents/financial_services_adapter.py` — 只读解释层，不输出买卖信号。

## 测试

360+ 单元测试，网络无关，`make test` 运行。

## 相关

- [[AMZ Quant 核心 Doctrine]]
- [[多因子选股核心因子]]
- [[9级验证阶梯]]
