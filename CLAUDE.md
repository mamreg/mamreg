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

- Benchmark/currency/country resolve from the **Yahoo ticker suffix** (`SUFFIX_*` maps).
- Call scoring (`alphaPeriods` → `gradeCall`): adjusted closes, date-driven. `abs` is the return **on
  the call**, not the share price — the price move for Buy/Hold, its **opposite for Sell**
  (`callSign`), so shorting a stock that falls 10% scores +10%. `rel = abs − benchRet` (the index's
  return stays raw), and **every** call type hits when `rel > 0`. `alphaTotal` compounds the
  per-stance `abs` for the lifetime row. Holdings use cost basis. NB `scoreCall`/`aggr`/`idxCell`
  are dead leftovers of the old flat-call model and still encode the old Sell rule — not in the
  render path; don't revive them.
- All live data goes through the Worker (Yahoo blocks direct browser calls — CORS). Don't add
  browser→Yahoo fetches.

## Verify a change

`cd worker && npx wrangler dev --port 8787 --local` + `python3 -m http.server` for the site; open
`http://localhost:<port>/#analysts/mark/calls`. Headless async (search/save) needs a real-time
capture (CDP), not `--virtual-time-budget`; pass `?token=localdevsecret` so save doesn't block on
the passphrase `prompt()`. `?ddtest=<q>` opens the ticker dropdown; `?pdftest` opens the PDF review.
