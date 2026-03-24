# Bracket Orders: Decision and Rationale

**Decision:** Disable broker-level bracket orders for Alpaca. The bot manages all exits (no TP/SL legs at the broker).

**Config:** `trading_config.json` → `day_trading.use_bracket_orders_alpaca: false`  
**Dashboard:** Control Center → uncheck "Use bracket orders (Alpaca)"

---

## Why not a middle ground (track and cancel brackets before execution)?

A possible middle ground would be: keep bracket orders but have the bot watch open orders and cancel the SL (or TP) leg when its logic says "hold" (e.g. price near stop but bot would not exit yet).

This was rejected because:

- **Reaction speed:** By the time the bot sees "price near stop" and sends "cancel SL", the stop may already have been hit. You're in a race with the market.
- **Data delay:** If the bot uses delayed data (e.g. 15 min), it may think price is fine while the live market has already traded through the stop.
- **Two sources of truth:** Exit rules would live in both the bracket (broker) and the bot (ExitEvaluator). Keeping them in sync and deciding when to override the bracket adds complexity and edge cases.
- **Partial fills / multiple legs:** Tracking which orders are bracket children, cancelling the right leg, and handling "cancel didn't go through in time" is brittle.

The bot can only **react** to market data; it cannot know what the market will do next. So "cancel ahead of execution" is still reactive and races the exchange.

**Conclusion:** One place for exit logic (the bot), no race with the broker. No brackets at broker.

---

## Evidence

- **MSFT sell at $399:** Bracket stop fired on a normal pullback; see `MSFT_sell_at_399_investigation.md`.
- **Historical study:** `bracket_vs_bot_exit_study_report.md` and `bracket_vs_bot_exit_round_trips.csv` — bot-managed exits would have improved P&L by ~$8.8k in the sample period (2026-01-01 to 2026-03-12).
