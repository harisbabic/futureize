Short answer: Yes for symbol → QuoteObj, no for ATR/RVOL on QuoteObj.
You can grab the current montage symbol and pass it into GetQuoteObj() (no hard-coded string). But ATR and RVOL are chart studies, not QuoteObj fields—pull them with GetStudyVal() from a Chart that’s on the same symbol.  ￼

Do it the right way (minimal pattern)

// 1) Get the current symbol from your named Montage
$mo  = GetWindowObj("myMontage");
$sym = $mo.GetTicker();   // returns current ticker

// 2) Inspect the Quote object for that symbol (non-modal inspector)
GetQuoteObj($sym).ShowObject();

// 3) Read QuoteObj fields (what it actually exposes)
$q = GetQuoteObj($sym);
$bid = $q.BID;  $ask = $q.ASK;  $last = $q.LAST;  $vol = $q.VOLUME;

// 4) Get ATR/RVOL from a Chart’s studies (not QuoteObj)
//    Make sure a Chart named "myChart" has ATR & RVOL studies added and named.
FocusWindow myChart;
$atr  = GetStudyVal("ATR14");     // whatever name you set in the study config
$rvol = GetStudyVal("RVOL");      // ditto

	•	GetTicker() on the Montage returns the current symbol; GetQuoteObj() accepts a variable (no quotes). QuoteObj has fields like BID/ASK/LAST/HI/LO/VOLUME/SYMBOL—no ATR/RVOL.  ￼
	•	GetStudyVal() returns the last value of a named study on the Chart. Name your studies in the Chart’s Study Configure, then reference those names. (If you call it from outside a Chart, focus the target Chart first.)  ￼
	•	Heads-up: GetStudyVal only sees bars currently displayed on that chart (per release notes). Keep the relevant range visible.  ￼
	•	RVOL is a built-in DAS study (ratio of current volume vs. average over a lookback). It can be shown in Montage UI, but scripting access is via GetStudyVal on a chart.  ￼

Quick “if” example

FocusWindow myChart;                     // ensure study context
$atr  = GetStudyVal("ATR14");
$rvol = GetStudyVal("RVOL");

if ($rvol >= 5) { Speak("RVOL five X"); }
if ($atr  >= 0.25) { Speak("ATR quarter"); }

One-liner inspector (optional)

You can skip temp vars altogether to peek what’s available:

GetQuoteObj(GetWindowObj("myMontage").GetTicker()).ShowObject();

This shows exactly what you can read from that object (and confirms ATR/RVOL aren’t there).  ￼

If you want, tell me the exact names you gave your ATR/RVOL studies and the window names you use (montage/chart). I’ll turn this into a drop-in hotkey that updates a Text Hot Button with live ATR/RVOL/BID/ASK without any popups.