# Prompt: FootJoy AU Competitive Price Tracker

## What this is

A daily price tracker for golf shoe SKUs across three Australian retailers (Golf Box, On Course, FootJoy AU), surfaced as a modern filterable dashboard hosted on GitHub Pages. The repo runs its own scraping schedule via GitHub Actions, commits the price history as JSON, and the static dashboard reads that JSON.

Source spreadsheet: `Price Tracker Sheet.xlsx`
- Sheet `Sheet1` columns: `Category`, `Brand`, `Product Name`, `Retailer`, `URL` (9 rows today)
- Sheet `Where to find price`: one row per retailer with a snippet of page source showing where the price lives

Categories in the current sheet: `Premium Performance Spikeless`, `Performance Spikeless`.
Brands: Ecco, Under Armour, FootJoy, Adidas.
All prices are AUD. Currency code should still be stored per row to stay future-proof.

## The Prompt (copy/paste this when ready)

Build a daily competitive price tracker for FootJoy Australia, run entirely from a single GitHub repo. Use `Price Tracker Sheet.xlsx` (committed to the repo) as the source of truth for what to track.

### Repo layout to produce

```
/
  Price Tracker Sheet.xlsx           source of truth (products + retailer selectors)
  scripts/scrape_prices.py           daily scraper
  data/prices.json                   append-only price history
  data/last_run.json                 last run status + per-URL errors
  data/archive/YYYY.json             rolled-off history (older than 18 months)
  .github/workflows/scrape.yml       daily GitHub Actions cron
  index.html                         single-file dashboard
  assets/styles.css                  optional split if index.html gets big
  assets/app.js                      optional split if index.html gets big
  README.md
```

### Scraper requirements (`scripts/scrape_prices.py`)

- Read `Price Tracker Sheet.xlsx`. From `Sheet1` collect every (Category, Brand, Product Name, Retailer, URL) row. From `Where to find price` collect each retailer's selector hint.
- Selector logic per retailer (derived from the sheet's sample source):
  - **Golf Box** (`golfbox.com.au`): parse JSON-LD `<script type="application/ld+json">` and read `offers.price` plus `offers.priceCurrency`. Fall back to a regex on `"price":\s*"([0-9.]+)"` if JSON-LD parsing fails.
  - **On Course** (`oncoursegolf.com.au`): same JSON-LD pattern as Golf Box.
  - **FootJoy AU** (`footjoy.com.au`): parse HTML and select `.product-price .price-sales`, strip whitespace and the `$` symbol. If the page renders a sale price, prefer the sale price and also record the original price as `price_was`.
- Handle JS-rendered pages by trying `requests` + BeautifulSoup first; if the price node is missing, retry with Playwright headless Chromium.
- Be polite: 2-3 second delay between requests to the same domain, realistic User-Agent, follow redirects, 30 second timeout.
- Output schema for each scrape (one row per product per retailer per day):

```json
{
  "date": "2026-04-27",
  "timestamp_utc": "2026-04-27T13:00:14Z",
  "category": "Premium Performance Spikeless",
  "brand": "FootJoy",
  "product": "Pro/SL",
  "retailer": "Golf Box",
  "url": "https://www.golfbox.com.au/...",
  "price": 159.99,
  "price_was": null,
  "currency": "AUD",
  "on_sale": false,
  "in_stock": true,
  "status": "ok",
  "error": null
}
```

- Append to `data/prices.json`. If an entry for the same (date, product, retailer) already exists, overwrite it rather than duplicating. Sort by date asc on save.
- Write `data/last_run.json` with: timestamp, total URLs, success count, failure count, list of failed URLs with their error messages.
- Exit code 1 if more than 30% of URLs fail (so the workflow surfaces breakage), but still commit whatever data was successfully scraped before exiting.

### GitHub Actions workflow (`.github/workflows/scrape.yml`)

- Triggers: `schedule: cron: "0 22 * * *"` (08:00 AEST = 22:00 UTC), plus `workflow_dispatch`.
- Steps: checkout, set up Python 3.12, `pip install` requirements, `playwright install chromium`, run scraper, commit `data/prices.json` and `data/last_run.json` back to `main` with author `github-actions[bot]`. Permissions block must grant `contents: write`.
- Upload the scraper log as a workflow artifact for debugging.

### Dashboard (`index.html`)

Single-file HTML/CSS/JS, no build step. Loads `data/prices.json` and `data/last_run.json` via fetch.

Visual style: clean and modern. Off-white page background (#FAFAFA), white cards with soft shadow and rounded 12px corners, sans-serif (Inter or system font stack), generous whitespace, subtle grid lines on charts. Dark mode toggle in the header. Mobile responsive, charts stack vertically below 768px.

Structure:

1. Header: "FootJoy AU Price Tracker" title, "Last updated DD MMM YYYY HH:MM AEST" pulled from `last_run.json`, dark mode toggle.
2. Toolbar: multi-select chips for Retailer (Golf Box / On Course / FootJoy), multi-select chips for Brand, segmented date range (7d / 30d / 90d / All). Filter state should update charts in place without a full rerender, and persist in the URL hash so the view is shareable.
3. One chart card per Category (in the order they appear in the spreadsheet).
4. Inside each chart card:
   - Line chart, X axis = date, Y axis = price (AUD), one line per (Product, Retailer) combination so that the same product across multiple retailers is directly comparable.
   - Legend below the chart with each line, color swatch, current price, and percent change over the selected window (green if down, red if up; this is from FootJoy's perspective so cheaper competitors are not necessarily good news, just label the indicator neutrally as "delta").
   - Hover tooltip: date, exact price, retailer, brand, on_sale flag, in_stock flag.
   - "Export CSV" button that exports the visible window for that chart.
5. Footer: link to the GitHub repo, link to the spreadsheet, link to the last workflow run.

Use Chart.js v4 from a CDN. No npm.

Banner behavior: if `last_run.json` shows the latest scrape failed entirely or is more than 36 hours old, show a yellow warning banner at the top of the page.

### README

Cover: how to add a new product (edit the xlsx, push), how to add a new retailer (add a row to `Where to find price`, then update `scripts/scrape_prices.py` with the corresponding parser branch), how to run the scraper locally, how to enable GitHub Pages, and where the cron is defined if you want to change the schedule.

### Acceptance test

After setup I should be able to: clone repo, run `python scripts/scrape_prices.py` locally and see prices populate; push to GitHub, manually dispatch the workflow, see a commit appear on `main` updating `data/prices.json`; open the published Pages URL and see two category charts with one line per product/retailer combo, filterable by retailer and brand, with a working dark mode toggle.

---

## Open questions to answer before kicking this off

1. **Backfill**: dashboard starts empty on day one. Acceptable, or do you want to seed with prices captured manually first? 
2. **Sale price handling**: when a retailer shows both "was" and "now", should the chart line follow the sale price (so promo dips show up) or the regular price (so the underlying RRP trend is clean)? My default in the prompt is "sale price as the line, was-price stored alongside for hover" but tell me if you'd rather invert.
3. **Out-of-stock**: skip the day, hold the previous price flat, or break the line? Default in the prompt is to record `in_stock: false` with the last seen price so the line stays continuous and the tooltip flags it.
4. **Cross-retailer same-product lines**: I assumed you want to see the same shoe at different retailers as separate lines on the same chart, since price competition across retailers is the whole point. Confirm.
5. **Privacy**: GitHub Pages is public. The repo will publish a list of competitor URLs and prices. If that's sensitive, make the repo private and use a paid GitHub plan to keep Pages private, or render the dashboard to a non-public host.
6. **Scraping resilience**: `golfbox.com.au` and `oncoursegolf.com.au` look like fairly conventional ecommerce, JSON-LD parsing should hold up. `footjoy.com.au` is the one to watch since the selector is HTML-class based, which breaks more easily on site updates. If selectors drift, the failure path is "edit the parser, push, done" rather than anything automated.
7. **Cadence**: daily at 08:00 AEST is my default. Tell me if you want twice-daily, weekly, or only on weekdays.
8. **Currency display**: AUD with `$` prefix; if you want explicit `A$` to disambiguate, say so.

## Suggested next message

Once you've answered the open questions, tell me to "build it per `PROMPT.md` with these decisions: [list]" and I'll lay down the repo structure, the scraper, the workflow, and the dashboard.
