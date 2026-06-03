# LSE scalper operations

## Execution model (intraday-only)

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

## Daily maintenance on the VM

| When | Command / cron | Purpose |
| :--- | :--- | :--- |
| After deploy (scalper) | `deploy_compose.sh` → `run_post_deploy_orchestrator` | Refresh top-20 CSV, archiver, ML pipeline |
| Weekdays ~07:00 UTC | `scripts/run_orchestrator.sh` | Same (manual or cron) |
| Daily 06:00 UTC | `scripts/vm_disk_guard.sh` | Truncate large logs; prune `trades_exports` &gt;3d; old `weekly_analysis/*.csv` |

Example cron (deploy user):

```cron
0 7 * * 1-5 /home/deploy/trading-bot/scripts/run_orchestrator.sh >> /home/deploy/trading-bot/logs/orchestrator_auto.log 2>&1
0 6 * * * /home/deploy/trading-bot/scripts/vm_disk_guard.sh >> /home/deploy/trading-bot/logs/disk_guard.log 2>&1
```

## Related docs

- [Alpha Optimizer (Top-20 monitor)](dashboard_alpha_optimizer.md)
- [ML Operational Lifecycle](ml_operational_lifecycle.md) — §4.5 `strategy_plan`
- [Bracket vs bot exit study](investigations/bracket_vs_bot_exit_study_report.md)
