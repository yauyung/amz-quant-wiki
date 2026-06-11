---
title: 多因子选股核心因子
created: 2026-06-11
updated: 2026-06-11
type: concept
tags: [factor, strategy]
sources: [原始资料/文章/amz_quant_美股多因子最终路线_20260531.md]
confidence: high
---

# 多因子选股核心因子

AMZ Quant 第一阶段选定的 4 个核心因子。月频、美股中大盘、long-only top quintile。

## 价格因子（优先）

### 12-1 Momentum
- 过去 12 个月收益，跳过最近 1 个月
- 纯价格因子，最适合做 pipeline 验证
- 使用 adjusted close

### Low Volatility
- 过去 60/126/252 日波动率
- 可扩展 beta、idiosyncratic volatility、max drawdown
- 作用：降低动量崩溃和组合波动

## 基本面因子

### Gross Profitability
- `(Revenue - COGS) / Total Assets`
- Novy-Marx (2013) 质量因子
- 注意：金融股/REIT 口径不可比
- ⚠️ 已修复：[[量纲混用]] → TTM

### Earnings Yield / Book-to-Market
- 价值因子
- 注意负值、行业差异、market cap 日期对齐
- ⚠️ 已修复：[[市值扭曲]] → 用 raw close

## 流动性因子

ADV、Amihud illiquidity 等不做 alpha，只做过滤/成本惩罚。

## 相关

- [[9级验证阶梯]] — 因子验证纪律
- [[AMZ Quant 核心 Doctrine]]
- [[量纲混用]] / [[市值扭曲]] / [[Survivorship 泄露]] — P0 缺陷与修复
