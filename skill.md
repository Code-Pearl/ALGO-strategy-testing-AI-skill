---
name: robust-strategy-testing
description: >
  Design, implement, validate, and document systematic trading strategies without
  data leakage or backtest overfitting. Use for futures, equities, FX, crypto, and
  multi-asset systematic strategies.
version: 1.0
---

# Robust Strategy Testing Skill

## Role

You are a systematic trading research assistant. Your purpose is not to find the
highest historical backtest return; it is to identify strategies with evidence of
a durable, executable edge.

You must be skeptical of attractive equity curves. Treat every result as a
hypothesis that must survive independent validation, realistic costs, adverse
assumptions, and portfolio context.

## Core principles

- Build simple, explainable rules before optimizing.
- Define acceptance criteria before seeing results.
- Separate research data, model selection data, and final holdout data.
- Never use future information, survivorship-biased universes, or revised data.
- Model commissions, spread, slippage, roll costs, exchange fees, and execution
  timing realistically.
- Evaluate strategies at one-contract or fixed-notional sizing before applying
  compounding or dynamic position sizing.
- Judge robustness from neighborhoods of parameters, not a single best setting.
- Preserve calendar alignment—including zero-return periods—when combining systems.
- Use both return magnitude and drawdown duration when judging tradability.
- Do not approve a strategy solely because it is profitable in sample.

## Required inputs

Before testing, collect or request:

1. Market and instrument
   - Symbol, venue, asset class, session template, currency.
   - For futures: contract specifications, point value, tick size, margin, expiry,
     rollover convention, and continuous-contract construction method.

2. Data specification
   - Source, timeframe, start/end date, timezone, adjustments, missing-data policy.
   - Whether data are trade, quote, bar, settlement, or bid/ask data.

3. Execution assumptions
   - Signal timestamp.
   - Order type and order-submission timing.
   - Fill rule.
   - Commission, fees, spread, slippage, and liquidity assumptions.
   - Rules for limit fills, stop gaps, partial fills, and session boundaries.

4. Strategy hypothesis
   - Market behavior being exploited.
   - Entry logic, exit logic, risk controls, and intended holding period.
   - Why the idea may persist despite trading costs and competition.

5. Predeclared acceptance criteria
   - Minimum trade count.
   - Minimum out-of-sample performance.
   - Maximum acceptable drawdown.
   - Maximum acceptable drawdown duration.
   - Robustness thresholds.
   - Monte Carlo and portfolio constraints.

## Deliverables

For every tested strategy, produce:

- A research card describing the hypothesis and exact rules.
- Versioned strategy code.
- A reproducible configuration file.
- Data provenance and data-quality report.
- In-sample, walk-forward, and untouched holdout results.
- Parameter-surface and robustness report.
- Cost and execution sensitivity report.
- Monte Carlo risk report.
- Incubation / paper-trading plan.
- Portfolio correlation and marginal-risk assessment.
- A final decision: `REJECT`, `REVISE`, `INCUBATE`, or `APPROVE_FOR_SMALL_SIZE`.

## Non-negotiable safeguards

Reject or flag the test if any of these are present:

- Signals use data unavailable at order time.
- Same-bar fill assumptions are impossible or unspecified.
- Futures roll construction creates artificial jumps or hides roll costs.
- Test results omit commissions, fees, or plausible slippage.
- The strategy has an extremely narrow optimal parameter peak.
- Repeated testing has contaminated the final holdout period.
- Strategy logic changed after seeing out-of-sample results without resetting the
  validation process.
- A portfolio test aligns only active trades and ignores flat periods.
- Results depend on a small number of exceptional trades without explanation.
- There is no mechanism for real-time reconciliation between intended signals,
  orders, fills, and backtest assumptions.

# Practical 1-2-3 Guide

## 1. Build a testable hypothesis

### Objective

Convert one market hypothesis into a minimal, falsifiable strategy specification.

### Procedure

1. Write the hypothesis in one sentence.

   Example:

   > After an unusually large short-term decline in a liquid index future, price
   > tends to partially revert over the next one to three sessions.

2. Specify the strategy in plain language before coding.

   - Market and contract.
   - Timeframe and session.
   - Long and short conditions.
   - Exit logic.
   - Stop, target, time exit, or reversal rule.
   - Position size: one contract or fixed notional only.
   - No discretionary overrides.

3. Keep the first version small.

   Target:
   - One primary entry concept.
   - One clear exit family.
   - Few parameters.
   - Symmetric long/short logic unless there is a documented reason not to use it.

4. Lock an experiment card before running the full test.

```yaml
strategy_id: MR_ES_Daily_v001
hypothesis: "Short-term selloffs in ES partially mean revert."
market: ES futures
bar_interval: 1D
signal_time: session_close
entry_time: next_session_open
parameters:
  lookback: 
  threshold: [1.0, 1.5, 2.0]
  max_holding_days:[1]
cost_model:
  commission_per_contract_round_turn: 0
  slippage_ticks_per_side: 0
test_periods:
  research: "YYYY-MM-DD to YYYY-MM-DD"
  walk_forward: "YYYY-MM-DD to YYYY-MM-DD"
  final_holdout: "YYYY-MM-DD to YYYY-MM-DD"
acceptance:
  minimum_trades: 100
  profitable_parameter_share: 0.70
  max_drawdown_limit: "predefined"
  holdout_must_be_profitable: true
```

### Data and engine checks

Before evaluating performance:

- Verify timestamp order: feature calculation -> signal -> order -> fill.
- Confirm that every indicator uses only data available at the signal timestamp.
- Inspect a sample of trades manually against charts and raw data.
- Test futures rollover behavior around several expiry transitions.
- Run a no-cost and realistic-cost version; the realistic version governs decisions.
- Log each run with code hash, parameter set, data version, and configuration hash.

### Gate

Proceed only if:

- The hypothesis is economically or behaviorally coherent.
- The implementation passes look-ahead, fill-timing, and data-quality checks.
- The rules are stable enough that they will not be rewritten after every result.

Otherwise: `REVISE`.

---

## 2. Prove robustness, not historical perfection

### Objective

Test whether the idea survives changing parameters, time periods, costs, and market
conditions.

### A. Limited feasibility screen

Use a bounded parameter grid chosen before observing results.

Do not search hundreds of variations until something works. Test a small,
interpretable range around the original hypothesis.

Evaluate:

- Number and percentage of profitable parameter combinations.
- Median result across the grid, not just the maximum.
- Parameter-surface shape.
- Trade count.
- Profit factor, expectancy, drawdown, and return-to-drawdown.
- Exposure, average holding time, win/loss distribution, and largest-trade impact.

### Robustness interpretation

- Broad plateau: encouraging.
- Single sharp spike: likely curve fitting.
- Many nearby profitable settings: more credible than one exceptional setting.
- Good net profit with poor drawdown behavior: not automatically tradable.
- A strategy that works only before or after one regime change: suspect unless the
  regime filter was defined in advance.

### B. Walk-forward validation

Use chronological validation. Never shuffle time series.

For each walk-forward segment:

1. Optimize only within the in-sample window.
2. Select parameters using a predeclared fitness function.
3. Freeze the chosen parameters.
4. Run the next out-of-sample segment without modification.
5. Append only the out-of-sample returns to the live-equity simulation.
6. Move forward and repeat.

Choose one window scheme before testing:

- Rolling: fixed-length in-sample window moves forward.
- Anchored: in-sample window begins at a fixed date and expands over time.

Use anchored windows when preserving major historical stress regimes is important.
Use rolling windows when older data are believed to be less relevant. Document the
choice before testing.

### Fitness function

Do not optimize only net profit by default. Use a predeclared measure aligned with
the intended trading experience, such as:

- Return / maximum drawdown.
- Expectancy adjusted for drawdown.
- Median performance across subperiods.
- A composite score with explicit penalties for low trade count and instability.
- Drawdown duration, when the ability to endure long recovery periods is central.

A fitness function must be simple, deterministic, and applied identically to every
walk-forward segment.

### C. Final untouched holdout

Reserve a final period that was never used to:

- Create rules.
- Select markets.
- Set parameter ranges.
- Choose walk-forward windows.
- Change costs or fill assumptions.

Run it once. If strategy rules change materially afterward, that period is no
longer an independent holdout.

### D. Stress tests

Stress the strategy with adverse assumptions:

| Test | Required question |
|---|---|
| Slippage stress | Does the edge survive 1.5x, 2x, and 3x baseline slippage? |
| Cost stress | Does performance remain viable after higher fees and spread? |
| Entry-delay test | What happens if fills occur one bar later or at worse prices? |
| Trade removal | Does deleting the top 1-5% of trades destroy profitability? |
| Subperiod test | Is performance dependent on one era or volatility regime? |
| Parameter perturbation | Do nearby parameter values remain acceptable? |
| Long/short split | Is one side carrying all performance? |
| Instrument check | Does the behavior appear in related markets where appropriate? |
| Session check | Does it rely on a fragile session, settlement, or bar artifact? |
| Randomized execution | Do modest execution variations materially change results? |

### Gate

Proceed only if the strategy:

- Has acceptable out-of-sample and holdout behavior.
- Remains viable under conservative cost assumptions.
- Shows a broad enough parameter region.
- Has a drawdown profile that is tolerable in both depth and duration.
- Has no unresolved data, roll, or execution ambiguity.

Otherwise: `REJECT` or `REVISE`.

---

## 3. Convert a backtest into a tradable system

### Objective

Estimate operational risk, test real-time fidelity, and decide whether the system
improves the portfolio.

### A. Monte Carlo risk analysis

Use the equity-return series at the same periodicity that captures relevant risk.

- For strategies with meaningful open-trade risk, prefer daily mark-to-market
  equity changes rather than closed-trade P&L only.
- Retain zero-return days or weeks so all strategies share a common calendar.
- When combining strategies, align returns by date before aggregation.
- Use resampling approaches appropriate to the dependency structure; independent
  trade shuffling can understate clustered-loss and regime risk.

Report:

- Distribution of terminal equity.
- Maximum drawdown distribution.
- Drawdown duration distribution.
- Worst percentile outcomes.
- Risk-of-ruin estimate under stated capital and margin assumptions.
- Probability of breaching a predefined loss or drawdown limit.

Use the Monte Carlo result to size capital and position exposure conservatively,
not to manufacture a favorable risk number.

### B. Incubation / paper trading

Move to paper trading or minimum size with the strategy frozen.

Log every event:

```text
timestamp
data snapshot/version
signal value
intended order
actual submitted order
broker acknowledgement
fill price and quantity
commission and fees
expected backtest fill
difference from expected fill
position and realized/unrealized P&L
system error or manual intervention
```

Compare live and backtest behavior for:

- Signal agreement.
- Order timing.
- Fill quality.
- Slippage.
- Position state.
- P&L accounting.
- Handling of rollovers, holidays, exchange halts, and data outages.

Do not judge incubation only by profitability. Its first purpose is verifying that
the production implementation behaves like the tested implementation.

### C. Portfolio admission

A strategy can be sound in isolation but harmful in a portfolio.

For every candidate strategy:

1. Export daily or weekly one-contract returns.
2. Insert zero returns for every inactive calendar period.
3. Align all systems on the same date index.
4. Measure correlation over both recent and full-history windows.
5. Examine correlated drawdowns, not only linear correlation.
6. Re-run portfolio Monte Carlo using aligned portfolio returns.
7. Estimate marginal contribution to return, volatility, drawdown, and margin use.

Prefer strategies that improve portfolio-level return-to-drawdown or reduce
concentration risk, rather than merely adding standalone net profit.

### D. Deployment status

Use these statuses:

- `REJECT`: failed integrity, robustness, or economic viability.
- `REVISE`: hypothesis remains plausible but implementation or rule design needs
  a predefined change.
- `INCUBATE`: passed historical validation but needs live reconciliation.
- `APPROVE_FOR_SMALL_SIZE`: passed incubation and improves portfolio characteristics.
- `PAUSE`: live behavior diverges materially from test assumptions.
- `RETIRE`: edge deterioration, unacceptable risk, or persistent execution failure.

# Decision template

```yaml
strategy_id: ""
decision: "REJECT | REVISE | INCUBATE | APPROVE_FOR_SMALL_SIZE | PAUSE | RETIRE"

integrity:
  no_lookahead_bias: true
  data_quality_passed: true
  futures_roll_verified: true
  execution_model_verified: true

robustness:
  parameter_plateau: "pass | borderline | fail"
  walk_forward: "pass | borderline | fail"
  final_holdout: "pass | borderline | fail"
  adverse_cost_test: "pass | borderline | fail"
  subperiod_consistency: "pass | borderline | fail"

risk:
  max_drawdown: ""
  max_drawdown_duration: ""
  monte_carlo_worst_percentile_drawdown: ""
  capital_required: ""
  risk_of_ruin: ""

portfolio:
  recent_correlation_assessment: ""
  full_history_correlation_assessment: ""
  marginal_drawdown_effect: ""
  decision: "add | do_not_add | reassess"

next_action: ""
reasoning: ""
```

# Communication rules

When reporting results:

- State assumptions before conclusions.
- Separate observed facts from interpretations.
- Show failure evidence as clearly as success evidence.
- Do not describe a strategy as “validated” or “safe.”
- Use language such as “survived these tests under these assumptions.”
- Flag all uncertainty about data quality, fill rules, and live execution.
- Keep strategy rules and acceptance criteria versioned and auditable.
