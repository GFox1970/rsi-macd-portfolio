# LSE scalper operations

## Execution model (intraday-only)

When `strategy.active` is `lse_momentum_v1` (Broker v2), LSE entries use **LSE Momentum v1** only — not the legacy Strategic Judgement / ML / agent stack. See [broker_v2_lse_momentum.md](broker_v2_lse_momentum.md).

When `day_trading.lse_scalping.mode` is `intraday_only`:

| Mechanism | Scalper behaviour |
| :--- | :--- |
| **IBKR entry** | Limit buy only — no broker TP/SL legs (`use_bot_managed_exits` in `trading_bot/scalper/mode.py`) |
| **Exits** | Same-day `ExitEvaluator` + `force_flat` + LSE sell helpers |
| **Runner partial exit** | Skipped for `.L` |
| **StagedExit** | Skipped for `.L` |
| **Portfolio breakeven TP** | Skipped for `.L` |
| **Track B / holdable profiles** | Disabled via config |

Planned TP/SL are still written to `data/bracket_targets.json` and `strategy_plan` on the decision log for dashboard and ML.

## Safety guards (v1 live)

| Guard | Config | Behaviour |
| :--- | :--- | :--- |
| Legacy LSE buy block | `strategy.lse_momentum_v1` active | `Strategic Decision` cannot buy `.L` symbols |
| Post-stop buy cooldown | `day_trading.cooldown_minutes_after_stop` (default 60) | No re-buy on same symbol after stop exit; state in `data/shared/stop_buy_cooldown.json` |
| Coach legacy leak alert | `broker_coach` cron | `alert_legacy_leak` if scalper logs show legacy `.L` BUY while v1 active |

## Broker Coach cron

| When (UTC) | Script | Purpose |
| :--- | :--- | :--- |
| ***/15 8–16 Mon–Fri** | `scripts/run_broker_coach.sh` | Parse scalper docker logs, refresh stale momentum, bounded pilot overrides |

Runs on the **VM host** (needs `docker` CLI). Audit: `logs/broker_coach.jsonl`. See [broker_coach.md](broker_coach.md).

## Candidate selection

The bot now runs two competing approaches (see [candidate_selection_approaches.md](candidate_selection_approaches.md)):

| Approach | Script | Output | Active when |
| :--- | :--- | :--- | :--- |
| **Session Momentum** (Approach 2) | `weekly_analysis/session_momentum_scan.py` | `data/session_momentum_top20.csv` | File ≤ 4h old during LSE hours |
| **Structural** (Approach 1) | `weekly_analysis/find_top_day_trading_candidates.py` | `weekly_analysis/top_candidates/top_20_day_trading_candidates.csv` | Fallback when momentum CSV is stale |

The bot's watchlist loader (Priority 3a) automatically prefers the momentum CSV when fresh.

## Daily maintenance on the VM

| When (UTC) | Command / cron | Purpose |
| :--- | :--- | :--- |
| **08:00 Mon–Fri** | `scripts/run_session_momentum_scan.sh` | Refresh momentum watchlist before LSE open |
| After deploy (scalper) | `deploy_compose.sh` → post-deploy block | Also runs session scan + orchestrator (structural + ML) |
| Weekdays ~07:00 | `scripts/run_orchestrator.sh` | Structural top-20 refresh, archiver, ML pipeline |
| Daily 06:00 | `scripts/vm_disk_guard.sh` | Truncate large logs; prune `trades_exports` >3d; old `weekly_analysis/*.csv` |

Example cron (deploy user):

```cron
# Session momentum scan — must run BEFORE LSE open (08:00 UTC)
0 8 * * 1-5 /home/deploy/trading-bot/scripts/run_session_momentum_scan.sh >> /home/deploy/trading-bot/logs/session_momentum_scan.log 2>&1

# Nightly orchestrator (structural discovery + ML + archiver)
0 7 * * 1-5 /home/deploy/trading-bot/scripts/run_orchestrator.sh >> /home/deploy/trading-bot/logs/orchestrator_auto.log 2>&1

# Disk guard
0 6 * * * /home/deploy/trading-bot/scripts/vm_disk_guard.sh >> /home/deploy/trading-bot/logs/disk_guard.log 2>&1
```

## Related docs

- [Alpha Optimizer (Top-20 monitor)](dashboard_alpha_optimizer.md)
- [ML Operational Lifecycle](ml_operational_lifecycle.md) — §4.5 `strategy_plan`
- [Bracket vs bot exit study](investigations/bracket_vs_bot_exit_study_report.md)
