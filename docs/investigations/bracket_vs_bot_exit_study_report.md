# Bracket vs Bot-Managed Exit Study

**Period:** 2026-01-01 to 2026-03-12  
**Generated:** 2026-03-13 16:00 UTC

## Summary

| Metric | Value |
|--------|--------|
| Round trips (closed positions) | 67 |
| **Total actual P&L (bracket fills)** | $-21,382.11 |
| **Total simulated P&L (bot-managed)** | $-12,601.07 |
| **Improvement (simulated − actual)** | $8,781.04 |
| Trades where bot would have done better | 32 |
| Trades where bot would have done worse | 35 |
| Bracket closed at loss, bot would have been profitable | 0 |
| Bracket profitable, bot would have been even better | 15 |

## Interpretation

- **Actual P&L** = realized outcome from broker bracket order fills (TP/SL legs).
- **Simulated P&L** = outcome if the bot had managed exits using simplified rules on daily bars (ADR harvest, blow-off, 2% SL, 10% trailing retrace).
- **Improvement** > 0 suggests bot-managed exits would have improved results over bracket fills in this period.

## Round-trip detail

See `bracket_vs_bot_exit_round_trips.csv` for per-trade columns: symbol, entry/exit dates and prices, actual vs simulated P&L, `sim_exit_reason`, and `improvement_usd`.

## Methodology

1. Fetched Alpaca CLOSED orders from 2026-01-01 to 2026-03-12.
2. Built round trips by FIFO matching buys to sells per symbol.
3. For each round trip, fetched daily OHLCV (yfinance) and ran a simplified exit simulation: first bar where one of (emergency SL -2%, ADR harvest 0.75×ADR, blow-off 1.4×ADR, trailing 10% retrace) triggered; exit price = that bar's close.
4. Compared actual realized P&L to simulated P&L.

---

## Conclusions and next steps

### What the numbers show

- **Aggregate:** Bot-managed exits would have lost **$8,781 less** in this period (actual bracket outcome −$21,382 vs simulated −$12,601). So broker-level brackets are meaningfully worse than the bot’s exit logic in aggregate.
- **Trade count:** 32 trades would have been better with the bot, 35 worse — but the *size* of improvements outweighs the deteriorations, hence the positive total improvement.
- **Winners:** In **15** trades the bracket was already profitable but the bot would have been **even better** (e.g. bracket TP or early exit left upside on the table).
- **Losers:** **0** trades where the bracket closed at a loss and the bot would have been profitable. So the gain isn’t from “bracket turned winners into losers”; it’s from better exit *levels* (less loss or more profit per trade) when the bot’s rules (ADR harvest, trailing, 2% SL) are used instead of fixed bracket TP/SL.

### Recommended next steps

1. **Turn off bracket orders for Alpaca**  
   - In the dashboard: **Control Center → uncheck “Use bracket orders (Alpaca)”**, or  
   - In `trading_config.json`: set `day_trading.use_bracket_orders_alpaca` to `false`.  
   New Alpaca buys will be limit-only; the bot will manage all exits (ExitEvaluator, runner, etc.).

2. **Leave existing behaviour unchanged for now**  
   - No code change required; the study used the existing sim logic.  
   - Optionally re-run the study periodically (e.g. monthly) to compare new round trips to the same simulated rules.

3. **Monitor after switching**  
   - Track realized P&L over the next few weeks and compare to the study period.  
   - If you later want a tighter comparison, you could log the bot’s actual exit reason (e.g. from ExitEvaluator) per close and compare to the study’s `sim_exit_reason`.

4. **Keep the study as a guard rail**  
   - Use this report and `bracket_vs_bot_exit_round_trips.csv` as the documented basis for “bracket off, bot-managed exits on.”  
   - If you re-enable brackets in future, re-run the script and review the summary before committing.

*This study supports the case for disabling broker-level bracket orders and letting the bot manage exits.*
