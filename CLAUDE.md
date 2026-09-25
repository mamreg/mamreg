# CLAUDE.md — Research Portal (mamreg)

Dashboard to **record & grade stock calls by analyst**, plus holdings by country. Static
`index.html` on GitHub Pages (`mamreg/mamreg`) **+ a Cloudflare Worker** (`worker/`) that proxies
Yahoo (live search + prices) and stores the book in KV.

## Architecture

- **`index.html`** — the whole app: one file, inline CSS + vanilla JS, Chart.js + pdf.js via CDN.
  `WORKER` constant = the Cloudflare Worker base URL (auto-uses `http://127.0.0.1:8787` on localhost).
  **No buttons** — edits auto-save to the Worker/KV (debounced `pushBook`, one-time `WRITE_TOKEN`
  passphrase). Prices are fetched **live** from the Worker (`fetchChart`→`chartToData` builds the
  same `PRICES{quotes,closes}` the old static feed did); cached in localStorage ~15 min.
- **`worker/worker.js`** — routes `/search`, `/chart` (Yahoo proxy, CORS, ~15-min edge cache) and
  `GET/PUT /book` (KV `BOOK_KV`, PUT needs `Bearer WRITE_TOKEN`). Deploy per `worker/README.md`.
- **`data/book.json`** — SEED only (used until KV has data). **`scripts/*.py`** are local-audit
  only (not in the runtime path). `.github/workflows/deploy.yml` just deploys the static site.

## Two tabs

- **Analysts** → per-analyst summary + **Alpha** (open table + Closed table below) and
  **Coverage List**. Alpha columns: Entry date · Ticker(search dropdown) · Stock(auto) · Call · Executed(tick, one per dated call — `history[].executed`) ·
  Entry cost · Return · Index return(+bench tag) · Return vs index · Result · Close.
- **Holdings** → grouped by country (HK/CN/IN/TW/KR/ASEAN); Cost · LTP · 1D/1W/1M/YTD · Return.
  PDF upload → heuristic parse → review modal.

## Invariants (don't quietly change)

- Benchmark/currency/country resolve from the **Yahoo ticker suffix** (`SUFFIX_*` maps). Before adding
  or changing a benchmark, CHECK IT HAS HISTORY: Yahoo answers `/chart?range=2y` for some index
  symbols with a **single** point (today's level), which silently kills every index return. Known
  today-only symbols: `000300.SS`, `399300.SZ`, `000016.SS`, `399006.SZ`, `^SET.BK`, `PSEI.PS`,
  `^PHDOW`. A-shares therefore use `510300.SS` (Huatai-PB CSI 300 ETF) and Thailand `TDEX.BK`
  (ThaiDEX SET50) — same currency, real history, and total-return like the stock's adjusted closes.
  **Philippines (.PS) has no working local-currency index** — it still maps to `PSEI.PS` and shows
  the "⚠ no index" marker; the only live alternative is `EPHE`, which is USD and would mix FX into
  alpha. `vsIndexCell` renders that marker whenever a call is priced but its benchmark is not.
- Row order (`ideaOrder(byClose)` returns the comparator): **★ high-conviction first**, then a date
  ascending — the open table uses the displayed **Entry date** (`alphaCurrent().date`, so
  `ideaOrder(false)`), the **Closed** table uses **`closeDate`** (`ideaOrder(true)`). Rows missing
  that date sink to the bottom of their block; ties break by name. Sort the `.filter()` copies only
  — never reorder `BOOK.calls` itself.
- Entry price: the adjusted close on the call date, **unless** that dated call carries `entryPx` — an
  IPO/subscription price keyed in by hand (`entryPxModal`, the ✎ beside the cell, or the optional
  field in the log-call modal). A valid `entryPx` (> 0) always wins; anything else falls back to the
  close. The **benchmark leg still runs from the call date**, so the index return is unaffected.
  `alphaPeriods` returns `entryPx` + `manualEntry` so the cell can flag a hand-set price.
- Call scoring (`alphaPeriods` → `gradeCall`): adjusted closes, date-driven. `abs` is the return **on
  the call**, not the share price — the price move for Buy/Hold, its **opposite for Sell**
  (`callSign`), so shorting a stock that falls 10% scores +10%. `rel = abs − benchRet` (the index's
  return stays raw), and **every** call type hits when `rel > 0`. `alphaTotal` compounds the
  per-stance `abs` for the lifetime row. Holdings use cost basis. NB `scoreCall`/`aggr`/`idxCell`
  are dead leftovers of the old flat-call model and still encode the old Sell rule — not in the
  render path; don't revive them.
- **Staying on the current build:** Pages serves `index.html` with `max-age=600`, so an open tab keeps
  running old code and can show numbers a later fix has already corrected. `checkVersion()` HEADs the
  page (`cache:"no-store"`) on load, on every 20s poll and on `visibilitychange`, comparing the ETag
  (Last-Modified on a plain local server). On a change it reloads — but only when `_dirty` is false and
  no modal is open, with a `sessionStorage` guard against loops; otherwise it shows `#updBanner`.
- All live data goes through the Worker (Yahoo blocks direct browser calls — CORS). Don't add
  browser→Yahoo fetches.

## Verify a change

`cd worker && npx wrangler dev --port 8787 --local` + `python3 -m http.server` for the site; open
`http://localhost:<port>/#analysts/mark/calls`. To observe anything on a timer (the version check,
polling), do NOT use `--virtual-time-budget` — it exhausts before real network replies. Launch plain
`--headless=new` with no capture flag, mutate/kill from a background script, then read the **server
access log** (`python3 -u -m http.server`): one GET per load, one HEAD per check, so a second GET right
after a HEAD proves the self-reload. Check the port is free first — a stale server silently keeps 8000. Headless async (search/save) needs a real-time
capture (CDP), not `--virtual-time-budget`; pass `?token=localdevsecret` so save doesn't block on
the passphrase `prompt()`. `?ddtest=<q>` opens the ticker dropdown; `?pdftest` opens the PDF review.
