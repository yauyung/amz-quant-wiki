---
source_path: /root/quant/notes/tradingagents_arch.md
ingested: 2026-06-11
sha256: b4e89df1539fd35d4d7ce05f5160d522555965a0dd9990b089a57df2eb98f4df
---

# TradingAgents 架构分析笔记

> Phase 1 产出 · amz quant · 2026-05-30
> 源码版本：**v0.2.5**（`/root/quant/tradingagents/`，commit 见 .git）
> 论文：arXiv 2412.20138 《TradingAgents: Multi-Agents LLM Financial Trading Framework》

---

## 0. TL;DR（给赶时间的人）

- **本质**：用 LangGraph 把 ~12 个 LLM Agent 串成一张有向图，模拟一家交易公司的决策链：
  **分析师团队 → 多空研究员辩论 → 研究经理拍板 → 交易员下计划 → 风控三方辩论 → 投资组合经理终裁**。
- **入口**：`TradingAgentsGraph(...).propagate(ticker, date)` → 返回 `(完整状态, 决策)`，决策被归一化成 5 档评级
  `Buy / Overweight / Hold / Underweight / Sell`。
- **数据层**：所有数据通过 `dataflows/interface.py` 的 `route_to_vendor()` 路由到 vendor（当前 `yfinance` / `alpha_vantage`）。
  **这就是我们接 ccxt 的唯一切入点**——新增一个 `ccxt` vendor 即可，不用动 Agent 代码。
- **加密支持现状**：仓库里已有 `asset_type="crypto"` 参数，但**只改了 prompt 措辞**（"asset" vs "company"），
  **没有任何 ccxt / 加密数据源**。真正的数据适配要我们自己写（Phase 2）。
- **LLM 配置（本项目）**：用 `provider="openrouter"` 走通用 OpenAI 兼容 Chat Completions，指向小米 MiMo。
  详见 §6。

---

## 1. 运行方式

### 1.1 编程入口（我们要用的）
```python
from tradingagents.graph.trading_graph import TradingAgentsGraph
from tradingagents.default_config import DEFAULT_CONFIG

ta = TradingAgentsGraph(
    selected_analysts=["market", "social", "news", "fundamentals"],  # 可裁剪
    debug=True,
    config=DEFAULT_CONFIG.copy(),
)
final_state, decision = ta.propagate("NVDA", "2026-01-15")   # 美股
final_state, decision = ta.propagate("BTC/USDT", "2026-01-15", asset_type="crypto")  # 加密
```
- `propagate()` 返回 `(final_state: dict, decision: str)`，`decision` ∈ {Buy, Overweight, Hold, Underweight, Sell}。
- `final_state` 里有每个环节的完整报告（见 §3 AgentState）。

### 1.2 CLI 入口
`python -m cli.main` 或安装后的 `tradingagents` 命令（交互式选 ticker/日期/模型/辩论深度）。本项目不依赖 CLI。

### 1.3 关键文件地图
| 关心什么 | 看哪 |
|---|---|
| 主编排器 | `tradingagents/graph/trading_graph.py` |
| 图结构（节点+边） | `tradingagents/graph/setup.py` |
| 路由条件（何时调工具/换人/停） | `tradingagents/graph/conditional_logic.py` |
| 初始状态 | `tradingagents/graph/propagation.py` |
| 全局配置 | `tradingagents/default_config.py` |
| 共享状态结构 | `tradingagents/agents/utils/agent_states.py` |
| 各 Agent 定义 | `tradingagents/agents/{analysts,researchers,managers,risk_mgmt,trader}/` |
| **数据源路由（接 ccxt 处）** | `tradingagents/dataflows/interface.py` |
| LLM 工厂 | `tradingagents/llm_clients/factory.py` + `openai_client.py` |
| 决策→评级 | `tradingagents/graph/signal_processing.py` + `agents/utils/rating.py` |

---

## 2. LangGraph 多 Agent 图结构

整张图用 `langgraph.graph.StateGraph(AgentState)` 构建（`setup.py`）。节点之间通过**普通边**和
**条件边（conditional_edges）**连接，条件函数都在 `conditional_logic.py`。

### 2.1 全景图（默认 4 分析师 + 全流程）

```
                                   START
                                     │
                ┌────────────────────▼─────────────────────┐
                │            分析师团队（顺序执行）            │   每个分析师是一个
                │   market → social → news → fundamentals    │   "ReAct 小循环"：
                └────────────────────┬─────────────────────┘   见 §2.2
                                     │ (最后一个分析师的 Clear 节点)
                                     ▼
                            ┌──────────────────┐
                            │  Bull Researcher  │◄────────────┐
                            └────────┬──────────┘             │
                  should_continue_debate                      │ 轮流辩论
                  ┌──────────┴───────────┐                    │ count < 2*max_debate_rounds
                  ▼                      ▼                     │
          ┌──────────────┐      ┌────────────────┐            │
          │Bear Researcher│─────▶│ (回到 Bull)     │────────────┘
          └──────┬───────┘      └────────────────┘
                 │ count ≥ 2*max_debate_rounds
                 ▼
        ┌──────────────────┐
        │ Research Manager  │   (深度模型) 把多空辩论裁成"投资计划"+评级
        └────────┬─────────┘
                 ▼
            ┌─────────┐
            │ Trader  │   把投资计划落成一个具体"交易提案"
            └────┬────┘
                 ▼
        ┌────────────────────┐
        │ Aggressive Analyst  │◄──────────────────────┐
        └─────────┬──────────┘                        │
        should_continue_risk_analysis                 │ 三方循环
        ┌─────────┴─────────┐                          │ Aggressive→Conservative→Neutral
        ▼                   ▼                          │ count < 3*max_risk_discuss_rounds
┌────────────────┐  ┌──────────────┐                  │
│Conservative An. │  │Neutral Analyst│──────────────────┘
└───────┬────────┘  └──────┬───────┘
        └─────────┬─────────┘
                  │ count ≥ 3*max_risk_discuss_rounds
                  ▼
         ┌────────────────────┐
         │ Portfolio Manager   │   (深度模型) 终裁，产出 final_trade_decision
         └─────────┬──────────┘
                   ▼
                  END  ──▶ SignalProcessor 解析出 5 档评级
```

### 2.2 分析师小循环（ReAct 模式，每个工具型分析师都一样）

```
   [Analyst 节点] ──tool_calls?──► 有 ──► [tools_<key> 工具节点] ──► 回到 [Analyst 节点]
        │ 无 tool_calls（即写完报告）
        ▼
   [Msg Clear 节点]  (清空 messages，塞一个 "Continue" 占位，给下一个分析师干净上下文)
        ▼
   下一个分析师 / Bull Researcher
```
- 路由函数 `should_continue_<key>()`：看最后一条消息有没有 `tool_calls`，有就去工具节点，没有就去 Clear 节点。
- `create_msg_delete()`（`agent_utils.py`）：用 `RemoveMessage` 删光历史消息 + 加一个 `HumanMessage("Continue")`，
  避免把上一个分析师的工具消息污染给下一个（也是为了 Anthropic 兼容）。

### 2.3 三组"控制循环"的计数规则（`conditional_logic.py`）
| 循环 | 节点 | 停止条件 | 默认轮数 |
|---|---|---|---|
| 分析师工具循环 | 单个分析师 ↔ 其工具节点 | LLM 不再发 tool_calls | 不限（直到写报告） |
| 多空辩论 | Bull ↔ Bear | `investment_debate_state.count ≥ 2 × max_debate_rounds` | `max_debate_rounds=1` → 2 次发言 |
| 风控辩论 | Aggressive→Conservative→Neutral 轮转 | `risk_debate_state.count ≥ 3 × max_risk_discuss_rounds` | `max_risk_discuss_rounds=1` → 3 次发言 |

> 注意：`selected_analysts` 可裁剪。**只传 `["market"]` 就只跑技术分析师**——这是我们做加密 demo 的关键，
> 因为 social/news/fundamentals 依赖美股专属数据（StockTwits/Reddit/yfinance 财报），对 BTC 不可用。

---

## 3. 共享状态 AgentState（`agent_states.py`）

所有节点读写同一个 `AgentState`（继承自 LangGraph `MessagesState`，带 `messages` 通道）。关键字段：

| 字段 | 写入者 | 含义 |
|---|---|---|
| `company_of_interest` | 初始 | 标的（ticker） |
| `asset_type` | 初始 | `"stock"` / `"crypto"`（只影响 prompt 措辞） |
| `trade_date` | 初始 | 决策日 |
| `market_report` | Market Analyst | 技术面报告 |
| `sentiment_report` | Sentiment Analyst | 情绪报告 |
| `news_report` | News Analyst | 宏观/新闻报告 |
| `fundamentals_report` | Fundamentals Analyst | 基本面报告 |
| `investment_debate_state` | Bull/Bear/ResearchMgr | 多空辩论历史 + count + 裁决 |
| `investment_plan` | Research Manager | 投资计划（含评级） |
| `trader_investment_plan` | Trader | 交易员提案 |
| `risk_debate_state` | 风控三方 + PM | 风控辩论历史 + count + 终裁 |
| `final_trade_decision` | Portfolio Manager | **最终决策文本（含 `Rating: X`）** |
| `past_context` | 初始（来自记忆日志） | 同标的历史决策 + 跨标的教训，注入 PM |

---

## 4. 每个 Agent 的 Prompt 设计

> 角色对标（SPEC §1.2）：技术分析师＝策略Agent，基本面＝研究Agent，情绪＝数据Agent，交易员＝执行Agent，风控经理＝风控Agent。

### 4.1 分析师团队（Analyst Team）——产出 4 份报告

| Agent | 模型 | 用工具？ | Prompt 要点 |
|---|---|---|---|
| **Market / Technical** `market_analyst.py` | quick | ✅ `get_stock_data`,`get_indicators` | 给出 11 个指标的"使用手册"（SMA/EMA/MACD/RSI/BOLL/ATR/VWMA/MFI），要求**最多选 8 个互补指标**，先 `get_stock_data` 再逐个 `get_indicators`，最后写带 Markdown 表格的详细报告。**唯一只依赖 OHLCV 的分析师 → 加密可用**。 |
| **Sentiment** `sentiment_analyst.py` | quick | ❌（预取数据塞进 prompt） | 预先抓 3 路数据：Yahoo 新闻 + StockTwits（按 cashtag）+ Reddit（wsb/stocks/investing），注入 prompt。教 LLM 读多空比、找跨源背离、按热度加权。**依赖美股专属源，BTC 基本拿不到**。 |
| **News** `news_analyst.py` | quick | ✅ `get_news`,`get_global_news` | 让 LLM 调工具搜个股新闻 + 宏观新闻，写"当前世界状态"报告。 |
| **Fundamentals** `fundamentals_analyst.py` | quick | ✅ `get_fundamentals`,`get_balance_sheet`,`get_cashflow`,`get_income_statement` | 财报/估值/财务史综合分析。**加密无公司财报，不适用**。 |

分析师通用 system 框架（`market/news/fundamentals` 共用）：
> "You are a helpful AI assistant, collaborating with other assistants... 若有最终结论，用
> `FINAL TRANSACTION PROPOSAL: **BUY/HOLD/SELL**` 开头让团队停。You have access to tools: {tool_names}..."

### 4.2 研究员团队（Researcher Team）——多空辩论

| Agent | Prompt 要点 |
|---|---|
| **Bull Researcher** `bull_researcher.py` | "你是看多分析师"，吃 4 份报告 + 辩论历史 + 对方上一句，强调成长/护城河/正面指标，**逐条反驳空头**，要求对话式而非罗列。每次发言 `count += 1`。 |
| **Bear Researcher** `bear_researcher.py` | 镜像版"看空分析师"：强调风险/竞争劣势/负面信号，逐条反驳多头。 |

辩论由 `should_continue_debate` 调度：Bull/Bear 交替，直到 `count ≥ 2*max_debate_rounds` 转交 Research Manager。

### 4.3 经理 & 交易员（结构化输出）

| Agent | 模型 | 结构化 Schema | Prompt 要点 |
|---|---|---|---|
| **Research Manager** `research_manager.py` | **deep** | `ResearchPlan` | "辩论主持人"，评判这轮辩论，给**可执行投资计划 + 5 档评级**（Buy/Overweight/Hold/Underweight/Sell），证据均衡时才 Hold。 |
| **Trader** `trader.py` | quick | `TraderProposal` | 基于投资计划给**具体买/卖/持提案**，推理须锚定分析师报告。 |
| **Portfolio Manager** `portfolio_manager.py` | **deep** | `PortfolioDecision` | "终裁"：综合风控辩论 + 投资计划 + 交易提案 + 历史教训(`past_context`)，给**最终决策 + 5 档评级**，渲染成带 `**Rating**: X` 的 markdown。 |

> 结构化输出走 `with_structured_output`（`structured.py`）；若模型不支持则**优雅降级为自由文本**，
> 再由 `parse_rating()` 启发式抠出评级（抠不到默认 `Hold`）。→ **对 MiMo 这类第三方模型很鲁棒**。

### 4.4 风控团队（Risk Management）——三方辩论

| Agent | 立场 |
|---|---|
| **Aggressive Debator** `aggressive_debator.py` | 激进：力挺高风险高回报，逐条反驳保守/中立。 |
| **Conservative Debator** | 保守：强调风险控制。 |
| **Neutral Debator** | 中立：平衡两方。 |

`should_continue_risk_analysis` 调度轮转 Aggressive→Conservative→Neutral，`count ≥ 3*max_risk_discuss_rounds` 后交 Portfolio Manager。

### 4.5 决策归一化
`SignalProcessor.process_signal()` → `parse_rating(final_trade_decision)` → 返回 5 档之一（纯启发式，无额外 LLM 调用）。

---

## 5. 数据层与 Vendor 路由（**Phase 2 的核心切入点**）

### 5.1 调用链
```
Agent 绑定的工具 (@tool get_stock_data / get_indicators / get_news / ...)
        │  agents/utils/*_tools.py
        ▼
route_to_vendor(method, *args)          # dataflows/interface.py
        │  按 config["data_vendors"][category] 选 vendor，带 fallback 链
        ▼
VENDOR_METHODS[method][vendor]          # 具体实现
   ├─ "yfinance":      dataflows/y_finance.py
   └─ "alpha_vantage": dataflows/alpha_vantage*.py
```

### 5.2 工具 → 类别 → vendor 映射
| 类别 (`data_vendors` key) | 工具 | 默认 vendor |
|---|---|---|
| `core_stock_apis` | `get_stock_data` | yfinance |
| `technical_indicators` | `get_indicators` | yfinance |
| `fundamental_data` | `get_fundamentals`/`get_balance_sheet`/`get_cashflow`/`get_income_statement` | yfinance |
| `news_data` | `get_news`/`get_global_news`/`get_insider_transactions` | yfinance |

- `config["tool_vendors"]`（工具级）优先于 `config["data_vendors"]`（类别级）。
- `route_to_vendor` 有 fallback：只有 `AlphaVantageRateLimitError` 才触发切换 vendor。

### 5.3 两个关键函数的输出格式（ccxt 适配必须对齐）
**`get_stock_data`（`get_YFin_data_online`）** 返回字符串：
```
# Stock data for AAPL from 2024-01-01 to 2024-02-01
# Total records: N
# Data retrieved on: ...

Date,Open,High,Low,Close,Volume
2024-01-02,...
```
**`get_indicators`（`get_stock_stats_indicators_window`）** 返回字符串：
```
## rsi values from <before> to <curr_date>:

2024-01-15: 56.3
2024-01-14: N/A: Not a trading day (weekend or holiday)
...

<该指标的使用说明文字>
```
- 指标计算靠 `stockstats.wrap(df)` 然后 `df[indicator]`；`df` 列须为 `Date,Open,High,Low,Close,Volume`。
- `load_ohlcv()`（`stockstats_utils.py`）负责取数 + 缓存（CSV）+ **按 curr_date 截断防未来函数**。

> **接 ccxt 的两种思路**（Phase 2 详细设计）：
> 1. **新增 `ccxt` vendor**：在 `VENDOR_METHODS["get_stock_data"]`/`["get_indicators"]` 注册 ccxt 实现，
>    并设 `config["data_vendors"]["core_stock_apis"]="ccxt"`、`["technical_indicators"]="ccxt"`。
>    ——最干净，不动任何 Agent 代码。
> 2. ccxt 拉 5m K 线 → 造出 `Date/Open/High/Low/Close/Volume` 的 DataFrame → 复用 `stockstats` 算指标，
>    输出对齐上面两种格式。

---

## 6. LLM 配置（本项目 = 小米 MiMo，已实测）

`llm_clients/factory.py`：provider ∈ OpenAI 兼容集合 `{openai, xai, deepseek, qwen, glm, minimax, ollama, openrouter}` → 走 `OpenAIClient`。

**坑：`provider="openai"` 会强制 `use_responses_api=True`（打 `/v1/responses`），第三方兼容端点大概率不支持。**

✅ **本项目正确配置**（用 openrouter 这个"通用 OpenAI 兼容透传"通道）：
```python
import os
os.environ["OPENROUTER_API_KEY"] = "sk-c5duun9tu27agfx7sshjqu13khma240mqkm4rvyxlk17xjsp"
config = DEFAULT_CONFIG.copy()
config["llm_provider"]   = "openrouter"                       # 走 Chat Completions，不碰 Responses API
config["backend_url"]    = "https://api.xiaomimimo.com/v1"    # 显式 base_url 优先于 provider 默认
config["deep_think_llm"]  = "mimo-v2.5-pro"
config["quick_think_llm"] = "mimo-v2.5-pro"
```
- ⚠️ **模型名是小写 `mimo-v2.5-pro`**（SPEC 里写的 `MiMo-V2.5-Pro` 会报 400 "Not supported model"）。
  实测可用模型：`mimo-v2-flash/pro`、`mimo-v2.5`、`mimo-v2.5-pro`、`*-tts`、`mimo-v2-omni`。
- `openrouter` 在 validator 里"任意模型放行"，不会因模型名未知报错。
- **实测结论**：✅ Chat Completions 可用；✅ **支持 function/tool calling**（分析师依赖它）；
  ⚠️ `mimo-v2.5-pro` 是**推理模型**（带 `reasoning_content`，trivial prompt 也烧 ~83 reasoning tokens、~5s）。
  → max_tokens 不能给太小（否则 content 为空）；→ **每根 K 线都跑全流程不现实**（Phase 3 要控频）。
- 能力表 `capabilities.py`：`mimo-*` 命中 `_DEFAULT`（function_calling 结构化输出），不匹配 minimax/deepseek 的特殊分支。

---

## 7. 持久化（了解即可，本次不深用）
- **决策日志**：每次跑完追加到 `~/.tradingagents/memory/trading_memory.md`，下次同标的会取实际收益做反思并注入 PM。
  （会触发 `yfinance` 取收益算 alpha vs SPY——**对加密/BTC 会失败，但只是 warning，不阻断**。）
- **Checkpoint**：`--checkpoint` 开启后每个节点存 SQLite，崩溃可续跑。本次默认关。

---

## 8. 对照 SPEC 的适配结论（承上启下）

| SPEC 阶段 | 结论 |
|---|---|
| L1 探索 | ✅ 图结构、Agent prompt、数据路由已厘清（本文档）。 |
| L2 加密适配 | 切入点 = `dataflows` 注册 `ccxt` vendor；BTC 用 `selected_analysts=["market"]` 单分析师最稳；情绪/新闻/基本面对 BTC 不可用。 |
| L3 Freqtrade 集成 | 因 MiMo 是推理模型 + 每决策 ~9 次 LLM 调用，**不能每根 5m K 线都调**；策略须"低频调用 + 缓存 + 可关闭 LLM 的回测安全模式"，回测能跑通不报错即达标。 |

**已验证的事实**：框架在 `/root/quant-env` 可正常 import；ccxt 能拉 BTC/USDT 5m；MiMo（`mimo-v2.5-pro`）可用且支持工具调用。
