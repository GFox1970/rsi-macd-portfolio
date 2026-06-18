# Candidate Selection Approaches

The bot supports multiple competing approaches for identifying which LSE symbols to trade each session. Each approach tags its decisions in `meta.approach` inside `enhanced_decision_log.jsonl` so the nightly ML pipeline can evaluate them against each other.

---

## Approach 1 — Structural (historical ADV/ADR)

**Script:** `weekly_analysis/find_top_day_trading_candidates.py`  
**Output:** `weekly_analysis/top_candidates/top_20_day_trading_candidates.csv`  
**When it runs:** Nightly via `daily_orchestrator.py` (post-market or pre-market)  
**`meta.approach` tag:** `"structural"`

Ranks all 841 symbols in `all_symbols.txt` by a composite score of:

```
composite_score = consistency × avg_daily_range_pct × log10(ADV_250)
```

Lookback: 250 trading days (≈1 year).  Strengths: liquidity-safe, stable. Weakness: slow to pick up short-term momentum — may miss names that are moving *right now*.

---

## Approach 2 — Session Momentum Scan ⭐ new

**Script:** `weekly_analysis/session_momentum_scan.py`  
**Output:** `data/session_momentum_top20.csv`  
**When it runs:** At LSE open each trading day (08:00 UTC cron or `scripts/run_session_momentum_scan.sh`)  
**`meta.approach` tag:** `"session_momentum"`

Scans the 100 `.L` symbols in real-time using the first 12 × 5-minute bars of the session and ranks by a three-factor momentum score:

| Factor | Metric | Weight |
|---|---|---|
| Volume spike | today's session vol ÷ 20-day avg (normalised cap 3×) | 33% |
| ADR consumed | intraday range ÷ ADR (normalised cap 1.5×) | 33% |
| RSI direction | distance from 50 (oversold/overbought signal) | 33% |

A symbol must clear a minimum **volume spike ≥ 1.2×** and **ADR ≥ 0.5%** to qualify.

The bot's watchlist loader (Priority 3a in `get_symbols_to_trade`) prefers this CSV over the structural CSV when:
- The file is ≤ 4 hours old, **and**
- LSE intraday-only mode is active

### Configuration (`trading_config.json` → `day_trading.session_momentum_scan`)

```json
{
  "top_n": 20,
  "volume_spike_min": 1.2,
  "min_adr_pct": 0.005,
  "lookback_bars": 12,
  "adr_lookback_days": 20,
  "max_workers": 6,
  "sell_harvest_fraction": 0.75,
  "buy_buffer_pct": 0.003
}
```

### VM cron setup

Add to the `deploy` user's crontab (`crontab -e`):

```cron
# Run session momentum scan at 08:00 UTC Mon–Fri (before LSE open at 08:00)
0 8 * * 1-5  /home/deploy/trading-bot/scripts/run_session_momentum_scan.sh >> /home/deploy/trading-bot/logs/cron.log 2>&1
```

---

## Approach 3 — Catalyst + Liquidity (future)

**Script:** TBD  
**`meta.approach` tag:** `"catalyst"`

Pre-market filtering based on RNS news announcements, gap magnitude (today open vs. yesterday close), and a minimum ADV floor. Not yet implemented — reserved for future competition.

---

## Competition Scoring

**Script:** `ml_pipeline/approach_competition.py`  
**Output:** `data/approach_competition_scores.csv`, `data/approach_competition_summary.json`  
**When it runs:** Nightly via `daily_orchestrator.py` (after `export_shadow_data`)

Reads the last 30 days of `enhanced_decision_log.jsonl`, groups by `meta.approach`, and computes:

- `picks_total` — total BUY decisions tagged to this approach
- `fill_rate` — what fraction of attempted orders were filled
- `mean_shadow_pnl_1h` — mean P&L per mille 1h after signal
- `harvest_hit_rate` — fraction of decisions where price ≥ `sell_price_target` within 1h
- `win_rate` — fraction with positive 1h P&L

The **leader** (highest `harvest_hit_rate`) is written to `data/approach_competition_summary.json`.

### Dashboard integration

The Alpha Optimizer tab reads `approach_competition_summary.json` to display a live scoreboard. The bot does **not** auto-promote the leader — that decision remains manual. The competition data feeds into the nightly ML training as additional feature context.

---

## Decision Log `meta` schema (extended)

```jsonc
"meta": {
  "mode": "live",            // "live" | "test" | "shadow"
  "approach": "session_momentum",  // approach tag (see above)
  "agent_confidence": 0.73,
  "is_stale": false,
  "tags": []
}
```

Legacy entries without `meta.approach` are treated as `"structural"` by the competition scorer.
