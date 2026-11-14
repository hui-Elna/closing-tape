# GFNA V7 — A 股行情采集 + 量化选股系统

> 从「新闻聚合器」改造而来：新闻链路降级为**情绪因子输入**，主线变成
> **行情采集 → 本地 PostgreSQL 单一事实源 → 指标/因子 → 条件树选股 → 回测/纸面跟踪**。
>
> 版本说明：V5 打通本地全链路基线；V6 完成行情呈现、算法中心与视觉系统；
> V7 增加市场状态/配置建议、指数对比与 6 宽基牛熊估值。

---

## 0. 一句话说明它现在能做什么

- 采集**沪深主板 / 创业板 / 科创板**股票池、日线（含复权因子）、板块日线与成分（SCD2）、指数（A 股 + 美股）、两融（含融资买入额）、中债国债收益率、风格指数（创业板50 / 红利低波50）、6 个宽基指数、资金流、历史估值与新闻；
- **股票池只保留沪深主板 / 创业板 / 科创板**，北交所与其它标的不采集、不进入广度/温度/因子/策略/搜索；
- 在本地库上计算 MA/RSI/MACD/BOLL/ATR/KDJ/动量/波动/量比，以及六维因子（价值/质量/动量/波动/流动性/情绪）；
- 用**条件树 + 六个内置策略（A3/A4 + B1–B5）**选出股票，结果带 `run_id`、参数快照、数据版本，**可复现**；
- 对任一策略跑**异步回测**：T+1 开盘成交、费用/滑点、涨跌停与停牌不可成交、后复权收益、基准超额；支持**参数敏感性矩阵**与**纸面组合跟踪**；
- 前端页面：
  - `/` 新闻情报：新闻列表、搜索、AI 摘要；
  - `/dashboard` 盘后分析：市场阶段 / 配置建议 / 一周温度计 Hero，多周期指数、美股、两融、板块 treemap、领涨领跌、AI 复盘；
  - `/index-compare` 指数对比：创业板50 / 红利低波50 风格轮动 + 6 宽基牛熊位置 × 估值四面板；
  - `/algo-center` 选股中心：策略天梯、多策略共识、信号逐条解释、参数矩阵、回测、CSV 导出；
  - `/paper` 纸面组合、`/watchlist` 自选信号、`/jobs` 任务监控、`/stocks/:code` 个股详情（蜡烛图 + MA/BOLL + 成交量 + 回测买卖标记）；
- 后端启动后**自动补最近缺失交易日**，任务监控页提供「一键补数」作为手动兜底。

**它不是“能稳定赚钱的策略”**，而是一套让决策可复现、可归因、可证伪的数据与信号系统。回测 ≠ 实盘。

---

## 1. 快速开始（从零到第一份选股结果）

```bash
# 1) 依赖（建议在仓库根目录建虚拟环境；国内可用镜像）
cd C:\docker-projects\grna
python -m venv .venv
.venv\Scripts\activate            # Linux/macOS: source .venv/bin/activate
pip install -r backend/requirements.txt -i https://mirrors.aliyun.com/pypi/simple/

# 2) 配置（DEEPSEEK_API_KEY 可留空，AI 日报会自动降级为数据摘要）
copy backend\.env.example backend\.env

# 3) 建库 + 建表 + 分区 + 视图 + 策略种子
cd backend
python scripts\bootstrap_db.py

# 4) 首次回补历史（断点续传）
python scripts\pipeline_cli.py calendar
python scripts\pipeline_cli.py universe
# 腾讯批量通道单次约 1023 个交易日（近 4 年），更早用 backfill 分段补
python scripts\pipeline_cli.py history_fast --start 2023-01-01
python scripts\pipeline_cli.py backfill --start 2021-01-01 --end 2022-12-31
# 资金流源单只约 300 日；估值源覆盖 2018 年起
python scripts\pipeline_cli.py moneyflow_history
python scripts\pipeline_cli.py valuation_fast --start 2018-01-01

# 5) 跑一次最新交易日全链路（指数/板块/两融/中债/温度/状态/选股/信号统计）
python scripts\pipeline_cli.py all

# 6) 启动后端（默认 http://127.0.0.1:8000 ，文档 /docs）
python -m uvicorn app.main:app --reload --port 8000

# 7) 启动前端（另开终端）
cd ..\frontend
npm install
npm run dev                       # 默认 http://localhost:5173（Vite 可能只监听 [::1]）
```

> 顺序很重要：**日线 → 估值 → 指标 → 因子 → 选股**。
> `all` 是“最新交易日”的日更链路，不会回补历史；历史数据要先跑 `history_fast / moneyflow_history / valuation_fast`。
> 风格/宽基/中债等专题历史可单独回补：
> ```bash
> python scripts\backfill_style_indices.py --start 2014-06-01
> python scripts\backfill_broad_indices.py --start 2005-01-01
> python scripts\backfill_macro.py
> python scripts\backfill_temperature.py && python scripts\backfill_regime.py
> ```

> **更省事的一键启停**（Windows，脱离会话常驻，日志在 `logs/`）：
> ```powershell
> .\start-local.ps1          # 后端 8000 + 前端 5173（自动确保表结构）
> .\start-local.ps1 -Demo    # 空库时先灌演示数据
> .\start-local.ps1 -Stop    # 停止
> ```
>
> **访问边界（单机自用）**：本机启动与 Docker 部署都只绑定 `127.0.0.1`，页面和 API
> 不对局域网或公网开放。自选状态固定为 `app_watchlist_state(id=1)` 单行，同一后端的
> 所有浏览器共享一份；这是单用户设计，不做伪多租户隔离。需要跨设备访问时优先用
> SSH 隧道或 Tailscale；确有公网需求再在反向代理层加 Basic Auth，不要把端口改成
> `0.0.0.0`。

**Playwright（新闻爬虫用，可选）**：只装 Python 包不够，还需要浏览器内核：

```bash
python -m playwright install chromium      # Docker 镜像里已装好
```

**没有网络也能验收全链路**（确定性演示数据，不联网）：

```bash
python scripts\pipeline_cli.py demo
python scripts\pipeline_cli.py features
python scripts\pipeline_cli.py scores && python scripts\pipeline_cli.py sector_scores
python scripts\pipeline_cli.py algos
```

---

## 2. 架构

```
┌─ L1 采集 ingest ───────────────────────────────────────────┐
│  Provider 链 akshare → baostock · 批量通道 腾讯/新浪          │
│  东财 datacenter 估值 · 同花顺板块 · 中债收益率 · 中证指数     │
│  交易日历 · 限频 · 重试 · 断点续传 · 幂等 upsert             │
│  数据质量校验 · DB advisory lock · 自动补缺                   │
└───────────────────────────┬────────────────────────────────┘
                            ▼
┌─ L2 存储 PostgreSQL ───────────────────────────────────────┐
│  dim_stock · dim_sector_member · dim_stock_financial         │
│  dim_stock_dividend                                          │
│  fact_stock_daily(按年分区) · fact_feature_daily             │
│  fact_stock_score_daily · fact_moneyflow · fact_sector_daily │
│  sector_score · market_indices · margin_trading · macro_rates│
│  fact_market_daily · market_temperature_daily                │
│  market_regime_daily · daily_report                          │
│  stg_run/stg_signal/stg_signal_perf                          │
│  backtest_run/backtest_grid/backtest_grid_cell               │
│  paper_portfolio/paper_nav · app_watchlist_state             │
│  meta_data_quality · trade_calendar · ingest_job             │
└───────────────────────────┬────────────────────────────────┘
                            ▼
┌─ L3 分析 engine ───────────────────────────────────────────┐
│  indicators: MA/RSI(Wilder)/MACD/BOLL/ATR/KDJ/动量/波动/量比  │
│  factors: 价值/质量/动量/波动/流动性/情绪 → 分位               │
│  cross_section: 行业中性 z-score / winsorize / 分位            │
│  market: 广度 · 市场因子 · 温度计 · 阶段/配置 · 风格轮动       │
│  sector: 技术40/情绪35/消息25 · 龙头 · 共识 · 信号前瞻         │
└───────────────────────────┬────────────────────────────────┘
                            ▼
┌─ L4 选股 strategy ─────────────────────────────────────────┐
│  condition: AND/OR/between/in/top_pct/行业约束               │
│  strategies: A3/A4 + B1–B5 · runner: run_id/参数指纹/数据版本  │
│  backtest: T+1 成交/费用/涨跌停不可成交/后复权                 │
│  grid: 参数敏感性矩阵 · paper: 纸面组合跟踪                    │
└───────────────────────────┬────────────────────────────────┘
                            ▼
┌─ L5 服务与展示 ────────────────────────────────────────────┐
│  FastAPI /api/v1 · 统一错误码 · 新鲜度 · 自动补缺              │
│  React 看板 · 命令面板 · 数据新鲜度徽标 · CSV 导出             │
└────────────────────────────────────────────────────────────┘
```

**为什么不再调 `westock-tool`**：它是 117 行 mock；接真版本也无 SLA、5–30 秒响应、无法回测。
全市场日线落库后，选股本质是一次横截面运算——**毫秒级、可复现、参数真生效**。

---

## 3. 数据源与网络现实

| 数据 | 主源 | 兜底 / 备注 |
|---|---|---|
| 交易日历 | akshare `tool_trade_date_hist_sina` | baostock `query_trade_dates` |
| 股票池 | akshare `stock_info_a_code_name` | baostock `query_stock_basic`；只保留沪深主板/创业板/科创板 |
| 个股日线（历史） | **腾讯 `fqkline`**（`history_fast`，单只约 800 日） | **新浪 K 线 + qfq 因子**；腾讯连续失败触发 15 分钟熔断后自动切新浪 |
| 个股日线（日更） | akshare 东财 / 新浪 | baostock；东财 `push2*` 在本网络会重置 |
| 历史估值 | **东财 datacenter `RPT_VALUEANALYSIS_DET`**（2018 年起 PE/PB/PS/市值） | 本地财报+分红缓存（`valuation`）按 point-in-time 计算 PE_TTM/PB/股息率 |
| 个股资金流 | **新浪 `sina_moneyflow`**（最近约 300 日） | A2 已下线；资金流仍作为因子/展示数据保留 |
| 指数 | 新浪指数历史（A 股） | 中证指数历史（`000922`、`H30269`） |
| 美股指数 | 新浪/腾讯（道琼斯、标普、纳斯达克） | 盘后分析页按 T-1 展示 |
| 风格指数 | 创业板50 `sz399673`（新浪） | 红利低波50 `H30269`（中证指数） |
| 6 宽基 | 沪深300/上证/中证500/创业板/科创50（新浪） | 中证红利 `000922`（中证指数） |
| 板块/成分 | 东财板块接口 | **同花顺 `10jqka` 兜底**；名单不完整时标 `partial`，部分指标如实置空 |
| 板块资金流 | akshare 东财 | 同花顺 `fetch_net_inflow_map_sync`，来源标记 `ths` |
| 两融 | akshare `stock_margin_sse/szse` | 含融资买入额；T+1 披露，补跑时排在评分/日报之后 |
| 国债收益率 | akshare `bond_china_yield`（中债） | 1Y / 10Y，新增 `macro_rates` 表 |
| 新闻 | Scrapy + Playwright 公开页面 | DeepSeek 摘要；留空 Key 时降级为数据摘要 |

**本机实测（2026-11，重要）**：
- 东方财富 `push2*` 行情接口在本网络被重置（`RemoteDisconnected`），因此 `stock_zh_a_hist` / `*_spot_em` / 东财板块接口不可用；
- **腾讯、新浪、同花顺 `data.10jqka.com.cn`、中证指数、中债、巨潮、baostock、DeepSeek 可用**；
- 腾讯批量回补存在 IP 级 WAF：连续失败会自动进入冷却期并改走新浪，不必手工切换；
- 中证历史、两融等源可能 **T+1 更新**；页面按共同最新交易日计算，不补造缺失日期；
- 北向资金自 2024-08 起暂停披露，缺失不阻断评分。

> **历史估值口径**：`valuation_fast` 直接取东财估值分析历史（PE_TTM/PB/PS/市值）；
> `valuation` 仍保留“财报/分红缓存进库 + 本地推算”作为兜底：
> `EPS_TTM = 本期累计 + 上年年报 − 上年同期` → `PE = close / EPS_TTM`；
> `PB = close / BPS`；`股息率 = 近 12 个月每股分红 / close`。
> 用「报告期 + 90 天」作为市场可知日做 **point-in-time**，避免未来函数。

> 合规边界：AKShare 是对公开页面的二次封装，不是交易所授权接口。本系统限频
> （快照 ≤1 次/分钟、回补 ≤2 请求/秒）、固定 UA、**不实现任何反爬绕过/验证码/代理池**。
> 数据仅个人研究，不得对外分发或商用。

---

## 4. 六个内置策略（当前口径）

统一前置过滤：剔除 ST、停牌/退市、上市不足 60 个交易日、当日无行情。

| 编号 | 名称 | 条件 | 排序 |
|---|---|---|---|
| **A3** | 央企 × 综合评分 | `ownership='央企'`；六因子综合分 ≥ `min_score`(6.0/10) | 综合分降序 |
| **A4** | 高股息 × 低估值 | `PE_TTM>0`；`PE_TTM<pe_max`(15)；`dv_ttm≥div_min`(4%) | 股息率降序 |
| **B1** | 双均线动量突破 | EMA5/EMA20 近 3 日金叉；站上 EMA20 且 EMA20 上行；ROC10>0；量能 ≥ 5 日均量×倍数；收盘>MA60；20 日均额达标 | ROC10 降序 |
| **B2** | RSI6 超卖回归 | `RSI6<rsi_max`(20)；当日止跌 K 线；收盘>MA120×`trend_dev`(0.9)；60 日振幅>15%；20 日均额达标 | RSI6 升序 |
| **B3** | KDJ 低位金叉 | 今日/昨日 J<`j_max`(10)；当日 K 上穿 D；K<`k_max`(30)；收盘>EMA60；止跌 K 线；20 日均额达标 | KDJ_J 升序 |
| **B4** | 布林挤压突破 | 前一日带宽处于 60 日最低 `squeeze_pct`% 分位；收盘突破上轨且创 20 日新高；放量；20 日区间宽度<`range_max`；**沪深300 收盘 ≥ MA20×0.995** | 带宽分位升序 |
| **B5** | 布林下轨 × KDJ 双确认 | 最低价触及布林下轨；今日/昨日 J<`j_max`(0)；缩量；止跌 K 线；收盘>MA120 或偏离下轨≤`dev_max`%；**沪深300 当日跌幅 > -2%** | KDJ_J 升序 |

- **A1/A2 已下线**：按回测结果 A1 三年约 -16%，A2 约 -34% 且最大回撤约 59%，不具备实用性；
  代码与接口只保留 A3/A4/B1–B5。
- A3 的“央企”来自 `core/soe.py`：央企名录 + 名称规则，**未知不会被误判为央企**；
  `composite_score` 为**本地六因子模型**（0–10），不是腾讯评分。
- B4/B5 带**市场状态门槛**（沪深300 均线/跌幅），不是纯个股信号。
- 参数会同时进入**计算、缓存键（含参数指纹与数据版本）与 run 快照**——改参数必然产生新 run。

---

## 5. 回测、参数矩阵与纸面跟踪

| 项 | 取值 |
|---|---|
| 调仓 | 每 **20 个交易日**（按真实交易日历） |
| 持仓 | **20 只等权** |
| 成交 | **T 日收盘出信号 → T+1 开盘成交**（买入 ×(1+滑点)、卖出 ×(1−滑点)） |
| 成本 | 佣金 **万1**（双边，最低 5 元）、印花税 **万5**（仅卖出）、滑点 0.1% |
| 不可成交 | 涨停买不进、跌停卖不出、停牌、无数据、不足 1 手 |
| 收益 | **后复权**（`close × adj_factor`），缺复权因子时在 metrics 标 `adjusted=false` |
| 基准 | 沪深300（`sh000300`） |
| 起点 | 自动裁剪到该策略**数据可用起点**，响应带 `start_trimmed=true` 与 `effective_start` |
| 结束 | 默认取库内最新交易日（不晚于今天），避免把非交易日写进区间 |
| 指标 | 总收益/年化/最大回撤/夏普/索提诺/卡玛/胜率/盈亏比/换手/成交数/基准收益/超额/波动 |
| 派生视图 | 月度收益、回撤曲线、20 日滚动超额、行业暴露 |
| 执行方式 | `POST /backtest/{code}` 立即返回 `bt_id`，后台执行；前端轮询状态 |

- **参数敏感性矩阵**：`POST /backtest/{code}/grid`，对 `rebalance_days × max_positions` 组合批量回测，
  单格失败可重试，结果落 `backtest_grid*`。
- **纸面组合**：17:30 任务按策略信号维护虚拟组合（水位 + 60 日重叠窗口，每周全量校准），
  `/paper` 展示净值、超额、回撤、持仓与明日待执行；也可手动 sync。
- **防未来函数**：代码层禁止 `.shift(-`，`assert_no_lookahead()` 静态扫描本包；
  测试里可用“把 T+1 价格换成 T 日价格”来验证检测会失败。

---

## 6. 接口

基础契约见 [`docs/V5-API-CONTRACT.md`](docs/V5-API-CONTRACT.md)；
V6/V7 增量见 [`docs/GFNA_V6_产品升级任务书.md`](docs/GFNA_V6_产品升级任务书.md)、
[`docs/GFNA_V7_市场状态与配置仪表盘任务书.md`](docs/GFNA_V7_市场状态与配置仪表盘任务书.md)、
[`docs/GFNA_V7_指数对比看板.md`](docs/GFNA_V7_指数对比看板.md)、
[`docs/GFNA_V7_牛熊估值看板.md`](docs/GFNA_V7_牛熊估值看板.md)。

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/health` | 数据库状态 / 最新交易日 / 数据就绪 |
| GET | `/api/v1/market/dashboard` | 盘后分析（指数/美股/板块/评分/龙头/两融/日报/新鲜度） |
| GET | `/api/v1/market/overview` | 市场状态（阶段 / 配置建议 / 一周温度计 / 因子依据） |
| GET | `/api/v1/market/index-compare` | 风格轮动（R、滚动分位、三态仓位、解读） |
| GET | `/api/v1/market/bull-bear` | 6 宽基牛熊位置 × 估值（四面板、PE/ERP 快照） |
| GET | `/api/v1/market/indices` | 指数日线（支持多日序列） |
| GET | `/api/v1/market/temperature` | 市场温度计 |
| GET | `/api/v1/market/sectors` | 板块日线 + 评分 |
| GET | `/api/v1/market/sector/{code}` | 板块详情（三面子分 + 龙头 + 成分） |
| GET | `/api/v1/stocks/search` | 股票搜索 |
| GET | `/api/v1/stocks/{code}` | 个股详情（行情/指标/因子/板块） |
| GET | `/api/v1/stocks/{code}/kline` | K 线序列（含 MA/BOLL） |
| GET | `/api/v1/algo/list` | 策略清单（参数 schema） |
| GET | `/api/v1/algo/{code}` | 执行/读取策略结果（含 run 元信息） |
| POST | `/api/v1/algo/{code}/refresh` | 强制刷新策略 run |
| GET | `/api/v1/algo/{code}/runs` | 历史 run 列表 |
| GET | `/api/v1/algo/runs/{run_id}` | 历史 run 明细 |
| GET | `/api/v1/algo/ladder` | 策略天梯（5/10/20 日前瞻收益与胜率） |
| GET | `/api/v1/algo/consensus` | 多策略共识榜 |
| POST | `/api/v1/algo/hits` | 自选股近 N 日策略命中 |
| GET | `/api/v1/algo/{code}/export.csv` | 结果导出 |
| POST | `/api/v1/backtest/{code}` | 创建回测（立即返回，后台执行） |
| GET | `/api/v1/backtest/{bt_id}` | 回测结果（queued/running/success/failed） |
| GET | `/api/v1/backtest/{bt_id}/trades` | 成交明细 |
| POST | `/api/v1/backtest/{code}/grid` | 参数敏感性矩阵 |
| GET | `/api/v1/backtest/grid/{grid_id}` | 矩阵结果 |
| GET | `/api/v1/paper/portfolios` | 纸面组合列表 |
| GET | `/api/v1/paper/portfolio/{code}` | 纸面组合净值 / 持仓 / 待执行 |
| POST | `/api/v1/paper/portfolio/{code}/sync` | 手动重算纸面组合 |
| GET/PUT | `/api/v1/watchlist` | 自选状态（单行，所有浏览器共享） |
| GET | `/api/v1/meta/freshness` | 数据新鲜度（前端徽标唯一来源） |
| GET | `/api/v1/meta/jobs` | 采集/策略任务监控 |
| GET | `/api/v1/meta/trade-calendar` | 交易日历 |
| POST | `/api/v1/admin/ingest/{task}` | 手动触发采集（需 `X-Admin-Key`） |
| POST | `/api/v1/admin/catch-up` | 检查并补跑最近缺失交易日（需 `X-Admin-Key`） |
| POST | `/api/v1/admin/seed-demo` | 演示数据（仅 `ENV=demo`） |

新闻统计接口：`/api/v1/articles`、`/api/v1/search`、`/api/v1/stats/*`。
错误统一为 `{"ok": false, "error": {"code", "message", "detail"}}`，
HTTP 语义：参数 422 / 不存在 404 / 数据未就绪 409 / 外部源 503。

---

## 7. 测试与 CI

```bash
cd backend
..\.venv\Scripts\python.exe -m pytest tests -q          # 纯逻辑测试，不需要数据库

# 需要真实库的集成测试（先跑 demo + features/scores/sector_scores/algos）
..\.venv\Scripts\python.exe -m pytest tests/integration -q
```

Windows 下集成测试环境变量：

```powershell
$env:GFNA_TEST_DB=1
$env:SCHEDULER_ENABLED=0
$env:CATCH_UP_ENABLED=0
```

覆盖：指标已知值（Wilder RSI 参考值）、金叉无未来函数、因子单调性、条件树、
参数指纹、回测绩效对已知净值、回测未来函数检测、API 契约与统一错误体、
**参数真生效**（`pe_max` 变化 → 不同 run 与结果）、**可复现性**（同参同 run）、
补缺计划与残日判定、策略 run 来源口径、指数对比/牛熊估值、纸面组合、自选命中。

前端：

```bash
cd frontend
npm run build
npm run test:treemap
npm run test:sparkline
npm run test:palette
```

CI（`.github/workflows/ci.yml`）在 push/PR 时跑四组：
后端单元测试、后端集成测试（PostgreSQL 16）、前端构建 + 纯函数测试、
浏览器验收（demo 数据 + Playwright + preview）。

---

## 8. 目录结构

```
backend/
├── app/
│   ├── core/        配置、常量、错误码、日志、状态枚举、所有制规则(soe.py)
│   ├── db/          Base/Session/模型(models/)/分区与视图/种子
│   ├── ingest/      Provider 链、akshare/baostock/sina/tencent、EM 估值/分红、
│   │                同花顺板块、日历、质量校验、批量回补
│   ├── engine/      指标、因子、横截面、个股/板块评分、广度、市场因子、
│   │                温度计、市场状态、指数对比、牛熊估值、信号前瞻、共识、纸面
│   ├── strategy/    条件树、A3/A4 + B1–B5、runner、回测、绩效、参数矩阵
│   ├── api/v1/      新闻、行情、个股、选股、回测、纸面、自选、元信息、管理
│   ├── quality/     数据不变量巡检
│   ├── services/    新鲜度等共享服务
│   ├── catch_up.py  启动补缺（历史缺口 + 残日 + 当日）
│   ├── pipeline.py  任务编排（run_task / run_daily_pipeline）
│   ├── worker.py    定时任务（15:36 盘后 / 16:30 选股 / 17:10 信号统计 /
│   │                17:30 纸面 / 02:00 补齐 / 新闻）
│   └── fixtures/    离线确定性演示数据
├── alembic/         迁移
├── scripts/         bootstrap_db / pipeline_cli / backfill_* / check_invariants 等
└── tests/           单元 + integration
frontend/src/
├── api/ hooks/ types/ lib/
├── components/      图表、任务、回测、纸面、共识、天梯、命令面板等
└── pages/           Home / Dashboard / IndexCompare / AlgoCenter / Paper /
                     Watchlist / StockDetail / Jobs / About
docs/                V5 方案与契约、V6 任务书/遗留计划、V7 任务书与校准报告、evidence
.github/workflows/ci.yml
```

---

## 9. Docker 部署

```bash
# 前提：启动 Docker Desktop（否则报 npipe dockerDesktopLinuxEngine 找不到）
docker compose up -d --build        # 首次约 5–10 分钟
docker compose logs -f backend
# 浏览器：http://127.0.0.1:5173   (前端 Nginx，/api 反代到 backend:8000)
```

镜像已包含：

| 项 | 说明 |
|---|---|
| **Playwright Chromium** | `RUN python -m playwright install --with-deps chromium`（含 libnss3 等系统库） |
| 自动初始化 | 入口脚本先 `bootstrap_db.py`（建表/分区/视图/种子），再 `first_run.py` |
| **首次自动灌数据** | `SEED_DEMO_ON_EMPTY=1`（compose 默认）→ 空库时灌演示数据并算指标/评分/选股 |
| 自动补缺 | 默认 `SCHEDULER_ENABLED=1`、`CATCH_UP_ENABLED=1`，启动后补最近缺失交易日（最多 5 天） |
| 时区 | `TZ=Asia/Shanghai`（避免容器 UTC 导致 15:36 实际是北京 23:36） |
| 健康检查 | `/health`（含数据库连通性） |

> **关于“启动后没有数据”**：原因不是缺 Playwright（它只影响新闻爬虫），
> 而是空库没有任何行情。想直接用真实数据请设 `SEED_DEMO_ON_EMPTY=0`，再执行
> `docker compose exec backend python scripts/pipeline_cli.py all`；
> 需要历史行情时先跑 `history_fast / moneyflow_history / valuation_fast`。
>
> PostgreSQL 端口映射到宿主机 **5433**（避免与本机已装的 5432 冲突），容器内部仍用 `postgres:5432`。
> 所有端口只绑定 `127.0.0.1`；管理接口写操作需配置 `ADMIN_API_KEY`（留空仅允许本机访问）。

---

## 10. 常见问题

| 现象 | 原因 / 处理 |
|---|---|
| `pip install` 卡住/超时 | 用镜像：`-i https://mirrors.aliyun.com/pypi/simple/` |
| 导入即崩、缺 asyncpg/pandas | 依赖未装或装错解释器；确认使用仓库根 `.venv` |
| Windows 下 uvicorn 卡死 | 已在 `main.py` 设 `WindowsSelectorEventLoopPolicy`；不要移除 |
| **页面打开但没有数据** | 空库没有行情 → `first_run.py`/`pipeline_cli.py demo`，或按顺序回补真实数据 |
| **新闻爬虫报 `Executable doesn't exist ... chromium`** | Playwright 只装了 Python 包：`python -m playwright install chromium` |
| 东财接口 `RemoteDisconnected` | 本网络屏蔽东财 `push2*`；日线走腾讯/新浪，板块自动降级同花顺，估值走 datacenter |
| 腾讯回补出现 501/拦截页 | 已内置 WAF 熔断：连续失败后冷却 15 分钟并自动改走新浪；等冷却结束再重跑 |
| 板块热力图空 / 名单不完整 | 先跑 `pipeline_cli.py sector`；同花顺只暴露部分分页时标 `partial`，涨跌家数与市值如实置空 |
| A3/A4/B1–B5 无结果 | 先确认历史日线、估值、指标/因子、板块评分已就绪；B4/B5 还需要沪深300 指数 |
| 估值整体为空 | `valuation_fast` 覆盖 2018 年起；更早日期按设计留空，不外推 |
| 北向资金为空 | 2024-08 起暂停披露，属预期 |
| 北交所标的搜不到 | 股票池白名单只保留沪深主板/创业板/科创板，属预期 |
| 评分缺失显示“未评分” | 样本不足或该日未计算；先 `features`/`scores`，不再用 5.0 冒充 |
| 纸面组合 `pending` | 等待 17:30 任务，或在 `/paper` 触发 sync |
| 回测一直 `running` | 回测是后台任务；启动时会清理僵尸任务，也可用 `backtest_grid` 重试失败格 |
| `run` 记录查不到 | `get_db` 已改为成功即 commit；若自建会话请显式 commit |
| 启动没有自动补缺 | 检查 `SCHEDULER_ENABLED`、`CATCH_UP_ENABLED`、后端日志，或到「任务监控」点「一键补数」 |

---

## 11. 免责声明

本系统为工程技术项目，**不含投资建议**。所有信号仅供个人研究，不构成买卖依据。
量化价值在于把决策变得可复现、可归因、可证伪；回测漂亮的策略在实盘失效是常态。
市场有风险，投资需谨慎。
