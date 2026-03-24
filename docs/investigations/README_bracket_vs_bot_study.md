# Bracket vs Bot-Managed Exit Historical Study

## Purpose

This study compares **what actually happened** (broker bracket order TP/SL fills) with **what would have happened** if the bot had managed exits (no bracket at broker; exit logic on historical bars). It supports the decision to disable bracket orders at broker level and let the bot manage exits.

## How to run

From the project root, with Alpaca API keys set (e.g. in `.env`):

```bash
python3 scripts/bracket_vs_bot_exit_study.py
```

**Period:** 2026-01-01 through yesterday (UTC).  
**Output:**

- `docs/investigations/bracket_vs_bot_exit_round_trips.csv` – per–round-trip actual vs simulated P&L
- `docs/investigations/bracket_vs_bot_exit_study_report.md` – summary and interpretation

## Methodology (short)

1. Fetch Alpaca CLOSED orders for the date range.
2. Build round trips by FIFO matching buys to sells per symbol.
3. For each round trip, fetch daily OHLCV (yfinance) and run a **simplified bot-exit simulation**: first bar where one of (emergency SL -2%, ADR harvest 0.75×ADR, blow-off 1.4×ADR, trailing 10% retrace) triggers; exit price = that bar’s close.
4. Compare actual realized P&L to simulated P&L; report improvement and counts (e.g. “bracket closed at loss, bot would have been profitable”).

## Related

- **Decision and rationale:** `bracket_orders_decision.md` – why we disable brackets (no middle ground; bot manages exits).
- **MSFT sell at $399:** `MSFT_sell_at_399_investigation.md` – why the bracket stop fired and the option to turn off brackets.
- **Config:** `day_trading.use_bracket_orders_alpaca` and dashboard checkbox “Use bracket orders (Alpaca)”.
