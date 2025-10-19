Short punchline: your Signal v4.2 already nails magnitude and structure (Quiet-Regime ceiling → ACCEL/CROSS triggers, plus ATR/RVOL/EMA gates). What it doesn’t yet do is give RVOL a directional brain. The dRVOL idea I gave (CMF × RVOL, + body-efficiency & wick sanity) is additive, not redundant—it slots in as a new gate that reduces false “high-RVOL churn” and exhaustion spikes.

What’s different vs your Signal v4.2
	•	You have:
	•	M2 Quiet-Regime ceiling (percentile/top-X/peak×factor).
	•	M3 Composite window on CTF (kWindow, NetBody/SumBodies, nearTop/nearBot).
	•	M4 Raw triggers = ACCEL (|H| expansion, near extreme) > CROSS, single-side precedence.
	•	M5 Gates = RVOL (magnitude only), ATR/expansion, Structure (EMA bias), Trend (EMA stack), with bullGatesOK/bearGatesOK.
	•	You don’t yet have:
	•	Directional RVOL (CMF × RVOL) to tell who is winning when RVOL is big.
	•	Body-efficiency & wick sanity filters to avoid high-RVOL dojis / blow-off bars.
	•	VWAP alignment (optional) to suppress mean-reversion traps.

How they help each other
	•	Your M2–M4 decides “when a move is special.”
	•	A directional RVOL gate decides “if that special move is actually buy- (or sell-) dominated.”
	•	Result: fewer beacons in chop and better follow-through quality on the ones that remain.

⸻

Drop-in upgrade for v4.2 (add as “MODULE 6 — Directional Volume”)

Paste this below M5 and change one line in your gate combiner. Uses only built-ins you already use (request.security, ta.sum, etc.).

//──────────────────────────────────────────────────────────────────────────────
// MODULE 6 — Directional Volume (CMF × RVOL) + Eff/Wick + VWAP (optional)
//──────────────────────────────────────────────────────────────────────────────
versionComment := "M6 – Directional RVOL online"
grpM6 = "M6 — Directional Volume"

useDirVol     = input.bool(true,  "Use Directional RVOL gate", group=grpM6)
cmfLen        = input.int(20,     "CMF length", minval=2, group=grpM6)
useM6_STF     = input.bool(true,  "Feed = STF (off: CTF)", group=grpM6)
minDirLong    = input.float( 2.0, "dRVOL ≥ for LONG", step=0.1, group=grpM6)
minDirShort   = input.float(-2.0, "dRVOL ≤ for SHORT", step=0.1, group=grpM6)
effMin        = input.float(0.55, "Min body efficiency (|C−O|/(H−L))", step=0.01, group=grpM6)
wickMax       = input.float(0.40, "Max wick/body ratio", step=0.01, group=grpM6)
useVWAPgate   = input.bool(false, "Require VWAP alignment", group=grpM6)

m6tf = useM6_STF ? stfTf : ctfTf_eff
[h_, l_, c_, o_, v_] = request.security(syminfo.tickerid, m6tf, [high, low, close, open, volume], barmerge.gaps_off, barmerge.lookahead_off)
rng  = math.max(h_ - l_, syminfo.mintick)
clv  = rng > 0 ? (((c_ - l_) - (h_ - c_)) / rng) : 0.0
cmf  = ta.sum(clv * v_, cmfLen) / math.max(ta.sum(v_, cmfLen), 1.0)

// Reuse M5 RVOL calc at the same TF for consistency
volAvg_M6 = useM6_STF ? (rvolAvgType=="SMA" ? request.security(syminfo.tickerid, stfTf, ta.sma(volume, rvolLen))
                                            : request.security(syminfo.tickerid, stfTf, ta.ema(volume, rvolLen)))
                      : (rvolAvgType=="SMA" ? request.security(syminfo.tickerid, ctfTf_eff, ta.sma(volume, rvolLen))
                                            : request.security(syminfo.tickerid, ctfTf_eff, ta.ema(volume, rvolLen)))
volNow_M6 = request.security(syminfo.tickerid, m6tf, volume)
rvol_M6   = volAvg_M6 > 0 ? (volNow_M6 / volAvg_M6) : na
drvol     = cmf * rvol_M6

// Body efficiency & wicks on M6 TF
body = math.abs(c_ - o_)
uw   = h_ - math.max(c_, o_)
lw   = math.min(c_, o_) - l_
uwr  = body > 0 ? (uw / body) : 999.0
lwr  = body > 0 ? (lw / body) : 999.0
eff  = body / rng

// VWAP gate on M6 TF (optional)
vwapM6   = request.security(syminfo.tickerid, m6tf, ta.vwap)
vwapLong = useVWAPgate ? (c_ > vwapM6) : true
vwapShort= useVWAPgate ? (c_ < vwapM6) : true

dirBullOK = useDirVol ? (not na(drvol) and drvol >=  minDirLong and eff >= effMin and uwr <= wickMax and vwapLong)  : true
dirBearOK = useDirVol ? (not na(drvol) and drvol <=  minDirShort and eff >= effMin and lwr <= wickMax and vwapShort) : true

// (Optional) tiny HUD line
if barstate.islast
    label.new(bar_index, na, "M6 dRVOL="+str.tostring(drvol, "#.##")+"  CMF="+str.tostring(cmf, "#.##")+"  Eff="+str.tostring(eff, "#.##"),
              xloc=xloc.bar_index, yloc=yloc.price, color=color.new(color.navy, 85), textcolor=color.white, size=size.tiny)

Now change one line in your gate combiner (M5)

Find:

bullGatesOK = atrOK and rvolOK and structBullOK and trendBull
bearGatesOK = atrOK and rvolOK and structBearOK and trendBear

Replace with:

bullGatesOK = atrOK and rvolOK and dirBullOK and structBullOK and trendBull
bearGatesOK = atrOK and rvolOK and dirBearOK and structBearOK and trendBear

That’s it. You keep your ACCEL/CROSS + ATR/Structure/Trend DNA, but you only “greenlight” when volume’s abnormal AND skewed to your side with sane bar anatomy.

⸻

Quick presets (tune later)
	•	Premarket (thin): minRVOL 5–8, minDirLong 3.0 / minDirShort −3.0, effMin 0.60, wickMax 0.35, useVWAPgate = true.
	•	RTH (liquid): minRVOL 2–3, minDirLong 2.0 / minDirShort −2.0, effMin 0.55, wickMax 0.40, useVWAPgate = false/true by symbol.

⸻

What you’re not asking (but should)
	1.	Validation loop: How often does dRVOL-gated ACCEL beat ACCEL alone on follow-through over the next N bars? Define N (e.g., 3/5/10), and measure per hour-of-day.
	2.	Side-effects: Does dRVOL filter out too many winners in low-float runners? You might need lower dRVOL in the first 3 mins after a halt resume.
	3.	Context gates:
	•	HOD/LOD proximity (e.g., within 0.5–1.0% for continuation plays).
	•	SSR state (treat signals differently when SSR is on).
	•	Spread/float filters for your $2–$20 small-cap universe.
	4.	Exit logic: Does dRVOL flip negative before your current exit? If yes, test early-exit on dRVOL cross of 0 (or CMF cross) vs. your existing stop/target logic.
	5.	Multi-TF agreement: Is 1-min dRVOL > 0 while 5-min dRVOL < 0? Decide which dominates (I usually require same-sign or skip).
	6.	Deployment: Which pieces must live in DAS (execution) vs TradingView (signal visualization)? Plan the mapping now.

⸻

Bottom line
	•	Your v4.2 is strong on range/accel+structure.
	•	Adding directional RVOL (CMF×RVOL) + bar anatomy + optional VWAP is the cleanest upgrade to reduce noisy high-RVOL signals.
	•	The patch above is minimal and doesn’t fight your module layout—think of it as M6 that plugs into bullGatesOK/bearGatesOK.

If you want, I’ll re-emit a ready-to-paste v4.3 with this M6 block integrated, plus a HUD row (“DIR: ✓/✖”) and alertconditions that only fire when the full gate stack (incl. M6) passes.