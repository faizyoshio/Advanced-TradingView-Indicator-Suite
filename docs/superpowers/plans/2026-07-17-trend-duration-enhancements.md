# Trend Duration Stabilization and Enhancements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver a non-repainting, resource-bounded Pine v6 Trend Duration indicator with correct duration statistics, numerical confidence, ADX, MTF confirmation, exhaustion, progress, pivot S/R zones, and unambiguous alerts.

**Architecture:** Replace the implicit boolean trend with an explicit `-1/0/+1` confirmed-bar state machine. Compute directional samples before all dependent statistics, request completed HTF direction inside the requested context, and keep drawings/tables bounded through paired arrays and fixed capacities.

**Tech Stack:** TradingView Pine Script v6, PowerShell static source checks, TradingView Pine Editor/Bar Replay/Profiler.

## Global Constraints

- Keep `Trend Duration Forecast` as an `indicator()` overlay.
- Preserve the ChartPrime copyright and MPL-2.0 notice at lines 1–2.
- HTF direction must use expression `[1]` with `barmerge.lookahead_on`.
- Trend state is `-1`, `0`, or `+1`; duration does not begin before the first confirmed direction.
- Sample input is bounded to 2–20; history table is exactly 3×23 cells.
- S/R zones default to 10 and are bounded to 1–25.
- Progress bar always contains exactly ten blocks.
- Start events are only `0 → ±1`; reversal events are only `+1 ↔ -1`.
- All alerts and lifecycle transitions occur on confirmed bars.
- The script uses no more than three unique `request.*()` contexts.
- Preserve UTF-8 text and LF line endings.

---

## File Structure

- Modify `Trend Duration Forecast`: all indicator calculations, drawings, tables, and alerts remain in the established standalone source.
- Create `tools/Test-PineStatic.ps1`: repository-local, deterministic checks for Pine version, source invariants, forbidden regressions, and resource declarations.
- Modify `docs/superpowers/specs/2026-07-17-tradingview-enhancements-design.md` only if implementation reveals a genuine contradiction; otherwise leave the approved spec unchanged.

### Task 1: Static Guardrail and Bounded Inputs

**Files:**
- Create: `tools/Test-PineStatic.ps1`
- Modify: `Trend Duration Forecast:11-126`

**Interfaces:**
- Consumes: the two source files at repository root.
- Produces: `tools/Test-PineStatic.ps1 -Scope Trend`, returning exit code 0 only when Trend invariants hold.

- [ ] **Step 1: Create the failing Trend static checks**

```powershell
param([ValidateSet('All','Trend','Luxy')][string]$Scope = 'All')
$root = Split-Path -Parent $PSScriptRoot
$trendPath = Join-Path $root 'Trend Duration Forecast'
$luxyPath = Join-Path $root 'Luxy BIG beautiful Dynamic ORB - v5.pine'
$failures = [System.Collections.Generic.List[string]]::new()

function Assert-Match([string]$Text, [string]$Pattern, [string]$Message) {
    if ($Text -notmatch $Pattern) { $script:failures.Add($Message) }
}
function Assert-NoMatch([string]$Text, [string]$Pattern, [string]$Message) {
    if ($Text -match $Pattern) { $script:failures.Add($Message) }
}

if ($Scope -in @('All','Trend')) {
    $trend = Get-Content -Raw -Encoding UTF8 -LiteralPath $trendPath
    Assert-Match $trend '(?m)^//@version=6\s*$' 'Trend must use Pine v6.'
    Assert-Match $trend 'samples\s*=\s*input\.int\([\s\S]*?minval\s*=\s*2[\s\S]*?maxval\s*=\s*20' 'Trend samples must be bounded 2..20.'
    Assert-Match $trend 'table\.new\([^\n]*3\s*,\s*23' 'Trend history table must be 3x23.'
}
if ($Scope -in @('All','Luxy')) {
    $luxy = Get-Content -Raw -Encoding UTF8 -LiteralPath $luxyPath
    Assert-Match $luxy '(?m)^//@version=6\s*$' 'Luxy must use Pine v6.'
}
if ($failures.Count -gt 0) {
    $failures | ForEach-Object { Write-Error $_ }
    exit 1
}
Write-Host "Pine static checks passed for scope: $Scope"
```

- [ ] **Step 2: Run the check and confirm it fails on the old table/input/state**

Run: `powershell -ExecutionPolicy Bypass -File tools/Test-PineStatic.ps1 -Scope Trend`

Expected: exit 1 with failures for the sample maximum and 3×23 table.

- [ ] **Step 3: Bound all numeric inputs and add display-position inputs**

Use these exact bounds and ordering-safe inputs in `Trend Duration Forecast`:

```pine
length = input.int(50, "Smoothing Length", minval = 2, maxval = 500)
trendLength = input.int(3, "Trend Detection Sensitivity", minval = 1, maxval = 20)
samples = input.int(10, "Trend Sample Size", minval = 2, maxval = 20)
adxLen = input.int(14, "ADX DI Length", minval = 1, maxval = 100, group = grp_enhance)
adxSmooth = input.int(14, "ADX Smoothing", minval = 1, maxval = 100, group = grp_enhance)
exhaustWarn = input.float(1.5, "Exhaustion Warning Multiplier", minval = 1.0, maxval = 10.0, step = 0.1, group = grp_enhance)
exhaustDangerInput = input.float(2.0, "Exhaustion Danger Multiplier", minval = 1.1, maxval = 20.0, step = 0.1, group = grp_enhance)
exhaustDanger = math.max(exhaustDangerInput, exhaustWarn + 0.1)
historyPositionInput = input.string("Bottom Right", "History Table Position", options = ["Top Left", "Top Right", "Bottom Left", "Bottom Center", "Bottom Right"], group = grp_enhance)
adxPositionInput = input.string("Top Right", "ADX Table Position", options = ["Top Left", "Top Right", "Bottom Left", "Bottom Center", "Bottom Right"], group = grp_enhance)
progressPositionInput = input.string("Bottom Center", "Progress Table Position", options = ["Top Left", "Top Right", "Bottom Left", "Bottom Center", "Bottom Right"], group = grp_enhance)
```

Add the position mapper once before table declarations:

```pine
f_tablePosition(string value) =>
    switch value
        "Top Left" => position.top_left
        "Top Right" => position.top_right
        "Bottom Left" => position.bottom_left
        "Bottom Center" => position.bottom_center
        => position.bottom_right
```

- [ ] **Step 4: Replace the history table allocation and rerun static checks**

```pine
var table tbl = table.new(f_tablePosition(historyPositionInput), 3, 23)
```

Run: `powershell -ExecutionPolicy Bypass -File tools/Test-PineStatic.ps1 -Scope Trend`

Expected: exit 0 and `Pine static checks passed for scope: Trend`.

- [ ] **Step 5: Commit the bounded-input scaffold**

```powershell
git add -- 'Trend Duration Forecast' 'tools/Test-PineStatic.ps1'
git commit -m "test: add Pine static guardrails and Trend bounds"
```

### Task 2: Confirmed Trend State Machine and Duration Samples

**Files:**
- Modify: `Trend Duration Forecast:128-218`
- Modify: `tools/Test-PineStatic.ps1`

**Interfaces:**
- Produces: `trendDir` (`int -1/0/+1`), `trendBars` (`int`), `trendStartBull`, `trendStartBear`, `reversalToBull`, `reversalToBear`, and bounded `bullishCount`/`bearishCount` arrays.
- Consumers: all later confidence, MTF, exhaustion, drawing, and alert tasks.

- [ ] **Step 1: Add failing assertions for the explicit state machine**

Append inside the Trend scope:

```powershell
Assert-Match $trend 'var\s+int\s+trendDir\s*=\s*0' 'Trend must use a three-state integer direction.'
Assert-Match $trend 'bool\s+reversalToBull' 'Trend must expose bullish reversal events.'
Assert-Match $trend 'trendBars\s*:=\s*1' 'A new confirmed trend must begin at one bar.'
Assert-NoMatch $trend 'var\s+trend\s*=\s*bool\(na\)' 'Trend must not initialize bool(na).'
Assert-NoMatch $trend 'if\s+trend\s+or\s+not\s+trend' 'Trend tautology must be removed.'
```

Run the Trend check; expect exit 1 for all four new assertions.

- [ ] **Step 2: Replace boolean state and transition logic**

```pine
var int trendDir = 0
var int trendBars = 0
var array<int> bullishCount = array.new<int>()
var array<int> bearishCount = array.new<int>()

f_hmaDir(float source, int sensitivity) =>
    ta.rising(source, sensitivity) ? 1 : ta.falling(source, sensitivity) ? -1 : 0

int detectedDir = f_hmaDir(hma, trendLength)
bool trendStartBull = false
bool trendStartBear = false
bool reversalToBull = false
bool reversalToBear = false
int completedDir = 0
int completedBars = 0

if barstate.isconfirmed
    if trendDir == 0 and detectedDir != 0
        trendDir := detectedDir
        trendBars := 1
        trendStartBull := detectedDir == 1
        trendStartBear := detectedDir == -1
    else if detectedDir != 0 and detectedDir != trendDir
        completedDir := trendDir
        completedBars := trendBars
        reversalToBull := detectedDir == 1
        reversalToBear := detectedDir == -1
        if completedDir == 1 and completedBars > 0
            array.push(bullishCount, completedBars)
            if array.size(bullishCount) > samples
                array.shift(bullishCount)
        else if completedDir == -1 and completedBars > 0
            array.push(bearishCount, completedBars)
            if array.size(bearishCount) > samples
                array.shift(bearishCount)
        trendDir := detectedDir
        trendBars := 1
    else if trendDir != 0
        trendBars += 1
```

Replace every directional ternary with `trendDir == 1` / `trendDir == -1`, and replace `TrendCount` reads with `trendBars`.

- [ ] **Step 3: Make transition drawing consume the explicit event**

```pine
bool trendEvent = trendStartBull or trendStartBear or reversalToBull or reversalToBear
bool trendChanged = reversalToBull or reversalToBear
if trendEvent
    line.delete(LengthLine)
    label.delete(LabelProbLen)
    label.delete(exhaustLabel)
```

Use `trendEvent` to create the active trend label on both initial start and reversal; use `trendChanged` only for actions that require a completed prior trend, such as S/R creation. Do not push duration samples anywhere else; remove the old pushes and the old `TrendCount := 0` block.

- [ ] **Step 4: Run static checks**

Run: `powershell -ExecutionPolicy Bypass -File tools/Test-PineStatic.ps1 -Scope Trend`

Expected: exit 0 and `Pine static checks passed for scope: Trend`.

- [ ] **Step 5: Commit the state-machine correction**

```powershell
git add -- 'Trend Duration Forecast' 'tools/Test-PineStatic.ps1'
git commit -m "fix: correct Trend duration state machine"
```

### Task 3: Confidence, ADX, and Safe History Rendering

**Files:**
- Modify: `Trend Duration Forecast:178-210,399-475`
- Modify: `tools/Test-PineStatic.ps1`

**Interfaces:**
- Consumes: bounded duration arrays from Task 2.
- Produces: `bullAvg`, `bearAvg`, `bullConfidence`, `bearConfidence`, `f_confText()`, and safe independent table loops.

- [ ] **Step 1: Add failing assertions for numerical confidence and safe loops**

```powershell
Assert-Match $trend 'f_confidence\(' 'Trend must define numerical confidence.'
Assert-Match $trend 'math\.max\(0\.0,\s*math\.min\(100\.0' 'Confidence must be clamped 0..100.'
Assert-NoMatch $trend 'bearishCount\.get\(i\)[\s\S]{0,300}for\s+i\s*=\s*0\s+to\s+bullishCount\.size' 'One array length must not index the other array.'
```

Run the Trend check; expect exit 1 for missing confidence.

- [ ] **Step 2: Implement averages, CV, confidence, and grades**

```pine
f_average(array<int> values) => array.size(values) > 0 ? array.avg(values) : na
f_cv(array<int> values) =>
    float result = na
    if array.size(values) >= 2
        float mean = array.avg(values)
        result := mean > 0 ? array.stdev(values) / mean * 100.0 : na
    result
f_confidence(array<int> values) =>
    float cv = f_cv(values)
    na(cv) ? na : math.max(0.0, math.min(100.0, 100.0 - cv))
f_confText(float confidence) =>
    na(confidence) ? "N/A" : str.tostring(confidence, "#") + "% " + (confidence >= 80 ? "High" : confidence >= 60 ? "Medium" : confidence >= 40 ? "Low" : "Very Low")

float bullAvg = f_average(bullishCount)
float bearAvg = f_average(bearishCount)
float bullConfidence = f_confidence(bullishCount)
float bearConfidence = f_confidence(bearishCount)
```

- [ ] **Step 3: Render history arrays independently**

```pine
if barstate.islast
    table.clear(tbl, 0, 0, 2, 22)
    table.cell(tbl, 0, 0, "#", text_color = color.gray)
    table.cell(tbl, 1, 0, "Trend ↑", text_color = coloUP)
    table.cell(tbl, 2, 0, "Trend ↓", text_color = coloDN)
    if array.size(bullishCount) > 0
        for i = 0 to array.size(bullishCount) - 1
            int row = i + 1
            table.cell(tbl, 0, row, str.tostring(row), text_color = color.gray)
            table.cell(tbl, 1, row, str.tostring(array.get(bullishCount, array.size(bullishCount) - 1 - i)), text_color = chart.fg_color)
    if array.size(bearishCount) > 0
        for i = 0 to array.size(bearishCount) - 1
            int row = i + 1
            table.cell(tbl, 0, row, str.tostring(row), text_color = color.gray)
            table.cell(tbl, 2, row, str.tostring(array.get(bearishCount, array.size(bearishCount) - 1 - i)), text_color = chart.fg_color)
    table.cell(tbl, 0, 21, "Avg", text_color = color.gray)
    table.cell(tbl, 1, 21, na(bullAvg) ? "N/A" : str.tostring(bullAvg, "#.#"), text_color = coloUP)
    table.cell(tbl, 2, 21, na(bearAvg) ? "N/A" : str.tostring(bearAvg, "#.#"), text_color = coloDN)
    if showConfidence
        table.cell(tbl, 0, 22, "Conf.", text_color = color.gray)
        table.cell(tbl, 1, 22, f_confText(bullConfidence), text_color = coloUP)
        table.cell(tbl, 2, 22, f_confText(bearConfidence), text_color = coloDN)
```

Replace every direct drawing read of `bullishCount.avg()` / `bearishCount.avg()` with `bullAvg` / `bearAvg`. On `trendEvent`, always create the direction label, using `"Probable Length: N/A"` until that direction has a completed sample. Create `LengthLine` and `LabelProbLen` only when the direction-specific average is not `na`; otherwise keep both IDs `na`. Dynamic updates check `not na(TrendUP)` / `not na(TrendDN)` before setting label text and check `not na(LengthLine)` / `not na(LabelProbLen)` before setters.

- [ ] **Step 4: Keep ADX display bounded and clear stale cells**

Allocate `adxTbl` with `f_tablePosition(adxPositionInput)`, call `table.clear(adxTbl, 0, 0, 1, 3)` on the last bar, then populate only when `showADX`. Preserve DMI direction and the existing 15/25/40 strength bands.

Run the Trend static check; expected exit 0.

- [ ] **Step 5: Commit statistical rendering**

```powershell
git add -- 'Trend Duration Forecast' 'tools/Test-PineStatic.ps1'
git commit -m "feat: add bounded confidence and ADX displays"
```

### Task 4: Confirmed Multi-Timeframe Direction

**Files:**
- Modify: `Trend Duration Forecast:64-84,151-167,389-397,470-475`
- Modify: `tools/Test-PineStatic.ps1`

**Interfaces:**
- Produces: `mtfTF1Valid`, `mtfTF2Valid`, `mtfDir1`, `mtfDir2`, and `mtfAligned`.

- [ ] **Step 1: Add failing non-repaint assertions**

```powershell
Assert-Match $trend 'request\.security\([^\n]+f_confirmedHmaDir\(\)[^\n]+lookahead\s*=\s*barmerge\.lookahead_on' 'MTF direction must request confirmed context direction.'
Assert-Match $trend 'timeframe\.in_seconds\(mtfTF1\)\s*>\s*timeframe\.in_seconds\(timeframe\.period\)' 'MTF1 must be validated as higher than chart.'
Assert-NoMatch $trend 'ta\.rising\(hmaMTF' 'MTF slope must not be calculated on merged chart-timeframe values.'
```

- [ ] **Step 2: Implement confirmed requested-context direction**

```pine
f_confirmedHmaDir() =>
    float requestedHma = ta.hma(close, length)
    int requestedDir = ta.rising(requestedHma, trendLength) ? 1 : ta.falling(requestedHma, trendLength) ? -1 : 0
    requestedDir[1]

bool mtfTF1Valid = timeframe.in_seconds(mtfTF1) > timeframe.in_seconds(timeframe.period)
bool mtfTF2Valid = timeframe.in_seconds(mtfTF2) > timeframe.in_seconds(timeframe.period)
int mtfDir1 = request.security(syminfo.tickerid, mtfTF1, f_confirmedHmaDir(), lookahead = barmerge.lookahead_on)
int mtfDir2 = request.security(syminfo.tickerid, mtfTF2, f_confirmedHmaDir(), lookahead = barmerge.lookahead_on)
bool mtfConfigValid = mtfTF1Valid and mtfTF2Valid
bool mtfAligned = mtfConfigValid and trendDir != 0 and trendDir == mtfDir1 and trendDir == mtfDir2
```

- [ ] **Step 3: Update the diamond and ADX-table MTF row**

The diamond condition is `showMTF and mtfAligned`; color is `trendDir == 1 ? coloUP : coloDN`. The table shows `Invalid TF` in red when `not mtfConfigValid`, `Aligned` in green when aligned, and `Diverging` in orange otherwise.

- [ ] **Step 4: Run static checks and compile in Pine Editor**

Run the Trend static check; expected exit 0. Paste the full script into Pine Editor and choose **Save → Add to chart**; expected zero compile errors and exactly two unique HTF requests.

- [ ] **Step 5: Commit MTF confirmation**

```powershell
git add -- 'Trend Duration Forecast' 'tools/Test-PineStatic.ps1'
git commit -m "fix: make Trend MTF confirmation non-repainting"
```

### Task 5: Exhaustion, Progress, Forecast, and Alerts

**Files:**
- Modify: `Trend Duration Forecast:204-216,308-397,478-507`
- Modify: `tools/Test-PineStatic.ps1`

**Interfaces:**
- Consumes: `trendDir`, `trendBars`, averages, start/reversal events.
- Produces: confirmed `exhaustWarnEvent`, `exhaustDangerEvent`, exact ten-block progress text, capped forecast x-coordinate, and six non-duplicating alert conditions.

- [ ] **Step 1: Add failing alert/progress guards**

```powershell
Assert-Match $trend 'int\s+filledBlocks\s*=\s*math\.min\(10,\s*math\.max\(0' 'Progress blocks must be clamped 0..10.'
Assert-Match $trend 'math\.min\(bar_index\s*\+\s*500' 'Forecast must cap future bar_index.'
Assert-Match $trend 'trendStartBull[^\n]+title\s*=\s*"New Bullish Trend"' 'Bullish start alert must use only the start event.'
Assert-Match $trend 'reversalToBull[^\n]+title\s*=\s*"Bullish Reversal"' 'Bullish reversal alert must be separate.'
```

- [ ] **Step 2: Implement confirmed exhaustion edges**

```pine
float currentAvg = trendDir == 1 ? bullAvg : trendDir == -1 ? bearAvg : na
bool exhaustedWarn = barstate.isconfirmed and showExhaustion and not na(currentAvg) and trendBars > currentAvg * exhaustWarn
bool exhaustedDanger = barstate.isconfirmed and showExhaustion and not na(currentAvg) and trendBars > currentAvg * exhaustDanger
bool exhaustWarnEvent = exhaustedWarn and not exhaustedDanger and not exhaustedWarn[1]
bool exhaustDangerEvent = exhaustedDanger and not exhaustedDanger[1]
```

- [ ] **Step 3: Build an exact ten-block progress string**

```pine
float progressPct = not na(currentAvg) and currentAvg > 0 ? trendBars / currentAvg * 100.0 : na
int filledBlocks = na(progressPct) ? 0 : math.min(10, math.max(0, int(math.round(math.min(progressPct, 100.0) / 10.0))))
string progressBar = ""
for i = 0 to 9
    progressBar += i < filledBlocks ? "█" : "░"
string progressText = na(progressPct) ? "N/A" : progressBar + " " + str.tostring(progressPct, "#") + "%"
```

Create `progTbl` at `f_tablePosition(progressPositionInput)`, clear it on the last bar, and populate only when `showProgress`.

- [ ] **Step 4: Cap forecasts and declare unambiguous alerts**

```pine
int projectedBars = na(currentAvg) ? 0 : math.min(500, int(math.round(currentAvg)))
int forecastX2 = math.min(bar_index + 500, bar_index + projectedBars)

alertcondition(trendStartBull, title = "New Bullish Trend", message = "{{ticker}}: first confirmed bullish HMA trend.")
alertcondition(trendStartBear, title = "New Bearish Trend", message = "{{ticker}}: first confirmed bearish HMA trend.")
alertcondition(reversalToBull, title = "Bullish Reversal", message = "{{ticker}}: confirmed HMA reversal to bullish.")
alertcondition(reversalToBear, title = "Bearish Reversal", message = "{{ticker}}: confirmed HMA reversal to bearish.")
alertcondition(exhaustWarnEvent, title = "Trend Exhaustion Warning", message = "{{ticker}}: trend duration crossed the warning threshold.")
alertcondition(exhaustDangerEvent, title = "Trend Exhaustion Danger", message = "{{ticker}}: trend duration crossed the danger threshold.")
```

- [ ] **Step 5: Run checks and commit**

Run the Trend static check; expected exit 0.

```powershell
git add -- 'Trend Duration Forecast' 'tools/Test-PineStatic.ps1'
git commit -m "feat: correct Trend exhaustion progress and alerts"
```

### Task 6: Bounded Pivot Support/Resistance Zones

**Files:**
- Modify: `Trend Duration Forecast:114-145,218-251,343-372`
- Modify: `tools/Test-PineStatic.ps1`

**Interfaces:**
- Produces paired arrays `srBoxes`, `srLabels`, and `srDirections`; direction `1` means resistance and `-1` means support.

- [ ] **Step 1: Add failing zone-bound assertions**

```powershell
Assert-Match $trend 'maxZones\s*=\s*input\.int\(10[^\n]+minval\s*=\s*1[^\n]+maxval\s*=\s*25' 'S/R zones must be bounded 1..25.'
Assert-Match $trend 'array\.new<box>' 'S/R zones must use boxes.'
Assert-Match $trend 'array\.shift\(srBoxes\)' 'Oldest S/R boxes must be removed.'
```

- [ ] **Step 2: Add inputs, pivots, and paired arrays**

```pine
pivotStrength = input.int(3, "Pivot Strength", minval = 1, maxval = 10, group = grp_enhance)
pivotMaxAge = input.int(100, "Maximum Pivot Age", minval = 10, maxval = 500, group = grp_enhance)
maxZones = input.int(10, "Maximum S/R Zones", minval = 1, maxval = 25, group = grp_enhance)
zoneBreakMode = input.string("Fade", "Broken Zone Action", options = ["Fade", "Delete"], group = grp_enhance)
float pivotHigh = ta.pivothigh(high, pivotStrength, pivotStrength)
float pivotLow = ta.pivotlow(low, pivotStrength, pivotStrength)
var float lastPivotHigh = na
var float lastPivotLow = na
var int lastPivotHighBar = na
var int lastPivotLowBar = na
if not na(pivotHigh)
    lastPivotHigh := pivotHigh
    lastPivotHighBar := bar_index - pivotStrength
if not na(pivotLow)
    lastPivotLow := pivotLow
    lastPivotLowBar := bar_index - pivotStrength
var array<box> srBoxes = array.new<box>()
var array<label> srLabels = array.new<label>()
var array<int> srDirections = array.new<int>()
```

- [ ] **Step 3: Create and bound zones on reversals**

```pine
float zoneAtr = ta.atr(14)
f_addZone(float level, int pivotBar, int direction, float atrAtTransition) =>
    float halfWidth = atrAtTransition * 0.15
    box zone = box.new(pivotBar, level + halfWidth, bar_index, level - halfWidth, extend = extend.right, bgcolor = color.new(direction == 1 ? coloDN : coloUP, 88), border_color = direction == 1 ? coloDN : coloUP)
    label zoneLabel = label.new(pivotBar, level, direction == 1 ? "Resistance" : "Support", style = label.style_none, textcolor = direction == 1 ? coloDN : coloUP, size = size.tiny)
    array.push(srBoxes, zone)
    array.push(srLabels, zoneLabel)
    array.push(srDirections, direction)
    if array.size(srBoxes) > maxZones
        box.delete(array.shift(srBoxes))
        label.delete(array.shift(srLabels))
        array.shift(srDirections)

if showSR and reversalToBear and not na(lastPivotHighBar) and bar_index - lastPivotHighBar <= pivotMaxAge
    f_addZone(lastPivotHigh, lastPivotHighBar, 1, zoneAtr)
if showSR and reversalToBull and not na(lastPivotLowBar) and bar_index - lastPivotLowBar <= pivotMaxAge
    f_addZone(lastPivotLow, lastPivotLowBar, -1, zoneAtr)
```

- [ ] **Step 4: Fade or delete broken zones without index drift**

Iterate backward only when the array is non-empty. Resistance breaks above box top; support breaks below box bottom. In Delete mode, delete/remove all three paired entries. In Fade mode, set `extend.none`, set the right edge to `bar_index`, raise background transparency to 96, and set the paired direction to zero. Keeping faded entries in the same bounded arrays ensures total historical boxes remain capped by `maxZones`.

```pine
if barstate.isconfirmed and array.size(srBoxes) > 0
    for i = array.size(srBoxes) - 1 to 0
        box zone = array.get(srBoxes, i)
        int direction = array.get(srDirections, i)
        bool broken = direction == 1 ? close > box.get_top(zone) : direction == -1 ? close < box.get_bottom(zone) : false
        if broken
            label zoneLabel = array.get(srLabels, i)
            if zoneBreakMode == "Delete"
                box.delete(zone)
                label.delete(zoneLabel)
            else
                box.set_extend(zone, extend.none)
                box.set_right(zone, bar_index)
                box.set_bgcolor(zone, color.new(direction == 1 ? coloDN : coloUP, 96))
                array.set(srDirections, i, 0)
            if zoneBreakMode == "Delete"
                array.remove(srBoxes, i)
                array.remove(srLabels, i)
                array.remove(srDirections, i)
```

- [ ] **Step 5: Run checks and commit**

Run the Trend static check; expected exit 0.

```powershell
git add -- 'Trend Duration Forecast' 'tools/Test-PineStatic.ps1'
git commit -m "feat: add bounded Trend pivot zones"
```

### Task 7: Trend Integration Verification and Documentation

**Files:**
- Modify: `Trend Duration Forecast`
- Modify: `tools/Test-PineStatic.ps1`

**Interfaces:**
- Consumes all prior Trend tasks.
- Produces a compiled, replay-tested Trend indicator and a clean task-level diff.

- [ ] **Step 1: Add final source invariants**

```powershell
$requestCount = ([regex]::Matches($trend, 'request\.security\(')).Count
if ($requestCount -gt 3) { $failures.Add("Trend has $requestCount request.security sites; maximum is 3.") }
Assert-NoMatch $trend 'for\s+i\s*=\s*0\s+to\s+(filledBlocks|emptyBlocks)\s*-\s*1' 'Progress must not use negative loop endpoints.'
Assert-Match $trend 'max_boxes_count\s*=\s*300' 'Trend must retain its declared box capacity.'
```

- [ ] **Step 2: Run repository checks and inspect the diff**

Run:

```powershell
powershell -ExecutionPolicy Bypass -File tools/Test-PineStatic.ps1 -Scope Trend
git diff --check
git diff --stat HEAD~6..HEAD
git status --short
```

Expected: static checks pass, `git diff --check` emits nothing, and only the Trend source/test tool plus approved docs are involved.

- [ ] **Step 3: Compile and exercise the validation matrix in TradingView**

Compile the exact UTF-8 file in Pine Editor. Test initial flat/rising/falling history; 0/1/2/20 samples; 0%, 100%, and >100% progress; requested TF below/equal/above chart; unfinished HTF reload; warning-to-danger escalation; more than 25 transitions; both Fade and Delete; and all six alerts configured once per bar close.

Expected: zero compile/runtime errors, no duplicate start/reversal alert, stable completed-HTF values after reload, and no table overlap using defaults.

- [ ] **Step 4: Capture Profiler evidence**

Record total runtime with enhancements off and all enhancements on over the same symbol/history. Expected: enhanced total below 2× baseline, no execution timeout, and any block over 25% reviewed for repeated work.

- [ ] **Step 5: Commit final Trend verification adjustments**

```powershell
git add -- 'Trend Duration Forecast' 'tools/Test-PineStatic.ps1'
git commit -m "test: verify Trend Duration enhancements"
```
