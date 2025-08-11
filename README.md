# TitoSniperBot-ultra-pro-

//@version=5
indicator("TitoSniperBot Ultra Pro - Debug Friendly FULL v1", overlay=true, max_labels_count=500)

// =====================
// = USER INPUTS       =
// =====================
/* Sessions */
londonSession = input.session("0700-1000", "London Session (UTC)")
newYorkSession = input.session("1200-1500", "New York Session (UTC)")
asianSession = input.session("0000-0300", "Asian Session (UTC)")

/* Scoring & alerts */
scoreThreshold = input.int(80, "Score threshold to alert (0-100)")
alertCooldownBars = input.int(10, "Alert cooldown (bars)")

/* Liquidity Grab */
lookbackLG = input.int(10, "LG lookback (bars)")
volSpikeLG = input.float(2.0, "LG volume spike multiplier")

/* Pause-Then-Go */
ptgATRLen = input.int(5, "PTG ATR length")
ptgATRpauseLen = input.int(10, "PTG pause ATR lookback")

/* FVG */
fvgGapLookback = input.int(2, "FVG gap lookback (bars)")

/* VWAP / ATR / RelVol / HTF EMA */
useVWAPFilter = input.bool(true, "Use VWAP trend filter")
atrPeriod = input.int(14, "ATR period")
atrThreshold = input.float(0.0, "Min ATR (0 = disabled)")
relVolPeriod = input.int(14, "Relative volume period")
relVolMin = input.float(1.0, "Relative volume minimum multiplier")
htfEMA_tf = input.timeframe("60", "HTF EMA timeframe")
htfEMA_len = input.int(200, "HTF EMA length")

/* Order block */
obLookback = input.int(20, "OB lookback (bars)")

/* Volume Profile */
vpLookback = input.int(50, "VP lookback (bars)")
vpThreshold = input.float(1.5, "VP volume spike multiplier")

/* RSI & Stochastic */
rsiPeriod = input.int(14, "RSI period")
rsiOverbought = input.int(70, "RSI overbought")
rsiOversold = input.int(30, "RSI oversold")
stoK = input.int(14, "Stochastic %K period")
stoD = input.int(3, "Stochastic %D period")
stoSmooth = input.int(3, "Stochastic smoothing")

/* MACD */
macdFast = input.int(12, "MACD fast")
macdSlow = input.int(26, "MACD slow")
macdSignal = input.int(9, "MACD signal")

/* Supertrend */
useSupertrend = input.bool(true, "Use Supertrend filter")
superFactor = input.float(3.0, "Supertrend factor")
superATRlen = input.int(10, "Supertrend ATR len")

/* Heikin-Ashi filter */
useHAfilter = input.bool(true, "Use Heikin-Ashi filter")

/* Price action patterns */
pinBarBodyPct = input.float(30.0, "Pin bar max body % of range")

/* TP/SL multipliers (ATR-based) */
atrMultSL = input.float(1.5, "SL = ATR *")
atrMultTP1 = input.float(1.0, "TP1 = ATR *")
atrMultTP2 = input.float(2.0, "TP2 = ATR *")
atrMultTP3 = input.float(3.0, "TP3 = ATR *")

/* Position sizing (informational) */
riskPercent = input.float(1.0, "Risk % per trade")

/* Dashboard and visuals */
showDashboard = input.bool(true, "Show Dashboard (last bar)")
dashX = input.int(0, "Dashboard X offset")
dashY = input.int(15, "Dashboard Y offset")

// =====================
// = SESSIONS & BASICS =
// =====================
inLondon = not na(time(timeframe.period, londonSession))
inNewYork = not na(time(timeframe.period, newYorkSession))
inAsian = not na(time(timeframe.period, asianSession))
inAnySession = inLondon or inNewYork or inAsian

// Basic indicators
vw = ta.vwap(close)
atr = ta.atr(atrPeriod)
relVol = volume / ta.sma(volume, relVolPeriod)
htfEMA = request.security(syminfo.tickerid, htfEMA_tf, ta.ema(close, htfEMA_len))

// =====================
// = CORE PATTERN LOGIC =
// =====================

// --- 1) Liquidity Grab (LG) — simple stop-hunt detection ---
// Bullish LG: price dips below recent low (lookback), then closes back above that low, with volume spike
recentLow = ta.lowest(low, lookbackLG)
recentHigh = ta.highest(high, lookbackLG)
volAvgLG = ta.sma(volume, lookbackLG)
bullishLG = (low < recentLow) and (close > recentLow) and (volume > volAvgLG * volSpikeLG)
bearishLG = (high > recentHigh) and (close < recentHigh) and (volume > volAvgLG * volSpikeLG)

// --- 2) Pause-Then-Go (PTG) ---
// Low volatility "pause" (ATR at low) followed by a momentum breakout candle
ptgATR = ta.atr(ptgATRLen)
ptgATRmin = ta.lowest(ptgATR, ptgATRpauseLen)
strongBull = (close > open) and ((close - open) > ptgATR)
strongBear = (open > close) and ((open - close) > ptgATR)
bullPTG = (ptgATR < ptgATRmin) and strongBull
bearPTG = (ptgATR < ptgATRmin) and strongBear

// --- 3) Fair Value Gap (FVG) / imbalance ---
// Simple approach: gap between candle bodies / highs-lows
bullFVG = low > high[fvgGapLookback]
bearFVG = high < low[fvgGapLookback]

// --- 4) Order Block (OB) simplified (engulfing proxy) ---
bullOB = (close > open) and (close[1] < open[1]) and (close > close[1])
bearOB = (close < open) and (close[1] > open[1]) and (close < close[1])

// --- 5) Volume Profile filter (visible-range proxy) ---
vpHigh = ta.highest(high, vpLookback)
vpLow = ta.lowest(low, vpLookback)
vpAvg = ta.sma(volume, vpLookback)
vpSpike = volume > vpAvg * vpThreshold
vpBuy = (close < vpLow * 1.01) and vpSpike
vpSell = (close > vpHigh * 0.99) and vpSpike

// --- 6) Momentum filters: RSI & Stochastic ---
rsi = ta.rsi(close, rsiPeriod)
rsiBuy = rsi < rsiOversold
rsiSell = rsi > rsiOverbought

stochK = ta.sma(ta.stoch(close, high, low, stoK), stoSmooth)
stochD = ta.sma(stochK, stoD)
stochBuy = (stochK < 20) and (stochK > stochD)
stochSell = (stochK > 80) and (stochK < stochD)

// --- 7) MACD trend --- 
[macdLine, macdSignal, _] = ta.macd(close, macdFast, macdSlow, macdSignal)
macdBuy = macdLine > macdSignal
macdSell = macdLine < macdSignal

// --- 8) Divergence simple (RSI) ---
// NOTE: this is simplistic — full divergence detection is more complex
bullDiv = ta.lowestbars(low, 5) == 0 and rsi > ta.lowest(rsi, 5)
bearDiv = ta.highestbars(high, 5) == 0 and rsi < ta.highest(rsi, 5)

// --- 9) HTF confirmation (HTF EMA + HTF RSI) ---
htfRSI = request.security(syminfo.tickerid, htfEMA_tf, ta.rsi(close, rsiPeriod))
htfBuy = (htfRSI < rsiOversold) and (close > htfEMA)
htfSell = (htfRSI > rsiOverbought) and (close < htfEMA)

// --- 10) Heikin-Ashi filter ---
var float haOpen = na
haClose = (open + high + low + close) / 4
haOpen := na(haOpen[1]) ? (open + close) / 2 : (haOpen[1] + haClose[1]) / 2
haBull = haClose > haOpen
haBear = haClose < haOpen
haOkBuy = useHAfilter ? haBull : true
haOkSell = useHAfilter ? haBear : true

// --- 11) Supertrend (light) ---
var float upperBand = na
var float lowerBand = na
atrST = ta.atr(superATRlen)
upperBasic = (high + low) / 2 + superFactor * atrST
lowerBasic = (high + low) / 2 - superFactor * atrST
upperBand := nz(upperBand[1], upperBasic)
lowerBand := nz(lowerBand[1], lowerBasic)
upperBand := close[1] > upperBand[1] ? math.max(upperBasic, upperBand[1]) : upperBasic
lowerBand := close[1] < lowerBand[1] ? math.min(lowerBasic, lowerBand[1]) : lowerBasic
superBuy = close > upperBand
superSell = close < lowerBand

// --- 12) Price Action patterns (Pin bar, Engulfing, Inside) ---
body = math.abs(close - open)
rng = high - low
bodyPct = rng == 0 ? 0 : (body / rng) * 100
pinBull = (bodyPct < pinBarBodyPct) and (close > open) and (low == ta.lowest(low, 3))
pinBear = (bodyPct < pinBarBodyPct) and (close < open) and (high == ta.highest(high, 3))
engulfBull = (close > open) and (open[1] > close[1]) and (close > open[1]) and (open < close[1])
engulfBear = (close < open) and (open[1] < close[1]) and (close < open[1]) and (open > close[1])
inside = (high < high[1]) and (low > low[1])
paBuy = pinBull or engulfBull or inside
paSell = pinBear or engulfBear or inside

// =====================
// = SCORING (100 pts)  =
// =====================
// Weights - tuneable
w_LG = 20
w_PTG = 15
w_FVG = 12
w_Session = 8
w_VWAP = 8
w_ATR = 5
w_HTFEMA = 7
w_RelVol = 5
w_OB = 5

// New filters (total remaining)
w_VP = 5
w_RSI = 5
w_STO = 5
w_MACD = 5
w_DIV = 5
w_HTF = 7
w_HA = 3
w_SUP = 5
w_PA = 5

// initialize
scoreBuy = 0.0
scoreSell = 0.0

// Original scoring contributors
scoreBuy += bullishLG ? w_LG : 0
scoreSell += bearishLG ? w_LG : 0

scoreBuy += bullPTG ? w_PTG : 0
scoreSell += bearPTG ? w_PTG : 0

scoreBuy += bullFVG ? w_FVG : 0
scoreSell += bearFVG ? w_FVG : 0

scoreBuy += inAnySession ? w_Session : 0
scoreSell += inAnySession ? w_Session : 0

scoreBuy += (useVWAPFilter and close > vw) ? w_VWAP : 0
scoreSell += (useVWAPFilter and close < vw) ? w_VWAP : 0

scoreBuy += (atr > atrThreshold ? w_ATR : 0)
scoreSell += (atr > atrThreshold ? w_ATR : 0)

scoreBuy += (close > htfEMA ? w_HTFEMA : 0)
scoreSell += (close < htfEMA ? w_HTFEMA : 0)

scoreBuy += (relVol > relVolMin ? w_RelVol : 0)
scoreSell += (relVol > relVolMin ? w_RelVol : 0)

scoreBuy += bullOB ? w_OB : 0
scoreSell += bearOB ? w_OB : 0

// New filters scoring
scoreBuy += vpBuy ? w_VP : 0
scoreSell += vpSell ? w_VP : 0

scoreBuy += rsiBuy ? w_RSI : 0
scoreSell += rsiSell ? w_RSI : 0

scoreBuy += stochBuy ? w_STO : 0
scoreSell += stochSell ? w_STO : 0

scoreBuy += macdBuy ? w_MACD : 0
scoreSell += macdSell ? w_MACD : 0

scoreBuy += bullDiv ? w_DIV : 0
scoreSell += bearDiv ? w_DIV : 0

scoreBuy += htfBuy ? w_HTF : 0
scoreSell += htfSell ? w_HTF : 0

scoreBuy += haOkBuy ? w_HA : 0
scoreSell += haOkSell ? w_HA : 0

scoreBuy += superBuy ? w_SUP : 0
scoreSell += superSell ? w_SUP : 0

scoreBuy += paBuy ? w_PA : 0
scoreSell += paSell ? w_PA : 0

// small bonus for order block strength (example)
scoreBuy := scoreBuy + (bullOB ? 0 : 0)
scoreSell := scoreSell + (bearOB ? 0 : 0)

// Clamp scores (just in case)
scoreBuy := math.min(scoreBuy, 100)
scoreSell := math.min(scoreSell, 100)

// =====================
// = ALERT & COOLDOWN  =
// =====================
var int lastBuyBar = na
var int lastSellBar = na
canAlertBuy = na(lastBuyBar) or (bar_index - lastBuyBar > alertCooldownBars)
canAlertSell = na(lastSellBar) or (bar_index - lastSellBar > alertCooldownBars)

buySignalFinal = (scoreBuy >= scoreThreshold) and canAlertBuy
sellSignalFinal = (scoreSell >= scoreThreshold) and canAlertSell

if buySignalFinal
    lastBuyBar := bar_index
if sellSignalFinal
    lastSellBar := bar_index

// Position sizing (info only)
positionSize = atr == 0 ? 0 : math.round(riskPercent / (atr / close) * 100, 2)

// TP/SL on entry bar (ATR multiples)
var float entryBuy = na
var float entrySell = na
if buySignalFinal
    entryBuy := close
    tp1Buy := entryBuy + atr * atrMultTP1
    tp2Buy := entryBuy + atr * atrMultTP2
    tp3Buy := entryBuy + atr * atrMultTP3
    slBuy := entryBuy - atr * atrMultSL
else
    tp1Buy := na; tp2Buy := na; tp3Buy := na; slBuy := na; entryBuy := na

if sellSignalFinal
    entrySell := close
    tp1Sell := entrySell - atr * atrMultTP1
    tp2Sell := entrySell - atr * atrMultTP2
    tp3Sell := entrySell - atr * atrMultTP3
    slSell := entrySell + atr * atrMultSL
else
    tp1Sell := na; tp2Sell := na; tp3Sell := na; slSell := na; entrySell := na

// Alerts
alertcondition(buySignalFinal, title="Tito BUY", message="🔥 TITO BUY 🔥\nScore: " + str.tostring(scoreBuy) + "\nEntry: {{close}}\nTP1: " + str.tostring(tp1Buy) + "\nTP2: " + str.tostring(tp2Buy) + "\nTP3: " + str.tostring(tp3Buy) + "\nSL: " + str.tostring(slBuy))
alertcondition(sellSignalFinal, title="Tito SELL", message="⚠️ TITO SELL ⚠️\nScore: " + str.tostring(scoreSell) + "\nEntry: {{close}}\nTP1: " + str.tostring(tp1Sell) + "\nTP2: " + str.tostring(tp2Sell) + "\nTP3: " + str.tostring(tp3Sell) + "\nSL: " + str.tostring(slSell))

// =====================
// = PLOTTING & DASH  =
// =====================
// Session backgrounds
bgcolor(inLondon ? color.new(color.blue, 90) : na)
bgcolor(inNewYork ? color.new(color.red, 90) : na)
bgcolor(inAsian ? color.new(color.green, 90) : na)

// Arrows
plotshape(buySignalFinal, location=location.belowbar, color=color.green, style=shape.triangleup, size=size.large, title="Buy Arrow")
plotshape(sellSignalFinal, location=location.abovebar, color=color.red, style=shape.triangledown, size=size.large, title="Sell Arrow")

// TP/SL lines
plot(tp1Buy, title="TP1 Buy", color=color.new(color.green, 60), linewidth=1, style=plot.style_linebr)
plot(tp2Buy, title="TP2 Buy", color=color.new(color.green, 40), linewidth=1, style=plot.style_linebr)
plot(tp3Buy, title="TP3 Buy", color=color.new(color.green, 20), linewidth=1, style=plot.style_linebr)
plot(slBuy,  title="SL Buy",  color=color.new(color.red, 0), linewidth=2, style=plot.style_linebr)

plot(tp1Sell, title="TP1 Sell", color=color.new(color.red, 60), linewidth=1, style=plot.style_linebr)
plot(tp2Sell, title="TP2 Sell", color=color.new(color.red, 40), linewidth=1, style=plot.style_linebr)
plot(tp3Sell, title="TP3 Sell", color=color.new(color.red, 20), linewidth=1, style=plot.style_linebr)
plot(slSell,  title="SL Sell",  color=color.new(color.green, 0), linewidth=2, style=plot.style_linebr)

// Dashboard (last bar)
var label dash = na
if showDashboard and barstate.islast
    label.delete(dash)
    sessionName = inLondon ? "London" : inNewYork ? "New York" : inAsian ? "Asian" : "None"
    dashText = "TitoSniperBot Ultra PRO\nSession: " + sessionName + "\nScoreBuy: " + str.tostring(scoreBuy, "#.0") + "\nScoreSell: " + str.tostring(scoreSell, "#.0") + "\nATR: " + str.tostring(atr, format.mintick) + "\nRelVol: " + str.tostring(relVol, format.mintick) + "\nPosSize%: " + str.tostring(positionSize)
    dash := label.new(bar_index + dashX, high + dashY, dashText, style=label.style_label_left, color= buySignalFinal ? color.green : sellSignalFinal ? color.red : color.gray, textcolor=color.white, size=size.small)

// Small helpful plots for debugging (optional)
plot(close > vw ? 1 : 0, title="VWAP Bias (1=above)", display=display.none)
plot(atr > atrThreshold ? 1 : 0, title="ATR OK (1=above)", display=display.none)

// =====================
// = TODOs for community =
// =====================
// 1) Improve Liquidity Grab detection to include wick shape, orderflow proxies, and non-repainting rules.
// 2) Replace simple FVG with multi-bar fair value gap zones and "filled" detection (now simple gap check).
// 3) Add persistent storage of entry prices and partial TP hit tracking (Pine's bar-based alerting is limited).
// 4) Optimize divergence detection to avoid false positives (use peaks/troughs library approach).
// 5) Performance: arrays/line management for many FVG/OB lines (clean-up on old bars).
// 6) Optional: add input toggles to re-balance all weights quickly from UI.
// 7) Integrate a webhook-friendly JSON message template for Discord with all fields formatted.
// 8) Make this pine script a version 6 to be able to be used in trendingview indicators
