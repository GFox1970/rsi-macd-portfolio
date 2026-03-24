## Trading Bot Audit — 2026-03-16

**Scope:** Re‑audit of day‑trading performance with focus on exits, capital efficiency, and whether we can justify **holding positions when a stock is fundamentally undervalued** rather than treating everything as same‑day scalps.  
**Data used:**  
- `data/trades_exports/orders_all.csv`  
- `data/intraday_top_20_with_target.csv`  
- Prior investigation: `bracket_vs_bot_exit_study_report.md`

---

## 1. Recap of prior bracket vs bot-exit study

- **Period:** 2026-01-01 to 2026-03-12  
- **Finding:** Bot-managed exits (ADR harvest, blow-off, trailing, emergency SL) would have produced **~$8.8k less loss** vs broker bracket orders on the same round trips:
  - Actual P&L (brackets): **−$21,382.11**  
  - Simulated P&L (bot exits): **−$12,601.07**  
  - Improvement: **+$8,781.04**  
- The improvement came from **better exit levels** (less loss / more profit per trade), not from flipping losers to winners:
  - 15 trades: bracket profitable, bot would be **even better**.  
  - 0 trades: bracket loss, bot would have been profitable.  
- Decision: **prefer bot-managed exits over broker-level brackets** for Alpaca.

**Conclusion:** The bot’s exit framework is structurally better than static brackets and worth continuing to invest in.

---

## 2. Fresh efficiency snapshot (2026-03-03 to 2026-03-04)

Command run:

```bash
python3 trading_bot/core/performance_analyzer.py --all
```

This uses:
- Intraday data: `data/intraday_top_20_with_target.csv`
- Trade log: `data/trades_exports/orders_all.csv`
- Metric: **Efficiency** = Actual Return % / Perfect Trade Return %

**Observed sample (per symbol/date):**

- Symbols: ALB, COIN, CVNA, HOOD, MRNA, MSTR, Q, SMCI, SNDK  
- Dates: 2026-03-03 and 2026-03-04  
- For **every** symbol/date pair:
  - `efficiency_pct = 0.0`  
  - `actual_pnl` is **small and negative** (−$1 to about −$89)  
  - `reason` from classifier: **"Early exit (Profit target too conservative)"**

**Interpretation:**

- The **market offered intraday opportunities** (perfect trades exist) in these names, yet:
  - Our actual trades captured **essentially none** of the theoretical potential.
  - We systematically **exited early at a loss**, rather than participating in the move.
- This is consistent with:
  - **Overly conservative exits** for the type of entries we are taking, and/or  
  - Entries occurring in noisy or fading conditions where a 2%‑style loss is realized quickly.

---

## 3. Strengths vs weaknesses (2026‑03‑16 audit)

### 3.1 What the bot does well

- **Exit framework beats brackets:** Prior study shows bot-managed exits are materially better than static broker brackets in aggregate.
- **Closed-loop performance tooling:**
  - `PerformanceAnalyzer` to compute perfect vs actual efficiency.
  - `perfect_trades.csv` generation for ML labels and hindsight learning.
  - Nightly **Sentinel + Healer** loop to detect gaps and suggest fixes.
- **Regime-awareness exists in design:**
  - Macro regime and volume regime influence risk posture.
  - “Profit-first” modes can relax tight stops for small, volatile positions.

### 3.2 Where the bot is underperforming

- **Near-zero efficiency on recent trades:**
  - Every audited symbol/date in early March 2026 shows **0.0 efficiency** and **small realized losses**.
  - The classifier flags these as **Early exit (profit target too conservative)** rather than “loss from huge adverse move.”
- **Top 20 selection is purely trading/ADR-driven:**
  - We select high-ADR, liquid names without knowing if they are **fundamentally strong or undervalued**.
  - In weak / declining markets, this can leave us long high‑beta names that are not worth holding beyond the day.
- **No “okay to hold because undervalued” concept:**
  - Exit logic treats all longs as short-term trades that must be flattened relatively quickly.
  - We lack a **symbol classification** that says: “this is a high-quality, undervalued company where holding through noise is acceptable.”
- **Capital efficiency issues:**
  - Small, repeated losses in names like HOOD, Q, SMCI, etc. erode P&L without offsetting large winners in this sample.

---

## 4. Design direction: Top 20 candidates that we can hold because they are undervalued

### 4.1 Goal

Redefine the **Top 20 day-trading candidates** so they are not just “highest ADR” but also:

1. **Tradable intraday** (high ADR, liquid, acceptable spreads), and  
2. **Regime‑aligned** (showing relative strength vs their sector/index when the broader tape is healthy), and  
3. **Fundamentally strong and/or undervalued**  
   - So that, when market conditions are not ideal for intraday exits, we can **hold with conviction** instead of being forced into a 2%‑style loss.

This creates two clear behavioural paths:

- **INTRADAY_ONLY:** High‑ADR, purely technical/momentum names. We scalp them and avoid multi‑day exposure.  
- **CAN_HOLD_UNDERVALUE:** High‑ADR **and** fundamentally attractive names (like BYD‑type examples) where we are comfortable holding through volatility.

---

## 5. Proposed symbol profile & scoring model

### 5.1 Symbol profile fields

For each candidate symbol, define a daily `symbol_profile` with at least:

- `tradability_score`  
  - Based on ADR, average volume, and bid-ask spread.
- `regime_score`  
  - Based on relative strength vs sector ETF and broad index; macro and volume regime (bull/bear/volatile, high/low volume).
- `valuation_score`  
  - Based on simple valuation metrics such as:
    - P/E vs sector median.
    - EV/EBITDA vs sector median.
    - Price vs analyst consensus target or fair‑value proxy.
- `quality_score`  
  - Based on business quality:
    - Positive EPS or improving EPS trend.
    - Revenue growth above a small threshold.
    - Reasonable leverage / no severe balance sheet red flags.
- `can_hold` (boolean)  
  - True when the symbol is judged “good company at undemanding valuation.”
- `valuation_bucket` (enum)  
  - e.g. `UNDERVALUED`, `FAIR`, `EXPENSIVE`.

### 5.2 First-pass numeric rules for “undervalued, can hold”

These are **starting thresholds** we can tune with experience:

- **UNDERVALUED if any of:**
  - P/E < **0.80 × sector median P/E** (for positive‑EPS companies), **or**
  - EV/EBITDA < **0.80 × sector median EV/EBITDA**, **or**
  - Current price < **0.80 ×** analyst consensus target / fair‑value estimate.
- **QUALITY filter (all of):**
  - Positive trailing‑twelve‑month EPS **or** revenue growth > **5% YoY**.
  - Debt-to-equity below a chosen ceiling (e.g. < **1.5** where data is available), or clear evidence of improving leverage.
  - No hard exclusion flags (e.g. recent bankruptcy, going‑concern warnings).

Then:

- `can_hold = True` if:
  - `valuation_bucket == UNDERVALUED` **and** `quality_score` passes minimum thresholds.
- Else:
  - `can_hold = False` (treated as **INTRADAY_ONLY**).

### 5.3 Regime-aware Top 20 scoring

Define a composite score:

```text
Top20_score = w_tradability * tradability_score
            + w_regime      * regime_score
            + w_valuation   * valuation_score
            + w_quality     * quality_score
```

With weights changing by macro regime:

- **Bull / strong regime:**
  - `w_tradability = 0.5`, `w_regime = 0.2`, `w_valuation = 0.2`, `w_quality = 0.1`
- **Bear / volatile regime:**
  - `w_tradability = 0.3`, `w_regime = 0.3`, `w_valuation = 0.3`, `w_quality = 0.1`

We then:

- Rank all candidates by `Top20_score`.  
- Select:
  - Up to **N_intraday** purely intraday names (e.g. 10–15), and  
  - Up to **N_holdable** undervalued names (e.g. 5–10) where `can_hold = True`.

---

## 6. Exit behaviour based on profile

### 6.1 INTRADAY_ONLY symbols

For symbols where `can_hold = False`:

- Use existing **tight intraday exits**:
  - Same‑day emergency SL around **2%** or ADR‑scaled.
  - ADR‑based harvest targets and trailing retrace logic.
  - Time‑based Staged Exit for multi‑day positions to free capital quickly.
- Bias strongly towards **being flat by end of day** unless profit-first rules explicitly advise otherwise for tiny notionals.

### 6.2 CAN_HOLD_UNDERVALUE symbols

For symbols where `can_hold = True`:

- Use **wider, catastrophe‑only stops** rather than tight 2% exits:
  - Example: **8–15%** downside stop, optionally ADR‑scaled (e.g. 1.5–2.0 × 20‑day ADR).
- Relax same‑day emergency SL:
  - Optionally **skip** same-day 2% stop when notional is above a conviction threshold and macro regime is not catastrophic.
- Extend time horizon for Staged Exit:
  - Allow **more days** underwater before triggering auto‑sell, to give undervalued names room to mean‑revert.
- Keep **take‑profit logic** (ADR harvest, trailing) active:
  - We still want to realize gains when a strong run occurs; “undervalued” is not a reason to ignore profit-taking.

Net effect: when a fundamentally solid, undervalued stock is temporarily dragged down by broad volatility, the bot is allowed to **hold through noise** instead of reflex-selling at −2% and crystallizing small losses.

---

## 7. Implementation worklist (from this audit)

1. **Add `symbol_profile` concept** to the daily orchestrator:
   - Extend Top 20 generation to compute `tradability_score`, `regime_score`, `valuation_score`, `quality_score`, `can_hold`, and `valuation_bucket`.
   - Integrate at least one robust fundamentals source (broker API or external fundamentals API) updated nightly.
2. **Refactor Top 20 ranking** to use the composite `Top20_score` with regime‑aware weights:
   - Implement separate budgets: `N_intraday` and `N_holdable`.
3. **Wire `can_hold` into intraday exit logic:**
   - For INTRADAY_ONLY: keep current tight stops and time limits.  
   - For CAN_HOLD_UNDERVALUE: wider catastrophe stop, relaxed same-day SL, longer Staged Exit horizon.
4. **Extend performance analytics:**
   - Track efficiency and P&L **by `can_hold` vs `intraday_only`** and **by valuation bucket**.
   - Confirm that undervalued holdable names behave better in down / volatile periods than pure ADR scalps.
5. **Re‑run this audit after changes:**
   - Re‑run `PerformanceAnalyzer --all` and a new bracket vs bot‑exit style study over a multi‑month window to evaluate:
     - Total P&L, drawdown.
     - Efficiency distribution (`Optimal`, `Early exit`, `Loss`, `No trades`) by symbol class and regime.

This audit supports evolving the strategy from **“high ADR only”** toward **“high ADR *and* fundamentally attractive”**, allowing the bot to **hold undervalued companies with confidence** rather than repeatedly realizing small, reactive losses.

