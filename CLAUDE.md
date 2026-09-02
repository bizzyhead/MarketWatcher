# MarketWatcher

A single self-contained `index.html` (~250 KB, ~4,700 lines). No build step, no
bundler, no dependencies, no framework. Editing the file *is* the build.

Live at https://marketwatcher.pages.dev, behind Cloudflare Access with a Gmail
sign-in policy (the account-wide "All Workers" application covers it).

Repo `github.com/bizzyhead/MarketWatcher`.

## Deploy

Push to `main`. Cloudflare Pages builds and promotes automatically in ~30s.

```
git add index.html
git commit -m "..."
git push
```

Build settings in Cloudflare (set once, do not change): framework preset None,
build command blank, output directory `/`, production branch `main`.

Hard-refresh (Ctrl+Shift+R) after deploying — the edge cache holds old HTML for
a minute or two.

## Hard rules

- **Keep it one file.** No external CSS or JS. Assets inline or as data URIs.
- **Plain ES5-style JavaScript.** `var`, `function`, no modules, no JSX, no
  optional chaining. It runs directly in the browser with no transpiler.
- **Never hardcode API keys.** The page is client-side; anything in the file is
  readable in view-source by anyone the Access policy admits, and lands in git
  history. Keys go in localStorage via the in-page API key button.
- **Verify frameability before building around any embed.** Check
  `x-frame-options` and CSP `frame-ancestors` with `curl -s -D - <url>` first.
  Birdeye, pump.fun and DexScreener's browse pages all block framing; only
  `dexscreener.com/{chain}/{addr}?embed=1` permits it.
- **Only CORS-open APIs.** Everything is fetched client-side. CoinGecko,
  CoinPaprika, DexScreener, RugCheck, api.blockchain.info (`?cors=true`) and
  api.alternative.me all send `Access-Control-Allow-Origin: *`.

## localStorage keys

All keys use the `mw*` prefix. Renaming one silently wipes that user's saved
data, so if you must, add a copy-old-to-new step to the `migrateKeys()` IIFE at
the top of the main script (never a bare rename).

- `mwGmeBtc`, `mwGmeEbay` — GME bitcoin cost basis, GME eBay-stake assumptions
  (live, on the GameStop tab).
- `mwCgKey`, `mwTradierKey`, `mwGlanceCollapsed` — CoinGecko key, Tradier token
  (GameStop *Options* tab), Dashboard glance-chart collapse state.
- `mwFmpKey` — reserved; fed the old Semiconductors sortable table (now a
  link-out), migrated forward in case a keyed table returns.

`migrateKeys()` at the top of the main script renames any legacy-prefixed key
onto its `mw*` name once per browser (matched by field suffix, only when the
`mw*` key is unset), then drops the stale key and clears leftovers from the
removed Portfolio / My Positions tabs.

## Layout

Primary sections (in order): Overview, News, Metals, Oil, FX, S&P 500, Nasdaq,
Bonds, Stocks, Bitcoin, Crypto. **Overview** is the default landing tab
(`class="panel active"` on `panel-overview`, `class="active"` on its primary
button; there is no JS init call). Clicking the **MarketWatcher** wordmark
(`#brandHome`) also calls `activateSection("overview")`.

Nav is 2–3 levels. Most sections are one-level bare sections: no `tabNav<Section>`
row, the primary button maps straight to `panel-<section>` — Overview, News, Oil,
S&P 500, Nasdaq, Bitcoin, and also **Metals** (`panel-gold`, one panel with four
quote tiles) and **FX** (`panel-usd`, three quote tiles; `data-section="usd"`
kept for the mapping). Two-level sections have a `tabNav<Section>` sub-tab row:
**Bonds** (Treasury Yields / Mortgage Rates) and **S&P 500** (S&P 500 / US Pulse
— US Pulse was folded in here). **Crypto** and **Stocks** add a middle "sub-main"
row (`nav.tabs.tabs-submain`, id `tabNavCrypto` / `tabNavStocks`): a sub-main
button either opens a panel directly (`data-tab` — Crypto's Dashboard, Watchlist,
Trending, Sentiment; Stocks' EBAY, Semiconductors, Purple List) or reveals its
own third-tier row (`data-group="x"` -> `tabNav<X>` — Crypto's HeatMap and
Market, Stocks' GameStop). Driven generically by `subMainSections` in the tabs
IIFE — add a section by following that shape, not by special-casing. Per-section
and per-group "last tab viewed" is remembered in memory (`lastTabInSection`,
`lastTabInGroup`), not persisted.

Panels are `<section class="panel" id="panel-*">`; only `panel-overview` is
`class="panel active"` at rest.

Key structures:

- `COINS[]` — the watchlist. `sub: true` marks the original sub-$0.01 screen
  (historical, permanent). "Major" is computed at render from
  `MAJOR_MCAP = 420e6`, never stored. `cat` uses exactly 14 fixed values.
- `TM_SCALE` — validated diverging palette for the treemap. Do not eyeball
  replacements; the two poles are reused by the hover sparkline so the colour
  always matches a number shown beside it.
- `squarify()` — hand-rolled squarified treemap (Bruls/Huizing/van Wijk).
- `OV_GROUPS` — the Overview landing grid: `[{label, tiles:[symbol…], wide?}]`.
  `buildOverview()` renders one `.ov-group` column per entry (a `wide` group is a
  2-col block); every tile is a TradingView `single-quote` widget. It also feeds
  the News picker. Editing the Overview is a one-line array change.
- `buildMetals()` / `METAL_TILES`, `buildFx()` / `FX_TILES` — the single-panel
  Metals and FX tabs: a fixed list of `[hostId, symbol]` pairs, each a
  `single-quote` widget, wired to a Refresh button and `onTabShown`.
- `positionTable()` and `simpleAsset()` are **gone** — the position tables and
  the per-metal / per-FX symbol-overview charts were all removed.

## Data sources and their limits

- **CoinGecko** — primary. Free tier is ~10 calls/min per IP and the page makes
  about six on load, so it throttles routinely. A `fetch` wrapper appends
  `x_cg_demo_api_key` when a key is stored. The stat strip falls back to
  CoinPaprika; the watchlist, market index and alt season tabs do not.
- **DexScreener** — prices the watchlist's on-chain (`dex:`) coins, which
  CoinGecko doesn't index. The Solana Memes tab (a DexScreener + RugCheck
  screen) was removed; that code is gone, but `.dex-chart` / `.detail-h` stay
  for the watchlist detail rows.
- **No free CORS stock quote API exists.** Stock/ETF/index/FX prices come from
  TradingView `single-quote` widgets (Overview tiles, Metals, FX, S&P 500,
  Nasdaq, GameStop, EBAY, Bonds Treasury). The GameStop *Options* tab can pull a
  live chain from **Tradier** (CORS-open, user token in `mwTradierKey`), falling
  back to link-outs.
- **Most price charts have been stripped.** The only `symbol-overview` chart
  left is S&P 500's SPY / 24h-CFD tabs (`sp500ChartHost`). Metals, FX, Bonds
  (Treasury), Nasdaq, GameStop and EBAY are quote tiles only. Oil, Semiconductors
  and Purple List are link-out cards (no widget at all) — Oil's `OIL_SYMBOLS` and
  Semiconductors' `SEMI_SYMBOLS` are kept solely to populate the News picker.
- **News tab** — one keyless `embed-widget-timeline` widget via `tvWidget`. A
  `<select>` (built by `newsBuildOptions()` from `OV_GROUPS` + `OIL_SYMBOLS` +
  `SEMI_SYMBOLS`, deduped) switches between market-level feeds and a per-symbol
  feed.
- **TradingView embeds** go through the shared `tvWidget(hostId, script,
  config, heightPx)` helper: concrete numeric `height` (never `"100%"`), host
  div needs `class="tradingview-widget-container"`. Widgets are built lazily on
  `onTabShown`.
- **Heavy third-party embeds are link-out cards, not iframes.** coinglass,
  kcex, chartexchange, fintel, unusualwhales, xsats and TradingView watchlist
  pages all either block framing or are too heavy to keep resident. The only
  live iframes left are the two Dashboard "glance" charts (blockchaincenter
  Pulse + cryptobubbles) — both default **collapsed** now (blanked `src`,
  `mwGlanceCollapsed` remembers a per-viewer override) — and the per-row
  DexScreener detail charts in the watchlist.

## Testing before pushing

There is no test suite. What catches real breakage:

```bash
# extract every inline <script> and syntax-check it — one unescaped quote
# inside a JS string silently kills the entire page
python3 - <<'PY'
import re, subprocess
h = open('index.html', encoding='utf-8').read()
for i, s in enumerate(re.findall(r'<script(?![^>]*\bsrc=)[^>]*>(.*?)</script>', h, re.S)):
    open(f'/tmp/s{i}.js', 'w', encoding='utf-8').write(s)
    r = subprocess.run(['node', '--check', f'/tmp/s{i}.js'], capture_output=True, text=True)
    print(i, 'FAIL' if r.returncode else 'OK', r.stderr[:300])
PY
```

Also check tag balance (`<div>` vs `</div>`, `<span>`, `<section>`, `<button>`)
after any bulk edit — past regex cleanups have left orphaned closing tags.

For visual changes, render with Playwright and actually look at the screenshot.

## Gotchas that have bitten before

- Three unescaped `"` inside JS strings once broke the whole script. `node
  --check` catches it; nothing else did.
- A detail row's `colspan` must match the column count — adding a column and
  forgetting it produces a subtly broken table.
- Git on Windows converts LF to CRLF on checkout, so a cloned `index.html` is
  ~3,900 bytes larger than the source. That gap is line endings, not content.
  Diff with `tr -d '\r'` before concluding anything.
- The clone lives inside OneDrive, which can corrupt `.git` by syncing
  mid-write. If lock-file weirdness appears, move the clone out of OneDrive.
- Attempts to "lock" an iframe to a scroll position do not work — pages reflow
  when the iframe viewport changes. This was tried and removed; a pane-height
  control replaced it.
- TradingView's free embed widgets gate many "real" index symbols — `SP:SPX`,
  `NASDAQ:NDX`, `TVC:US10Y`, `TVC:VIX`, `TVC:DXY`, `CME_MINI:ES1!` render a "only
  available on TradingView" notice. Substitutes that do render: `FRED:SP500`,
  `FRED:VIXCLS`, `FRED:DGS1/2/6MO/10/20/30`, `FOREXCOM:SPX500`, `CAPITALCOM:DXY`.
  `FRED:NASDAQ100` does not exist. ETFs (`AMEX:GLD/SLV/PPLT/CPER/SPY/QQQ`) and
  Coinbase crypto pairs always render. Nasdaq's index (NDX) and Semiconductors /
  Oil / Purple List are link-out cards precisely because of this.
- The Overview tile grid is generated from `OV_GROUPS` into `#ovGrid`; tiles are
  ~94px clipped `single-quote` widgets sized to fit one screen. `SPCX` resolves
  to SpaceX (`NASDAQ:SPCX`); `FWB:HY9` is SK Hynix's Frankfurt line (EUR).

## Deliberately not done

- Not published via the Artifact tool — its CSP blocks external iframes and
  fetches, which is most of what this page does.
- No server, no backend, no telemetry. All user data stays in localStorage.
