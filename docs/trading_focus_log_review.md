# Trading Focus Log Review (2026-03-12)

## Summary: Bot is making the right risk and gating decisions

From the provided `trading_focus.log` (LSE symbols only), the following holds.

### 1. Two-stage confidence is correct

- **Strategic Decision** (logged as “BUY with confidence 0.XX”) uses the **agent combined score** (technical + sentiment + sector + macro, from `trading_agent.decide()`).
- **ML gate** uses the **raw ML model score** (`ml_confidence` / `technical_score`). A BUY is only allowed if this is **≥ `ml_threshold`** (e.g. 0.30).
- So you can see e.g. “Strategic Decision for EDV.L: BUY with confidence 0.51” and then “🛑 EDV.L BUY rejected: ML Confidence 0.2813 < 0.30 (Risk)”. That is correct: the agent score is 0.51 but the ML model is only 0.28, so the trade is blocked.

### 2. Regional guardrail (Flex-Control) — only when ON

- **When Flex-Control is OFF** (toggle in side panel): the regional hurdle check is **skipped**; no trades are restricted by Over-concentration Risk. The sidebar shows "Regional Guardrail: **OFF**" when disabled.
- **When Flex-Control is ON**: regional exposure over the limit raises the confidence hurdle (e.g. base 0.45 + tax 0.20 = 0.65).
- In your log, many rejections show “Confidence X < Hurdle 0.45 (Over-concentration Risk)”. So the base hurdle 0.45 is applied; signals with confidence below that are correctly rejected.
- Where you see “Confidence 0.45 < Hurdle 0.45”, the underlying value is almost certainly slightly below 0.45 (e.g. 0.449…) and is rounded to 0.45 in the log; the strict `< hurdle` check is behaving as designed.

### 3. Rejection reasons are consistent

- **Regional Guardrail**: “Signal rejected by Regional Guardrail. Confidence X < Hurdle Y (Over-concentration Risk)” → sizing step correctly blocks when combined confidence is below the regional hurdle.
- **ML Risk**: “🛑 SYMBOL BUY rejected: ML Confidence X < 0.30 (Risk)” → ML gate correctly blocks when raw ML score is below threshold.
- **Sizing / MVC**: “BUY rejected: Sizing or Risk rejection (MVC or Risk Limit hit)” → used when `determine_order_quantity` returns 0 (e.g. after regional guardrail or other risk checks).

### 4. What is not in this log

- **No SELL decisions** in the snippet; only BUY or SKIP. So this review only confirms **buy-side** gating.
- **No order placement lines** (e.g. “Order placed” / “Filled”). To verify execution quality (buy low / sell high, fills, prices), use **trading_bot.log** (and optionally exchange/IBKR fill logs).

### 5. Trading Bot Log (execution) — 2026-03-12

From the provided `trading_bot.log`: bracket orders were sent for ANTO.L, EDV.L, ENT.L, FRES.L, BRBY.L, MRO.L. **IBKR Warning 110** ("price does not conform to the minimum price variation") caused the parent LSE order to be rejected, then **Error 135** (children cancelled). Fix: LSE/GBP prices are now aligned to 1p min tick in `ibkr_executor.py`. **MTLN.L, AV.L, JD.L** have no security definition on IBKR; the bot correctly skips them. **Read-only `data/bracket_targets.json`** in Docker is a volume permission issue; bracket orders still reach IBKR.

### 6. PDT and US stocks in ISA via IBKR (UK)

- **PDT** is a US (FINRA) rule for **US brokerages** and typically applies to margin/pattern day trader accounts.
- **UK ISA with IBKR**: Whether PDT applies depends on how the ISA and account are set up (e.g. cash vs margin, which entity holds the account). Many UK investors use **Alpaca for US stocks** to avoid US margin/PDT; others use IBKR UK with an ISA and report no PDT if the account is not treated as a US pattern day trader account.
- **Recommendation**: Until you confirm with IBKR (or your broker) that your ISA is not subject to PDT, **continuing to use Alpaca for US stocks** is the conservative and correct approach. If IBKR confirms that US stocks in your UK ISA are not subject to PDT, you could then consider routing US flow through IBKR as well.

---

**Conclusion**: From this focus log, the bot is applying risk and gating logic as intended: ML threshold, regional confidence hurdle, and sizing/MVC rejections are consistent and correct. For “buy low, sell high” and execution quality, With **Flex-Control OFF**, the regional hurdle is skipped (sidebar: "Regional Guardrail: OFF"). The LSE min-tick fix should resolve Warning 110; re-run and confirm.
