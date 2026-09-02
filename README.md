# MarketWatcher

Single-file markets dashboard. `index.html` is the entire site — no build step,
no dependencies, no bundler. Covers crypto, individual stocks (GameStop, EBAY,
a semiconductors watchlist), gold, the S&P 500, US macro (FRED), Treasury and
mortgage rates, and a personal position tracker.

Live at **https://marketwatcher.pages.dev** (behind a Cloudflare Access Gmail
sign-in policy).

## Deploying

This repo is connected to Cloudflare Pages. Any push to `main` deploys
automatically to production.

**Build settings in Cloudflare (set once at project creation):**

| Field                  | Value        |
| ---------------------- | ------------ |
| Framework preset       | None         |
| Build command          | *(blank)*    |
| Build output directory | `/`          |
| Production branch      | `main`       |

## Updating

Replace `index.html` and commit. That is the whole workflow.

Via git:
```
git add index.html && git commit -m "update dashboard" && git push
```

Via the GitHub website: edit `index.html` in place, commit to `main`, and
Cloudflare builds and promotes it within ~30 seconds. Hard-refresh
(Ctrl+Shift+R) afterwards — the edge cache holds old HTML for a minute or two.

## Notes

- All user-entered data (positions, cost basis, API keys, per-chart collapse
  state) lives in the browser's localStorage. Nothing personal is stored in
  this repo.
- The site is served behind Cloudflare Access — the repo being public or
  private does not change who can view the deployed page, but a public repo
  does mean anyone can read the HTML source, so **no secrets in the file**
  (API keys are entered in-page and kept in localStorage).
- Third-party data comes from CORS-open APIs where possible (CoinGecko,
  CoinPaprika, FINRA, TradingView widgets, FRED images); heavy or
  frame-blocking sources (coinglass, kcex, unusualwhales, etc.) are link-outs.
- See `CLAUDE.md` for the internal architecture, hard rules, and gotchas.
