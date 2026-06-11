---
title: 美股多因子路线图
created: 2026-06-11
updated: 2026-06-11
type: concept
tags: [strategy, factor, amz-quant]
sources: [原始资料/文章/amz_quant_美股多因子最终路线_20260531.md]
confidence: high
---

# 美股多因子路线图

AMZ Quant 12 周研究路线，问题驱动、分阶段推进。

## 当前阶段

**Phase: factor/data/QA MVP + review-only Agent adapter**

已完成 rung 0-7，rung 8（多因子合成）和 rung 9（组合净值回测）待启动。

## 12 周路线

| 周 | 阶段 | 关键交付 |
|----|------|---------|
| 1-2 | 数据底座 | universe + ticker-CIK 映射 + data QA |
| 3-4 | 基础因子 | 4 因子 + winsorize/z-score/行业中性化 |
| 5-6 | 验证 MVP | Alphalens 风格 IC/quantile/turnover 报告 |
| 7-8 | 稳健性 | Walk-forward + FF5 归因 |
| 9-10 | 事件增强 | SUE/PEAD + SEC filing events |
| 11-12 | Agent 解释层 | 候选解释 memo + paper portfolio |

## 不作为主线

1. LLM 直接输出买卖信号
2. 高频/日内统计套利
3. 复杂 long-short
4. 主观选股
5. 堆几十个因子做 p-hacking

## 相关

- [[AMZ Quant 核心 Doctrine]]
- [[多因子选股核心因子]]
- [[9级验证阶梯]]
- [[quant-equity]]
