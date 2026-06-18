# Broker v2: LSE Opening Momentum v1

**Status:** Active (replaces legacy RSI/MACD/ML/agent stack for LSE entries)

## What changed

The bot now runs a single falsifiable intraday strategy when `strategy.active` is `lse_momentum_v1`:

| Layer | Legacy | Broker v2 |
| :--- | :--- | :--- |
| Universe | 841 symbols | FTSE 100 `.L` (`weekly_analysis/ftse100_lse.txt`) |
| Pre-filter | Structural ADV rank | Session momentum **top 5** |
| Entry signal | RSI + MACD + XGBoost + Gemini CIO | Opening-range breakout + volume ≥ 1.5× |
| Entry window | All session | London **09:00–11:30** |
| Exit | ADR harvest, trailing, ML override | Fixed **−0.8%** stop, **+1.2%** target, time exit **15:30** |
| Sizing | MVC + conviction multipliers | Fixed **£700** notional |
| Max positions | 2 LSE | **1** LSE |

Legacy ML, Sentinel, Healer, and Strategic Agent are **bypassed on the buy path** when v1 is active. Exits use v1 rules for `.L` same-day positions. Force-flat at 16:25 remains via the scalper runner.

## Config (`trading_config.json`)

```json
"strategy": {
  "active": "lse_momentum_v1",
  "lse_momentum_v1": { ... }
}
```

Set `"active": "legacy"` (or any other value) to revert to the hybrid stack.

## Daily ops

| When (UTC) | Action |
| :--- | :--- |
| **08:00** Mon–Fri | `scripts/run_session_momentum_scan.sh` — scans FTSE100, writes top 5 |
| Market hours | `trading-bot-scalper` evaluates ORB breakouts on watchlist |
| **16:25** London | Scalper force-flat |

Optional: pause `run_orchestrator.sh` cron while validating v1 (ML retrain not needed for entries).

## Broker Coach (Phase 1)

Deterministic intraday monitor — **not** Gemini/Sentinel/Healer.

| When (UTC) | Action |
| :--- | :--- |
| ***/15 8–16** Mon–Fri | `scripts/run_broker_coach.sh` — tail v1 logs, tune bounded params |

Coach writes `data/shared/v1_pilot_overrides.json`; `load_settings()` merges overrides on each bot cycle.

**Dashboard:** Control Center → **Broker Coach** panel (metrics, last suggestion, clear overrides).

See [broker_coach.md](broker_coach.md) for rules and bounds.

## Go-live gate

Before disabling paper safeguards:

1. Run walk-forward backtest:
   ```bash
   python scripts/backtest_lse_momentum_v1.py --universe --last-days 90
   ```
2. Require **net P&L > 0**, profit factor **> 1.3**, max drawdown **< 8%** over 60 trading days.
3. Enable `day_trading.intraday_risk.enabled: true` with `session_drawdown_halt_pct: 0.015`.

## Safety guards (v1 live)

| Guard | Config / file | Behaviour |
| :--- | :--- | :--- |
| Legacy LSE buy block | `strategy.lse_momentum_v1` active | Strategic Decision cannot buy `.L` symbols |
| Post-stop buy cooldown | `day_trading.cooldown_minutes_after_stop` (default 60) | No re-buy on same symbol after stop exit |
| Coach legacy leak alert | `broker_coach` cron | Flags `Strategic Decision for *.L: BUY` in scalper logs |

## Key files

| File | Purpose |
| :--- | :--- |
| `trading_bot/strategy/lse_momentum_v1.py` | Entry/exit rules |
| `trading_bot/strategy/buy_guards.py` | Legacy block + stop cooldown helpers |
| `weekly_analysis/ftse100_lse.txt` | Universe |
| `weekly_analysis/session_momentum_scan.py` | Pre-market ranker (FTSE100 when v1 active) |
| `scripts/backtest_lse_momentum_v1.py` | Offline validation |
| `tests/test_lse_momentum_v1.py` | Unit tests |
