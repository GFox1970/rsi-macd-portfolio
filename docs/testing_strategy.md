# Testing Strategy Document

## 1. Objectives
-   Ensure the mathematical correctness of technical indicators (RSI, MACD).
-   Validate the Strategic Judgement Layer (ML Gating + Sentiment).
-   Guarantee 100% order flow integrity (no orphaned orders, correct sizing).
-   Prevent regressions across 3-tier architecture refactors.

## 2. Testing Levels

### 2.1 Unit Testing
-   **Focus**: Isolated functions and utility modules.
-   **Tools**: `pytest`, `unittest.mock`.
-   **Key Areas**: Indicator math, PDT logic (including reversal exceptions), configuration parsing, timestamp normalization.
-   **Enhanced Logic Validation**: Trailing stop-loss triggers, high-water mark persistence, and schema gating failures.
-   **Targets**: `trading_bot/core/*.py`, `ml_pipeline/*.py`.

### 2.2 Integration Testing
-   **Focus**: Component interactions and agent logic.
-   **Phase 3 Tests**: Specifically validate the `TradingAgent.decide()` flow with mocked signals (Mixed, Bullish, Bearish).
-   **Broker Mocks**: Use `MagicMock` to simulate Alpaca/IBKR response payloads without hitting live endpoints.
-   **Hard Gates**: Verify that "Hard Gates" (e.g., VIX > 30) correctly stop orders even if the ML score is high.
-   **Exit Scenarios**: Simulate "Parabolic Moves" (Price > 5% above entry) and retracements (1-2%) to verify trailing profit exits.

### 2.3 End-to-End (E2E) Testing
-   **Focus**: Full system lifecycle in a real-world environment.
-   **Mode**: `test_mode=True` (Dry-run) or Paper Trading.
-   **Validation**: The bot must run through a full 6.5-hour market session, fetch data, evaluate signals, and log outcomes without crashing.

## 3. Tools & Framework
-   **Runner**: `pytest`
-   **Coverage**: `pytest-cov` (Targeting >80% coverage).
-   **Automation**: GitHub Actions (`ci.yml`) runs the full suite on every commit.
-   **Logs Analysis**: Automated assertions against `enhanced_decision_log.jsonl` post-test.

## 4. QA Workflow
1.  **Develop**: Feature implementation + Unit Test.
2.  **Verify**: Run `pytest` locally.
3.  **Validate**: Push to branch, trigger CI pipeline.
4.  **UAT**: Deploy to VM, monitor Paper Trading for 24-48 hours.
5.  **Promote**: Manual sign-off for Live Trading.

## 5. Mocking Strategy
The system uses a "Mock-Heavy" approach for external dependencies:
-   **News/Sentiment**: Mocks return a 4-tuple `(score, sources, themes, forecast)`.
-   **Market Data**: Mocks return pre-defined DataFrames to ensure deterministic indicator results.
-   **Broker**: Mocks track `call_count` and `call_args` to verify that orders are placed with the expected prices and quantities.
