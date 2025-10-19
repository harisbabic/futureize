Here’s the low-latency checklist I’d use for DAS. Short, practical, and profit-minded.

1) Pick the right DAS servers (don’t just leave defaults)
	•	Setup → Network Setup…
	•	Ping all available Quote (and Backup Quote) servers from inside DAS during RTH (9:30–10:00 ET). Pick the ones with the lowest, stable ping and zero packet loss.
	•	Set a backup in a different POP (e.g., NY vs CHI) in case one hiccups.
	•	If you see “Auto/Smart” selection flapping between servers, lock it to your best pair to avoid mid-session hops.
	•	Trade server: Some brokers expose multiple trade gateways; if you can see/ping them, choose the lowest/stablest. If your broker fixes it on their side (common), ask them to pin you to their East-coast gateway for U.S. equities.

Reality: quotes and orders ride different paths. A fast quote server doesn’t guarantee fast order acks if your broker’s trade gateway is slow. Optimize both.

2) Subscribe to the right feeds (reduce “consolidation lag”)
	•	For L2, take the direct books you actually use to route:
NASDAQ TotalView, NYSE OpenBook, ARCA Book (and Cboe EDGX/EDGA if you route there).
These update faster than just NBBO or a vendor-consolidated view.
	•	If you never route to, say, IEX or BATS, consider not loading those L2 books; it cuts noise and CPU.

3) Trim DAS workload so your UI stays instantaneous
	•	Fewer Level 2 windows and Time & Sales streams open concurrently. Close orphaned symbols.
	•	Keep charts efficient: no exotic custom studies on every chart; limit intraday history length; avoid tick-by-tick on 10 symbols at once.
	•	Turn off window shadows/transparencies in Windows/macOS; run DAS on the primary GPU; set Windows Power Plan = High Performance.

4) Wire/network hygiene (this often beats any app tweak)
	•	Hard-wire Ethernet (no Wi-Fi). Cat6 to your router; avoid powerline/EoP.
	•	No VPN while trading; if you must, use a low-latency split-tunnel.
	•	Put your PC on a low-contention VLAN/SSID or a router QoS profile that prioritizes your PC’s UDP/TCP to DAS’s IPs/ports.
	•	Disable or pause bandwidth hogs: cloud sync, backups, game launchers, auto-updates, Zoom in the background.
	•	NTP time sync (accurate clock helps you judge slippage): use pool.ntp.org or a reliable time source.

5) DAS settings worth toggling/checking
	•	Auto-Reconnect on Quote and Trade connections (so brief blips self-heal).
	•	Heartbeat/Status windows visible (so you notice degradation immediately).
	•	If your build exposes it: keep quote update interval at the fastest setting you can run without CPU spikes (watch Windows Task Manager during the open).

6) Routing choices (microstructure reality)
	•	You’ll usually get faster acks/fills using direct ECN routes you see on L2 (e.g., NSDQ/ARCA/EDGX) vs generic “SMART” during heavy load.
	•	If you scalp micro-moves, avoid routes known to queue slowly for that symbol/time of day. Your own fills log will tell you which ones lag.

7) Proving your setup (quick test)
	1.	During the open, open Setup → Network Setup and run Ping to your current Quote/Trade servers. Note latency and jitter.
	2.	Place a few SIM marketable limit orders to ARCA/NSDQ/EDGX and log send→ack times in the Orders/Trades window.
	3.	Change only one thing (quote server, route, or workstation network) and re-test. Keep whatever reduces both quote staleness (L2 moving smoothly) and ack time.

8) When to call DAS/broker
	•	Ask DAS support which POP (e.g., NY/NJ) is closest to their exchange lines on your package and have them whitelist / suggest the best primary + backup for your location.
	•	Ask your broker which trade gateway you’re hitting and whether they can pin you to their East-coast gateway (for U.S. equities) and confirm co-lo/peering status to the venues you route most.

⸻

TL;DR: my “set and forget” defaults
	•	Quote/Backup: Manually pick the lowest-latency NY/NJ POPs measured during the open; avoid Auto if it flips.
	•	Feeds: Subscribe to TotalView, OpenBook, ARCA (and EDGX if you use it).
	•	Routes: Prefer direct ECN (NSDQ/ARCA/EDGX) for scalps; SMART only when you need it.
	•	Machine/Net: Wired Ethernet, no VPN, QoS priority, NTP synced, no background bandwidth hogs.
	•	Platform load: Minimal L2s & T&S, efficient charts.
	•	Monitor: Keep Status/Heartbeat up; if latency spikes, fail over to your Backup Quote immediately.

If you want, tell me your city/ISP and I’ll suggest which NY POPs usually test best and the order routes that tend to be snappiest for $2–$8 small-cap names.