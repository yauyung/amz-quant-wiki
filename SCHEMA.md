# Wiki Schema

## Domain

AMZ Quant（惊天骇地的量化）知识库。两条主线：

- **AI** — LLM、Agent 架构、模型训练/推理、alignment、多 Agent 系统
- **Quant** — 因子研究、交易策略、市场微观结构、风险管理、回测/验证

知识积累服务于 AI+Quant 交叉领域的研究与实践。Agent 从这里读取已编译的知识，而非每次都从原始文献重新推理。

## Conventions

- 文件名：小写英文，连字符，无空格（`gross-profitability-factor.md`）
- 中文术语可出现在内容和标题中，文件名用英文
- 每个 Wiki 页面必须以 YAML frontmatter 开头（见下方格式）
- 页面间用 `[[wikilinks]]` 链接（每页至少 2 个出站链接）
- 更新页面时，务必更新 `updated` 日期
- 新建页面必须添加到 `index.md`
- 每次操作必须追加到 `log.md`
- **溯源标记**：整合 3+ 来源的页面，在段落后追加 `^[raw/papers/source.md]`

## Frontmatter

```yaml
---
title: 页面标题
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: entity | concept | comparison | query | summary
tags: [从下方分类中选择]
sources: [raw/papers/xxx.md]
confidence: high | medium | low
contested: true  # 存在未解决的矛盾时设置
contradictions: [other-page-slug]
---
```

## Tag Taxonomy

### AI 线
- `model` — 模型架构、能力
- `agent` — Agent 系统、多 Agent 编排
- `training` — 训练方法、微调、RLHF
- `inference` — 推理优化、部署
- `alignment` — 对齐、安全
- `benchmark` — 评测、基准

### Quant 线
- `factor` — 因子定义、研究
- `strategy` — 交易策略
- `portfolio` — 组合构建、权重
- `risk` — 风险管理
- `backtest` — 回测方法、验证
- `market-microstructure` — 市场微观结构
- `data` — 数据源、数据质量

### 通用
- `person` — 人物
- `paper` — 论文
- `company` — 公司/组织
- `tool` — 工具/框架
- `comparison` — 对比分析
- `timeline` — 时间线
- `controversy` — 争议/辩论
- `prediction` — 预测/展望
- `amz-quant` — 内部项目相关

规则：页面使用的每个 tag 必须已出现在此分类中。新 tag 先加到这里再使用。

## Page Thresholds

- **创建页面**：当实体/概念出现于 2+ 来源，或是一个来源的核心主题
- **追加已有页面**：来源涉及已覆盖的内容
- **不创建页面**：只被顺带提及、次要细节、领域外内容
- **拆分页面**：超过 ~200 行时，拆为子主题并交叉链接
- **归档页面**：内容被完全替代时，移至 `_archive/`，从 index 移除

## Entity Pages

一人/一组织/一模型/一框架一页。包含：
- 概述 / 是什么
- 关键事实和日期
- 与其他实体的关系（[[wikilinks]]）
- 来源引用

## Concept Pages

一个概念或主题一页。包含：
- 定义 / 解释
- 当前知识状态
- 开放问题或争议
- 相关概念（[[wikilinks]]）

## Comparison Pages

并排对比分析。包含：
- 对比什么、为什么
- 对比维度（优先表格）
- 结论或综合判断
- 来源

## Update Policy

新信息与已存内容冲突时：
1. 检查日期——新来源通常替代旧来源
2. 若确实矛盾，同时记录两种观点，标注日期和来源
3. 在 frontmatter 标注 `contradictions: [page-name]`
4. Lint 时标记供人工审查
