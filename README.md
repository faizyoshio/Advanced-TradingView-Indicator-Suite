# 🚀 Advanced TradingView Indicator Suite: Dynamic ORB & Trend Duration Forecast (Modified & Perfected)

![TradingView](https://img.shields.io/badge/TradingView-Pine%20Script%20v6-131722?style=for-the-badge&logo=tradingview)
![License](https://img.shields.io/badge/License-MPL%202.0-blue?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Production%20Ready-success?style=for-the-badge)

A professional, institutional-grade suite of **two heavily enhanced Pine Script™ v6 trading indicators** engineered for high-precision intraday breakout trading, trend lifecycle forecasting, and automated risk management. 

These modified indicators (`MODIVED`) take the foundational concepts of Opening Range Breakouts (Mark Fisher) and Hull Moving Average (HMA) trend projection and elevate them into a comprehensive, multi-layered trading intelligence system. Both scripts have been extensively upgraded with real-time volatility adaptation, multi-timeframe (MTF) confirmation, institutional liquidity confluence, and automated position sizing—making them vastly superior, more resilient against market noise, and far more accurate than their original implementations.

---

## 📑 Table of Contents

- [🌟 Suite Overview](#-suite-overview)
- [📦 Indicator 1: Dynamic ORB (MODIVED)](#-indicator-1-dynamic-orb-modived)
  - [⚡ Why It's More Perfect Than the Original](#-why-its-more-perfect-than-the-original-key-enhancements)
  - [🛠️ Core Features & Architecture](#-core-features--architecture)
  - [📊 Dashboard & Risk Management](#-dashboard--risk-management)
- [📈 Indicator 2: Trend Duration Forecast (MODIVED)](#-indicator-2-trend-duration-forecast-modived)
  - [⚡ Why It's More Perfect Than the Original](#-why-its-more-perfect-than-the-original-key-enhancements-1)
  - [🛠️ Core Features & Analytics](#-core-features--analytics)
- [💻 Installation & Setup Guide](#-installation--setup-guide)
- [🎯 Step-by-Step Usage Guide & Trading Strategy](#-step-by-step-usage-guide--trading-strategy)
  - [1. Setup & Pre-Market Checks](#1-setup--pre-market-checks)
  - [2. Execution Rules (Dynamic ORB)](#2-execution-rules-dynamic-orb)
  - [3. Trend Lifecycle Management (Trend Duration Forecast)](#3-trend-lifecycle-management-trend-duration-forecast)
- [⚙️ Recommended Settings by Asset Class](#-recommended-settings-by-asset-class)
- [❓ FAQ & Troubleshooting](#-faq--troubleshooting)
- [⚠️ Disclaimer](#-disclaimer)

---

## 🌟 Suite Overview

| File Name | Script Name | Version | Primary Purpose |
| :--- | :--- | :---: | :--- |
| `Dynamic ORB - MODIVED.pine` | **Dynamic ORB - MODIVED** | `v6` | Multi-stage Opening Range Breakouts with GOD MODE quality scoring, automated risk management, RSI divergence, and VWAP bands. |
| `Trend Duration Forecast - MODIVED.pine` | **Trend Duration Forecast - MODIVED** | `v6` | Predictive trend duration cycle forecasting with ADX strength meter, statistical confidence ratings, MTF alignment, and exhaustion warnings. |

---

## 📦 Indicator 1: Dynamic ORB (MODIVED)

**File:** `Dynamic ORB - MODIVED.pine`

The **Dynamic Opening Range Breakout (Dynamic ORB - MODIVED)** is a complete intraday trading system that tracks market opening ranges across multiple time stages (**5, 15, 30, and 60 minutes**). While standard ORB indicators simply draw horizontal lines at opening highs and lows, this modified version integrates quantitative filtering algorithms to eliminate false breakouts ("bull/bear traps") and calculates precise order parameters before you execute.

### ⚡ Why It's More Perfect Than the Original (Key Enhancements)

#### 1. ⚡ GOD MODE Intelligence Layer (`grp_god`)
* **Breakout Quality Score (0–100, Graded A+ to D):** Every breakout is dynamically graded in real-time. The algorithm evaluates volume surge, trend alignment, Fair Value Gap (FVG) proximity, and volatility conditions to assign an instant grade right on the chart label and dashboard (`e.g., ⚡ 87 (A+) 🏆`).
* **Adaptive Buffer Scaled by ATR (`godAdaptiveBuffer`):** Replaces static percentage buffers with a dynamic calculation (`max(Breakout Buffer %, 10% of ATR)`). Volatile tickers automatically receive a wider breakout buffer to prevent false breakouts, while quiet tickers remain responsive.
* **Chop-Day Guard (`godChopGuard`):** Range-bound days kill traditional breakout strategies. If the indicator detects **2+ failed breakouts** within a single session, it triggers a `⚠️ CHOP DAY` warning on the dashboard and caps the Quality Score to `<= 40 (Grade C/D)`, protecting your capital from whipsaws.
* **Institutional Level Confluence (`🏆`):** Automatically detects when an ORB level aligns precisely with institutional reference points (Previous Day High/Low, Pre-Market High/Low). When confluent, the label awards the trophy badge (`🏆`).
* **Day Context Row:** Displays overnight gap percentage relative to yesterday's close and evaluates the opening range width compared to the 14-day Daily ATR.

#### 2. 📊 RSI Divergence Detection (`grp_rsi`)
* Automatically scans for momentum exhaustion at ORB boundaries. If price breaks out above ORB High but the Relative Strength Index (RSI 14) prints lower highs (Bearish Divergence), the indicator issues an immediate `⚠️ RSI Div` warning on the chart to alert you of a potential trap breakout.

#### 3. 📈 VWAP Standard Deviation Bands (`grp_vwapbands`)
* Plots real-time Volume-Weighted Average Price (VWAP) alongside customized inner (`±1σ`) and outer (`±2σ`) standard deviation bands. These bands act as dynamic support/resistance zones that converge with ORB boundaries during high-probability setups.

#### 4. 💪 Multi-Timeframe Momentum Score (`grp_momentum`)
* A composite momentum scoring engine that merges three powerful oscillators into a single 0–100 score:
  * **RSI (14)** for directional strength
  * **MACD (12, 26, 9)** for trend acceleration
  * **Stochastic (14)** for cycle timing
* Displays letter grades (`A` > 75, `B` > 60, `C` > 45, `D` > 30, `F` ≤ 30) directly on breakout labels to verify institutional buying/selling pressure.

#### 5. 🎯 Trailing Stop-Loss Automation (`grp_targets`)
* Once Take Profit 1 (`TP1`) is achieved, the indicator automatically activates a dynamic trailing stop (`ATR`, `Chandelier`, or `Percentage`) that tracks new price peaks, locking in profits and guaranteeing that a winning trade never turns into a loss.

#### 6. 📏 Previous Day & HMA Trend Confluence (`grp_pdl`, `grp_hma`)
* **Previous Day Levels:** Plots horizontal lines for Previous Day High (`PDH`), Low (`PDL`), and Close (`PDC`).
* **HMA Trend Filter:** Uses a 50-period Hull Moving Average to verify broader directional bias before confirming breakouts.

#### 7. 🌐 Smart Auto-Session & E-mini Futures 24/7 Override
* Automatically detects asset types. For **E-mini Futures (`ES, NQ, RTY, YM`)**, it features intelligent detection between `RTH Only (09:30–16:00 ET)` and `Electronic Full Day (24hr Globex)`, while fully supporting 24/7 Crypto markets and custom Forex sessions without timezone misalignment.

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
                         (Passed All Verification Checks)
                                        |
                                        v
                    +---------------------------------------+
                    |       AUTOMATED TRADE EXECUTION       |
                    |  • Calculate Position Size & Shares   |
                    |  • Plot Entry, TP1, TP2, TP3 & SL     |
                    |  • Trigger Non-Repainting Alerts      |
                    +---------------------------------------+
```

---

### 📊 Dashboard & Risk Management

The interactive heads-up display (HUD) can be placed anywhere on the chart (default: `Bottom Left`) and supports `Dark` and `Light` themes.

#### Position Sizing & Risk Calculator
Instead of guessing how many shares or contracts to trade, the built-in risk calculator computes the exact position size in real-time:
* **Risk Modes:** Set fixed dollar risk (`$ Amount`, e.g., `$150 per trade`) or account percentage (`% of Account`, e.g., `1.0% of $25,000`).
* **Max Position Limit (`maxPositionPct`):** Caps total position sizing (`e.g., max 25% of account`) so tight stop-losses never force over-leveraged orders.
* **Live Exchange Rate Conversion:** Automatically converts currency pairs (e.g., `EURUSD`, `USDJPY`, `USDINR`) so your risk calculations are always accurate in your native account currency.

---

## 📈 Indicator 2: Trend Duration Forecast (MODIVED)

**File:** `Trend Duration Forecast - MODIVED.pine`

The **Trend Duration Forecast - MODIVED** solves one of the hardest problems in technical analysis: **knowing when a trend is about to run out of fuel.** By tracking historical trend durations using a responsive **Hull Moving Average (HMA)** baseline, the indicator builds a statistical memory of average trend lengths and projects exact exhaustion zones in real-time.

### ⚡ Why It's More Perfect Than the Original (Key Enhancements)

#### 1. 🚀 ADX Trend Strength Meter (`showADX`)
* Integrates an **Average Directional Index (ADX 14)** gauge directly into the top-right HUD.
* Distinguishes between strong institutional trends (`ADX > 25`) and sideways, low-momentum chop (`ADX < 20`), ensuring you only trust duration projections when real market volume is driving the trend.

#### 2. 🎯 Forecast Confidence % & Statistical Accuracy (`showConfidence`)
* Calculates the historical consistency of past trends using the **Coefficient of Variation ($CV = \frac{\text{Standard Deviation}}{\text{Mean}} \times 100$)**.
* Displays a clear reliability rating on the chart:
  * `🟢 Tinggi (Accurate)` — $CV < 20\%$: Extremely consistent trend cycles; high confidence forecast.
  * `🟡 Sedang (Medium)` — $CV < 40\%$: Normal market variance; standard confidence.
  * `🟠 Rendah (Low)` — $CV < 60\%$: High variance; exercise caution.
  * `🔴 Sangat Rendah (Very Low)` — $CV \ge 60\%$: Erratic cycle lengths; do not rely on duration limits.

#### 3. 🔍 Multi-Timeframe Trend Confirmation (`showMTF`)
* Checks alignment across two user-defined higher timeframes (`mtfTF1` and `mtfTF2`, e.g., `15m` and `1H` when trading on a `5m` chart) without lookahead bias.
* Guarantees that you do not enter a short-term trend that is fighting against major macro institutional flows.

#### 4. ⚠️ Trend Exhaustion Early Warning System (`showExhaustion`)
* Continuously monitors the age of the active trend (`TrendCount`) against historical memory (`currentAvg`).
* **⚠️ Warning Zone (`1.5× Average Length`):** Alerts you when the trend is getting extended. Visual indicator changes to orange (`Waspada`).
* **🚨 Danger Zone (`2.0× Average Length`):** Alerts you that the trend is statistically overextended (`Bahaya`). The HMA line turns bright red/alert color, warning you to take profits immediately or prepare for a sharp mean-reversion reversal.

#### 5. 📊 Visual Progress Bar (`showProgress`)
* Displays an intuitive visual progress bar (`e.g., ████░░ 45%`) in the bottom-left dashboard, letting you see at a glance how much lifecycle capacity remains in the current move.

#### 6. 🧱 Automated Support/Resistance Transition Zones (`showSR`)
* Automatically plots persistent horizontal dotted reference lines (`srLines`) at the exact price levels where previous trends ended (`↑ End` / `↓ End`). These levels form strong historical pivot zones for future retests.

---

### 🛠️ Core Features & Analytics

```
========================= TOP RIGHT DASHBOARD =========================
| 📊 TREND DURATION & STRENGTH ANALYTICS                              |
|---------------------------------------------------------------------|
| Status Tren          : 🟢 NAIK (Bullish)                            |
| Kekuatan Tren (ADX)  : 🔥 32.4 (Tren Sangat Kuat)                  |
| Durasi Saat Ini      : 18 Bar                                       |
| Rata-rata Durasi     : 24 Bar                                       |
| Tingkat Akurasi      : 🟢 Tinggi (Akurat) (CV: 16.2%)               |
| Konfirmasi MTF       : ✅ SEARAH (15m: 🟢 | 60m: 🟢)                |
=======================================================================

========================= BOTTOM LEFT HUD =============================
| 📉 PROGRES DURASI TREN                                              |
| [██████████░░░░░░░░░░] 50% (12 / 24 Bar)                            |
| Status: Normal (Belum Kelelahan)                                    |
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

*(Both indicators run smoothly on the same chart. The HUD panels are pre-configured to sit in different corners (`Dynamic ORB` at Bottom Left/Top Left, `Trend Forecast` at Top Right/Bottom Left) so they never overlap or block your price bars).*

---

## 🎯 Step-by-Step Usage Guide & Trading Strategy

To achieve maximum win rates and optimal Risk/Reward, use the two indicators together as a unified system:

### 1. Setup & Pre-Market Checks
1. **Timeframe Selection:** Open a **5-minute chart** (Recommended for ORB strategies).
2. **Review Day Context (ORB Dashboard):**
   * Check the `Day Context` row. If there is a massive overnight gap (`> 2%`), expect higher volatility during the opening 15 minutes.
   * Look at the `Volatility Meter`. If `🔥🔥🔥 EXTREME (>3%)`, the `Adaptive Buffer` will automatically widen your breakout thresholds.
3. **Verify Macro Bias:**
   * Check `HTF Bias` (Daily/4H trend) and `MTF Confirmation` (on the Trend Duration Forecast HUD).
   * **Golden Rule:** Only take long trades if `HTF Bias` and `MTF Confirmation` are **Bullish/Aligned (`🟢`)**.

### 2. Execution Rules (Dynamic ORB)
1. **Wait for Stage Completion:**
   * Allow the first **15 minutes (`ORB 15`)** or **30 minutes (`ORB 30`)** to complete. Do not anticipate breakouts before the box closes.
2. **Identify High-Quality Breakouts:**
   * When a candle closes above `ORB High` or below `ORB Low`, inspect the **Quality Score label**:
     * **Grade A+ / A (`Score ≥ 75`):** Institutional confirmation! Execute full position size.
     * **Grade B (`Score 60–74`):** Solid setup. Execute standard position size.
     * **Grade C / D (`Score < 60`) or `⚠️ CHOP DAY` active:** **DO NOT ENTER.** The algorithm detected low volume, FVG mismatch, or range whipsaws.
3. **Execute & Set Orders:**
   * Look at the horizontal lines plotted instantly by the indicator:
     * **Entry:** Cyan (`Long`) or Orange (`Short`).
     * **Stop-Loss (SL):** Solid Red line (calculated dynamically using Smart Adaptive ATR).
     * **Take Profits (TP1, TP2, TP3):** Green/Cyan target lines.
   * Check the `Position Sizing` row on the dashboard for exact share count (`e.g., Buy 315 Shares`) to keep your dollar risk exact (`e.g., $150`).
4. **Manage the Trade:**
   * When `TP1` is hit, close 50% of the position. The indicator will automatically activate the **Trailing Stop-Loss** to secure remaining profits.

### 3. Trend Lifecycle Management (Trend Duration Forecast)
While holding your trade, monitor the `Trend Duration Forecast` HUD:
1. **Progress Bar Check:** As long as the progress bar is under `100%` (`e.g., ████░░ 55%`) and `ADX > 20`, hold your remaining position with confidence.
2. **Exhaustion Warning (`⚠️ Waspada` / `1.5×`):** When the trend exceeds 1.5× the average duration, tighten your manual stop-loss or trail closely using the HMA line.
3. **Danger Zone (`🚨 Bahaya` / `2.0×`):** When the warning turns red (`2.0× Average Length`), institutional profit-taking is imminent. **Exit your remaining position immediately** even if `TP3` has not been reached.

---

## ⚙️ Recommended Settings by Asset Class

### 📈 U.S. Stocks & Equities (NVDA, TSLA, AAPL, AMD)
* **Dynamic ORB:**
  * `Session Mode:` Auto-Detect (or `Custom: 0930-1600`)
  * `Enable ORB Stages:` `ORB 15` and `ORB 30` enabled (`ORB 5` disabled to filter opening noise)
  * `GOD MODE:` `ON` (`Adaptive Buffer: ON`, `Chop-Day Guard: ON`)
  * `Volume Filter:` `ON` (`MA Length: 20`, `Min Volume: 1.5x`, `Strong: 2.0x`)
  * `Stop Method:` `Smart Adaptive` (`Risk Mode: % of Account` -> `1.0%`)
* **Trend Duration Forecast:**
  * `HMA Length:` `50`
  * `ADX Length:` `14` (`Show ADX: ON`)
  * `MTF Timeframes:` `TF1: 15` (`15m`), `TF2: 60` (`1H`)

### ⚡ E-mini Futures (ES, NQ, YM, RTY)
* **Dynamic ORB:**
  * `Session Mode:` Auto-Detect
  * `Futures Trading Hours:` `Electronic Full Day (24hr)` for swing/overnight or `RTH Only (9:30-16:00)` for pure day trading.
  * `Signal Mode:` `Track Cycles` (`Max Cycles: 6`)
  * `Retest Buffer:` `0.1%` (`Min Distance for Retest: 1.5%`)
  * `Stop Method:` `ATR` (`Multiplier: 1.5x`)
* **Trend Duration Forecast:**
  * `HMA Length:` `40`
  * `Samples:` `15` (Captures frequent intraday cycles)

### ₿ Crypto 24/7 (BTCUSD, ETHUSD, SOLUSD)
* **Dynamic ORB:**
  * `Session Mode:` Custom (`0000-2359:1234567`) or Auto-Detect
  * `Volume Filter:` `OFF` (Crypto volume can be erratic across decentralized exchanges; rely on `GOD MODE Adaptive Buffer` + `RSI Divergence`)
  * `RSI Divergence Filter:` `ON` (`Lookback: 5`)
  * `Stop Method:` `Smart Adaptive` or `Scaled ATR` (Automatically enforces `>= 1.0x ATR` minimums for crypto volatility protection)
* **Trend Duration Forecast:**
  * `HMA Length:` `60`
  * `Exhaustion Danger Threshold:` `2.2x` (Crypto trends often run further than equities before exhausting)

---

## ❓ FAQ & Troubleshooting

#### Q1: Why don't I see any breakout labels appearing on my chart?
* **Answer:** You likely have `GOD MODE Chop-Day Guard` or `Volume Filter` active during a low-volume or sideways session. If the market is moving inside yesterday's range with volume under `1.5x MA`, the indicator intentionally suppresses signals to save you from false breakouts. To test, temporarily disable `Volume Filter` or check if `⚠️ CHOP DAY` is displayed on your HUD.

#### Q2: The ORB lines look shifted or are calculating during strange hours. How do I fix this?
* **Answer:** This is always a TradingView chart timezone mismatch. Right-click anywhere on your chart → **Settings** → **Symbol** → **Timezone**, and verify it matches the exchange native timezone (e.g., `New York UTC-4/-5` for U.S. stocks, `UTC` for crypto). If using `Session Mode: Auto-Detect`, TradingView handles timezone shifts automatically when set correctly.

#### Q3: Can I set up automated mobile or webhook alerts?
* **Answer:** Yes! Both scripts include pre-compiled alert conditions (`enableAlerts`, `alertBreakouts`, `alertRetests`, `alertBullish`, `alertExhaustDanger`). In TradingView, press `Alt+A` (or click **Alerts** → **Create Alert**), select `Dynamic ORB - MODIVED` or `Trend Duration Forecast - MODIVED` under **Condition**, choose **Any alert function call**, and enable **Notifications** (App/Email/Webhook).

#### Q4: What does the trophy icon (`🏆`) on the breakout label mean?
* **Answer:** The trophy indicates **Institutional Level Confluence**. It means the breakout level sits directly on top of the Previous Day High/Low (`PDH/PDL`) or Pre-Market High/Low. These setups carry significantly higher institutional participation and win rates.

---

## ⚠️ Disclaimer

These indicators (`Dynamic ORB - MODIVED` and `Trend Duration Forecast - MODIVED`) are provided for educational, quantitative research, and informational purposes only. They do not constitute financial, investment, or trading advice. Trading equities, futures, forex, and cryptocurrencies carries substantial financial risk and may result in the loss of your entire capital. Past performance, backtested results, and statistical duration forecasts do not guarantee future performance. Always perform your own independent research (`DYOR`) and practice strict risk management before deploying capital in live markets.
