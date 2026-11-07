# PROJECT SEREN — DEVELOPMENT HANDOFF v1.0

**Prepared:** November 7, 2026  
**Status:** Working browser-based Coinbase USD spot-market analysis prototype. NOT a validated or automated trading system.  
**Purpose:** Preserve verified progress, product requirements, technical design, open questions, and exact next steps so development can resume in a fresh conversation without repeating setup.

## 1. Project identity and user experience

**Name:** Seren (Project Seren). The user calls their AI assistant Seren and wants an approachable, conversational assistant with an eventual voice interface, perhaps an avatar, and richer market analysis. The initial priority is a **free, iPad-first, browser-based Coinbase breakout scanner**. Do not promise that the website is literally the ChatGPT assistant or that ChatGPT can see the scanner's live output without a separate integration.

**Visual identity:** Dark near-black/navy background `#09151d`, luminous cyan `#00f0e8`, pale foreground text `#e7f7f7`. User explicitly likes the design and envisions a sophisticated dark AI trading terminal with ranked candidates and a dedicated analysis panel. CYNEIA PROTOCOL shares the cyan aesthetic, but Seren is a separate project.

**Working style:** One verified increment at a time, with short, explicit iPad Safari/GitHub instructions. User does not know coding; provide exact insertion points or complete copy/paste files. Avoid unnecessary installations, subscriptions, risky large changes, and rabbit holes. Be honest about what is verified vs. planned.

## 2. Devices and platform constraints

- Primary development/testing device: **iPad A16, Safari**.
- Secondary HP laptop: Intel N200, 4 GB RAM, Windows 11. Previous Linux Mint XFCE live-USB experiment did not make Coinbase Advanced reliable (browser freezing). Keep Windows for manual trading; do not restart Linux troubleshooting unless asked.
- Free GitHub account and public GitHub Pages hosting; no backend or paid infrastructure yet.
- Initial embedded ChatGPT HTML preview failed with `TypeError: Load failed`; standalone GitHub Pages and direct Safari Coinbase API requests succeeded.

## 3. Authoritative URLs

- **Repository:** https://github.com/cryptowolf15633/Seren-coinbase-scanner
- **Live GitHub Pages scanner:** https://cryptowolf15633.github.io/Seren-coinbase-scanner/
- **Coinbase Exchange public products API:** https://api.exchange.coinbase.com/products
- **BTC 15-minute candles:** https://api.exchange.coinbase.com/products/BTC-USD/candles?granularity=900

GitHub Pages configured from branch `main`, folder `/(root)`. The root file must be lowercase **`index.html`**. GitHub deployment may lag a commit, and Safari can cache an older version: refresh and retest before assuming code is broken.

## 4. Trading scope and strategy hypothesis

- Exchange/product: **Coinbase Advanced spot trading**, **USD-quoted pairs**; scan all active eligible Coinbase USD spot markets, not a hand-picked watchlist. Actual exchange eligibility, liquidity, and tradability must be filtered and verified.
- **Long-only, no leverage.** User manually reviews and executes trades. No automatic order placement.
- Earlier personal BB%B (20,2) + ALMA (9, .85, 6) strategy was explicitly **set aside** to test a new Seren breakout strategy; do not quietly revert to it.
- **Experimental breakout strategy v1 (not validated):** Daily established uptrend and price above rising 50 EMA; 4-hour bullish confirmation and resistance context; 30-minute volatility compression near resistance, a completed candle closing above resistance with increased volume, and a successful retest/hold before considering an entry.
- Initial **backtesting hypotheses**, not promises: around 3% take profit and 1% stop loss; account for Coinbase trading fees, spread, slippage, and realistic order fills. Optimize/validate only after a sound historical test design. Avoid look-ahead bias and overfitting.
- Refresh near each 30-minute candle close; eventual 24/7 monitoring/alerts require a reliably running service or equivalent scheduled infrastructure. A static GitHub Pages page cannot run unattended when Safari is closed.

## 5. Verified working features (as of Nov 7, 2026)

**Test 1 — Market discovery: PASSED.** Browser fetches `/products`, filters for `quote_currency === 'USD'`, `status === 'online'`, and excludes `trading_disabled`, `cancel_only`, `limit_only`, `auction_mode`; displays sorted pair names. A screenshot showed **391 active USD pairs** at that moment. The number changes over time. These filters are an initial eligibility screen, not a full liquidity/tradability guarantee.

**Test 2 — Historical candles: PASSED.** Direct Safari request to BTC-USD candles with `granularity=900` returned numerical data. Coinbase Exchange candles do **not** support `granularity=1800` directly (the direct test returned `unsupported granularity`). Supported 15-minute candles use `900` seconds. Candle rows are `[time, low, high, open, close, volume]`.

**Test 3 — 30-minute candle construction: PASSED for BTC-USD.** Hosted page retrieves BTC 15-minute candles, excludes incomplete intervals, sorts by timestamp, pairs two consecutive 15-minute candles on aligned 30-minute boundaries, and computes: open = first open; close = second close; high = max highs; low = min lows; volume = sum. It refuses to form a 30-minute candle when one half is absent. Screenshot showed **174 completed 30-minute candles**. The latest 30-minute candle is completed, not the currently forming candle.

**Test 4 — Relative volume: PASSED for BTC-USD.** Compares latest completed 30-minute candle volume against the **preceding 20 completed 30-minute candles** (not including latest): `RVOL = latest.volume / average(previous20.volume)`. An initial `>= 1.5x` threshold labels `ELEVATED VOLUME`; otherwise `Normal volume`. Screenshot: latest volume **16.1787 BTC**, preceding-20 average **63.8820 BTC**, RVOL **0.25x**, `Normal volume`. Threshold is exploratory, not a validated signal.

**Test 5 — Rolling resistance: PASSED for BTC-USD.** `resistance = max(high of preceding20)`; `distanceToResistance = ((latest.close - resistance) / resistance) * 100`; breakout status is `CLOSED ABOVE RESISTANCE` only if latest completed candle closes above the prior-20 high. Screenshot: resistance **$76,086.99**, latest close **$75,989.63**, distance **-0.13%**, `No confirmed breakout`. This is a simple rolling-high proxy, not proof of a repeatedly tested support/resistance zone. The screenshots show plausible calculations and output; independent data validation/backtesting remains pending.

**Important:** Latest tested page showed all five features working. No reason to replace the entire file or reset the repository.

## 6. Current code architecture and key logic

The live prototype is a single root `index.html` containing HTML, CSS, and browser JavaScript. There are two independent buttons:

1. `Scan Coinbase USD Pairs`: fetches and displays filtered USD product list.
2. `Test BTC Candles`: fetches `https://api.exchange.coinbase.com/products/BTC-USD/candles?granularity=900`, builds completed 30-minute OHLCV candles, and displays latest candle, preceding-20 average volume, RVOL, volume label, rolling-high resistance, distance to resistance, and breakout status.

Relevant code fragments (the repository's `index.html` is the authoritative source):

```js
const previous20 = candles30.slice(-21, -1);
const averageVolume = previous20.reduce((sum, c) => sum + c.volume, 0) / 20;
const relativeVolume = averageVolume > 0 ? latest.volume / averageVolume : null;
const volumeSignal = relativeVolume === null ? 'Unavailable' : relativeVolume >= 1.5 ? 'ELEVATED VOLUME' : 'Normal volume';
const resistance = Math.max(...previous20.map(c => c.high));
const distanceToResistance = ((latest.close - resistance) / resistance) * 100;
const breakoutStatus = latest.close > resistance ? 'CLOSED ABOVE RESISTANCE' : 'No confirmed breakout';
```

Candles from Coinbase may omit intervals without trades. Before using this for live signals, confirm that the previous 20 candles are genuinely **contiguous** 30-minute intervals; the current logic requires both halves of each 30-minute candle but does not explicitly ensure the 20 selected historical 30-minute candles are contiguous. Also verify API history limits, timezone alignment, stale data, error handling, and rate limits.

## 7. Immediate next development steps (in recommended order)

1. **Preserve documentation:** Save this handoff as `docs/HANDOFF.md` in the existing GitHub repository. Leave `index.html` unchanged. Optionally keep a local copy in Files/iCloud Drive. Create a ChatGPT Project called `PROJECT SEREN` and put future conversations there if available.
2. **Candle continuity/data-quality check:** Verify the latest candle is the expected most recently completed half-hour and that the prior 20 candles are contiguous. Surface stale/missing data rather than producing misleading signals. This should come before relying on a live scanner.
3. **Volume + price breakout combined condition:** Show whether latest completed close is above prior-20 high **and** RVOL is above the experimental threshold; do not call this a trade entry.
4. **Compression metric:** Define and test a reproducible measure of narrowing 30-minute ranges/volatility near resistance, with enough historical context.
5. **4-hour and daily trend filters:** Fetch appropriate supported candle granularities; build/verify 4h data if needed; obtain sufficient daily history to compute rising EMA50. Avoid comparing incomplete higher-timeframe candles.
6. **Retest detection:** Specify objective retest window, tolerance, invalidation, and close-based confirmation. No hindsight-based labels.
7. **Multi-pair scanning:** Start with a small controlled batch, then expand to all eligible USD markets while respecting API rate limits, paging/history limits, errors, concurrency, liquidity filters, and UI responsiveness on iPad.
8. **Backtest/forward-test:** Include survivorship bias, realistic execution, fees, spread, slippage, and out-of-sample testing. No profitability claims until demonstrated.
9. **UX and alerts:** Ranked candidates, explanatory panels, clear timestamps, status/errors; later consider reliable 24/7 alerts with appropriate hosting. Voice/avatar and richer AI integrations are longer-term ambitions, not existing capabilities.

## 8. Organization plan

Use **one GitHub repository** for code and docs:

```text
Seren-coinbase-scanner/
├── index.html                 # Working hosted scanner; do not move
└── docs/
    ├── HANDOFF.md             # This living handoff (create first)
    ├── ROADMAP.md             # Milestones and priorities (later)
    └── TEST_LOG.md            # Verified results and screenshots (later)
```

GitHub can create the `docs/` folder when adding a new file named `docs/HANDOFF.md`. Avoid creating empty folders. Commit after each verified feature with descriptive messages; use GitHub history to recover prior versions. Update `HANDOFF.md` after each milestone; add date and version number. Keep experimental code changes separate from documentation changes where practical. If a ChatGPT Project is used, link the repository and keep the handoff file available there; a ChatGPT Project is organizational convenience, not a replacement for independent backup.

## 9. Instructions for a future assistant

Continue Project Seren from this document and the **actual current GitHub `index.html`**, not from a guessed earlier version. Ask the user for the file or relevant snippet if code access is unavailable. First verify the most recent working test and avoid overwriting code blindly. Give **one concrete iPad-friendly step at a time**, with precise code insertion points and test expectations. Explain technical limits honestly. Maintain long-only/manual-execution scope, free-first approach, cyan/dark style, and friendly but grounded collaboration. Never treat RVOL, rolling-high proximity, or breakout labels alone as a validated buy recommendation.

**Next action on handoff completion:** Create `docs/HANDOFF.md` on GitHub; then return to candle continuity/data-quality verification before expanding breakout logic.
