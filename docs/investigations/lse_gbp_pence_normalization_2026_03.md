# LSE/GBP (Pence) Normalization Fix

**Date:** 2026-03-16  
**Issue:** Bot dashboard showed IBKR LSE position values 100× too large (e.g. £-43,720 instead of £-437.20).

## Root cause

- IBKR reports UK (LSE) positions in **GBp (pence)**. `avgCost` and bar prices from IBKR for `.L` symbols are in pence.
- The dashboard builds position rows from `fetch_open_positions()` and used these values as **pounds**, so portfolio value, P&L, cost basis, and all derived metrics were 100× too large.

## Fix applied

- **File:** `trading_bot/core/orders_dashboard.py`  
- In `fetch_open_positions()`, for IBKR positions where `yf_sym.endswith(".L")` and `currency == "GBP"`:
  - Convert `avg_cost` from pence to pounds (× 0.01).
  - Convert `current_price` from pence to pounds when it is clearly in pence (heuristic: `current_price > 1000`), since IBKR bars are in pence and YFinance returns pounds.
- All downstream values (`unrealized_pl`, `market_value`, `avg_entry_price`, `market_price`) are then in pounds and display correctly.

## Knock-on effects checked

| Area | Data source | Effect |
|------|-------------|--------|
| **Operational Plan (app.py)** | `p_df` from `fetch_open_positions()` | Fixed: portfolio value, cost basis, P&L, exit forecasts, SL, orchestrator buy/sell now in correct £. |
| **Capital employed (app.py)** | Same `positions_df` when broker `capital_active` not used | Fixed: IBKR sum of `market_value` now correct. |
| **Positions table / Strategic Holdings** | Same `positions_df` | Fixed: `unrealized_pl` and notional in £. |
| **CapitalRiskManager** | Raw `get_all_positions()` from broker | Unchanged: already applies `_get_scale_factor(symbol)` (0.01 for `.L`) when summing exposure; uses raw broker positions, not dashboard DataFrame. |
| **Exit elevator / staged exit** | Bot’s own position state (broker) | Unchanged: no use of dashboard `positions_df`; scaling handled in risk/execution path where needed. |
| **FX / convert_to_usd** | Orders use `GBp` for `.L` in preprocess; `convert_to_usd` divides by 100 for GBp | Unchanged; orders path already correct. |

## Edge cases

- **High-priced LSE names:** Threshold 1000 avoids treating £500–1000+ prices as pence; only values > 1000 are scaled (typical pence range for LSE is hundreds to tens of thousands).
- **YFinance fallback:** When current price comes from YFinance it is already in pounds (< 1000), so it is not scaled.

---

## 2026-03-17: Bot trading logic fix (BLOW-OFF TOP / Parabolic Hold)

**Issue:** Logs showed "BLOW-OFF TOP DETECTED (+9720% vs ADR)" and "Parabolic Hold active. Profit +10693%" for LSE stocks. These percentages are impossible and caused by the same pence/pounds mix: `current_price` from IBKR historical bars (pence) vs `entry_price` from broker position (pounds).

**Fix:** In `trading_bot/core/trading_bot.py`, before passing to `exit_evaluator`, normalize `current_price` and `entry_price` for LSE:
- When `symbol.endswith(".L")` or when `current_price/entry_price` is 50–150 (unit mismatch) and `current_price > 500`: scale any value > 1000 by 0.01 (pence → pounds).

**Effect:** Exit logic (blow-off top, ADR harvest, parabolic hold, tactical pilot, stop-loss) now receives consistent units and no longer triggers on phantom 9000%+ profit.

---

## 2026-03-17: JD.L / MTLN.L IBKR contract errors

**Issue:** Bot attempted to buy JD.L and MTLN.L; IBKR rejected with "Error 200: No security definition has been found". Orders failed with "Contract qualification failed".

**Cause:** These symbols are in the orchestrator/candidate universe but either (a) do not exist on LSE, (b) use a different ticker on IBKR, or (c) require a different exchange. JD.com trades as JD on NASDAQ, not JD.L on LSE.

**Mitigation:** Consider filtering out or mapping symbols that IBKR cannot qualify before routing bracket orders. Alternatively, maintain an IBKR-tradeable symbol allowlist for LSE.
