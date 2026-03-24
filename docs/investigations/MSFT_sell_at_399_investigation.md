# MSFT Sell at $399 – Investigation Summary

**Date:** 2026-03-13  
**Context:** MSFT rose to ~$404 (US morning) then was sold at ~$399 (408 shares in two fills: 314 + 94 at ~$399.39–$399.40). User questioned whether the bot made a mistake, whether data was 15 minutes old, and whether Alpaca bracket orders should be disabled.

---

## 1. What happened

- **Sell execution:** Two Alpaca FILLs at 02:59:25 PM (Mar 13, 2026): Sell 314 MSFT (~$399.39) and Sell 94 MSFT (~$399.40), total 408 shares.
- **Price context:** Intraday chart shows MSFT peaking above $404 (~13:45 UTC / 9:45 AM EST), then declining; by the time of the sells the price was around $399–400.

---

## 2. Did the bot “decide” to sell at $399?

**No.** The sell was almost certainly triggered by the **bracket order’s stop-loss leg**, not by the bot issuing a new sell at that moment.

- US (Alpaca) buys are placed as **bracket orders**: limit buy + take-profit (TP) + **stop-loss (SL)**.
- SL is sent to Alpaca as a fixed `stop_price`. When the market trades at or through that price, Alpaca executes the sell.
- So: once the bracket was placed, the sell at ~$399 was the **SL leg firing** when price fell from the morning high to that level.

---

## 3. How the stop price is set

- **Source:** `DayTradingManager.calculate_buy_sell_targets` / `calculate_buy_sell_targets_with_ml` return `(buy_p, sell_p, stop_p)`.
- **Stop formula:**  
  `dynamic_stop_pct = max(0.01, stop_loss_pct, adr_pct * 0.75)`  
  then `stop_p = buy_target * (1 - dynamic_stop_pct)`.
- **Config:** `day_trading.stop_loss_pct` is 0.02 (2%). So the effective stop is at least 2% below the **buy target** (the limit price used for the bracket).
- **Risk multiplier:** In `_process_symbol_buy_path`, `risk_multiplier` can **tighten** the stop (move it closer to entry). So a 2% distance can be reduced (e.g. in volatile/systemic regimes), which can produce a stop near $399 if entry was around $403–404.

So for a sell to occur at **$399**:

- Either the **buy target (limit)** used for the bracket was around **$404** and the effective stop was ~1–1.25% below that (e.g. after risk multiplier or a slightly tighter dynamic stop), or  
- The buy target was lower (e.g. ~$400) and the **risk multiplier** tightened the stop upward to ~$399.

In both cases the **bracket’s stop-loss** is the mechanism that sold at ~$399; the bot did not separately “choose” to sell at that price.

---

## 4. Is the data 15 minutes old?

- **Data path for US symbols:**  
  `load_historical_data` → `get_historical_data` (data_factory): **IBKR first**, then **Alpaca**, then **YFinance**.
- If the process uses **Alpaca** for bars and the account/data plan is **delayed** (e.g. 15‑min delay), then the bot can place brackets using **stale** `last_close` and thus stale `buy_p` / `stop_p`.
- The code already treats Alpaca as a potentially delayed source and applies a **20‑minute** staleness threshold for “blocking” actions; between **3–20 minutes** it only **warns**:  
  `"Operating on DELAYED data ... High risk for stop-loss accuracy!"`
- So **yes**: 15‑minute delayed data is possible for Alpaca-sourced bars, and that can make the bracket’s stop (and TP) less aligned with the real-time move to $404 and back to $399.

---

## 5. Should we stop placing bracket orders?

**Trade-offs:**

| Approach | Pros | Cons |
|----------|------|------|
| **Keep brackets** | Automatic TP/SL at broker; no dependency on bot being up to cancel. | SL can fire on normal pullbacks (e.g. $404 → $399); delayed data makes SL/TP levels less accurate. |
| **No brackets (limit buy only)** | Bot controls all exits; can avoid selling on brief dips. | Exits only when the bot runs and decides; no automatic protection if bot is down or slow. |

**Recommendation:**

- **Short term:** Add a **config switch** (e.g. `day_trading.use_bracket_orders_alpaca` or `use_bracket_orders`) so you can **disable** bracket orders for Alpaca and use **limit buys only**; the bot then exits via its own logic (e.g. ExitEvaluator, runner protocol). This stops “uncontrollable” bracket-driven sells at a fixed SL.
- **If you keep brackets:** Prefer **real-time or low-latency data** for US symbols when setting bracket levels, and/or **widen** the stop (e.g. more ADR-based, or larger `stop_loss_pct`) so normal pullbacks like $404→$399 don’t trigger the SL.

---

## 6. Root causes (summary)

1. **Bracket stop-loss** – The sell at ~$399 was the bracket’s SL leg, not a separate bot sell decision.
2. **Stop level** – Stop was likely at or just above $399 (from buy target ~$403–404 and/or risk multiplier tightening).
3. **Data delay** – Alpaca can be delayed (e.g. 15 min); the code allows it up to 20 min with a warning, so brackets can be placed on stale prices.
4. **Volatility** – A ~1.25% drop from $404 to $399 is a normal intraday pullback; a tight SL will often get hit in that situation.

---

## 7. Actions taken / suggested

- **Done:**
  - This investigation doc.
  - **Config option** `day_trading.use_bracket_orders_alpaca` (default `true`). When `false`, Alpaca buys are **limit-only** (no TP/SL at broker); the bot controls all exits via ExitEvaluator/runner logic.
  - **Dashboard:** Control Center sidebar checkbox **"Use bracket orders (Alpaca)"** to toggle the setting; change is persisted to `trading_config.json`.
- **Suggested (when brackets are kept):**
  - Consider **ADR-based or wider** stops for volatile names, and **real-time (or low-delay) data** for US symbols when available.
  - Optionally **log** the exact `(buy_p, sell_p, stop_p)` and data source/timestamp when placing brackets, to make future post-trade reviews easier.
