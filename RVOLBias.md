Yep—this is very doable in Pine Script on TradingView (and on most other charting platforms). Below is a drop-in Pine v5 indicator that implements the exact idea we discussed:
	•	CMF (Chaikin-style “directional” volume) over N bars
	•	RVOL (current volume ÷ average volume)
	•	dRVOL = CMF × RVOL as the core bias metric
	•	Body-efficiency + wick sanity filters
	•	Optional VWAP alignment
	•	Optional HTF (higher-timeframe) confirmation
	•	Visuals + alert conditions for long/short bias

⸻


//@version=5
indicator("Directional RVOL Bias [HxB]", overlay=false, max_labels_count=500)

//––––– Inputs
lenCMF      = input.int(20,  "CMF length", minval=2)
lenRVOL     = input.int(20,  "RVOL lookback", minval=2)
rvolType    = input.string("SMA", "RVOL average", options=["SMA","EMA"])
minRVOL     = input.float(3.0, "Min RVOL to consider", step=0.1)

effMin      = input.float(0.55, "Min body efficiency (|C-O|/(H-L))", step=0.01)
wickMax     = input.float(0.40, "Max wick/body ratio", step=0.01)

useVWAP     = input.bool(true, "Require VWAP alignment")
useHTF      = input.bool(false, "Use HTF confirmation")
htfTf       = input.timeframe("5", "HTF for confirmation")

thrLong     = input.float(2.0,  "dRVOL long threshold", step=0.1)
thrShort    = input.float(-2.0, "dRVOL short threshold", step=0.1)

//––––– Helpers
rng  = high - low
rngS = math.max(rng, syminfo.mintick)

// Chaikin-style close location value (directional weight, -1..+1)
clv  = rng > 0 ? (((close - low) - (high - close)) / rngS) : 0.0
mfv  = clv * volume
cmf  = ta.sum(mfv, lenCMF) / math.max(ta.sum(volume, lenCMF), 1.0)

// RVOL (current volume divided by average volume)
volAvg = rvolType=="SMA" ? ta.sma(volume, lenRVOL) : ta.ema(volume, lenRVOL)
rvol   = volAvg > 0 ? (volume / volAvg) : na

// Directional RVOL
drvol = cmf * rvol

// Body efficiency & wicks (to avoid churn/exhaustion)
body = math.abs(close - open)
uw   = high - math.max(close, open)
lw   = math.min(close, open) - low
uwr  = body > 0 ? (uw / body) : 999.0
lwr  = body > 0 ? (lw / body) : 999.0
eff  = body / rngS

// VWAP + HTF confirmation
vwap = ta.vwap
htfOkLong  = useHTF ? request.security(syminfo.tickerid, htfTf, drvol,  barmerge.gaps_off, barmerge.lookahead_off) > 0 : true
htfOkShort = useHTF ? request.security(syminfo.tickerid, htfTf, drvol,  barmerge.gaps_off, barmerge.lookahead_off) < 0 : true
vwapLong   = useVWAP ? close > vwap : true
vwapShort  = useVWAP ? close < vwap : true

// Signals
longOK  = (rvol >= minRVOL) and (drvol >=  thrLong) and (eff >= effMin) and (uwr <= wickMax) and vwapLong  and htfOkLong
shortOK = (rvol >= minRVOL) and (drvol <=  thrShort) and (eff >= effMin) and (lwr <= wickMax) and vwapShort and htfOkShort

//––––– Plots
col = longOK ? color.new(color.lime, 0) : shortOK ? color.new(color.red, 0) : color.new(color.gray, 40)
plot(drvol, "dRVOL (CMF × RVOL)", color=col, style=plot.style_histogram, linewidth=2)
plot(cmf,   "CMF", color=color.new(color.teal, 0))
plot(rvol,  "RVOL", color=color.new(color.orange, 0))

h0 = hline(0, "zero", color=color.gray)
hL = hline(thrLong,  "long-thr", color=color.new(color.lime, 50))
hS = hline(thrShort, "short-thr", color=color.new(color.red, 50))

bgcolor(longOK ? color.new(color.lime, 85) : shortOK ? color.new(color.red, 85) : na)

// Compact dashboard label (last bar only)
if barstate.islast
    var label dash = na
    txt = "dRVOL " + str.tostring(drvol, format.mintick) +
          " | CMF " + str.tostring(cmf,   format.mintick) +
          " | RVOL " + str.tostring(rvol,  format.mintick) +
          " | Eff " + str.tostring(eff,   format.mintick)
    dash := na(dash) ? label.new(bar_index, drvol, txt, style=label.style_label_down, textcolor=color.white, color=col) :
                       label.set_x(dash, bar_index), label.set_y(dash, drvol), label.set_text(dash, txt), label.set_color(dash, col)

// Alerts
alertcondition(longOK,  title="Directional RVOL LONG",  message="dRVOL LONG bias")
alertcondition(shortOK, title="Directional RVOL SHORT", message="dRVOL SHORT bias")

How to use (fast)
	•	Paste into a new Pine script, add to your 1-min or 5-min chart.
	•	Tune minRVOL, thrLong/Short, and effMin/wickMax per symbol class (e.g., small caps vs. large caps).
	•	Turn on alerts for Directional RVOL LONG/SHORT.
	•	Optional: enable Use HTF with htfTf="5" when on 1-min charts.

⸻

What about other platforms?
	•	Yes: NinjaTrader (C#), Sierra Chart (ACSIL / C++), MultiCharts (EasyLanguage), and Thinkorswim (thinkScript) can all compute CMF, RVOL, and the filters above.
	•	Bonus on order-flow platforms (Sierra, Ninja + Order Flow, Quantower, ATAS, Bookmap): you can upgrade from CMF→true CVD (cumulative volume delta) using uptick/downtick or bid/ask classified ticks. That’s closer to the “is buying dominating?” ground truth than CMF.
	•	TradingView limitation: no native tick classification or bid/ask volume in Pine; CMF+RVOL is the best robust proxy on TV without broker APIs.

⸻

What you might be missing (and easy upgrades)
	•	Context gates: Add HOD proximity (e.g., close within X% of session high for longs), or require HTF trend via a slow EMA.
	•	Session handling: Treat premarket vs. RTH separately—raise minRVOL premarket to avoid noise.
	•	Structure overlay: Require price above session VWAP + anchored VWAP from ORB or prior pivot for longs (mirror for shorts).
	•	A/B thresholds: Log signals for 10–20 sessions and optimize minRVOL, effMin, and dRVOL thresholds by hour of day.

If you want, tell me your default TFs and small-cap vs. large-cap focus and I’ll ship you two tuned presets (premarket + RTH) plus an alternate version that paints bar colors on the price chart instead of a histogram.