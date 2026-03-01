# Capital Management & Reinvestment Principles

This document outlines the "Darwinian Scaling" model for the trading bot, ensuring it remains viable at small scales (UK ISA / Alpaca Seed) and earns the right to manage larger capital.

## 1. Initial Scale (The "Seed" Phase)
- **UK (IBKR ISA)**: Initial capital £1,000 - £2,000. Regulated by a £20,000 annual ISA limit.
- **US (Alpaca)**: Initial capital $1,200 - $2,400.
- **Goal**: Demonstrate resilience and fee-efficiency before scaling.

## 2. Performance-Based Reinvestment (PBR)
Additional capital is not "given"; it is "earned."
- **Matching Rule**: Profitability is rewarded with fresh capital injections.
- **Scaling Tiers**:
    - **Tier 1 (Recovery)**: If win rate < 30%, capital is held static. Focus on "Sentinel Audits."
    - **Tier 2 (Stability)**: Win rate 30-50%. Incremental capital matching (e.g., 20% of base).
    - **Tier 3 (Growth)**: Win rate > 50% + positive ROI. Aggressive capital matching (up to 60% of base).

## 3. Scale-Aware Fee Optimization
Broker fees represent a significant percentage of small portfolios. The bot must adapt its tactics based on current capital.

| Portfolio Size | Strategy Type | Fee Tolerance | Min Notional |
| :--- | :--- | :--- | :--- |
| < $5,000 | **Sniper** (Low freq, high conviction) | < 2% of expected move | ~$300 - $500 |
| $5k - $50k | **Tactical** (Medium freq) | < 5% of expected move | ~$150 - $300 |
| > $50k | **Beast** (High volume allowed) | Standard | Standard |

## 4. Operational Guardrails
- **Fee Toxicity Gate**: Orders where fees > 10% of the statistical expected move (ADR) are automatically blocked.
- **Sector Concentration Gate**: No single sector (e.g., Technology, Energy) can exceed 30% of total portfolio exposure.
- **Concentration Risk**: At the "Seed" phase, the bot is restricted to a maximum of 3-5 simultaneous open positions to avoid spreading capital too thin across multiple fee-paying entries.
