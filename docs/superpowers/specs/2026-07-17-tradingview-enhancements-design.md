# TradingView Indicator Stabilization and Enhancement Design

Date: 2026-07-17  
Status: Approved design, pending implementation plan

## 1. Objective

Stabilize and complete the two Pine Script v6 indicators in this workspace:

- `Luxy BIG beautiful Dynamic ORB - v5.pine`
- `Trend Duration Forecast`

The work will correct existing defects first, then implement all requested enhancements without converting either script into a strategy. Luxy trade statistics will remain an explicitly synthetic, bar-based simulation rather than broker-emulator results.

## 2. Guiding Decisions

1. Preserve both scripts as `indicator()` overlays and keep existing user-facing behavior available.
2. New features must be independently switchable. Disabling them must preserve the corrected baseline behavior.
3. Higher-timeframe filters use confirmed HTF bars to prevent reload repainting.
4. All trade lifecycle decisions happen on confirmed chart bars.
5. If a chart candle touches both stop and target and intrabar order is unknowable, the stop wins. This conservative policy is documented in tooltips.
6. A trailing stop activates after TP1 is confirmed and becomes effective on the next bar, avoiding impossible same-bar sequencing assumptions.
7. Only one simulated Luxy trade can be active at a time. New signals may still be annotated but cannot overwrite active trade state.
8. Previous-day values use completed daily bars and remain stable throughout the current day.
9. The ChartPrime/MPL-2.0 notice remains in the Trend Duration source. Any reused Trend Duration logic in Luxy will retain appropriate attribution.
10. Luxy ORB signals are supported only where their construction is accurate. Charts above five minutes receive a visible warning and do not claim accurate 5-minute ORB behavior.
11. Table dimensions, drawing retention, request counts, and loop bounds are explicit and bounded rather than left to garbage collection.

## 3. Scope

### 3.1 Luxy ORB stabilization

Correct these existing issues before enabling new filters:

- Fix the malformed escape in the session-statistics tooltip.
- Decouple dashboard rendering from `enableHTF`.
- Move HTF calculation before breakout/GOD scoring and make it independent of dashboard visibility.
- Make stock regular-session behavior exclude extended bars when extended hours are disabled.
- Replace bar-local futures auto-detection with stable session behavior.
- Prevent active trade objects and state from being overwritten by later breakouts.
- Define deterministic behavior when long and short entries become pending together.
- Assign breakout label IDs so failed-break relabeling works.
- Align target labels and session R accounting with actual entry-to-target risk distances.
- Track the most recent direction instead of treating session-long `everHadBreak*` flags as current direction.
- Show an explicit timeframe warning and suppress inaccurate intraday ORB signals above five-minute charts.
- Preserve dynamic-alert behavior and ensure alert events reset per bar.
- Resize the Luxy dashboard from 2×28 to 2×40 cells. The maximum all-features path must remain below row 40; the implementation will assert the row budget in comments and test the maximum configuration.

Session behavior is deterministic:

- Stocks, funds, indexes, ETFs, and depositary receipts use `session.ismarket` for regular hours. Enabling extended hours uses the configured premarket/after-hours windows instead.
- Crypto and forex use their loaded continuous/native sessions unless the user selects Custom.
- Custom mode uses the configured session string and symbol timezone.
- Futures `RTH Only` uses the configured RTH window; `Electronic Full Day` uses the electronic session. The existing `Auto-Detect` option is retained for compatibility but deliberately maps to safe RTH behavior because a stable historical inference cannot be made from bar-local volume. Its tooltip will disclose this.
- A trade closing at session end uses a cached last confirmed in-session close, never the close of the first out-of-session bar.
- Session reset occurs on the confirmed `false → true` in-session transition.

### 3.2 Luxy requested enhancements

#### RSI divergence

- Use confirmed price pivots with configurable left/right strength and a maximum pivot separation.
- Bearish divergence: price higher high with RSI lower high.
- Bullish divergence: price lower low with RSI higher low.
- A divergence is valid for filtering only if its pivot was already confirmed on or before the breakout bar and its confirmation is within the configured lookback window. Later pivot confirmation never cancels or retroactively changes a breakout.
- Modes: `Off`, `Label Only`, and `Filter Breakouts`.
- Labels explicitly indicate that pivot divergence is confirmed after the right-side pivot delay.
- `Label Only` follows the same no-retroactivity rule; it annotates the breakout using information available on that bar.

#### Session VWAP deviation bands

- Calculate VWAP and volume-weighted variance from the selected Luxy session anchor.
- Reset accumulators at each Luxy session start.
- Plot VWAP, ±1σ, and ±2σ using existing band multipliers and colors.
- Bands are display/confluence data by default and do not block breakouts.
- Dashboard shows whether price is inside, above, or below the outer bands.
- Use `hlc3` as the source and population variance: `sumV += volume`, `sumPV += hlc3 × volume`, `sumPV2 += hlc3² × volume`, `VWAP = sumPV / sumV`, and `variance = max(sumPV2 / sumV - VWAP², 0)`.
- With zero cumulative volume, all VWAP values are `na`; the dashboard shows `N/A`. Plots are `na` outside the selected session and never carry stale bands into the next session.
- Hand-check case: equal volumes at prices 10 and 12 produce VWAP 11 and σ 1.

#### Multi-timeframe momentum score

- Calculate directional RSI, MACD histogram, and Stochastic components in each requested context.
- Combine chart timeframe plus two user-selected higher timeframes.
- Calculate the full component tuple inside each `request.security()` context.
- Use the last completed HTF bar via expression offset `[1]` with `barmerge.lookahead_on`. Validate that both requested timeframes exceed the chart timeframe.
- Produce bullish and bearish scores on a 0–100 scale plus grades A–F.
- Show the direction-appropriate score on breakout labels and the latest score in the dashboard.
- Optional minimum-score filtering is separate from display enablement.
- Per context, use RSI directly on 0–100, Stochastic %K directly on 0–100, and normalize MACD histogram as `clamp(50 + 50 × hist / max(3 × EMA(abs(hist), 20), syminfo.mintick), 0, 100)`.
- Context bullish score is the equal-weight mean of the three components; bearish score is `100 - bullish`.
- Composite weighting is 50% chart timeframe, 30% HTF1, and 20% HTF2.
- Grades use the existing promised bands: A for `≥75`, B for `≥60`, C for `≥45`, D for `≥30`, and F below 30.
- Until every enabled context has valid data, display `N/A` and do not reject a breakout on the momentum filter.

#### Trailing stop

- Detect the TP1 milestone independently of `showTP1`; display visibility never controls trade logic. Activate only after confirmed TP1 and no earlier than the next chart bar.
- Support ATR, Chandelier, and percentage methods.
- Long stops can only rise; short stops can only fall.
- Chandelier uses a bounded configurable lookback.
- Update the existing stop line/label rather than creating unbounded new objects.
- Fire one trailing-stop alert when the trailed stop closes a trade.
- Preserve the original stop when the calculated trail would loosen risk.
- TP1 is an activation milestone only; no partial exit is booked. A trailing exit realizes `(exit - entry) / initialRisk` for a long and `(entry - exit) / initialRisk` for a short.
- On every bar, test the previously effective stop before processing targets. Only afterward calculate a ratcheted trail from the current bar's high/low, and make that new trail effective on the following bar. Thus the current bar can never stop against a level derived from its own unknown intrabar path.

#### Win/loss streak and drawdown tracker

- Update statistics only once in the central trade-finalization block.
- Track current and maximum win/loss streaks for the active Luxy session.
- Treat zero-R exits as breakeven: they reset both current streaks and count as neither win nor loss.
- Track cumulative realized R, peak cumulative R, current drawdown, and maximum session drawdown in R.
- Clearly label these as synthetic session statistics.
- Use the same realized R value for result labels, streaks, drawdown, and dashboard totals.

#### Previous-day levels

- Request previous daily high, low, and close as one confirmed tuple.
- Plot PDH, PDL, and PDC with independent visibility toggles.
- Reuse the same tuple for GOD confluence and gap calculations.
- Preserve the existing Dashed/Dotted/Solid style input with three bounded line objects. Create/update at most one PDH, one PDL, and one PDC line for the active session, delete the prior session's objects, and extend them right. Global plots are not used because they cannot preserve dash styles.

#### Trend Duration integration

- Port only the corrected HMA direction, duration, average-duration, confidence, and exhaustion primitives.
- Do not copy Trend Duration's independent tables into Luxy.
- Modes: `Info Only` and `Filter Breakouts`.
- Filter mode requires HMA direction to match breakout direction.
- Dashboard row shows HMA direction, current duration, expected duration, confidence, and exhaustion state.
- HMA integration remains chart-timeframe by default; MTF momentum supplies the separate MTF confirmation layer.

### 3.3 Trend Duration stabilization and enhancements

#### Trend state and duration accounting

- Replace `bool(na)` initialization with an explicit direction state: `-1`, `0`, or `+1`.
- Do not count duration until the first confirmed slope direction exists.
- On reversal, save the completed trend without assigning the reversal bar to it; start the new trend at one bar.
- Trim history immediately after push and recalculate averages afterward.

#### ADX strength meter

- Keep DMI/ADX direction and strength display.
- Add safe input bounds.
- Make strength thresholds configurable or central constants with documented defaults.
- Clear stale table cells when display options are disabled.

#### Forecast confidence percentage

- Continue using coefficient of variation as the consistency input.
- Derive display confidence as `clamp(100 - CV, 0, 100)`.
- Display both numerical confidence and a categorical grade.
- Require at least two completed samples; otherwise display `N/A`.
- Describe confidence as historical-duration consistency, not a probability of future price movement.
- Confidence grades are High for `≥80`, Medium for `≥60`, Low for `≥40`, and Very Low below 40.

#### Multi-timeframe confirmation

- Compute HMA direction inside each requested timeframe.
- Use confirmed higher-timeframe bars through expression `[1]` plus `barmerge.lookahead_on`.
- Validate requested timeframes and show a configuration warning instead of silently using lower-timeframe data.
- Calculate chart direction before alignment, eliminating one-bar stale reversal status.

#### Exhaustion warning

- Evaluate warning and danger thresholds from the completed directional sample average.
- Enforce positive multipliers and `danger > warning`.
- Trigger warning/danger only once on confirmed threshold crossings.
- Escalating from warning to danger creates one danger event without duplicating the warning.

#### Progress bar

- Guard zero/full loop counts so the bar always contains exactly ten blocks.
- Cap the visual percentage at 100% while also showing the uncapped numeric duration percentage.
- Display `N/A` until an average exists.

#### Support/resistance zones

- Replace transition-HMA dotted lines with bounded pivot-based zones.
- Use configurable pivot strength with default 3 and bounds 1–10, plus a maximum pivot age with default 100 and bounds 10–500 bars.
- On bullish-to-bearish transition, create resistance from the most recent confirmed pivot high within the age limit. On bearish-to-bullish transition, create support from the most recent confirmed pivot low.
- Build a box with half-width `ATR(14) × 0.15`, extend it right, and label support/resistance.
- Keep a configurable maximum of 10 zones with bounds 1–25; delete the oldest box and label together.
- A confirmed close above a resistance box top or below a support box bottom breaks the zone. The user selects `Fade` or `Delete`; Fade stops extension and raises transparency, while Delete removes the paired objects.

#### Alerts

- Provide separate alert conditions for bullish start, bearish start, bullish-to-bearish reversal, bearish-to-bullish reversal, warning exhaustion, and danger exhaustion. Start means only `0 → ±1`; reversal means only `+1 ↔ -1`, so one transition cannot trigger both event classes.
- All alerts use confirmed-bar event booleans and meaningful TradingView placeholders where supported.

#### Resource and runtime safety

- Bound the sample input to 2–20 and replace the 100×100 table with 3×23 cells.
- Render bullish and bearish sample arrays independently and never index one using the other's length.
- Bound sample input and all TA lengths.
- Cap `bar_index` forecast projection to TradingView's 500-bar future limit.
- Avoid loops whose end value is negative; use explicit guards or `na` endpoints.
- Make Trend table positions configurable and default the progress table to Bottom Center, eliminating its current default collision with Luxy's Bottom Left dashboard. Existing users can select the old position.
- Bound Luxy divergence labels to 50, Trend S/R zones to 25, and PD lines to three. Keep Luxy at no more than 10 unique `request.*()` contexts and Trend at no more than three.

## 4. Component Boundaries and Data Flow

Both scripts remain standalone because Pine indicators cannot read each other's internal state.

Luxy execution order:

1. Inputs and validation.
2. Session state and reset event.
3. Cached chart/HTF calculations.
4. ORB construction and active-range selection.
5. RSI, VWAP, momentum, and HMA feature calculations.
6. Breakout qualification and label/alert creation.
7. Pending-entry resolution with a single-active-trade guard.
8. Confirmed TP, trailing-stop, fixed-stop, and EOD handling.
9. Central trade finalization and statistics.
10. Plots, dashboard, and global alert conditions.

Trend Duration execution order:

1. Inputs and validation.
2. Confirmed chart and HTF direction calculations.
3. Transition event and duration history update.
4. Averages, confidence, and exhaustion state.
5. Forecast/S/R drawings and bounded cleanup.
6. Tables and plots.
7. Confirmed event alerts.

## 5. Failure Handling and User Feedback

- Invalid timeframe selections show a warning table/label and disable only the affected MTF filter.
- Missing volume disables volume-weighted features gracefully and displays `N/A` rather than blocking all signals.
- Insufficient pivot or duration history produces no divergence/confidence decision.
- Unsupported Luxy chart timeframes display a clear warning and suppress ORB signals instead of emitting inaccurate ranges.
- Empty arrays are never indexed.
- All optional tables clear stale cells when turned off.
- Runtime objects are bounded and deleted in matched groups.
- Luxy's dashboard shows a configuration warning within its 40-row budget rather than creating a fourth standalone table.

## 6. Verification Strategy

### Static checks

- Preserve UTF-8/LF encoding and MPL attribution.
- Scan for conflict markers, malformed escapes, unused enhancement inputs/state, unbounded object arrays, and unsafe negative loop bounds.
- Review the complete diff and confirm no unrelated files changed.
- Recompute source hashes after each implementation phase for traceability.

### TradingView compile and runtime checks

- Compile both scripts independently as Pine v6 with zero errors.
- Add both indicators together and confirm dashboard/table positions do not overlap by default.
- Test stock RTH and extended sessions, CME RTH/electronic modes, crypto custom sessions, and a DST boundary.
- Test Luxy on 1-minute and 5-minute charts; verify the explicit warning/suppression above five minutes.
- Exercise every ORB stage combination and all new feature toggles.
- Test long/short lifecycle, TP1 activation, each trail mode, same-bar TP/SL, EOD exit, and a new signal during an active trade.
- Compare live/replay/reload behavior at higher-timeframe boundaries for non-repainting confirmation.
- Test Trend Duration with empty, one-sample, two-sample, and full histories; initial rising/falling/flat data; 0%, 100%, and >100% progress; and more transitions than the S/R zone limit.
- Confirm each alert fires once on its documented event.
- Inspect Pine Profiler output on long histories and multiple symbol types.
- Record baseline and all-features Profiler totals. The all-features configuration must remain below twice the corrected baseline runtime and must not hit TradingView's execution timeout; any block consuming over 25% of total runtime is reviewed for avoidable repeated work.
- Verify RSI divergence when the second pivot is confirmed before, exactly on, and after a breakout; only the first two may affect that breakout.
- Verify TP1 logic with its plot hidden, trailing-exit R arithmetic, and prior-stop-before-new-trail sequencing.
- Verify a `0 → direction` event emits only a start alert and `+1 ↔ -1` emits only a reversal alert.
- Verify the Luxy dashboard with every optional row enabled stays within its 2×40 allocation.

## 7. Acceptance Criteria

- Both scripts compile and run in TradingView Pine v6 without runtime errors in the validation matrix.
- Every requested enhancement has working calculations and visible output; no feature is only an unused input or variable.
- Confirmed HTF features do not change after reload.
- Luxy produces one coherent active trade lifecycle and consistent realized-R statistics.
- Trend Duration begins counting only after a real trend is established and never performs unsafe array/table access.
- Optional features can be disabled without leaving stale objects or table content.
- Resource usage stays within TradingView limits.
- Luxy uses at most 40 dashboard rows, 50 retained divergence labels, three PD lines, and ten unique request contexts; Trend uses a 3×23 history table, at most 25 zones, and three unique request contexts.
- Documentation/tooltips clearly disclose synthetic statistics, pivot confirmation delay, confirmed-HTF latency, and unsupported Luxy timeframes.

## 8. Out of Scope

- Converting Luxy into a `strategy()` or claiming broker-accurate backtest results.
- Modeling commissions, slippage, partial fills, or tick-level sequencing.
- Supporting lower-timeframe reconstruction for Luxy on chart timeframes above five minutes.
- Sharing runtime state directly between the two indicators.
- Rewriting unrelated visual styling or renaming the user-owned source files.
