# FootJoy AU Competitive Price Tracker

Daily scrape of golf shoe prices across Australian retailers (Golf Box, On Course, FootJoy AU), surfaced as a static dashboard hosted on GitHub Pages. Everything runs from this repo: GitHub Actions does the daily scrape, commits the data file back to the repo, and the dashboard reads that file.

```
.
├── Price Tracker Sheet.xlsx     source of truth for products + retailer selectors
├── scripts/scrape_prices.py     daily scraper (run locally or via Actions)
├── data/
│   ├── prices.json              real price history (populated by Actions)
│   ├── prices.example.json      30 days of demo data so the dashboard renders before the first real run
│   └── last_run.json            timestamp + per-URL success/failure summary
├── .github/workflows/scrape.yml daily cron at 22:00 UTC (08:00 AEST)
├── index.html                   the dashboard
├── requirements.txt
└── README.md
```

## First-time setup

1. Push this repo to GitHub.
2. **Settings → Pages → Source:** "Deploy from a branch", select `main` and root (`/`). Save. After the first publish the dashboard is live at `https://<your-username>.github.io/<repo>/`.
3. **Settings → Actions → General → Workflow permissions:** "Read and write permissions" so the daily job can commit price data back to the repo.
4. **Actions tab → "Scrape prices daily" → Run workflow** to trigger the first scrape manually. After it completes, `data/prices.json` will have one row per product per retailer for today, and the dashboard will start showing real data instead of the demo set.

The cron is set to 22:00 UTC daily, which is 08:00 AEST. To change it, edit `.github/workflows/scrape.yml`.

## Running the scraper locally

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python -m playwright install chromium
python scripts/scrape_prices.py
```

The script reads `Price Tracker Sheet.xlsx`, scrapes each URL, and updates `data/prices.json` and `data/last_run.json` in place. Same-day entries are overwritten rather than duplicated, so it is safe to re-run.

## Adding a product

1. Open `Price Tracker Sheet.xlsx`, go to `Sheet1`, add a new row with `Category`, `Brand`, `Product Name`, `Retailer`, and `URL`.
2. Commit the spreadsheet. The next scheduled run picks it up automatically.

## Adding a new retailer

1. Add a row to the `Where to find price` tab in the spreadsheet with the retailer name and a sample of the product page source so future-you remembers where the price lives.
2. In `scripts/scrape_prices.py`, add a parser function for the retailer (e.g. `parse_<retailer>(html)`) that returns `{"price": float, "currency": str, "in_stock": bool, "on_sale": bool, "price_was": float | None}`.
3. Register it in the `RETAILER_PARSERS` dict at the bottom of the parsers section, keyed by the retailer name as it appears in `Sheet1` (lowercase, spaces preserved).
4. Add a row to the spreadsheet pointing at a real product URL on that retailer to confirm it scrapes cleanly.

## How the parsers work today

| Retailer  | Method                                                                                  | Failure mode                                              |
| --------- | --------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| Golf Box  | `<script type="application/ld+json">` → `offers.price`, fallback regex on `"price":...` | If the site moves to client-side-only rendering, Playwright fallback kicks in. |
| On Course | Same JSON-LD pattern as Golf Box                                                        | Same.                                                     |
| FootJoy AU | `.product-price .price-sales` (and `.price-standard` when on sale)                     | If FootJoy changes class names, edit the selector in `parse_footjoy`. |

When the static-HTML pass cannot find a price, the scraper retries with Playwright (headless Chromium) which executes JavaScript. If both fail and there is a previous price for that (Product, Retailer), the scraper records a `carry_forward` row with `in_stock: false` and the previous price, so the chart line stays continuous and the tooltip flags the issue. If there is no prior price, the URL is logged as a hard failure in `data/last_run.json`.

The workflow exits nonzero (and the Actions run is marked failed) if more than 30% of URLs fail in a single run, so silent breakage gets surfaced.

## Sale-price behaviour

When a retailer shows both a "was" and a current sale price, the **chart line follows the sale price** so promotional dips show up. The original price is stored alongside as `price_was` and shown on hover, and sale days are drawn with a slightly larger dot.

If you would rather the line track the regular RRP instead, swap `price` and `price_was` in `parse_footjoy` (and add equivalent logic to the JSON-LD parsers via `priceSpecification`).

## Dashboard notes

- Loads `data/prices.json` first; if empty, falls back to `data/prices.example.json` and shows a "demo data" banner. The first real scrape replaces this transparently.
- Multi-select chips for Retailer and Brand. Date range is 7d / 30d / 90d / All. Filter state is encoded in the URL hash so any view is shareable.
- Each chart card shows one line per (Product, Retailer). Clicking a legend row hides that line.
- "Export CSV" exports the visible window for that chart.
- Banner appears if the latest scrape is older than 36 hours or had any failures.
- Dark mode toggle in the header. Choice is persisted in the URL hash.

## Privacy

This repo is public if you turn on GitHub Pages with the default settings. That means the list of competitor URLs and their daily prices is publicly readable. If that is sensitive, either:
- Make the repo private and use a GitHub plan that supports private Pages, or
- Host the dashboard on an internal-only host (e.g. an internal S3 bucket) and keep this repo private.

## Troubleshooting

- **Workflow ran but no commit appeared:** check the workflow run log; the commit step is skipped only when there are zero changes (which is correct on a no-op rerun).
- **Workflow shows red X:** open the run, look at `data/last_run.json` for the failed URL list, then try the URL in a browser to see whether the page changed structure or is gating with a Cloudflare challenge.
- **Dashboard is blank:** open the browser dev console. Most likely cause is `data/prices.json` is `[]` and `data/prices.example.json` is missing — confirm both exist in the repo.
- **Charts look stretched on mobile:** the layout already collapses to a single column below 768px. If a specific chart looks off, file an issue with a screenshot.

## What is intentionally NOT in this repo

- **No paid scraping API.** If a retailer starts blocking the GitHub Actions runner with Cloudflare or similar, the fix is either to add a small per-retailer wait, switch that retailer's parser to Playwright by default, or accept the failure rate. A paid scraping API can be added later if a single retailer becomes consistently unreachable.
- **No database.** All data lives in `data/prices.json` plus yearly archives. After 18 months, older entries should be moved to `data/archive/<year>.json` and the dashboard's default window kept rolling so it stays fast.
- **No alerting.** If you want a Slack ping when something moves more than 5%, add a second workflow that diffs the latest two days in `prices.json` and posts via webhook.
