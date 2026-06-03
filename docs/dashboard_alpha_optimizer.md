# Alpha Optimizer (Top-20 Swing Strategy Monitor)

## Purpose

The **Alpha Optimizer** Streamlit tab is a **strategy-only** view for the orchestrator’s **top-20 ADR universe**. It does not duplicate portfolio P&amp;L or recovery logic — that lives on **Strategic Holdings**.

| Tab | Responsibility |
| :--- | :--- |
| **Strategic Holdings** | Open book, cost basis, unrealized P&amp;L, recovery signals |
| **Alpha Optimizer** | Orchestrator rank, ADR entry/harvest bands, charts, planned vs actual |
| **Health Audit & Logs** | Sentinel, self-corrections detail, raw logs |

## Data sources

1. **Primary — orchestrator CSV** (latest row per symbol):
   - `data/intraday_top_20_with_target.csv`
   - `data/top_20_day_trading_candidates.csv`
   - `weekly_analysis/top_candidates/top_20_day_trading_candidates.csv`
2. **Fields used:** `symbol`, `buy_price_target`, `sell_price_target`, `avg_daily_range_pct`, `rank`, `rsi`, optional `close` for board distance-to-entry.
3. **Fallback — live recompute:** `DayTradingManager.calculate_buy_sell_targets()` when a symbol is missing from CSV.
4. **Charts:** `data_factory.get_historical_data()` (IBKR → Alpaca → YFinance), with LSE pence normalised via `lse_price_to_gbp_pounds()`.
5. **Valuation panel:** `data/symbol_profiles/latest_symbol_profiles.csv` (or weekly_analysis mirror).

## UI layout

1. **Hero metrics** — top-20 count, held-in-top-20 count, last orchestrator run time, buy buffer, ML threshold.
2. **Top-20 strategy board** — sortable table (rank, ADR%, entry, harvest, dist to entry, held badge).
3. **Symbol drill-down** — default timeframe **Intraday (Last 3 Days)**; chart overlays:
   - Buy support (green)
   - ADR harvest / hard limit (blue)
   - Blow-off top (yellow, when ADR &gt; 0)
   - Risk guard / stop (red)
4. **Planned vs Actual** expander — FIFO round-trips vs orchestrator harvest plan + tail of `enhanced_decision_log.jsonl` (including `strategy_plan` when present).
5. **Self-corrections** — collapsed; full history remains under Health Audit.

## Symbol universe

- Default picker: **top-20 only** (orchestrator list).
- Optional checkbox: *Include open positions not in top-20* (off by default in scalper-focused workflows).

## Currency display

Listing currency is inferred from the ticker (`symbol_listing_currency()` in `trading_bot/core/fx_utils.py`):

- `.L` → GBP (£)
- `.TO` → CAD (C$)
- `.PA` / `.DE` / `.AS` → EUR (€)
- otherwise USD ($)

## Code map

| Component | Path |
| :--- | :--- |
| Streamlit render | `trading_bot/core/alpha_optimizer_ui.py` |
| Tab entry (thin wrapper) | `trading_bot/core/app.py` → `render_alpha_optimizer_tab()` |
| Tests | `tests/test_alpha_optimizer_ui.py` |

## Operational notes

- If the board is empty, run the **daily orchestrator** so `intraday_top_20_with_target.csv` is produced.
- **FALLBACK DATA** banner means YFinance (or non-IBKR) bars — targets may still be valid from CSV.
- LSE display bugs (100× prices) are documented in `docs/investigations/lse_gbp_pence_normalization_2026_03.md`.

## Related documentation

- [ML Operational Lifecycle](ml_operational_lifecycle.md) — §4.5 planned vs actual and `strategy_plan`
- [Data Architecture](data_architecture.md) — decision log schema
- [System Architecture](system_architecture.md) — Streamlit observability
