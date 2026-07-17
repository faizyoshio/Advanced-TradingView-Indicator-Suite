# Luxy ORB Stabilization and Enhancements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn Luxy's unused enhancement scaffold into a deterministic Pine v6 ORB indicator with non-repainting confluence filters, coherent synthetic trade management, bounded resources, and accurate session statistics.

**Architecture:** Keep the existing `ORBData` state machine and large standalone indicator, but impose a strict execution order: session/reset, cached calculations, ORB build, feature calculations, breakout qualification, one-trade lifecycle, final statistics, then visuals/alerts. Reuse one confirmed daily tuple and completed-HTF requests, update existing drawing IDs instead of creating per-bar objects, and gate unsupported chart timeframes explicitly.

**Tech Stack:** TradingView Pine Script v6, PowerShell static source checks from the Trend plan, TradingView Pine Editor/Bar Replay/Profiler.

## Global Constraints

- Execute `2026-07-17-trend-duration-enhancements.md` first; Luxy ports its corrected HMA primitives.
- Keep Luxy as an `indicator()` overlay; synthetic statistics are not broker-emulator backtests.
- Preserve the disclaimer/credits and add ChartPrime/MPL attribution for ported Trend Duration primitives.
- HTF and previous-day values use expression `[1]` with `barmerge.lookahead_on`.
- All breakout and trade transitions occur on confirmed chart bars.
- One synthetic trade may be active; new signals cannot overwrite it.
- A previously effective stop is tested before targets; a newly ratcheted trail becomes effective on the next bar.
- TP1 activation is independent of TP1 visibility and books no partial exit.
- Same-bar stop/target ambiguity resolves conservatively in favor of the stop.
- Charts above five minutes show a warning and suppress ORB signals.
- Dashboard allocation is exactly 2×40 and all-features usage remains below 40 rows.
- Retain at most 50 RSI-divergence labels, three previous-day lines, and ten unique `request.*()` contexts.
- Preserve UTF-8 text and LF line endings.

---

## File Structure

- Modify `Luxy BIG beautiful Dynamic ORB - v5.pine`: preserve the single-file indicator architecture while grouping each new calculation near existing caches and consumers.
- Modify `tools/Test-PineStatic.ps1`: extend the shared static checks created by the Trend plan with Luxy-specific invariants.
- Do not create a strategy companion or Pine library.

### Task 1: Correct Session, Timeframe, Dashboard, and Compile Blockers

**Files:**
- Modify: `Luxy BIG beautiful Dynamic ORB - v5.pine:112-215,387-464,1246-1567,2564-2613,3221`
- Modify: `tools/Test-PineStatic.ps1`

**Interfaces:**
- Produces: deterministic `inSession`, `sessionStarted`, `sessionEnded`, `lastInSessionClose`, `orbTimeframeSupported`, and a 2×40 dashboard.

- [ ] **Step 1: Add failing Luxy foundation checks**

Append inside the Luxy scope:

```powershell
Assert-NoMatch $luxy 'statsTooltip\s*\+=\s*"\\═' 'Luxy contains an invalid tooltip escape.'
Assert-Match $luxy 'table\.new\([^\n]*2\s*,\s*40' 'Luxy dashboard must be 2x40.'
Assert-Match $luxy 'orbTimeframeSupported\s*=\s*timeframe\.isminutes\s+and\s+timeframe\.multiplier\s*<=\s*5' 'Luxy must gate ORB signals above five minutes.'
Assert-NoMatch $luxy 'if\s+showDashboard\s+and\s+enableHTF' 'Dashboard visibility must not depend on HTF.'
```

Run: `powershell -ExecutionPolicy Bypass -File tools/Test-PineStatic.ps1 -Scope Luxy`

Expected: exit 1 for all four assertions.

- [ ] **Step 2: Replace session detection with deterministic rules**

```pine
bool isStockFamily = syminfo.type == "stock" or syminfo.type == "fund" or syminfo.type == "index" or syminfo.type == "dr" or syminfo.type == "etf"
bool isContinuousMarket = syminfo.type == "crypto" or syminfo.type == "forex"
bool inCustomSession = not na(time(timeframe.period, customSession, syminfo.timezone))
bool inExtendedSession = not na(time(timeframe.period, extendedPreMarket, syminfo.timezone)) or session.ismarket or not na(time(timeframe.period, extendedAfterHours, syminfo.timezone))
bool inFuturesRTH = not na(time(timeframe.period, "0930-1600:23456", "America/New_York"))
bool inFuturesElectronic = not na(time(timeframe.period, "1800-1700:1234567", "America/Chicago"))

bool inSession = switch
    sessionMode == "Custom" => inCustomSession
    isStockFamily => enableExtendedHours ? inExtendedSession : session.ismarket
    syminfo.type == "futures" => futuresTradingHours == "Electronic Full Day (24hr)" ? inFuturesElectronic : inFuturesRTH
    isContinuousMarket => true
    => session.ismarket

bool tradingDayChanged = ta.change(time_tradingday) != 0
bool sessionStarted = barstate.isconfirmed and inSession and (not inSession[1] or tradingDayChanged)
bool sessionEnded = barstate.isconfirmed and not inSession and inSession[1]
var float lastInSessionClose = na
float sessionBoundaryClose = sessionStarted or sessionEnded ? lastInSessionClose : na
if inSession and barstate.isconfirmed
    lastInSessionClose := close
```

Update the Auto-Detect tooltip to say it intentionally uses safe RTH behavior. Drive ORB-construction reset from `sessionStarted`. Remove trade-price/target flags, active-trade drawings, and session-statistics resets from the early ORB reset block; Task 7 finalizes any prior active trade at `sessionBoundaryClose`, then resets trade/session state on `sessionStarted`. This ordering also works when an RTH-only feed has no out-of-session bars between two trading days.

- [ ] **Step 3: Gate unsupported chart timeframes and fix the tooltip escape**

```pine
bool orbTimeframeSupported = timeframe.isminutes and timeframe.multiplier <= 5
bool canDetectBreakout = orbTimeframeSupported and not isHTF and enableBreakout and not na(activeORB) and barstate.isconfirmed
```

Replace the malformed statistics divider with a normal string literal beginning `"════════` and no backslash.

- [ ] **Step 4: Resize and decouple the dashboard**

```pine
if showDashboard
    if na(dashTable)
        dashTable := table.new(tablePos, 2, 40, border_width = 1)
```

Compute HTF state outside this block. Within the dashboard, render HTF cells only when `enableHTF`; render a single warning row when `not orbTimeframeSupported`. Add `// Maximum configured row index: 38 of 39` beside the final row increment after all features are implemented.

- [ ] **Step 5: Run checks and commit**

Run the Luxy static check; expected exit 0.

```powershell
git add -- 'Luxy BIG beautiful Dynamic ORB - v5.pine' 'tools/Test-PineStatic.ps1'
git commit -m "fix: stabilize Luxy sessions dashboard and timeframe gate"
```

### Task 2: Confirmed HTF/Daily Data, Previous-Day Lines, and Existing State Bugs

**Files:**
- Modify: `Luxy BIG beautiful Dynamic ORB - v5.pine:522-523,917-951,1524-1567,1680-1701,1895-2178,2731-2738,3016`
- Modify: `tools/Test-PineStatic.ps1`

**Interfaces:**
- Produces: confirmed `htfBullish`, `htfBearish`, `prevDayHigh/Low/Close`, bounded PD line IDs, and `latestSignalDirection`.

- [ ] **Step 1: Add failing confirmed-data checks**

```powershell
Assert-Match $luxy 'request\.security\(syminfo\.tickerid,\s*"D",\s*\[high\[1\],\s*low\[1\],\s*close\[1\],\s*ta\.atr\(14\)\[1\]\][^\n]+lookahead\s*=\s*barmerge\.lookahead_on' 'Previous-day tuple and daily ATR must use completed daily values.'
Assert-Match $luxy 'var\s+int\s+latestSignalDirection\s*=\s*0' 'Luxy must track the latest signal direction.'
Assert-Match $luxy 'lastBreakUpLabel\s*:=' 'Breakout-up label ID must be retained.'
Assert-Match $luxy 'lastBreakDownLabel\s*:=' 'Breakout-down label ID must be retained.'
```

- [ ] **Step 2: Replace daily and HTF requests with completed-bar requests**

```pine
[prevDayHigh, prevDayLow, prevDayClose, godDailyATR] = request.security(syminfo.tickerid, "D", [high[1], low[1], close[1], ta.atr(14)[1]], lookahead = barmerge.lookahead_on)

f_confirmedHtfData() =>
    float requestedMA = ta.ema(close, htfEMA)
    [close[1], open[1], requestedMA[1]]

[confirmedHtfClose, confirmedHtfOpen, confirmedHtfMA] = request.security(syminfo.tickerid, htfTF, f_confirmedHtfData(), lookahead = barmerge.lookahead_on)
float confirmedHtfDistance = confirmedHtfMA > 0 ? math.abs((confirmedHtfClose - confirmedHtfMA) / confirmedHtfMA) * 100.0 : 0.0
bool confirmedHtfStrong = confirmedHtfDistance >= htfMinStrength
bool htfBullish = enableHTF and (htfMethod == "Price vs MA" ? confirmedHtfClose > confirmedHtfMA and confirmedHtfStrong : confirmedHtfClose > confirmedHtfOpen)
bool htfBearish = enableHTF and (htfMethod == "Price vs MA" ? confirmedHtfClose < confirmedHtfMA and confirmedHtfStrong : confirmedHtfClose < confirmedHtfOpen)
string htfBiasText = not enableHTF ? "Disabled" : htfBullish ? "Bullish" : htfBearish ? "Bearish" : "Neutral/Weak"
```

Use this state in breakout filtering, GOD scoring, and dashboard rendering. Remove the old dashboard-local HTF assignment.

- [ ] **Step 3: Create exactly three style-preserving previous-day lines**

```pine
f_lineStyle(string value) => value == "Dotted" ? line.style_dotted : value == "Solid" ? line.style_solid : line.style_dashed
var line pdhLine = na
var line pdlLine = na
var line pdcLine = na
if sessionStarted
    line.delete(pdhLine)
    line.delete(pdlLine)
    line.delete(pdcLine)
    pdhLine := enablePDL and showPDH ? line.new(bar_index, prevDayHigh, bar_index + 1, prevDayHigh, extend = extend.right, color = pdlColor, style = f_lineStyle(pdlStyle)) : na
    pdlLine := enablePDL and showPDL ? line.new(bar_index, prevDayLow, bar_index + 1, prevDayLow, extend = extend.right, color = pdlColor, style = f_lineStyle(pdlStyle)) : na
    pdcLine := enablePDL and showPDC ? line.new(bar_index, prevDayClose, bar_index + 1, prevDayClose, extend = extend.right, color = color.new(pdlColor, 25), style = f_lineStyle(pdlStyle)) : na
```

- [ ] **Step 4: Repair label IDs and current direction**

Initialize `var int latestSignalDirection = 0`, set it to `1`/`-1` when confirmed breakout labels are created, and assign each new breakout label to `lastBreakUpLabel`/`lastBreakDownLabel`. Dashboard direction and header color use `latestSignalDirection`, not sticky session-long flags.

- [ ] **Step 5: Run checks and commit**

Run the Luxy static check; expected exit 0.

```powershell
git add -- 'Luxy BIG beautiful Dynamic ORB - v5.pine' 'tools/Test-PineStatic.ps1'
git commit -m "fix: confirm Luxy HTF daily levels and signal state"
```

### Task 3: Confirmed RSI Divergence and Session VWAP Bands

**Files:**
- Modify: `Luxy BIG beautiful Dynamic ORB - v5.pine:339-348,656-663,1381-1382,1524-1703,1895-2010,2417-2562`
- Modify: `tools/Test-PineStatic.ps1`

**Interfaces:**
- Produces: `bearishRsiDivReady`, `bullishRsiDivReady`, separate confirmation bars, `sessionVWAP`, `sessionSigma`, and four band series.

- [ ] **Step 1: Add failing feature-usage checks**

```powershell
Assert-Match $luxy 'ta\.pivothigh\(high,\s*rsiPivotLeft,\s*rsiPivotRight\)' 'Luxy must calculate confirmed price high pivots.'
Assert-Match $luxy 'bearishRsiDivBar\s*<=\s*bar_index' 'Bearish divergence must not use future confirmation.'
Assert-Match $luxy 'bullishRsiDivBar\s*<=\s*bar_index' 'Bullish divergence must not use future confirmation.'
Assert-Match $luxy 'sessionPV2' 'Luxy VWAP must track the second weighted moment.'
Assert-Match $luxy 'math\.sqrt\(math\.max\(sessionPV2\s*/\s*sessionVolume\s*-\s*sessionVWAP\s*\*\s*sessionVWAP' 'Luxy must calculate population VWAP sigma.'
```

- [ ] **Step 2: Replace RSI bool input with explicit mode and pivot controls**

```pine
rsiDivMode = input.string("Off", "RSI Divergence", options = ["Off", "Label Only", "Filter Breakouts"], group = grp_rsi)
rsiLength = input.int(14, "RSI Length", minval = 2, maxval = 100, group = grp_rsi)
rsiPivotLeft = input.int(3, "Pivot Left", minval = 1, maxval = 10, group = grp_rsi)
rsiPivotRight = input.int(3, "Pivot Right", minval = 1, maxval = 10, group = grp_rsi)
rsiPivotMaxSeparation = input.int(60, "Maximum Pivot Separation", minval = 2, maxval = 300, group = grp_rsi)
rsiDivLookback = input.int(20, "Confirmation Lookback", minval = 1, maxval = 100, group = grp_rsi)
```

- [ ] **Step 3: Implement confirmed divergence state**

```pine
float rsiValue = ta.rsi(close, rsiLength)
float pricePivotHigh = ta.pivothigh(high, rsiPivotLeft, rsiPivotRight)
float pricePivotLow = ta.pivotlow(low, rsiPivotLeft, rsiPivotRight)
var float priorHighPrice = na
var float priorHighRsi = na
var float priorLowPrice = na
var float priorLowRsi = na
var int priorHighBar = na
var int priorLowBar = na
var int bearishRsiDivBar = na
var int bullishRsiDivBar = na
if not na(pricePivotHigh)
    int pivotBar = bar_index - rsiPivotRight
    float pivotRsi = rsiValue[rsiPivotRight]
    bool newBearishDiv = not na(priorHighPrice) and pivotBar - priorHighBar <= rsiPivotMaxSeparation and pricePivotHigh > priorHighPrice and pivotRsi < priorHighRsi
    if newBearishDiv
        bearishRsiDivBar := bar_index
    priorHighPrice := pricePivotHigh
    priorHighRsi := pivotRsi
    priorHighBar := pivotBar
if not na(pricePivotLow)
    int pivotBar = bar_index - rsiPivotRight
    float pivotRsi = rsiValue[rsiPivotRight]
    bool newBullishDiv = not na(priorLowPrice) and pivotBar - priorLowBar <= rsiPivotMaxSeparation and pricePivotLow < priorLowPrice and pivotRsi > priorLowRsi
    if newBullishDiv
        bullishRsiDivBar := bar_index
    priorLowPrice := pricePivotLow
    priorLowRsi := pivotRsi
    priorLowBar := pivotBar
bool bearishRsiDivFresh = not na(bearishRsiDivBar) and bearishRsiDivBar <= bar_index and bar_index - bearishRsiDivBar <= rsiDivLookback
bool bullishRsiDivFresh = not na(bullishRsiDivBar) and bullishRsiDivBar <= bar_index and bar_index - bullishRsiDivBar <= rsiDivLookback
bool bearishRsiDivReady = bearishRsiDivFresh
bool bullishRsiDivReady = bullishRsiDivFresh
bool rsiAllowsLong = rsiDivMode != "Filter Breakouts" or not bearishRsiDivReady
bool rsiAllowsShort = rsiDivMode != "Filter Breakouts" or not bullishRsiDivReady
```

Append divergence text only when it is already confirmed and fresh on the breakout bar. Keep an array of those annotation labels and delete the oldest above 50.

- [ ] **Step 4: Implement session-anchored VWAP variance**

```pine
var float sessionVolume = 0.0
var float sessionPV = 0.0
var float sessionPV2 = 0.0
if sessionStarted
    sessionVolume := 0.0
    sessionPV := 0.0
    sessionPV2 := 0.0
if inSession and volume > 0
    sessionVolume += volume
    sessionPV += hlc3 * volume
    sessionPV2 += hlc3 * hlc3 * volume
float sessionVWAP = inSession and sessionVolume > 0 ? sessionPV / sessionVolume : na
float sessionSigma = not na(sessionVWAP) ? math.sqrt(math.max(sessionPV2 / sessionVolume - sessionVWAP * sessionVWAP, 0.0)) : na
float vwapUpper1 = sessionVWAP + sessionSigma * vwapBandMult1
float vwapLower1 = sessionVWAP - sessionSigma * vwapBandMult1
float vwapUpper2 = sessionVWAP + sessionSigma * vwapBandMult2
float vwapLower2 = sessionVWAP - sessionSigma * vwapBandMult2
```

Plot these five values at global scope only when `enableVWAPBands`; use `na` outside session.

- [ ] **Step 5: Run checks and commit**

Run the Luxy static check; expected exit 0.

```powershell
git add -- 'Luxy BIG beautiful Dynamic ORB - v5.pine' 'tools/Test-PineStatic.ps1'
git commit -m "feat: add Luxy RSI divergence and VWAP bands"
```

### Task 4: Multi-Timeframe Momentum Score

**Files:**
- Modify: `Luxy BIG beautiful Dynamic ORB - v5.pine:350-356,660-663,1524-1703,1895-2010,2905-2950`
- Modify: `tools/Test-PineStatic.ps1`

**Interfaces:**
- Produces: `bullMomentumScore`, `bearMomentumScore`, `momentumGrade`, `momentumConfigValid`, and direction-specific filter booleans.

- [ ] **Step 1: Add failing formula/request assertions**

```powershell
Assert-Match $luxy 'f_momentumContext\(' 'Luxy must define one context momentum function.'
Assert-Match $luxy '0\.50\s*\*\s*chartBullMomentum\s*\+\s*0\.30\s*\*\s*htf1BullMomentum\s*\+\s*0\.20\s*\*\s*htf2BullMomentum' 'Momentum timeframe weights must be 50/30/20.'
Assert-Match $luxy 'f_confirmedMomentum\(\)[^\n]+lookahead\s*=\s*barmerge\.lookahead_on' 'HTF momentum must use completed bars.'
```

- [ ] **Step 2: Add timeframe and threshold inputs**

```pine
momTF1 = input.timeframe("15", "Momentum HTF 1", group = grp_momentum)
momTF2 = input.timeframe("60", "Momentum HTF 2", group = grp_momentum)
momMinScore = input.float(60.0, "Minimum Direction Score", minval = 0, maxval = 100, step = 5, group = grp_momentum)
momFilterBreakouts = input.bool(false, "Filter Breakouts by Score", group = grp_momentum)
```

- [ ] **Step 3: Implement exact score formulas and completed HTF requests**

```pine
f_momentumContext() =>
    float r = ta.rsi(close, momRsiLen)
    [macdLine, signalLine, hist] = ta.macd(close, momMacdFast, momMacdSlow, momMacdSig)
    float histScale = math.max(3.0 * ta.ema(math.abs(hist), 20), syminfo.mintick)
    float macdScore = math.max(0.0, math.min(100.0, 50.0 + 50.0 * hist / histScale))
    float lowestLow = ta.lowest(low, momStochLen)
    float highestHigh = ta.highest(high, momStochLen)
    float stochScore = highestHigh > lowestLow ? 100.0 * (close - lowestLow) / (highestHigh - lowestLow) : 50.0
    (r + macdScore + stochScore) / 3.0
f_confirmedMomentum() => f_momentumContext()[1]
f_momentumGrade(float score) => na(score) ? "N/A" : score >= 75 ? "A" : score >= 60 ? "B" : score >= 45 ? "C" : score >= 30 ? "D" : "F"

float chartBullMomentum = f_momentumContext()
float htf1BullMomentum = request.security(syminfo.tickerid, momTF1, f_confirmedMomentum(), lookahead = barmerge.lookahead_on)
float htf2BullMomentum = request.security(syminfo.tickerid, momTF2, f_confirmedMomentum(), lookahead = barmerge.lookahead_on)
bool momentumConfigValid = timeframe.in_seconds(momTF1) > timeframe.in_seconds(timeframe.period) and timeframe.in_seconds(momTF2) > timeframe.in_seconds(timeframe.period)
bool momentumReady = momentumConfigValid and not na(chartBullMomentum) and not na(htf1BullMomentum) and not na(htf2BullMomentum)
float bullMomentumScore = momentumReady ? 0.50 * chartBullMomentum + 0.30 * htf1BullMomentum + 0.20 * htf2BullMomentum : na
float bearMomentumScore = momentumReady ? 100.0 - bullMomentumScore : na
string momentumGradeLong = f_momentumGrade(bullMomentumScore)
string momentumGradeShort = f_momentumGrade(bearMomentumScore)
bool momentumFilterEnabled = enableMomentumScore and momFilterBreakouts
bool momentumAllowsLong = not momentumFilterEnabled or not momentumReady or bullMomentumScore >= momMinScore
bool momentumAllowsShort = not momentumFilterEnabled or not momentumReady or bearMomentumScore >= momMinScore
```

- [ ] **Step 4: Wire score into breakout labels and qualification**

Long qualification includes `momentumAllowsLong`; short includes `momentumAllowsShort`. Label text uses the direction score/grade only when `enableMomentumScore and momentumReady`; otherwise it omits the score. Dashboard shows `Invalid TF`, `N/A`, or both directional scores.

- [ ] **Step 5: Run checks and commit**

Run the Luxy static check; expected exit 0.

```powershell
git add -- 'Luxy BIG beautiful Dynamic ORB - v5.pine' 'tools/Test-PineStatic.ps1'
git commit -m "feat: add Luxy confirmed MTF momentum score"
```

### Task 5: Corrected HMA Trend Duration Integration

**Files:**
- Modify: `Luxy BIG beautiful Dynamic ORB - v5.pine:372-376,656-676,1381-1395,1524-1703,1895-2010,2905-2950`
- Modify: `tools/Test-PineStatic.ps1`

**Interfaces:**
- Produces: `hmaDir`, `hmaBars`, bounded directional duration arrays, averages/confidence, exhaustion state, and breakout filter booleans.

- [ ] **Step 1: Add failing HMA-state assertions**

```powershell
Assert-Match $luxy 'var\s+int\s+hmaDir\s*=\s*0' 'Luxy HMA must use explicit three-state direction.'
Assert-Match $luxy 'hmaFilterMode\s*!=\s*"Filter Breakouts"\s+or\s+hmaDir\s*==\s*1' 'Long HMA filter must require bullish direction.'
Assert-Match $luxy 'f_hmaConfidence' 'Luxy must expose HMA duration confidence.'
```

- [ ] **Step 2: Port the corrected confirmed state machine**

Use a separate namespace of variables so it cannot collide with Trend filters:

```pine
hmaSamples = input.int(10, "Duration Samples", minval = 2, maxval = 20, group = grp_hma)
hmaWarnMult = input.float(1.5, "Exhaustion Warning", minval = 1.0, maxval = 10.0, step = 0.1, group = grp_hma)
var int hmaDir = 0
var int hmaBars = 0
var array<int> hmaBullDurations = array.new<int>()
var array<int> hmaBearDurations = array.new<int>()
float hmaValue = ta.hma(close, hmaLength)
int hmaDetectedDir = ta.rising(hmaValue, hmaTrendLen) ? 1 : ta.falling(hmaValue, hmaTrendLen) ? -1 : 0
if barstate.isconfirmed and enableHMAFilter
    if hmaDir == 0 and hmaDetectedDir != 0
        hmaDir := hmaDetectedDir
        hmaBars := 1
    else if hmaDetectedDir != 0 and hmaDetectedDir != hmaDir
        array<int> completed = hmaDir == 1 ? hmaBullDurations : hmaBearDurations
        if hmaBars > 0
            array.push(completed, hmaBars)
            if array.size(completed) > hmaSamples
                array.shift(completed)
        hmaDir := hmaDetectedDir
        hmaBars := 1
    else if hmaDir != 0
        hmaBars += 1
```

- [ ] **Step 3: Add averages, confidence, exhaustion, and filters**

```pine
f_hmaAvg(array<int> values) => array.size(values) > 0 ? array.avg(values) : na
f_hmaConfidence(array<int> values) =>
    float result = na
    if array.size(values) >= 2
        float mean = array.avg(values)
        float cv = mean > 0 ? array.stdev(values) / mean * 100.0 : na
        result := na(cv) ? na : math.max(0.0, math.min(100.0, 100.0 - cv))
    result
float hmaAvg = hmaDir == 1 ? f_hmaAvg(hmaBullDurations) : f_hmaAvg(hmaBearDurations)
float hmaConfidence = hmaDir == 1 ? f_hmaConfidence(hmaBullDurations) : f_hmaConfidence(hmaBearDurations)
bool hmaExhausted = not na(hmaAvg) and hmaBars > hmaAvg * hmaWarnMult
bool hmaAllowsLong = not enableHMAFilter or hmaFilterMode != "Filter Breakouts" or hmaDir == 1
bool hmaAllowsShort = not enableHMAFilter or hmaFilterMode != "Filter Breakouts" or hmaDir == -1
```

Add the direction-appropriate boolean to breakout qualification and one compact dashboard row containing direction, `hmaBars/hmaAvg`, confidence, and exhaustion.

- [ ] **Step 4: Add attribution and run checks**

Add a source comment near the ported block: `// HMA duration concepts adapted from Trend Duration Forecast [ChartPrime], MPL-2.0; locally corrected for confirmed state and bounded samples.`

Run the Luxy static check; expected exit 0.

- [ ] **Step 5: Commit HMA integration**

```powershell
git add -- 'Luxy BIG beautiful Dynamic ORB - v5.pine' 'tools/Test-PineStatic.ps1'
git commit -m "feat: integrate corrected HMA duration into Luxy"
```

### Task 6: Single-Trade Lifecycle and Next-Bar Trailing Stop

**Files:**
- Modify: `Luxy BIG beautiful Dynamic ORB - v5.pine:358-362,664-667,1384-1387,1707-1834,2218-2415`
- Modify: `tools/Test-PineStatic.ps1`

**Interfaces:**
- Produces: `tradeActive`, `initialRisk`, `effectiveStop`, `nextTrailStop`, `trailActivationBar`, and `exitReason`/`realizedR` finalized exactly once.

- [ ] **Step 1: Add failing lifecycle assertions**

```powershell
Assert-Match $luxy 'var\s+bool\s+tradeActive\s*=\s*false' 'Luxy must gate one active trade.'
Assert-Match $luxy 'bar_index\s*>\s*trailActivationBar' 'Trail must become effective after the TP1 bar.'
Assert-Match $luxy 'float\s+stopToTest\s*=\s*effectiveStop' 'Luxy must test the prior effective stop.'
Assert-Match $luxy 'nextTrailStop' 'Luxy must stage a next-bar trail.'
```

- [ ] **Step 2: Add explicit lifecycle state and entry guard**

```pine
var bool tradeActive = false
var float initialRisk = na
var float effectiveStop = na
var float nextTrailStop = na
var int trailActivationBar = na
var float trailHighWatermark = na
var float trailLowWatermark = na
var bool trailEffective = false
float realizedR = na
string exitReason = ""
bool shouldClose = false
bool canOpenTrade = not tradeActive
trailLookback = input.int(22, "Chandelier Lookback", minval = 5, maxval = 100, group = grp_targets)
```

Pending entries create targets only when `canOpenTrade`; if both directions are due, cancel both and emit one ambiguity warning rather than selecting short. On entry, set `tradeActive := true`, `initialRisk := math.abs(entry - sl)`, `effectiveStop := sl`, `trailEffective := false`, and clear staged trail state.

- [ ] **Step 3: Calculate target touches independently of visibility**

```pine
bool isLongTrade = orbTradeDirection == 1
bool tp1Touched = not na(orbTP1Price) and bar_index >= orbEntryBar and (isLongTrade ? high >= orbTP1Price : low <= orbTP1Price)
bool tp1_5Touched = not na(orbTP1_5Price) and bar_index >= orbEntryBar and (isLongTrade ? high >= orbTP1_5Price : low <= orbTP1_5Price)
bool tp2Touched = not na(orbTP2Price) and bar_index >= orbEntryBar and (isLongTrade ? high >= orbTP2Price : low <= orbTP2Price)
bool tp3Touched = not na(orbTP3Price) and bar_index >= orbEntryBar and (isLongTrade ? high >= orbTP3Price : low <= orbTP3Price)
```

These booleans are calculated before the lifecycle block but applied only after the prior effective stop survives. Label/line updates remain conditional on their corresponding `showTP*` input.

- [ ] **Step 4: Test prior stop, then stage the next trail**

```pine
float trailChandelierHigh = ta.highest(high, trailLookback)
float trailChandelierLow = ta.lowest(low, trailLookback)
if tradeActive and not orbLinesFrozen and barstate.isconfirmed
    if enableTrailingStop and not na(nextTrailStop) and bar_index > trailActivationBar
        float promotedStop = isLongTrade ? math.max(effectiveStop, nextTrailStop) : math.min(effectiveStop, nextTrailStop)
        trailEffective := trailEffective or promotedStop != effectiveStop
        effectiveStop := promotedStop
        nextTrailStop := na
    float stopToTest = effectiveStop
    bool stopTouched = isLongTrade ? low <= stopToTest : high >= stopToTest
    if stopTouched
        orbSLHit := true
        orbSLPrice := stopToTest
        realizedR := isLongTrade ? (stopToTest - orbEntryPrice) / initialRisk : (orbEntryPrice - stopToTest) / initialRisk
        exitReason := trailEffective ? "Trailing Stop" : "Stop Loss"
    else
        if not orbTP1Hit and tp1Touched
            orbTP1Hit := true
            trailActivationBar := enableTrailingStop ? bar_index : na
            trailHighWatermark := high
            trailLowWatermark := low
        if not orbTP1_5Hit and tp1_5Touched
            orbTP1_5Hit := true
        if not orbTP2Hit and tp2Touched
            orbTP2Hit := true
        if not orbTP3Hit and tp3Touched
            orbTP3Hit := true
        if enableTrailingStop and orbTP1Hit and bar_index >= trailActivationBar
            trailHighWatermark := math.max(nz(trailHighWatermark, high), high)
            trailLowWatermark := math.min(nz(trailLowWatermark, low), low)
            float atrTrail = isLongTrade ? trailHighWatermark - cachedATR * trailATRMult : trailLowWatermark + cachedATR * trailATRMult
            float chandelierTrail = isLongTrade ? trailChandelierHigh - cachedATR * trailATRMult : trailChandelierLow + cachedATR * trailATRMult
            float percentTrail = isLongTrade ? trailHighWatermark * (1.0 - trailPct / 100.0) : trailLowWatermark * (1.0 + trailPct / 100.0)
            float candidate = trailMethod == "Chandelier" ? chandelierTrail : trailMethod == "Percentage" ? percentTrail : atrTrail
            nextTrailStop := isLongTrade ? math.max(effectiveStop, candidate) : math.min(effectiveStop, candidate)
```

Update the existing SL line/label to `effectiveStop` after promotion; do not draw `nextTrailStop` early.

- [ ] **Step 5: Finalize exit R and commit**

Trailing/fixed stop R is `(effectiveStop - orbEntryPrice) / initialRisk` for long and `(orbEntryPrice - effectiveStop) / initialRisk` for short. Final-target R is `(target - orbEntryPrice) / initialRisk` for long and `(orbEntryPrice - target) / initialRisk` for short, not hard-coded 1/1.5/2/3. Assign `exitReason` to the exact closing target or stop, pass `realizedR` into Task 7's single finalization block, and set `tradeActive := false` there once.

The `bar_index >= trailActivationBar` condition stages the first candidate on the TP1 bar. The promotion block at the top runs only when `bar_index > trailActivationBar`, so that candidate becomes effective on exactly the following bar. Run the Luxy static check; expected exit 0.

```powershell
git add -- 'Luxy BIG beautiful Dynamic ORB - v5.pine' 'tools/Test-PineStatic.ps1'
git commit -m "fix: add deterministic Luxy trade and trailing lifecycle"
```

### Task 7: Coherent Streak and Drawdown Statistics

**Files:**
- Modify: `Luxy BIG beautiful Dynamic ORB - v5.pine:630-643,669-675,1366-1395,2324-2415,3185-3242`
- Modify: `tools/Test-PineStatic.ps1`

**Interfaces:**
- Consumes one finalized `realizedR` and `exitReason` from Task 6.
- Produces current/max streaks, cumulative session R, peak R, current drawdown R, and max drawdown R.

- [ ] **Step 1: Add failing central-statistics assertions**

```powershell
Assert-Match $luxy 'sessionEquityR\s*\+=\s*realizedR' 'Luxy drawdown must use finalized realized R.'
Assert-Match $luxy 'currentDrawdownR\s*=\s*sessionPeakR\s*-\s*sessionEquityR' 'Luxy must calculate peak-to-current drawdown.'
Assert-Match $luxy 'realizedR\s*==\s*0' 'Breakeven handling must be explicit.'
```

- [ ] **Step 2: Replace unused tracking variables with explicit R state**

```pine
var int winStreak = 0
var int lossStreak = 0
var int maxWinStreak = 0
var int maxLossStreak = 0
var float sessionEquityR = 0.0
var float sessionPeakR = 0.0
var float currentDrawdownR = 0.0
var float maxDrawdownR = 0.0
```

Reset all eight values on `sessionStarted`.

- [ ] **Step 3: Finalize boundary exits, then update every statistic exactly once**

```pine
bool boundaryCloseDue = freezeOnEOD and tradeActive and (sessionEnded or sessionStarted) and not na(sessionBoundaryClose)
if boundaryCloseDue
    realizedR := orbTradeDirection == 1 ? (sessionBoundaryClose - orbEntryPrice) / initialRisk : (orbEntryPrice - sessionBoundaryClose) / initialRisk
    exitReason := "EOD Close"
    shouldClose := true

if shouldClose and not trade_closed
    trade_closed := true
    tradeActive := false
    session_trades += 1
    session_total_rr += realizedR
    sessionEquityR += realizedR
    sessionPeakR := math.max(sessionPeakR, sessionEquityR)
    currentDrawdownR := sessionPeakR - sessionEquityR
    maxDrawdownR := math.max(maxDrawdownR, currentDrawdownR)
    session_best_r := math.max(session_best_r, realizedR)
    session_worst_r := math.min(session_worst_r, realizedR)
    last_trade_result := exitReason + " (" + (realizedR >= 0 ? "+" : "") + str.tostring(realizedR, "#.##") + "R)"
    alertTrailingStopTriggered := exitReason == "Trailing Stop"
    if realizedR > 0
        session_wins += 1
        winStreak += 1
        lossStreak := 0
        maxWinStreak := math.max(maxWinStreak, winStreak)
    else if realizedR < 0
        session_losses += 1
        lossStreak += 1
        winStreak := 0
        maxLossStreak := math.max(maxLossStreak, lossStreak)
    else
        winStreak := 0
        lossStreak := 0

if sessionStarted
    session_wins := 0
    session_losses := 0
    session_total_rr := 0.0
    session_trades := 0
    session_best_r := 0.0
    session_worst_r := 0.0
    sessionEquityR := 0.0
    sessionPeakR := 0.0
    currentDrawdownR := 0.0
    maxDrawdownR := 0.0
    winStreak := 0
    lossStreak := 0
    maxWinStreak := 0
    maxLossStreak := 0
    last_trade_result := ""
    trade_closed := false
    tradeActive := false
    orbTP1Hit := false
    orbTP1_5Hit := false
    orbTP2Hit := false
    orbTP3Hit := false
    orbSLHit := false
    orbLinesFrozen := false
    orbEntryPrice := na
    orbSLPrice := na
    effectiveStop := na
    nextTrailStop := na
    trailActivationBar := na
    trailEffective := false
```

Move the existing entry/SL/TP line-and-label deletion statements from the early reset block to the end of this `sessionStarted` block, preserving all twelve `line.delete`/`label.delete` calls and their assignments to `na`. Delete the old `current_trade_r` hard-coded 1/1.5/2/3R selection and its duplicate result-text/statistics block. Target, fixed-stop, trailing-stop, and EOD paths must set only `realizedR`, `exitReason`, and `shouldClose`; this block is the only writer of totals, best/worst R, streaks, drawdown, `last_trade_result`, and the trailing alert event.

- [ ] **Step 4: Render synthetic-stat disclosure and metrics**

The session row text includes `W/L`, total R, current streak, and max drawdown. Tooltip begins `Synthetic bar-based session statistics — not broker-emulator results.` and lists current/max win/loss streak plus current/max drawdown R.

- [ ] **Step 5: Run checks and commit**

Run the Luxy static check; expected exit 0.

```powershell
git add -- 'Luxy BIG beautiful Dynamic ORB - v5.pine' 'tools/Test-PineStatic.ps1'
git commit -m "feat: add Luxy streak and drawdown statistics"
```

### Task 8: Dashboard, Alerts, Resource Bounds, and Feature-Off Parity

**Files:**
- Modify: `Luxy BIG beautiful Dynamic ORB - v5.pine:2564-3298`
- Modify: `tools/Test-PineStatic.ps1`

**Interfaces:**
- Consumes all feature/state outputs from Tasks 1–7.
- Produces a bounded dashboard and one trailing-stop alert without stale cells or feature-disabled side effects.

- [ ] **Step 1: Add final row/resource assertions**

```powershell
Assert-Match $luxy 'Maximum configured row index:\s*38 of 39' 'Luxy must document its dashboard row budget.'
$luxyRequests = ([regex]::Matches($luxy, 'request\.(security|security_lower_tf)\(')).Count
if ($luxyRequests -gt 10) { $failures.Add("Luxy has $luxyRequests request sites; maximum is 10.") }
Assert-Match $luxy 'array\.size\(rsiDivLabels\)\s*>\s*50' 'RSI divergence labels must be capped at 50.'
Assert-Match $luxy 'alertcondition\([^\n]*alertTrailingStopTriggered' 'Luxy must expose a trailing-stop alert.'
```

- [ ] **Step 2: Add three bounded feature rows**

Within the 2×40 dashboard, add exactly one row each:

```pine
// VWAP row: "N/A", "Inside ±2σ", "Above +2σ", or "Below -2σ".
// Momentum row: invalid timeframe, N/A, or "Bull 72 B | Bear 28 F".
// HMA row: "↑ 12/18 | 82%" plus "EXH" when exhausted.
```

Clear `dashTable` over columns 0–1 and rows 0–39 on the last bar before rebuilding it. Leave the final comment `// Maximum configured row index: 38 of 39` after the statistics row.

- [ ] **Step 3: Add trailing alert and feature-off behavior**

Declare `alertTrailingStopTriggered` as a per-bar boolean, set it only when the effective trailed stop closes a trade, and add:

```pine
alertcondition(alertTrailingStopTriggered and enableAlerts and alertTradeManagement and enableTrailingStop, title = "Trailing Stop Hit", message = "{{ticker}} trailing stop hit at {{close}}.")
```

When each enhancement is off: RSI does not filter/annotate; VWAP plots are `na`; momentum does not request-filter labels; trailing uses the original stop; HMA does not filter; PD objects are deleted/absent; dashboard rows are omitted without leaving old cells.

- [ ] **Step 4: Run static checks and inspect source diff**

```powershell
powershell -ExecutionPolicy Bypass -File tools/Test-PineStatic.ps1 -Scope Luxy
git diff --check
git status --short
```

Expected: static checks pass, no whitespace errors, and only Luxy plus the shared test tool have implementation changes.

- [ ] **Step 5: Commit dashboard integration**

```powershell
git add -- 'Luxy BIG beautiful Dynamic ORB - v5.pine' 'tools/Test-PineStatic.ps1'
git commit -m "feat: integrate Luxy enhancement dashboard and alerts"
```

### Task 9: Full Luxy and Combined-Indicator Verification

**Files:**
- Modify: `Luxy BIG beautiful Dynamic ORB - v5.pine`
- Modify: `Trend Duration Forecast` only if combined compilation reveals an actual source defect in that script.
- Modify: `tools/Test-PineStatic.ps1`

**Interfaces:**
- Produces verified standalone and combined indicators with recorded static, compile, replay, and profiler evidence.

- [ ] **Step 1: Run all repository checks**

```powershell
powershell -ExecutionPolicy Bypass -File tools/Test-PineStatic.ps1 -Scope All
git diff --check
git status --short --branch
```

Expected: checks pass, diff check emits nothing, and the worktree contains only intentional current-task changes.

- [ ] **Step 2: Compile each exact source independently**

Paste each UTF-8 file into TradingView Pine Editor, save, and add to chart. Expected: zero compiler errors and no runtime red-exclamation markers.

- [ ] **Step 3: Execute Luxy functional matrix**

Test 1m/5m and the warning on 15m; stock RTH/extended, ES/NQ RTH/electronic/Auto-safe-RTH, crypto, forex, custom session, and a DST boundary; every ORB-stage combination; long/short breakouts; failed break/retest; hidden TP1; all three trails; same-bar stop/target; EOD exit; new signal during active trade; RSI pivots confirmed before/on/after breakout; valid/invalid MTF momentum; PD style toggles; HMA Info/Filter; zero-volume history where available; all feature-off parity.

Expected: no repaint after reload for HTF/daily values, no overwritten active trade, conservative same-bar stop result, next-bar trail, correct realized R/streak/drawdown, and no objects beyond specified bounds.

- [ ] **Step 4: Test both indicators together and profile**

Use default positions: Luxy Bottom Left, Trend history Bottom Right, ADX Top Right, progress Bottom Center. Expected: no table overlap. Record corrected-baseline and all-features Luxy Profiler totals on the same symbol/history; enhanced runtime must remain below 2× baseline and avoid timeouts. Review any block above 25% for repeated requests or object work.

- [ ] **Step 5: Review final diff and commit verification repairs**

```powershell
git diff --check
git diff --stat a71ad4c..HEAD
git status --short
git add -- 'Luxy BIG beautiful Dynamic ORB - v5.pine' 'Trend Duration Forecast' 'tools/Test-PineStatic.ps1'
git commit -m "test: verify Luxy and Trend indicators"
```

Expected: the commit includes only verification-driven repairs; if no repairs were needed, do not create an empty commit.
