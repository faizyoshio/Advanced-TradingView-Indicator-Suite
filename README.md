# 🚀 Advanced TradingView Indicator Suite: Dynamic ORB & Trend Duration Forecast

![TradingView](https://img.shields.io/badge/TradingView-Pine%20Script%20v6-131722?style=for-the-badge&logo=tradingview)
![License](https://img.shields.io/badge/License-MPL%202.0-blue?style=for-the-badge)
![Validation](https://img.shields.io/badge/Manual%20TradingView%20Smoke%20Test-Passed-success?style=for-the-badge)
![Backtest](https://img.shields.io/badge/Strategy%20Backtest-Not%20Available-lightgrey?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Experimental%20%7C%20Known%20Limitations-orange?style=for-the-badge)

A suite of **two Pine Script™ v6 chart indicators** for visual Opening Range Breakout (ORB) analysis, trend-duration context, and synthetic risk planning.

Both files use `indicator()`, not `strategy()`. They do not place broker-emulator orders and do not provide a TradingView Strategy Tester backtest. The validation badge records the manual compile/runtime smoke test documented below; it is not a profitability, accuracy, non-repainting, or production-readiness claim.

---

## 📑 Table of Contents

- [🌟 Suite Overview](#-suite-overview)
- [🔗 Source References & Attribution](#-source-references--attribution)
- [✅ Validation Results](#-validation-results-2026-07-17)
- [📦 Indicator 1: Dynamic ORB (MODIVED)](#-indicator-1-dynamic-orb-modived)
  - [⚡ Implemented Enhancements](#-implemented-enhancements)
  - [🛠️ Core Features & Architecture](#-core-features--architecture)
  - [📊 Dashboard & Risk Management](#-dashboard--risk-management)
- [📈 Indicator 2: Trend Duration Forecast (MODIVED)](#-indicator-2-trend-duration-forecast-modived)
  - [⚡ Implemented Analytics](#-implemented-analytics)
  - [🛠️ Core Features & Analytics](#-core-features--analytics)
- [💻 Installation & Setup Guide](#-installation--setup-guide)
- [🎯 Illustrative Usage Workflow](#-illustrative-usage-workflow)
  - [1. Setup & Pre-Market Checks](#1-setup--pre-market-checks)
  - [2. Execution Rules (Dynamic ORB)](#2-execution-rules-dynamic-orb)
  - [3. Trend Lifecycle Management (Trend Duration Forecast)](#3-trend-lifecycle-management-trend-duration-forecast)
- [⚙️ Example Settings by Asset Class](#-example-settings-by-asset-class-not-validated)
- [❓ FAQ & Troubleshooting](#-faq--troubleshooting)
- [⚠️ Disclaimer](#-disclaimer)

---

## 🌟 Suite Overview

| File Name | Script Name | Version | Primary Purpose |
| :--- | :--- | :---: | :--- |
| `Dynamic ORB - MODIVED.pine` | **Dynamic ORB - MODIVED** | `v6` | Multi-stage Opening Range Breakouts with heuristic quality scoring, synthetic risk planning, RSI divergence, and VWAP bands. |
| `Trend Duration Forecast - MODIVED.pine` | **Trend Duration Forecast - NODIVED** (current internal title) | `v6` | Historical trend-duration summaries with ADX, coefficient-of-variation ratings, MTF context, and exhaustion warnings. |

---

## 🔗 Source References & Attribution

Both indicators in this suite are modified (`MODIVED`) and enhanced versions built upon open-source or community Pine Script™ foundations on TradingView. We gratefully acknowledge the original authors and reference scripts:

| Script in Suite | Original Base Indicator | Original Author | Source Reference Link |
| :--- | :--- | :--- | :--- |
| **Dynamic ORB - MODIVED** | **BIG & beautiful Dynamic ORB** | **Luxy** | [TradingView Script: AZUUpYlW](https://id.tradingview.com/script/AZUUpYlW-Luxy-BIG-beautiful-Dynamic-ORB/) |
| **Trend Duration Forecast - MODIVED** | **Trend Duration Forecast** | **ChartPrime** | [TradingView Script: L3SJFfAQ](https://id.tradingview.com/script/L3SJFfAQ-Trend-Duration-Forecast-ChartPrime/) |

*Note: All enhancements, localization, GOD MODE analytics, MTF checks, and risk calculation modifications (`MODIVED`) in this repository build upon these original concepts while retaining Pine Script™ v6 compatibility.*

---

## ✅ Validation Results (2026-07-17)

The Pine sources at commit `df43f5f` were pasted into the TradingView Pine Editor and attached to a standard-candlestick `BITSTAMP:BTCUSD` chart with default inputs. Both files compiled as Pine Script v6 and rendered without a visible compiler error or runtime red-exclamation during the smoke test.

### Three-timeframe smoke matrix

| Chart timeframe | Dynamic ORB observed snapshot | Trend Duration smoke result | Assessment |
| :---: | :--- | :--- | :--- |
| `5m` | Rendered; dashboard showed ORB60 active, score `53 (C) DOWN`, `TP2 Hit (+2R)`, and `1W / 0L / 100%` internal session statistics. | Compiled and rendered without a visible runtime error. | **Smoke pass.** This is the most defensible tested chart resolution for the current ORB implementation. |
| `15m` | Rendered; dashboard showed score `83 (A) DOWN` and the same one-trade internal session result. | Compiled and rendered without a visible runtime error. | **Smoke pass with aggregation limitation.** ORB5 cannot be reconstructed precisely from a 15-minute candle. |
| `60m` | Rendered entry/SL/TP drawings; dashboard showed score `58 (B) DOWN` and the same one-trade internal session result. | Compiled and rendered HMA/exhaustion visuals without a visible runtime error. | **Smoke pass with aggregation limitation.** Only the 60-minute range is aligned to the chart; lower ORB stages are not precise. |

The displayed `1W / 0L / 100%` value is a **single synthetic, session-local observation**, not a measured win rate. It changes with symbol, data feed, loaded history, timeframe, session, and inputs.

### Validation scope

| Check | Method | Result |
| :--- | :--- | :--- |
| Pine v6 compilation | TradingView Pine Editor, Add/Update on chart | **Pass** for both files |
| Runtime smoke test | `BTCUSD` at `5m`, `15m`, and `60m` | **Pass**: both indicators rendered; no visible runtime error |
| Documentation diff check | `git diff --check` | **Pass** |
| Strategy Tester backtest | Search for `strategy()` and `strategy.*` | **Not applicable**: both scripts are `indicator()` |
| GitHub Actions / CI | Repository inspection | **Not configured**; the badge above is a manual test record |
| Profitability, win rate, drawdown | Broker-emulator report or independent dataset | **Not measured** |

### Known limitations found by audit

- **Higher-timeframe reload repaint risk:** active HTF values are requested with `lookahead_off` but without a confirmed-bar offset. They can change while the HTF candle is open and may differ after a reload. See TradingView's [repainting guidance](https://www.tradingview.com/pine-script-docs/concepts/repainting/) and [other timeframes FAQ](https://www.tradingview.com/pine-script-docs/faq/other-data-and-timeframes/).
- **ORB precision above 5 minutes:** the script does not reconstruct lower-timeframe data. On a `15m` or `60m` chart, ORB5/ORB15/ORB30 can be based on aggregated chart candles rather than the exact requested minute windows.
- **Auto-session anomaly:** during the `BTCUSD` smoke test, Auto-Detect displayed `REGULAR HOURS (09:30-16:00)` even though crypto trades 24/7. Verify the session manually and prefer Custom `0000-2359:1234567` for this use case.
- **Synthetic statistics only:** entries, targets, stops, and session statistics are inferred from chart-bar OHLC. They do not model tick order, commissions, slippage, partial fills, or broker execution, and they reset by session.
- **Partially wired controls:** static reference checks found that the FVG/pullback filter inputs and the selected Chandelier/Percentage trailing methods are not consumed by the breakout/trailing logic. The active trailing calculation is ATR-based.
- **Trend “accuracy” is not predictive accuracy:** the displayed rating is derived from the coefficient of variation of prior trend durations; it is a consistency descriptor, not an out-of-sample forecast score.
- **Naming mismatch:** the Trend file is named `MODIVED`, while its current `indicator()` title is `NODIVED`.

Passing this smoke matrix proves that the scripts compiled and rendered in the recorded setup. It does **not** prove non-repainting behavior, trading edge, profitability, broker execution, or correctness for other symbols/sessions.

---

## 📦 Indicator 1: Dynamic ORB (MODIVED)

**File:** `Dynamic ORB - MODIVED.pine`  
**Base Reference:** [BIG & beautiful Dynamic ORB by Luxy](https://id.tradingview.com/script/AZUUpYlW-Luxy-BIG-beautiful-Dynamic-ORB/)

The **Dynamic Opening Range Breakout (Dynamic ORB - MODIVED)** is a visual intraday analysis indicator that tracks opening ranges across **5, 15, 30, and 60-minute stages**. It adds configurable filters and synthetic entry/target/stop planning; it cannot eliminate false breakouts or execute orders.

### ⚡ Implemented Enhancements

#### 1. ⚡ GOD MODE Intelligence Layer (`grp_god`)
* **Breakout Quality Score (0–100, Graded A+ to D):** The current score combines volume, HTF bias, opening-range width versus ATR, gap direction, level confluence, prior breaks, and chop state. FVG proximity is displayed separately and is not currently part of `f_godScore()`.
* **Adaptive Buffer Scaled by ATR (`godAdaptiveBuffer`):** Replaces the static percentage alone with `max(Breakout Buffer %, 10% of ATR)`, widening the threshold when ATR is larger.
* **Chop-Day Guard (`godChopGuard`):** After **2+ failed breakouts** within a session, the dashboard shows `⚠️ CHOP DAY` and caps the Quality Score at `40`. It does not block the breakout condition.
* **Reference-Level Confluence (`🏆`):** Marks proximity between an ORB boundary and Previous Day or Pre-Market High/Low. The icon is a heuristic marker, not evidence of institutional participation.
* **Day Context Row:** Displays overnight gap percentage relative to yesterday's close and evaluates the opening range width compared to the 14-day Daily ATR.

#### 2. 📊 RSI Divergence Detection (`grp_rsi`)
* When enabled (default: off), compares price and RSI peaks around an ORB breakout and can append an `⚠️ RSI Div` warning to the chart label.

#### 3. 📈 VWAP Standard Deviation Bands (`grp_vwapbands`)
* When enabled (default: off), plots real-time Volume-Weighted Average Price (VWAP) alongside customized inner (`±1σ`) and outer (`±2σ`) standard-deviation bands as visual context.

#### 4. 💪 Composite Momentum Score (`grp_momentum`)
* When enabled (default: off), merges three chart-timeframe oscillators into a single 0–100 score:
  * **RSI (14)** for directional strength
  * **MACD (12, 26, 9)** for trend acceleration
  * **Stochastic (14)** for cycle timing
* Displays letter grades (`A` > 75, `B` > 60, `C` > 45, `D` > 30, `F` ≤ 30) on breakout labels. All three inputs use the chart timeframe; this is not a multi-timeframe score.

#### 5. 🎯 Trailing Stop-Loss Automation (`grp_targets`)
* Once Take Profit 1 (`TP1`) is achieved, the indicator can move a visual trailing stop. The current calculation is ATR-based; the Chandelier and Percentage selectors are not yet wired into the trailing logic, and no profit guarantee is implied.

#### 6. 📏 Previous Day & HMA Trend Confluence (`grp_pdl`, `grp_hma`)
* **Previous Day Levels:** When enabled, plots horizontal lines for Previous Day High (`PDH`), Low (`PDL`), and Close (`PDC`).
* **HMA Trend Filter:** When enabled (default: off), uses a 50-period Hull Moving Average as visual context. It blocks opposing breakouts only when `Mode Filter HMA` is set to `Filter Breakouts`; the default is `Info Only`.

#### 7. 🌐 Session Modes & E-mini Futures Override
* Provides Auto-Detect, Custom, `RTH Only`, and `Electronic Full Day` controls. Session boundaries must be verified per symbol/data feed; the audit observed an incorrect regular-hours label on `BTCUSD` Auto-Detect.

---

### 🛠️ Core Features & Architecture

```
                       +-----------------------------------+
                       |    OPENING RANGE BREAKOUT (ORB)   |
                       |    Stages: 5m | 15m | 30m | 60m    |
                       +-----------------+-----------------+
                                         |
               +-------------------------+-------------------------+
               | (Breakout Detected)                               |
               v                                                   v
    +--------------------+                              +--------------------+
    |  BULLISH BREAKOUT  |                              |  BEARISH BREAKOUT  |
    |  Close > ORB High  |                              |  Close < ORB Low   |
    +---------+----------+                              +---------+----------+
              |                                                   |
              +-------------------------+-------------------------+
                                        |
                                        v
                    +---------------------------------------+
                    |        GOD MODE & FILTER ENGINE       |
                    |  • Volume MA Check (≥ 1.5x / 2.0x)    |
                    |  • Trend Filter (VWAP / EMA / HTF)    |
                    |  • RSI Divergence & Momentum Score    |
                    |  • Adaptive Buffer & Chop-Day Guard   |
                    +-------------------+-------------------+
                                        |
                         (Eligible Confirmed-Bar Breakout)
                                        |
                                        v
                    +---------------------------------------+
                    |         VISUAL TRADE PLANNING         |
                    |  • Estimate Position Size & Shares    |
                    |  • Plot Entry, TP1, TP2, TP3 & SL     |
                    |  • Trigger Bar-Close Alert Conditions |
                    +---------------------------------------+
```

---

### 📊 Dashboard & Risk Management

The interactive heads-up display (HUD) supports five configured positions (default: `Bottom Left`) and `Dark` or `Light` themes.

#### Position Sizing & Risk Calculator
The built-in calculator estimates position size from the configured risk and visual stop distance:
* **Risk Modes:** Set fixed dollar risk (`$ Amount`, e.g., `$150 per trade`) or account percentage (`% of Account`, e.g., `1.0% of $25,000`).
* **Max Position Limit (`maxPositionPct`):** Applies a configured notional cap to the estimate.
* **Exchange Rate Conversion:** Requests daily FX rates for selected account currencies. The current conversion direction does not cover every symbol/account-currency combination correctly; verify the result independently before use.

---

## 📈 Indicator 2: Trend Duration Forecast (MODIVED)

**File:** `Trend Duration Forecast - MODIVED.pine`  
**Base Reference:** [Trend Duration Forecast by ChartPrime](https://id.tradingview.com/script/L3SJFfAQ-Trend-Duration-Forecast-ChartPrime/)

The **Trend Duration Forecast** tracks historical HMA-defined trend durations and compares the active trend with their averages. Its exhaustion zones are descriptive thresholds, not exact reversal forecasts.

### ⚡ Implemented Analytics

#### 1. 🚀 ADX Trend Strength Meter (`showADX`)
* Integrates an **Average Directional Index (ADX 14)** gauge directly into the top-right HUD.
* Uses the source thresholds `> 40` (very strong), `> 25` (strong), `> 15` (weak/moderate), and `≤ 15` (sideways). ADX does not measure volume or identify institutional activity.

#### 2. 🎯 Historical Duration Consistency (`showConfidence`)
* Calculates the dispersion of past trend durations using the **Coefficient of Variation ($CV = \frac{\text{Standard Deviation}}{\text{Mean}} \times 100$)**.
* Displays a consistency label on the chart. The UI currently calls this “accuracy”, but no predictive accuracy test is performed:
  * `🟢 Tinggi (Akurat)` — $CV < 20\%$: low relative dispersion in the stored sample.
  * `🟡 Sedang` — $CV < 40\%$: moderate relative dispersion.
  * `🟠 Rendah` — $CV < 60\%$: high relative dispersion.
  * `🔴 Sangat Rendah` — $CV \ge 60\%$: very high relative dispersion.

#### 3. 🔍 Multi-Timeframe Trend Confirmation (`showMTF`)
* Checks alignment across two user-defined timeframes (`mtfTF1` and `mtfTF2`, e.g., `15m` and `1H` when trading on a `5m` chart).
* Treat this as context only: the current active-HTF requests can change before their candles close and may repaint after reload.

#### 4. ⚠️ Trend Exhaustion Early Warning System (`showExhaustion`)
* Continuously monitors the age of the active trend (`TrendCount`) against historical memory (`currentAvg`).
* **⚠️ Warning Zone (`1.5× Average Length`):** Alerts you when the trend is getting extended. Visual indicator changes to orange (`Waspada`).
* **🚨 Danger Zone (`2.0× Average Length`):** Marks that the active duration is above the configured historical multiple. It does not prove that a reversal is imminent.

#### 5. 📊 Visual Progress Bar (`showProgress`)
* Displays a progress bar (`e.g., ████░░ 45%`) showing active duration as a percentage of the stored historical average.

#### 6. 🧱 Automated Support/Resistance Transition Zones (`showSR`)
* Automatically plots persistent horizontal dotted reference lines (`srLines`) where previous HMA trend states ended (`↑ End` / `↓ End`). They are visual references, not validated support/resistance probabilities.

---

### 🛠️ Core Features & Analytics

```
========================== TOP RIGHT ==================================
| Kekuatan Tren (ADX)    | value + strength label                     |
| Dominasi Arah          | DI+ / DI- direction                        |
| Tren Jangka Panjang    | MTF alignment                              |
=======================================================================

========================== BOTTOM RIGHT ================================
| Sample | Tren Naik (bar) | Tren Turun (bar)                         |
| ... stored duration history, averages, and CV consistency labels ... |
=======================================================================

========================== BOTTOM LEFT =================================
| Progres Perjalanan Tren | active duration / historical average      |
=======================================================================
```

---

## 💻 Installation & Setup Guide

Since these indicators are written in **TradingView Pine Script™ v6**, you can easily add them to your personal TradingView account without requiring any external server setup or paid third-party tools.

### Step 1: Open the Pine Editor
1. Log in to [TradingView.com](https://www.tradingview.com/) and open any live chart.
2. At the bottom panel of the chart window, click on the **Pine Editor** tab.

### Step 2: Add Indicator 1 (Dynamic ORB - MODIVED)
1. In the Pine Editor, click **Open** → **Create new blank indicator**.
2. Select all existing text (`Ctrl+A` or `Cmd+A`) and delete it.
3. Open `Dynamic ORB - MODIVED.pine` from this repository in your text editor, copy the entire source code, and paste it into the Pine Editor.
4. Click **Save** (top right of Pine Editor) and name it `Dynamic ORB - MODIVED`.
5. Click **Add to chart**.

### Step 3: Add Indicator 2 (Trend Duration Forecast - MODIVED)
1. In the Pine Editor, click **Open** → **Create new blank indicator** again.
2. Select all text and delete it.
3. Copy the entire source code from `Trend Duration Forecast - MODIVED.pine` and paste it into the Pine Editor.
4. Click **Save** and name it `Trend Duration Forecast - MODIVED`.
5. Click **Add to chart**.

*Both indicators can be added to the same chart, but their default bottom-left panels may overlap. Reposition or disable one HUD if needed.*

---

## 🎯 Illustrative Usage Workflow

The workflow below is an illustrative way to inspect both indicators together. It is not a validated strategy or a promise of a particular risk/reward outcome.

### 1. Setup & Pre-Market Checks
1. **Timeframe Selection:** Open a **5-minute chart** (Recommended for ORB strategies).
2. **Review Day Context (ORB Dashboard):**
   * Check the `Day Context` row to record the overnight gap and opening-range width; no gap threshold was validated by this audit.
   * Look at the `Volatility Meter`. If `🔥🔥🔥 EXTREME (>3%)`, the `Adaptive Buffer` will automatically widen your breakout thresholds.
3. **Verify Macro Bias:**
   * Check `HTF Bias` (Daily/4H trend) and `MTF Confirmation` (on the Trend Duration Forecast HUD).
   * Use `HTF Bias` and `MTF Confirmation` as context, remembering that active HTF values can change before the HTF candle closes.

### 2. Execution Rules (Dynamic ORB)
1. **Wait for Stage Completion:**
   * Allow the first **15 minutes (`ORB 15`)** or **30 minutes (`ORB 30`)** to complete. Do not anticipate breakouts before the box closes.
2. **Identify High-Quality Breakouts:**
   * When a candle closes above `ORB High` or below `ORB Low`, inspect the **Quality Score label**:
     * **Grade A+ (`Score ≥ 85`) / A (`70–84`):** Higher score under the current heuristic.
     * **Grade B (`55–69`) / C (`40–54`):** Mixed confirmation under the current heuristic.
     * **Grade D (`< 40`) or `⚠️ CHOP DAY`:** Lower score; investigate the underlying conditions before making an independent decision.
3. **Execute & Set Orders:**
   * Look at the horizontal lines plotted instantly by the indicator:
     * **Entry:** Cyan (`Long`) or Orange (`Short`).
     * **Stop-Loss (SL):** Solid Red line calculated with the selected Stop Method (default: `ATR`).
     * **Take Profits (TP1, TP2, TP3):** Green/Cyan target lines.
   * Treat the `Position Sizing` row as an estimate and verify contract multipliers, account currency, FX direction, fees, and broker rules independently.
4. **Manage the Trade:**
   * When `TP1` is hit, the indicator can activate its visual ATR trailing stop. Position exits and partial fills remain the user's responsibility.

### 3. Trend Lifecycle Management (Trend Duration Forecast)
While holding your trade, monitor the `Trend Duration Forecast` HUD:
1. **Progress Bar Check:** Compare the active duration with the historical average; do not use the percentage as a standalone holding rule.
2. **Exhaustion Warning (`⚠️ Waspada` / `1.5×`):** Treat the threshold as a prompt to review risk, price structure, and other evidence.
3. **Danger Zone (`🚨 Bahaya` / `2.0×`):** This means the configured duration multiple was crossed; it does not prove institutional profit-taking or require an automatic exit.

---

## ⚙️ Example Settings by Asset Class (Not Validated)

These presets are starting points copied from the available controls. They were not included in the three-timeframe smoke matrix and have no measured performance claim.

### 📈 U.S. Stocks & Equities (NVDA, TSLA, AAPL, AMD)
* **Dynamic ORB:**
  * `Session Mode:` verify Auto-Detect against the exchange; otherwise use `Custom: 0930-1600`
  * `Enable ORB Stages:` example: enable `ORB 15` and `ORB 30`, disable `ORB 5`; compare alternatives before use
  * `GOD MODE:` `ON` (`Adaptive Buffer: ON`, `Chop-Day Guard: ON`)
  * `Volume Filter:` `ON` (`MA Length: 20`, `Min Volume: 1.5x`, `Strong: 2.0x`)
  * `Stop Method:` `Smart Adaptive` (`Risk Mode: % of Account` -> `1.0%`)
* **Trend Duration Forecast:**
  * `HMA Length:` `50`
  * `ADX Length:` `14` (`Show ADX: ON`)
  * `MTF Timeframes:` `TF1: 15` (`15m`), `TF2: 60` (`1H`)

### ⚡ E-mini Futures (ES, NQ, YM, RTY)
* **Dynamic ORB:**
  * `Session Mode:` verify the selected futures mode against the exchange/data feed
  * `Futures Trading Hours:` `Electronic Full Day (24hr)` for swing/overnight or `RTH Only (9:30-16:00)` for pure day trading.
  * `Signal Mode:` `Track Cycles` (`Max Cycles: 6`)
  * `Retest Buffer:` `0.1%` (`Min Distance for Retest: 1.5%`)
  * `Stop Method:` `ATR` (`Multiplier: 1.5x`)
* **Trend Duration Forecast:**
  * `HMA Length:` `40`
  * `Samples:` `15` as an example history-window size

### ₿ Crypto 24/7 (BTCUSD, ETHUSD, SOLUSD)
* **Dynamic ORB:**
  * `Session Mode:` Custom (`0000-2359:1234567`) until Auto-Detect is verified for the selected crypto feed
  * `Volume Filter:` `OFF` (Crypto volume can be erratic across decentralized exchanges; rely on `GOD MODE Adaptive Buffer` + `RSI Divergence`)
  * `RSI Divergence Filter:` `ON` (`Lookback: 5`)
  * `Stop Method:` `Smart Adaptive` or `Scaled ATR` (Automatically enforces `>= 1.0x ATR` minimums for crypto volatility protection)
* **Trend Duration Forecast:**
  * `HMA Length:` `60`
  * `Exhaustion Danger Threshold:` `2.2x` as an example; calibrate independently for the selected market and timeframe

---

## ❓ FAQ & Troubleshooting

#### Q1: Why don't I see any breakout labels appearing on my chart?
* **Answer:** Confirm that a selected ORB stage has completed, `Enable Breakout` is on, and the bar closes outside the active range plus buffer. The volume and trend filters can block a signal. `Chop-Day Guard` currently caps the quality score; it does not by itself suppress the breakout condition.

#### Q2: The ORB lines look shifted or are calculating during strange hours. How do I fix this?
* **Answer:** Check the symbol/exchange session, data feed, chart resolution, and `Session Mode`. Pine calculations use the symbol timezone, not the chart display timezone. Auto-Detect is not reliable for every asset in the current implementation; use a verified Custom session when the dashboard hours are wrong (for crypto, try `0000-2359:1234567`).

#### Q3: Can I set up automated mobile or webhook alerts?
* **Answer:** Yes. For Dynamic ORB, choose the relevant named condition or **Any alert() function call** when appropriate. Trend Duration uses named `alertcondition()` entries, so select the desired bullish, bearish, warning, or danger condition individually. Configure webhook/mobile delivery in TradingView and test the alert on non-live capital first.

#### Q4: What does the trophy icon (`🏆`) on the breakout label mean?
* **Answer:** The trophy indicates proximity to a configured reference level such as Previous Day High/Low (`PDH/PDL`) or Pre-Market High/Low. The repository does not contain evidence that this icon implies a particular institutional participation level or win rate.

---

## ⚠️ Disclaimer

These indicators (`Dynamic ORB - MODIVED` and `Trend Duration Forecast - MODIVED`) are provided for educational, quantitative research, and informational purposes only. They do not constitute financial, investment, or trading advice. Trading equities, futures, forex, and cryptocurrencies carries substantial financial risk and may result in the loss of your entire capital. The repository contains no Strategy Tester backtest; observed chart outputs and statistical duration summaries do not guarantee future performance. Always perform your own independent research (`DYOR`) and practice strict risk management before deploying capital in live markets.
