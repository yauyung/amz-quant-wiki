---
title: Survivorship 泄露
created: 2026-06-11
updated: 2026-06-11
type: concept
tags: [data, backtest, factor]
sources: [原始资料/文章/claude_code_audit_20260611.md]
confidence: high
---

# Survivorship 泄露

用当前成分股回测历史是量化研究中最常见的 look-ahead bias。Claude Code 审计（2026-06-11）将此列为三个 P0 缺陷之二。

## 问题

基本面验证 runner（`run_gross_profitability_validation.py`、`run_valuation_validation.py`）在 `facts_df["ticker"].unique()` 上运行——即 **50 个当前 S&P 500 成分股**。

## 后果

- 已退市的股票不在样本中 → 回测结果系统性高估
- 已加入的新兴股票在历史回测中存在 → 存活者偏差
- 直接违反 CLAUDE.md Rule 5

## 解决方案（已实现）

- 使用 `build_historical_approx_universe`（Wikipedia S&P 500 历史修订 + 当前成分回溯）
- 基本面 runner 通过 `universe=` 参数接入
- Universe 缺失时明确告警，不静默回退

## 局限性

Historical-approx universe 只回溯到 2024-01，仍不是 CRSP 级别。

## 相关

- [[量纲混用]]
- [[市值扭曲]]
- [[AMZ Quant 核心 Doctrine]]
