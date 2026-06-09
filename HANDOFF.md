# Market Terminal — Session Handoff

> Purpose: let a fresh Claude Code session (e.g. on a **Windows** PC) continue exactly where the macOS session left off. This conversation's chat history does **not** transfer across machines, but everything that matters (code, git history, decisions, gotchas, next steps) is captured here + in git.

Last synced commit: **`b5793f6`** — "Make the entire UI English". Always `git pull` first and confirm you're at or past this.

---

## 1. What this is
- A single-file, Bloomberg-style **market terminal**: live stocks, indices, crypto, FX, commodities, macro, news, screener, calendar, portfolio, alerts — all in one screen.
- **Everything lives in one file:** `index.html` (~8,860 lines: one big `<style>` ~lines 7–768, one big `<script>`). No build step, no dependencies, no framework. Vanilla HTML/CSS/JS.
- **Repo:** https://github.com/archixusa/market-terminal  ·  **Live site:** https://archixusa.github.io/market-terminal/ (GitHub Pages deploys `main` automatically on push).
- Local path on the Mac: `/Users/archixusa/Desktop/DD/work/market-terminal/` (was symlinked from `~/Desktop/work/` earlier).

## 2. Run / deploy
- **Run locally:** `cd` into the repo, then `python3 -m http.server 8000` (Windows: `py -m http.server 8000`), open `http://localhost:8000/index.html`. Must be served over http:// (not file://) so API CORS works.
- **Verify a change in-browser before pushing.** Syntax-check the script first:
  `python3 -c "import re;open('/tmp/x.js','w').write(max(re.findall(r'<script(?![^>]*src=)[^>]*>(.*?)</script>',open('index.html').read(),16),key=len))" && node --check /tmp/x.js`
- **Deploy:** commit + `git push origin main`. Pages updates in ~1 min. (Project rule: only commit/push when the user asks.)

## 3. Moving to Windows (the practical migration)
1. Install Claude Code (native): PowerShell → `irm https://claude.ai/install.ps1 | iex`. Also install **Git for Windows** (so the Bash tool + git work).
2. `claude` → log in via browser.
3. `git clone https://github.com/archixusa/market-terminal.git`
4. Re-add user-level MCP servers you use (`claude mcp add ...`) — these are machine-level, they don't travel with the repo.
5. Open the repo, tell Claude to read this HANDOFF.md + the user's `CLAUDE.md`, and continue.

**What does NOT transfer:** this chat's history (sessions are local-only; `--resume` doesn't work cross-machine), user-level `~/.claude` config/memory, and any running background workflow. **macOS-only:** the `computer-use` MCP (desktop control). The "Claude in Chrome" extension must be re-installed on Windows for in-browser testing.

## 4. Architecture & HARD-WON GOTCHAS (read before editing)
- **`LIVE` is a top-level `const`, NOT on `window`.** Reference `LIVE.loaded[sym]` / `LIVE.sourceFor[sym]` directly. `window.LIVE` is always `undefined` — this exact bug broke the whole FX tab once. (`LIVE.loaded[sym]` = `{price, pct, chg, open, high, low, prevClose, regularClose, week52High/Low?, _source}`. `LIVE.fx` = array of `{pair, rate, pct}`.)
- **No API keys required.** Only a shared Finnhub key is embedded (public repo, intentional). Twelve Data has **no** default key, so anything routed through `fetchChartTwelve`/`tdFetch` returns null for normal users — don't rely on it.
- **Charts are simulated.** Finnhub's candle endpoint is premium → `fetchChartFinnhub` returns null on the free tier, so all charts fall back to `simulateSeries(sym, range)`, which is **anchored to the live quote** (last close = live price) so the headline number is real even though the intraday shape is synthetic. Crypto charts ARE real (CoinGecko `fetchChartCoinGecko`).
- **Finnhub is rate-limited & throttled:** `fhFetch` has a 6-wide concurrency limiter + a 60-calls/min budget gate; `fetchQuoteFinnhub` caches 60s; `fetchFundamentals` caches 1h. Don't add big parallel fetch loops — you'll starve live quotes and get sim fallbacks.
- **Indices/VIX/DXY use ETF proxies** via `fetchFinnhubProxy(displaySym, etf, scale)` (e.g. SPY×10.01=SPX). Scale factors in `ETF_PROXIES` were calibrated **2026-06-06** against live ground truth; recompute `scale = indexLevel / etfPrice` if they drift.
- **Tabs:** nav-tab `<span>`s in `#topbar` call `switchTab(key)`, which shows `#overlay`, sets `#overlay-title`, and calls a render fn that fills `#overlay-body`. Shared "✕ Back to Markets" closes it. To add a tab: add the span, add an `else if` branch in `switchTab`, write the render fn.

## 5. Keyless data sources (all work in-browser, CORS-OK)
| Asset | Source / helper |
|---|---|
| US stocks / ETFs / crypto quotes | Finnhub — `fetchQuoteFinnhub`, `fetchFundamentals` |
| Crypto (live + real history) | CoinGecko — `fetchCryptoLive`, `fetchChartCoinGecko` (MATIC id = `polygon-ecosystem-token`) |
| Indices / VIX / DXY | Finnhub ETF proxy — `fetchFinnhubProxy` + `ETF_PROXIES` |
| Oil / Gas | Finnhub ETF proxy (USO/BNO/UNG) primary, EIA fallback — `COMMODITY_PROXIES`, `fetchEIAPrice` |
| Metals (XAU/XAG/XPT/XPD/HG) | gold-api.com — `fetchGoldApiPrice` |
| FX (17 pairs) | open.er-api.com — `fetchFXRates()` (Finnhub OANDA forex is premium → 403, don't use) |
| Treasury yields | treasury.gov CSV — `fetchTreasuryYields` |
| Fear & Greed | alternative.me (crypto) + CNN dataviz (stocks) |
| News | Finnhub `/news` + `/company-news` |
- **Yahoo Finance is unusable in-browser** (CORS-blocked; all free CORS proxies tested were dead/slow). Server-side curl works for one-off ground-truth checks only.

## 6. Recent work (this session)
- **Data accuracy overhaul** (`a69d8c9`): recalibrated all ETF→index scales (RUT was +34%, NKY −44%, VIX −41% off), oil via live ETF proxy (was a stale $112 vs real ~$90), FX via er-api (was all sim), MATIC→POL.
- **Chart anchoring** (`c570b3d`): the simulated chart was showing a stale base price in the headline while the live quote was correct → now re-anchors to the live quote (`_reanchorChart`).
- **Full bug audit** (`bec8f93`): fixed FX tab `window.LIVE` bug, screener fundamentals (now fetches top movers), calendar past-events filter, `loadMiniChart` null-deref, `fetchFngHistory` try/catch, WS chg/pct recompute, stale `regularClose`/`session` merge, real 52W range, CSV Infinity guard.
- **Full English UI** (`b5793f6`): converted all hardcoded Turkish strings to English (FX view, tab titles, watchlist manager, back button). The EN/TR toggle + `tr:` table remain (opt-in); default is English.

## 7. IN-FLIGHT when this handoff was written
- A background **Workflow** ("terminal-features-and-redesign") was running on the Mac to explore: a cohesive **design upgrade** (depth/motion/typography, theme-safe via the `--t-*` CSS vars) + net-new **features** (Command Palette ⌘K, "Market Pulse" overview hero, watchlist sparklines, sector-rotation/RRG view, global market-session clock, heatmap redesign). It only *produces a plan/code* (read-only); nothing was applied yet.
- **To resume on Windows:** just ask "continue the features + redesign" — re-running the exploration is cheap and safe (it doesn't edit the file). Implement features **sequentially** (one file → parallel edits would corrupt it); test each in-browser; commit per feature.

## 8. Known issues / next steps
- **Mega-cap heatmap** leaves empty space on the right (squarified-treemap flaw) — slated for redesign in the workflow above.
- **Macro tab** "Crypto Market" panel has a lot of empty space — could be enriched.
- Untracked file **`index 2.html`** exists in the repo dir (an iCloud duplicate). It's not tracked; delete it or ignore it — do NOT confuse it with `index.html`.
- Screener fundamentals populate only for the top ~24 movers (Finnhub budget) — acceptable, but a "load more" could help.

## 9. Design system (for any UI work)
- Token-based: `--t-bg/-bg2/-bg3, --t-border/-border-d, --t-text/-text-d, --t-muted/-muted-d, --t-accent (#f97316), --t-link, --t-up (#22c55e), --t-down (#ef4444)`. 4 themes via `body.theme-amber/green/light`. **All new CSS must use `var(--t-*)`** so it works across themes.
- Fonts: `Inter` for UI, `JetBrains Mono` for numbers/prices.

## 10. Project rules (from CLAUDE.md)
- Do exactly what's asked; prefer editing existing files; never save tests/working files to root; read a file before editing; never commit secrets/.env; **no `Co-Authored-By` trailer** unless `.claude/settings.json` sets `attribution.commit`; commit/push only when asked.
