---
title: 量纲混用（Flow/Stock Duration Mixing）
created: 2026-06-11
updated: 2026-06-11
type: concept
tags: [data, factor, backtest]
sources: [原始资料/文章/claude_code_audit_20260611.md]
confidence: high
---

# 量纲混用

基本面因子中最隐蔽的数据陷阱之一。Claude Code 审计（2026-06-11）将此列为三个 P0 缺陷之首。

## 问题

`pit_latest_fact` 按 `end` 字段选最新值，但不区分 10-Q（季度）和 10-K（年度）。对于流量型事实（GrossProfit、Revenues、NetIncomeLoss），10-Q 中是季度值，10-K 中是年度值，幅度差约 **4 倍**。

当 Q1 10-Q（end=3/31）晚于前一年 10-K（end=12/31）时，因子值静默从年度切换为季度，横截面排序被污染。

## 根因

1. SEC `extract_companyfacts_records` 丢弃了 XBRL `start` 字段
2. `CLAUDE.md` Rule 3 的 provenance 列表遗漏了 `start`
3. `pit_latest_fact` 不区分流量和存量事实

## 解决方案（已实现）

1. 保留 `start` 字段
2. 新增 `pit_ttm_fact`：对流量型事实取最近 4 个季度求和（TTM），不足 4 季度回退到年度值
3. 存量事实（Assets、Equity）继续用 `pit_latest_fact` 单值

## 相关

- [[市值扭曲]] — 另一个 P0 数据陷阱
- [[Survivorship 泄露]]
- [[多因子选股核心因子]]
