# Ghostbuster Protocol (v1.1.5)

The **Ghostbuster Protocol** is a critical safety layer designed to eliminate "phantom" P&L anomalies caused by stale broker data, missing cost basis, or currency mismatches (e.g., SLF/ORA issues).

## Core Mechanisms

### 1. Session Strictness (Today-Only Boundary)
- **Rule**: Stale execution data from previous calendar days is instantly discarded.
- **Implementation**: The system verifies the `UTC` calendar day for every fill. Any fill or report not matching the current session's date is suppressed.
- **Goal**: Prevents "resetting" broker sessions from injecting old trade segments into the current P&L calculation.

### 2. Zero-Cost Shield (Anchor Recovery)
- **Rule**: If a closing trade is detected without a matching opening trade in the current session, it is assumed to have **Zero Realized P&L**.
- **Implementation**: 
    - The system identifies "missing segments" (e.g., selling 100 shares when only 0 were bought today).
    - It deduces a "Ghost" entry price equal to the exit price.
    - **Avg Entry == Avg Exit** -> **$0.00 P&L**.
- **Goal**: Reconciles "unknown" historical trades gracefully without creating phantom profit spikes.

### 3. Anomaly Shield (Variance Gating)
- **Rule**: Broker-reported realized P&L is only trusted if the deduced entry price is mathematically plausible.
- **Implementation**: 
    - The system reconstructs the entry price from the broker's realized P&L report.
    - If `abs(deduced_entry - current_price) / current_price > 0.25`, the report is rejected.
- **Goal**: Blocks high-variance "Ghost profit" reports ($5k+ anomalies) that arise during broker maintenance or data glitches.

### 4. Cash-Only Guard (Margin Shield)
- **Rule**: International trades must be fully funded by settled cash in the specific broker sub-account.
- **Implementation**: 
    - Verifies **IBKR-specific settled cash** + 1.5x commission buffer.
    - Ignores cash in other currency buckets or accounts to prevent unintended margin borrowing.
- **Goal**: Shields the portfolio from margin debt and interest charges.

## Audit Trail
All Ghostbuster actions (reconciliations, report rejections, and cash-blocks) are logged to:
- `logs/ibkr_fills.jsonl`
- `logs/enhanced_decision_log.jsonl`
- `logs/trading_bot.log` (Strategic Focus entries)
