# Research Portal — calls by analyst

A single-page dashboard to **record and grade stock calls by analyst**, plus a light
**holdings** view grouped by country. Live at https://mamreg.github.io/mamreg/.

Static `index.html` on GitHub Pages + a tiny **Cloudflare Worker** (`worker/`) that proxies Yahoo
(live ticker search + prices) and stores your book in Cloudflare KV. No manual save/publish — edits
auto-save to the cloud. Prices are indicative (delayed ~15 min via the Worker's cache).

## Two tabs

- **Analysts** — one sub-tab per analyst (**Mark · Ziv · Aloy**, add more with ＋). Each analyst
  shows a **summary that switches scope** — a toggle **Auto · Alpha · Coverage · Total** picks whether
  the tiles reflect the timed Alpha calls, the Coverage list, or both combined (**Auto** follows the
  sub-tab you're on). Hit rate shows in **Alpha** scope only. Then:
  - **Alpha** — open calls on top, **Closed calls** table below (with exit date + exit price).
    Columns: Entry date · Ticker · Stock · Call · **Executed** (tick) · Entry cost · Return · Index return · Return vs
    index · Result · Close.
    **Returns are on the call, not the share price.** A **Sell** is scored the opposite way round to a
    Buy: sell a stock and it drops 10%, the call returns **+10%**. **vs Index** is that return minus
    the index's own return over the same dates, and any call with a positive **vs Index** is a **HIT**
    (Buy, Sell and Hold alike). The lifetime row in **History** compounds each stance the same way.
  - **Coverage List** — an editable list you key in yourself (independent of Alpha): **Entry date** ·
    Ticker · Stock · **Rating** (OW / N / UW) · **Entry price** · Last price · **Total return** · vs
    Index · **History** · **Comment**. The **Rating** is a coloured pill (OW / N / UW); click
    **Change** on a row to record a dated rating change (pick the new rating + effective date) — it's
    logged as a line in that stock's **History**, which shows the stock's return *during each stance*
    (absolute and vs index). **Entry date** = start of coverage (first stance); **Entry price** = its
    closing price; **Total return** = the stock's return since first covered (Entry price → Last
    price; the per-stance returns compound into it).
  - **Comment (💬)** — every Alpha call and coverage name has a 💬 you click to write/read the
    thesis behind it (gold when set, muted when empty).
- **Holdings** — positions grouped by **country** (HK · China · India · Taiwan · Korea · ASEAN):
  Stock · **Cost** · LTP · 1D · 1W · 1M · YTD · Return. Upload a statement **PDF** to auto-fill.

## How it fits together

```
index.html            single-file SPA (inline CSS+JS; Chart.js + pdf.js via CDN)
worker/worker.js       Cloudflare Worker: /search, /chart (Yahoo proxy) + /book (KV store)
data/book.json         SEED book (analysts·holdings·calls) — used until the KV store has data
scripts/*.py           stdlib-only fetcher + reference scorer (local auditing only, not runtime)
.github/workflows/deploy.yml   deploy the static site to Pages
```

The live book lives in the Worker's **KV** (auto-saved from the browser). `data/book.json` is only
the initial seed. Set the Worker URL as `WORKER` in `index.html`. See **`worker/README.md`** for the
one-time deploy.

### How to add a stock (no buttons, fully automatic)

- **A call:** Analysts → pick the analyst → **Alpha** → **＋ Add call** → set the **Entry date**,
  then **type in the Ticker box** and pick from the live dropdown. Name, entry cost, return, index
  return and result fill in automatically. Click **Close** (pick an exit date) to move it to the
  Closed table. Everything auto-saves to the cloud.
- **A coverage name:** Analysts → pick the analyst → **Coverage List** → **＋ Add name** → type the
  Ticker (dropdown) and set the **Rating** (OW / N / UW). Open **History** (▸) → **＋ Add rating
  change** to log a dated stance (back-date it to backtest); the return during each stance and the
  total since first covered compute automatically. Independent of your timed Alpha calls.
- **An IPO you subscribed to:** the entry defaults to the market close on the call date, which for a
  new listing is the **first day's close** rather than what you paid. Click the **✎** beside the
  entry price and key in your **subscription price**; the return then runs from your real cost. You
  can also fill it in when you log the call, and you can date the call **before** the listing day.
  A hand-set entry shows in **amber with a `*`**; reopen the ✎ and choose **Use market close** to
  undo it. The benchmark comparison still runs from the call date.
- **Executed:** tick the **Executed** box on a call once it has actually been acted on in the book
  (bought / sold), so readers can tell ideas from trades. One tick per dated call — a **Change** starts
  the new call unticked — and you can tick/untick calls logged earlier, including closed ones.
- **A comment/rationale:** click the **💬 Comment** on any coverage name (or 💬 on an Alpha call),
  type the thesis, **Save**.
- **A holding:** **Holdings** → **＋ Add holding** → type the ticker (dropdown) + cost; or upload a
  statement PDF and review.
- **Save passphrase:** the first time you edit, you're asked once for the Worker's `WRITE_TOKEN`
  (cached in your browser). That's the only prompt, ever.
- **Prices for a new ticker** appear only after you **Publish** (or Export + commit) — the fetcher
  reads tickers from the committed `book.json`, so until then the price shows "–".

### Ticker & benchmark conventions

Ticker = the **Yahoo symbol** (with exchange suffix). Benchmark, currency and country are resolved
from the suffix:

| Suffix | Country | Benchmark | | Suffix | Country | Benchmark |
|---|---|---|---|---|---|---|
| `.HK` | HK | Hang Seng | | `.KS/.KQ` | KR | KOSPI |
| `.SS/.SZ` | CN | CSI 300 | | `.SI` | ASEAN | STI |
| `.NS/.BO` | IN | Nifty 500 | | `.KL/.BK/.JK/.PS` | ASEAN | KLCI/SET/JKSE/PSEi |
| `.TW/.TWO` | TW | TAIEX | | *(bare)* | US | S&P 500 |

## PDF upload → Holdings

Fully in-browser. **Upload statement (PDF)** → `pdf.js` reads the text → a parser drafts the
holdings → a **review modal** (editable name / ticker / country / cost) → you confirm → rows are
added to Holdings. Add a Yahoo ticker so prices resolve. Optional: paste a **Claude API key** in
Settings for high-accuracy extraction on messy statements; without one a built-in heuristic parser
handles typical tabular statements. Nothing is saved until you click **Add** — then Export/Publish
to persist.

## Scoring / hit rate

For each dated call, with `end` = the next call's date, the close date, or *today* while it is open:

```
priceRet = adjClose(end)/adjClose(callDate) − 1     # how the SHARE PRICE moved
return   = priceRet for Buy and Hold, −priceRet for Sell
benchRet = the same sum on the ticker's benchmark  (never flipped)
vs Index = return − benchRet
```

The **return is on the call, not on the share price**. Sell a stock and it falls 10%, and the call
returns **+10%**. A call is a **HIT** whenever **vs Index** is positive, and that single rule covers
Buy, Sell and Hold alike. The lifetime row inside **History** compounds each stance the same way.
Returns use *adjusted* closes (splits/dividends clean) and are date-driven (the market close on the
call date). Holdings, by contrast, use your cost basis.

**The page keeps itself current.** GitHub Pages caches `index.html` for ten minutes, so a tab left open
used to keep running old code and could show numbers a later fix had already corrected. The page now
checks for a new build on load, every twenty seconds, and whenever you switch back to the tab, and
reloads itself when one lands. If you are mid-edit or have a dialog open it waits and shows a
**reload now** banner instead, so nothing you typed is lost.

**No index shown?** A call needs two price series: the stock and its benchmark. If the **vs Index**
cell reads **⚠ no index**, the stock is priced but the market's index has no usable history from the
data source. If the **entry price** reads ⚠ instead, the ticker itself did not resolve — type it again
and pick the exact listing from the dropdown, e.g. `002371` → **002371.SZ**, so the suffix sets the
benchmark and currency. China A-shares are benchmarked to the **CSI 300** via `510300.SS` and Thailand
to **SET50** via `TDEX.BK`, because the raw index symbols return only today's level. The **Philippines**
has no working local index, so `.PS` names show ⚠ no index.

> **`scripts/verify_calls.py` is legacy.** It audits the old flat-call seed in `data/book.json`
> against a frozen `data/prices.json`, not the live book in KV, and it still scores Sell the old way
> (`alpha < 0`). It is not wired into CI. Treat the app as the source of truth until it is rewritten
> for the dated-call model.

## Local development

```bash
python3 scripts/fetch_prices.py     # writes data/prices.json
python3 scripts/verify_calls.py     # audit + per-analyst scorecard
python3 -m http.server 8765         # open http://localhost:8765  (deep links: #analysts/mark/historical, #holdings)
```
