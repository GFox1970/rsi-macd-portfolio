# Intraday profitability implementation checklist

Tracked tasks to improve **real** net P&amp;L for day/intraday trading: simulation realism, execution quality, intraday risk, universe/regime gating, and learning from live fills.

**How to use:** Check items as you complete them. Update dates and owner initials in the [Progress log](#progress-log) section. Revisit [Decision checkpoints](#decision-checkpoints) before scaling size or promoting models.

---

## Principles (non-checklist)

- Prefer **config-driven** behavior (env / `day_trading` / risk config) over hard-coded thresholds.
- Define **success metrics** up front: net P&amp;L after costs, max intraday drawdown, trades/day, slippage vs mid.
- Roll out: **paper → small size → full size** per workstream.

---

## Phase A — Measurement & net-of-costs simulation

Foundation: one canonical intraday sim and honest costs so later changes are measurable.

### A.1 Canonical simulator

- [x] **A.1.1** Choose a single primary intraday backtest/sim path (extend one of: `trading_bot/core/performance_analyzer.py`, `scripts/run_two_week_backtest.py`, or `ml_pipeline/backtesting/` — avoid maintaining three divergent P&amp;L definitions).
- [x] **A.1.2** Document the chosen entrypoint (CLI args, inputs, outputs) in a short comment at top of script or in this file’s Progress log.
- [x] **A.1.3** Add configurable **commission** aligned with live broker assumptions (see `CapitalRiskManager` defaults as reference).

### A.2 Execution model (simulation)

- [x] **A.2.1** Add configurable **half-spread + slippage (bps)** to entry and exit in the canonical sim.
- [x] **A.2.2** Emit **gross vs net** P&amp;L and **fees as % of gross** in the standard report output.

### A.3 Walk-forward validation

- [x] **A.3.1** Implement walk-forward windows (train on A, test on strictly later B; repeat with configurable `--train-days`, `--test-days`, `--step` or equivalent).
- [x] **A.3.2** Export results to CSV + one-line summary (trade count, avg hold, max DD, net P&amp;L).

### A.4 Fill logging (minimal, for Phase B/C calibration)

- [x] **A.4.1** Log on fill: signal reference price (or bar mid), fill price, **slippage bps** (extend existing order/decision logging or `orders_dashboard` pipeline).
- [ ] **A.4.2** Optional weekly aggregation script or notebook query for slippage by symbol and time-of-day.

---

## Phase B — Execution quality

Tighten **when** and **how** orders are placed; measure real execution against assumptions.

**Config snippet (`day_trading` section):** spread uses Alpaca **SIP** latest quote when `ALPACA_*` keys are set; if no quote, optional last-bar `(High-Low)/Close` proxy. Session guard applies only to **US symbols** (no exchange suffix). Defaults are off / permissive so behavior is unchanged until you set thresholds.

```yaml
day_trading:
  session_entry_guard:
    enabled: false
    timezone: America/New_York
    skip_first_minutes: 15
    skip_last_minutes: 15
  spread_guard:
    max_spread_pct: 0        # e.g. 0.002 = 0.2% of mid; 0 disables
    fail_closed_no_quote: false
    allow_bar_spread_proxy: true
    enforce_in_test_mode: false
```

### B.1 Spread and liquidity

- [x] **B.1.1** Replace placeholder `_spread_guard` in `trading_bot/day_trading_manager.py` with real L1 bid/ask via **Alpaca data API** when keys are set (IBKR/native NBBO deferred).
- [x] **B.1.2** Add config: `max_spread_pct` (and fail closed: skip trade if quotes missing — document behavior).
- [x] **B.1.3** If L1 unavailable in an environment, document fallback (e.g. sim-only spread from last bar) and gate feature flag.

### B.2 Session rules

- [x] **B.2.1** Add config for **no new entries** in first/last N minutes of regular session (optional separate open vs close).
- [x] **B.2.2** Centralize in one helper invoked from the buy path; log when a session filter blocks an entry.

### B.3 Limit order policy

- [x] **B.3.1** Document intended limit policy (timeout, cancel, max deviation from signal).
- [x] **B.3.2** Minimal v1: **day TIF** on bracket paths + docstring; automated TTL/chase/depeg not implemented (defer to future if needed).

### B.4 Execution analytics

- [ ] **B.4.1** Compare live slippage distribution to sim assumptions from Phase A; **recalibrate** `slippage_bps` / spread model if systematically off.
- [ ] **B.4.2** Optional: per-symbol **slippage decay** input for Workstream D (down-rank poor executors).

---

## Phase C — Intraday-specific risk

Complement `CapitalRiskManager` portfolio caps with day-trading controls.

**Config (`day_trading.intraday_risk`):** all limits are **off** until `enabled: true`. Per-trade cap runs **after** final stop/TP adjustments, immediately before `place_bracket_order`. “Session” drawdown uses the **first broker equity seen each UTC calendar day** vs current equity (not US RTH session open — see open questions). State file: `data/shared/intraday_risk_state.json` (or `SHARED_DIR`).

```yaml
day_trading:
  intraday_risk:
    enabled: false
    max_per_trade_risk_pct_equity: 0.01
    max_concurrent_positions: 0
    max_new_buys_per_day: 0
    session_drawdown_halt_pct: 0
```

### C.1 Per-trade and concurrency

- [x] **C.1.1** **Per-trade risk**: cap $ at risk from entry to stop as % of equity (applied after final stop/TP adjustments in `_process_symbol_buy_path`, not inside `determine_order_quantity`).
- [x] **C.1.2** **Max concurrent day positions** and/or **max new day trades per session** (configurable).
- [x] **C.1.3** Persist or restore concurrency state across restarts if required for correctness (document choice).

### C.2 Intraday drawdown halt

- [x] **C.2.1** Track **session** P&amp;L vs session-start equity (broker or internal).
- [x] **C.2.2** If session P&amp;L below **-X%** equity, **block new buys** until next session (configurable).
- [x] **C.2.3** Reconcile with existing `daily_loss_limit` / `daily_drawdown_limit_pct` — single coherent story in code comments or `CAPITAL_PRINCIPLES.md` pointer.

---

## Phase D — Universe & regime

Fewer names; trade only when conditions match validated backtests.

**Universe** (`day_trading.universe`, `enabled: false` by default): applied in `get_symbols_to_trade()` after the dead-symbol block — CSV path passes the DataFrame for ADR / dollar-volume columns; watchlist paths only get max + whitelist.

**Entry regime gate** (`day_trading.entry_regime_gate`): first check inside `_process_symbol_buy_path` (before loading bars) — blocks **new** longs only; does not affect exits. `SYSTEMIC_CRASH` is still handled earlier in the main loop.

```yaml
day_trading:
  universe:
    enabled: false
    max_symbols: 0
    min_adr_pct: 0
    min_avg_dollar_volume: 0
    whitelist_file: data/symbol_whitelist.txt
  entry_regime_gate:
    enabled: false
    block_regimes: []
    max_vix: null
    min_vix: null
```

### D.1 Candidate universe

- [x] **D.1.1** Formalize **max symbols per day**, min **avg dollar volume** (or liquidity score), optional **whitelist** file.
- [x] **D.1.2** Wire filters into candidate pipeline (same stage that feeds intraday loop).

### D.2 Regime gating

- [x] **D.2.1** Expose **VIX / vol / trend** thresholds in config (reuse signals you already use in ML/backtest where possible).
- [x] **D.2.2** Single **entry gate**: skip new longs when regime = OFF (log reason).

### D.3 Symbol quality (optional)

- [ ] **D.3.1** Down-rank or temporarily exclude symbols with poor recent net P&amp;L or slippage (inputs from Phase A/B analytics).

---

## Phase E — Closed-loop learning from live fills

Train and promote models using realized outcomes, not only idealized labels.

### E.1 Data pipeline

- [x] **E.1.1** Extend `ml_pipeline/shadow_result_tracker.py` / export so training rows prefer **fill-based** entry/exit where available.
- [x] **E.1.2** Label definition: **realized P&amp;L after fees** (or agreed proxy) documented in `ml_operational_lifecycle.md` or training README.

### E.2 Promotion workflow

- [x] **E.2.1** Define **promotion criteria**: shadow/paper for N days, same net metrics as Phase A.
- [x] **E.2.2** Document manual vs automated promotion steps (who flips `ML_MODEL_PATH` or config).

### E.3 Automation

- [x] **E.3.1** Schedule or document cadence for `scripts/replay_recent_decisions.py` (backfill → export → retrain).
- [x] **E.3.2** Smoke-test full chain after deploy (one dry run logged).

---

## Decision checkpoints

| Checkpoint | When | Gate |
| :--- | :--- | :--- |
| **CP1** | End of Phase A | Net edge on walk-forward clears agreed threshold **after** costs; else pause sizing and revisit strategy/filters. |
| **CP2** | After Phase B | Live slippage vs sim assumptions; recalibrate execution model if bias &gt; X bps. |
| **CP3** | After Phase C + D | Review trades/day, concentration, regime OFF frequency; adjust universe and thresholds. |
| **CP4** | Before model promotion | E.2 criteria met on paper/shadow; no regression on CP1-style metrics. |

---

## Key file references

| Area | Files / dirs |
| :--- | :--- |
| Sim / analyzer | `trading_bot/core/performance_analyzer.py`, `trading_bot/core/execution_cost_model.py`, `scripts/run_intraday_backtest.py`, `scripts/run_two_week_backtest.py` (delegates), `ml_pipeline/backtesting/` (legacy / ML-specific) |
| Day trading & spread | `trading_bot/day_trading_manager.py` |
| Risk | `trading_bot/capital_risk_manager.py`, `trading_bot/core/intraday_risk.py`, `trading_bot/day_trading_manager.py` (`intraday_risk_*`), sizing in `trading_bot/core/trading_bot.py` |
| Shadow / ML | `ml_pipeline/shadow_result_tracker.py`, `ml_pipeline/export_shadow_data.py`, `ml_pipeline/training/train_mi_model.py`, `scripts/replay_recent_decisions.py`, `docs/ml_operational_lifecycle.md` §4 |
| Regime / agent | `trading_bot/core/trading_agent.py`, main loop in `trading_bot/core/trading_bot.py` |

---

## Progress log

| Date | Phase | Note (owner initials optional) |
| :--- | :--- | :--- |
| 2026-03-24 | A | Canonical sim: `PerformanceAnalyzer` + `ExecutionCostModel`; CLI `scripts/run_intraday_backtest.py`; walk-forward; fill log fields `signal_reference_price` / `slippage_bps` on `update_order_outcome`. |
| 2026-03-24 | B | `DayTradingManager.entry_session_guard_ok` / `spread_guard_ok` (Alpaca L1 + bar proxy); wired in `TradingBot` BUY path; bracket `place_bracket_order` docstring (day TIF, no auto-chase). |
| 2026-03-24 | C | `intraday_risk` config + `cap_qty_by_per_trade_risk`; `intraday_risk_allows_new_buy` / `record_intraday_buy_submitted`; state JSON; docstring in `is_within_daily_drawdown_limit`. |
| 2026-03-24 | D | `day_trading.universe` filters in `get_symbols_to_trade`; `entry_regime_gate` + `_entry_regime_allows_longs` in buy path; tests `test_universe_regime_gate.py`. |
| 2026-03-24 | E | `ShadowResultTracker.enrich_filled_executions` + `export_fill_training_csv`; backfill tail calls enrich; `replay_recent_decisions` → `train_mi_model.py`; ml_operational_lifecycle §4; `test_shadow_fill_enrich.py`. |
| | | |

---

## Open questions

Track blockers here (data subscriptions, PDT, broker API limits, etc.).

- [ ] **IBKR / non-Alpaca NBBO**: extend `spread_guard_ok` with broker-router snapshot when symbol routes away from Alpaca.
- [ ] **RTH session equity**: intraday drawdown halt currently uses **UTC day** first equity; optional upgrade to US market session open (or per-market).
- [ ] _Add items as discovered_
