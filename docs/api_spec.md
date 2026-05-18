# API & Integration Specification

## 1. External Broker Integrations

### 1.1 Alpaca Trade API (Primary)
-   **Version**: V2 (Trading & Market Data)
-   **Authentication**: API Key ID and Secret Key via `APCA-API-KEY-ID` and `APCA-API-SECRET-KEY` headers.
-   **Key Endpoints**:
    -   `POST /v2/orders`: Place bracket orders (Take Profit/Stop Loss).
    -   `GET /v2/positions`: Retrieve current active holdings.
    -   `GET /v2/account`: Monitor buying power and equity.
-   **Websockets**: `wss://paper-api.alpaca.markets/stream` for real-time order fill updates and account changes.

### 1.2 Interactive Brokers (IBKR)
-   **Integration Method**: IBKR Gateway via `ib_insync` (TWS API).
-   **Host/Port**: `ib-gateway:8888` (within Docker network, overridable via `IBKR_HOST`/`IBKR_PORT`). Gateway container also exposes `4002`; bot uses **8888** (socat) by default.
-   **Authentication**: Standard IBKR credentials configured on the Gateway container; the bot connects via the TWS API, not Client Portal REST.
-   **Market data type**: On connect, `reqMarketDataType(1)` (live). UK LSE requires an active **LSE Equities** subscription in IBKR.
-   **Historical bars (intraday loop)**:
    -   `IBKRBroker.get_historical_data()` — uses the **persistent** `self.ib` connection (`reqHistoricalData` on main thread). Default: `IBKR_HISTORICAL_USE_MAIN=true`.
    -   Do not spawn ephemeral connections per symbol in production (caused gateway timeouts before May 2026; see [VM Recovery](investigations/2026-05-18_vm_recovery.md)).
    -   Unified loader: `get_historical_data()` in `data_factory.py` → tagged `source=IBKR` in DataFrame attrs for staleness logic in `trading_bot.py`.
-   **Execution Flow**:
    -   `BrokerRouter` uses `IBKRBroker` to submit orders and read positions using the TWS API (`ib_insync`).
    -   A long-lived **account summary subscription** (`reqAccountSummary`) provides:
        -   `NetLiquidation` (equity), `TotalCashValue` (cash), `BuyingPower`, and `GrossPositionValue` (capital in positions).
    -   A long-lived **account updates subscription** (`reqAccountUpdates`) populates:
        -   `RealizedPnL` / `UnrealizedPnL` at the account level, used by the dashboard for **IBKR Day P&L**.
-   **Dashboard Consumption**:
    -   The Streamlit dashboard reads these live values via `IBKRBroker.get_account_summary()` and displays **NAV (equity)**, **cash**, **capital active**, and **Day P&L** directly from IBKR, with order-based P&L only used for historical period summaries and as a fallback.

## 2. Market Data Providers

### 2.0 Unified loader (runtime)
See `trading_bot/core/data_factory.py` — priority **IBKR → Alpaca (US) → YFinance**. International symbols (e.g. `*.L`) require IBKR.

### 2.1 Polygon.io
-   **Authentication**: Bearer token via `apiKey` query parameter.
-   **Usage**:
    -   **Sector Analysis**: Primary source for fetching daily bars for the `SectorAnalyzer`.
    -   **Intraday Outliers**: Used via `fetch_polygon_intraday_data.py` to identify high-volatility candidates during weekly analysis.
-   **Persistence**: Data is cached in `./data/sector_report.json`.

### 2.2 yfinance (Primary Prep)
-   **Usage**: The primary data source for the **ML Preparation Pipeline** and **Historical Ranking**.
-   **Protocol**: HTTPS (Scraping/REST fallback).

## 3. Internal Event Flows

### 3.1 Decision Logger (JSONL)
-   **Pattern**: Append-only log stream.
-   **Payload**: Structured JSON containing `market_context`, `agent_reasoning`, and `order_id`.
-   **Consumer**: `app.py` (Streamlit Dashboard) which tails the file for real-time UI updates.

## 4. Error Handling & Resilience
-   **Rate Limiting**: Implementation of exponential backoff for Alpaca (429 errors).
-   **Circuit Breakers**: If 3 consecutive API failures occur with a broker, the bot enters "Graceful Shutdown" mode to prevent stale-data orders.
-   **Payload Validation**: All outgoing order payloads are validated against the `OrderSchema` to prevent rejection due to malformed metadata.
