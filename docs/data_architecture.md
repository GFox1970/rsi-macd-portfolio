# Data Architecture Document

## 1. Overview
The system utilizes a hybrid data architecture, combining unstructured **JSONL** logs for high-frequency event auditing and structured **SQLite** storage for Machine Learning feature/outcome pairs.

## 2. Ingestion Layer
Data is ingested from four primary sources:
-   **Market Data (OHLCV) — intraday execution** (`trading_bot/core/data_factory.py`):
    1.  **IBKR** (primary) — US and international (e.g. `.L` LSE); real-time minute bars via main `IBKRBroker` session.
    2.  **Alpaca** — US symbols only when IBKR unavailable.
    3.  **YFinance** — delayed fallback; avoid for live entries when staleness guard is active.
-   **Market Data — preparation/training**: **yfinance** and cached CSVs under `data/historical` (orchestrator / weekly analysis). Yahoo errors on bare tickers (e.g. `STJ` vs `STJ.L`) do not block IBKR execution paths.
-   **Broker State (High-Fidelity)**: Live positions and prices via Alpaca/IBKR. **Primary source of truth for real-time exit decisions (TP/SL).**
-   **Macro Indicators**: FRED (Federal Reserve Economic Data) for VIX and interest rate series.

## 3. Data Storage Models

### 3.1 Decision Log (Audit Trail)
-   **Format**: JSONL (JSON Lines)
-   **Location**: `logs/enhanced_decision_log.jsonl`
-   **Primary Key**: `decision_id` (UUID + Timestamp)
-   **Key Fields**:
    -   `market_context`: Indicators (RSI, MACD), OHLCV, Sentiment.
    -   `portfolio_state`: Buying power, current holdings.
    -   `decision`: Action (BUY/SKIP/SELL), Confidence, Reasoning.
    -   `strategy_plan`: Optional ADR targets at decision time (`buy_target`, `sell_target`, `stop_target`, `adr_pct`, `source`) — written after target calculation in `TradingBot`; used by shadow backfill and the Alpha Optimizer planned-vs-actual panel.
    -   `order_outcome`: Fill price, qty, status, `signal_reference_price`, `slippage_bps`.
    -   `result`: Shadow prices (`price_after_1h`), proxies (`pnl_after_1h`), optional `execution` block after enrich.

### 3.2 IBKR Execution Log (Persistence)
-   **Format**: JSONL
-   **Location**: `logs/ibkr_fills.jsonl`
-   **Purpose**: Persistent storage of all IBKR trade executions to prevent data loss during bot restarts/redeployments.
-   **Fields**: `timestamp`, `symbol`, `side`, `qty`, `price`, `execId`, `orderId`, `permId`, `account`.

### 3.3 ML Feature DB (Training)
-   **Engine**: SQLite
-   **Location**: `ml_db/ml_data.db`
-   **Table: `ml_rows`**
| Column | Type | Description |
| :--- | :--- | :--- |
| `decision_id` | TEXT (PK) | Unique ID linking to the decision log. |
| `symbol` | TEXT | Stock ticker. |
| `timestamp` | TEXT | ISO8601 creation time. |
| `action` | TEXT | BUY/SELL/SKIP. |
| `json_row` | TEXT (JSON) | Flat map of all features at time of trade. |
| `outcome_profit` | REAL | Raw profit/loss from the trade. |
| `holding_period` | INTEGER | Time in minutes between Open and Close. |

### 3.3 State Management
-   **Positions**: `positions.json` stores current active trades to persist across container restarts.
-   **PDT Cache**: `pdt_state.json` tracks day-trade counts for regulatory compliance.

## 4. Data Flows

### 4.1 Feature Lifecycle
1.  **Extraction**: Indicators calculated by `IndicatorsCalculator`.
2.  **Normalization**: Values scaled/mapped for XGBoost input.
3.  **Logging**: `EnhancedDecisionLogger` writes to both JSONL and SQLite.
4.  **Labeling**: Post-trade "Outcomes" are computed by the `PerformanceAnalyzer` and pushed back into the SQLite `outcome_profit` column.
5.  **Training**: `XGBoostTrainer` queries SQLite `ml_rows` where `outcome_profit` is NOT NULL.

### 4.2 Data Synchronization
To maintain parity between the production environment (Hetzner VM) and the local development environment, data is synchronized via the **Data-Harvester** skill (`scripts/pull_vm_data.sh`). This script pulls down the latest XGBoost models, training datasets, and primary historic database (`ml_trading_data.db`) to enable realistic local backtesting and model iterations.

## 5. Retention & Refresh Rules
-   **Decision Logs**: Standard retention of 90 days. Older logs are archived to `.gz` format monthly.
-   **ML Data**: Retained indefinitely to support multi-year backtesting and longitudinal model analysis.
-   **Market Data (Refresh)**: Intermediate CSVs in `./data/historical` are refreshed **4 times daily** (HK, UK, US Prep and US Midday) via GitHub Actions.
-   **GHA-VM Parity**: Fresh raw intraday `.csv` files are automatically synchronized from GHA to the VM during each market prep session to ensure the orchestrator generates targets from the latest available market data.
