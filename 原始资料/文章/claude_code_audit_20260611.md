---
source_path: /root/quant/notes/claude_code_audit_20260611.md
ingested: 2026-06-11
sha256: 5b76244bb9991faebcd44f16635f8569a3c800ce693f40ea991d5ee96efa75b0
---

# quant-equity 深度代码审计报告

**日期**: 2026-06-11 | **审计方**: Claude Code (opus + xhigh effort) | **审计 commit**: `7d5ebae`

---

## 总评

这是一个**工程质量出色、但建立在无法支撑其结论的数据之上**的代码库。

工程纪律（纯函数、provenance 保留、look-ahead 防护、优雅降级、360 个网络无关测试、诚实的 caveat、正确的验证阶梯排序）在同类型研究代码中属于上乘。

但**三个已被证实的正确性问题**意味着：目前产出的所有"实盘面板"数字——gross profitability 和 valuation 的 rung 1–7 结果——不仅仅是统计上弱，而是**测量了错误的量**。项目自评"瓶颈是数据深度而非因子广度"方向正确，但严重低估了问题：数据不仅仅是浅，而是**以验证阶梯无法检测的方式出错了**，因为阶梯只对输入的值打分。

---

## 一、CRITICAL — 因子值的量纲不一致（流动/存量时间跨度混用）

### 问题

`pit_latest_fact`（`gross_profitability.py:333`）对每个 `(ticker, date)` 选择**财期 `end` 最新**的事实值，按 `filed` 打破平局。但**不区分 10-Q（季度）和 10-K（年度）**。

对于流量型事实——`GrossProfit`、`Revenues`、`CostOfRevenue`、`NetIncomeLoss`、EPS——10-Q 中是**季度值**，10-K 中是**年度值**。因为 Q1 10-Q 的 `end`（3月31日）晚于前一年 10-K（12月31日），一旦有 10-Q 提交，因子值就从年度静默切换为季度，幅度差约 **4 倍**。

### 后果（均已通读源码确认）

- **横截面不可比**：任意调仓日，有些股票最近提交的是 10-K（年度分子），有些是 10-Q（季度分子）。Gross profitability 和 earnings yield 在横截面上混合 ~4 倍量级差异，**毫无经济含义**。Rank IC、quantile sort、long-short spread、decay、cost、FF5——每个下游 rung——都在被污染后的排序上计算。`research_log.md` 里的 "rung 1–7 OK" 是对垃圾值的机制性跑通。
- **修复被结构性阻断**：`extract_companyfacts_records`（`sec_edgar.py:152`）丢弃了 XBRL `start` 字段——只保留 `end` 但不保留 `start`，因此**不重新拉取数据就无法恢复期间跨度**。更严重的是，`CLAUDE.md` Rule 3 规定的必保留字段列表（`ticker, cik, accn, filed, end, form, fy, fp, tag, unit, value`）**也遗漏了 `start`**。这是教义级别的 bug。

### 修复方向

保留 `start`；对流量型事实构建 **TTM（最近四个季度滚动）** 聚合，或者至少过滤为单一一致的时间跨度（例如仅用年度 / `fp == "FY"`）。在此之前，两个基本面因子**无法验证**。

---

## 二、CRITICAL — "实盘"验证跑在 survivorship-biased 的当前成分股 universe 上

### 问题

诚实的 historical-approximation universe（`universe.build_historical_approx_universe`）已存在且有测试——但**只接入了 `run_momentum_12_1.py` 和 `update_universe.py`**。基本面因子验证 runner 并没有用它。

`run_gross_profitability_validation.py:main()` 调用 `build_validation_panel` 时**没有传 `universe=` 参数**，ticker 轴落在 `facts_df["ticker"].unique()` 上限 50 个——即 **50 个当前 S&P 500 成分股**。`run_valuation_validation.py` 同理。

所以实际产出的 rung 1–7 证据，正是在 CLAUDE.md Rule 5 明确禁止的 universe 上生成的。代码标注为 `pipeline_validation` 且加了 caveat，这算诚实——但项目仍然**从此 universe 产出并归档了 IC/quantile/walk-forward 数字**，恰恰是容易诱发"从结果倒推解释"的配置。

更糟的是，historical-approx universe 本身也很弱：只回溯到 2024-01，通过从**当前**成员回溯 Wikipedia 变更重建，已退市股票缺少 sector/CIK 元数据——即使使用了也是 survivorship-leaning。

---

## 三、CRITICAL — 市值计算使用了复权收盘价，扭曲估值分母

### 问题

`market_frame.py` 设定 `PRICE_COLUMN = "adj_close"`，并在 `build_market_frame`（line 498）中以 `market_cap = shares_outstanding × adj_close` 计算市值，理由是"遵循 CLAUDE.md Rule 6"。

**这是对 Rule 6 的误用。** Rule 6 适用于**收益率/价格因子**（动量、低波），使用复权价是正确的。但市值**水平值**必须用**实际（未复权）收盘价**。yfinance 的 `adj_close` 对分红进行了回溯调整，因此对高分红股票，今天存储的 `adj_close` **系统性低于**真实历史价格，差距随回推时间扩大。

### 后果

`market_cap` 对**高分红股票系统性低估**，这机械性地**抬高了这些股票的 book-to-market 和 earnings yield**——往两个价值因子中**注入了分红率偏差**。这是一个方法论错误，不是 caveat 能覆盖的，且静默地将估值因子与分红策略耦合。

### 修复

用 `close`（未复权）× shares 算市值；`adj_close` 仅用于收益率计算。

---

## 四、HIGH — yfinance 复权价本身也不是 point-in-time

即使对价格因子，`adj_close` 在每次新分红/拆股时由 Yahoo 重新计算，所以某个 2023 年日期存储的值，会在 2025 年分红后**变化**。用今天的 `adj_close` 重建历史动量和前向收益，嵌入了前视的分红调整。这是数据源固有缺陷，但**项目内没有任何地方标注**。对于一个以"不撒谎的数据"为身份的项目，这至少需要明确的 caveat，最终应切换到 as-reported total-return 序列。

---

## 五、HIGH — 样本期太短、太近、重叠度太高，无法做任何推断

默认实盘网格（`market_frame.runner`）是**36 个月末**，截止约今天（2023-06 → 2026-05）。Walk-forward 在此基础上切 **8 个窗口、3 个月测试腿**，36 个月跨度内前向收益重叠。`research_log.md` 记录的 "8/8 usable windows, OOS mean IC 0.0102" 与噪声无异。n=36 月度横截面 × ~50 个名字，低于 Rank IC t 统计或 OOS 稳定性有意义的阈值。报告虽然声明了 t 统计膨胀问题，但持续产出 rung-7"结果"的节奏，会常态化"把噪声当信号"。

---

## 六、HIGH — config 承诺了预处理，但没有任何代码执行

`configs/base.yaml` 声明：
```yaml
preprocessing: { winsorize: true, zscore: true, sector_neutralize: true }
```
**没有任何代码读取它。** 因子以**原始值**流入 `rank_ic_by_date` / `quantile_returns`。两个实际后果：

- **无 winsorize / 异常值控制**：小分母的基本面比率产生极端值；单个名字可以主导薄横截面的均值 spread。Rank IC 相对稳健（它用排序），但 quantile **均值收益**、cost spread、FF5 因变量并不稳健。
- **无行业中性化**：Gross profitability 和 book-to-market 行业集中度极高（金融/REIT 没有有意义的 COGS；代码甚至注释了这一点但没有任何处理）。不做行业中性化，因子部分度量的其实是行业配置。这是"config 即愿望"，具有误导性——要么实现，要么删除。

---

## 七、HIGH — FF5 归因（rung 6）从未在真实数据上跑过

每次实盘跑都输出 `6: skipped_no_ff5`，因为 `ken_french_factors.parquet` 缺失。OLS 代码本身没问题（numpy、经典标准误、优雅降级），但 rung 6 端到端从未在真实输入上验证过。而且在 36 个月样本上 5 因子 + 截距，alpha t 统计量本身也没有意义。另外，月度 spread 用的是经典（非 Newey-West）标准误——月频可接受，但重叠收益归因时需要 HAC 选项。

---

## 八、MEDIUM — 没有共享的"面板核心"；look-ahead 逻辑被复制粘贴了 3 次

`_normalize_asof`、`_resolve_ticker_set`、`_validate_facts`、`_cik_map`、`_annotate_universe`、`_finalize`，以及按 ticker "重建完整月度网格、shift、剔除合成行"的前向收益逻辑，**在几乎所有因子模块中被重复实现**。日历正确的前向收益例程出现在 **三个地方**：`momentum.add_momentum_and_forward`、`low_volatility.add_forward_return`、`qa_metrics.forward_returns_by_horizon`。这是最高杠杆的重构项：今天修复一个 look-ahead bug 必须在三个模块中同步修改，且它们**必然漂移**。抽取 `panel_core`，包含 as-of join、universe 标注和前向收益引擎。

---

## 九、MEDIUM — long-short 框架对因子方向无知

`quantile_returns` 始终计算 `top − bottom`，`cost_sensitivity` 从正的 gross spread 推导 break-even。对于负异常因子（低波），spread 为负值，break-even/年化净收益数字变得无意义，而 `anomaly_sign` 只是一个**装饰性字符串**（`run_cost_sensitivity.py:51` 等），注入报告文本但从不翻转数学。目前只验证了正异常因子，暂未触发——但一旦低波上阶梯，rung 2/5 就会打错标签。sign 应是一等参数。

---

## 十、MEDIUM — "可得性" look-ahead 防护在唯一使用它的面板上是同义反复

`build_oos_validation_report` 在 `available_at > date` 时 fail-closed。设计好。但 gross-profitability runner 设定 `available_at = max(gp_filed, ta_filed)`，而每个操作数已被 `filed <= date` 选中。所以 `available_at <= date` **必然是 true**——这个防护**结构性无法触发**。原理上正确且有价值，但在真实面板上提供**虚假安全感**：一个绿色的"rung 7 fail-closed check passed"，但在结构上本来就无法失败。真正的 PIT 风险（时间跨度混用、restatement）在上游，这个防护看不见。

---

## 十一、MEDIUM — `backtest/` 和 `portfolio/` 空包；rung 8–9 缺失

`src/backtest/` 和 `src/portfolio/` 只有空的 `__init__.py`。这很诚实（rung 8–9 未启动），但结合发现 #1–3 意味着项目**不应该**下一步就推进多因子合成或组合净值回测——在时间跨度混用、survivorship-biased、分红扭曲的因子分数上构建组合，相当于把四个错误复合进一条净值曲线。

---

## 十二、MEDIUM — Agent/adapter 层只是一个模板桩，还不是"过程 alpha"

`build_mock_review_content` 确定性地将 artifact 自身的指标/caveat 转写为散文。`subprocess` 模式只用 `python -c` 测试假数据驱动过——**从没跑过真正的模型**。禁止内容正则防护（`scan_forbidden_content`）是合理的纵深防御，但轻易可绕过，不应被误认为对真实 LLM 的安全边界。作为未来的**类型化接口桩**这没问题，但文档/路线图应该叫它的真名：一个有合同的占位符，目前零 review 价值，不是工作 review agent。硬边界（禁止买卖、human_review_required）正确执行——这部分是好的。

---

## 代码质量 / 性能 / 基础设施

### 十三、MEDIUM — 规模化风险：完整 JSON 展开 + O(tickers×dates) PIT 循环

- `extract_companyfacts_records` 对每家公司展开 **全部** CompanyFacts JSON（所有 tag、所有 unit），然后下游代码过滤到 ~6 个 tag。500 个名字时内存浪费严重；摄入阶段没有 tag 投影。
- `pit_latest_fact` 是双层 Python 循环（`for ticker … for asof …`），每次迭代做布尔掩码——500 个名字 × 120 个月 ≈ 60k 次掩码切片。`pit_price_asof` 正确使用了 `merge_asof` 向量化；基本面路径没有。500 名字能跑，但是 universe 扩大后的明显瓶颈。

### 十四、MEDIUM — 依赖项未锁定；两个重型依赖未使用

`requirements.txt` **一个版本号都没锁**（`pandas`、`numpy`、`scipy`、`pyarrow`、`yfinance`…），而代码注释依赖特定的 Pandas 3 行为（`market_frame.py:373` 的 `merge_asof` dtype 修复、`Period.end_time`）。Pandas 小版本升级可能静默改变结果。另外 `scipy` 需要但未使用（代码有意避免），`duckdb` 已装但明确未接线。锁定版本；剔除死依赖。

### 十五、LOW — 测试启动脆弱

无 `conftest.py`，无 `pyproject.toml`/`pytest.ini`/`setup.py`。`import src…` 之所以能用，仅因 CI 从仓库根目录以 `python -m pytest` 调用（CWD 在 `sys.path` 上）。任何其他调用方式（子目录中 `pytest tests/...`）都会 import 失败。在根目录加 `conftest.py` 或 `pyproject.toml` 含 `pythonpath = ["."]`。

### 十六、LOW — 可复现性 & 静默数据缺失

- `computed_at`/`downloaded_at` = `now()` 写入每个面板，使输出在字节层面不可复现（仅影响 diff 能力，功能性无碍）。
- `download_yfinance_prices` 无逐 ticker 成功记账；如果一半 ticker 空返回，它们静默消失在覆盖率统计中而不报错。对 500 个名字的拉取，硬性 "缺失 > X%" 闸门可捕获静默 yfinance 失败。

---

## 值得保留的优点（重建时不要丢掉）

- ✅ 纯函数、无网络、合成数据单元测试 — 适合可测试量化逻辑的架构
- ✅ Provenance 保留（`accn/filed/end/form/…`）贯穿到每个面板行
- ✅ Look-ahead 概念被认真对待：按 `filed` 的 as-of join、仅向后价格 join、fail-closed walk-forward 防护、前向收益严格与因子分离
- ✅ 各处优雅降级（薄横截面跳过不造假；FF5 返回状态字典而非抛异常）
- ✅ 诚实、响亮的 caveat，正确的验证阶梯排序（IC → 单调性 → decay → turnover → cost → FF5 → OOS，再做合成）
- ✅ 真正好的测试覆盖率（360 个测试），干净的 CI artifact/secret guard

---

## 路线图修正建议

当前方向——在免费 CompanyFacts + yfinance 上继续加 runner/rung——是**在裂缝地基上盖高层**。在动 rung 8/9 之前：

| 优先级 | 行动 | 修复的发现 |
|--------|------|-----------|
| 🔴 P0 | 保留 `start` 字段；标准化流量事实时间跨度（TTM 或仅年度） | #1 量纲混用 |
| 🔴 P0 | 用未复权 `close × shares` 算市值；`adj_close` 仅用于收益率 | #3 市值扭曲 |
| 🔴 P0 | 将 historical-approx universe 接入基本面 runner | #2 survivorship |
| 🟠 P1 | 将历史数据扩展至远超 36 个月，然后再拿 OOS 数字当证据 | #5 样本太短 |
| 🟠 P1 | 实现或删除 `preprocessing:` 配置；基本面因子做行业中性化 | #6 伪预处理 |
| 🟠 P1 | 抽取共享 panel-core，消除 3× 重复的 look-ahead/forward-return 逻辑 | #8 代码重复 |

**发现 #1–3 不是增强项——是修正项。** 不修正它们，现有的基本面因子结果即使方向性上也无法解读。"不撒谎"的纪律在工程层面已经存在；现在需要把它应用到**数据语义**层面——真正的谎言就在这里。

---

*审计由 Claude Code (opus + xhigh effort) 执行，只读模式，未修改任何代码。*
