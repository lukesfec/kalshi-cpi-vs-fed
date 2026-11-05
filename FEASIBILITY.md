# Phase 0: Feasibility Check

Checked 2026-10-27. I probed every source below live, and the raw responses are cached in `data/raw/phase0/`.

## Verdict

The project is feasible at $0. All three data sources work without paying for anything, and since the Cleveland Fed's daily nowcast history is available directly, no fallback model is needed.

| Source | Works? | Auth | Cost |
|---|---|---|---|
| Kalshi market data (markets, historical, candlesticks) | Yes | None needed | $0 |
| FRED / ALFRED CPI (incl. first-release vintages) | Yes | CSV: none. API: free key | $0 |
| Cleveland Fed daily nowcast history | Yes | None | $0 |
| Kalshi fee schedule | Partly (see blockers) | n/a | $0 |

The usable head-to-head sample is about 53–55 CPI releases (Mar 2022 to Sep 2026). That's small, so every result will need wide confidence intervals and clustering by release.

---

## 1. Kalshi API

Base URL: `https://api.elections.kalshi.com/trade-api/v2`

### Authentication
Every market-data endpoint I use works without authentication: `/series`, `/markets`, `/historical/cutoff`, `/historical/markets`, `/historical/markets/{ticker}/candlesticks`, `/series/{series}/markets/{ticker}/candlesticks`, `/series/fee_changes`.

### Live vs. historical split
`GET /historical/cutoff` returned:
```
market_settled_ts:  2026-08-28T00:00:00Z
trades_created_ts:  2026-08-28T00:00:00Z
orders_updated_ts:  2026-10-13T00:00:00Z
```
- Markets settled before `market_settled_ts` are only in `/historical/markets`, and their candles are only at `/historical/markets/{ticker}/candlesticks`.
- Markets settled after it (plus open markets) are in `/markets`, with candles at `/series/{series}/markets/{ticker}/candlesticks`.
- Asking the live endpoint for a historical ticker's candles returns HTTP 404 (I checked). The cutoff moves forward over time, so ingestion has to call `/historical/cutoff` on each run and route each ticker to the right tier.
- The two tiers also use different field names. Historical candles have `price.close`, `volume`, `open_interest`, while live candles have `price.close_dollars`, `volume_fp`, `open_interest_fp`. The loader has to normalize both.

### CPI series tickers (found via `/series?category=Economics`, 871 series)
Headline CPI is series `KXCPI` ("CPI", monthly, source BLS).
- Older events keep the legacy prefix (`CPI-21JUN` … `CPI-24OCT`) but are filed under series `KXCPI`. Querying series `CPI` returns 0 markets. From `KXCPI-24NOV` on, events use the `KXCPI-` prefix.
- Market tickers look like `KXCPI-26JUL-T0.2`, meaning "Jul 2026 CPI MoM > 0.2%".

Related series that could be extensions (the Cleveland Fed nowcasts these too):

| Series | What | Settled events | First event |
|---|---|---|---|
| `KXCPI` | Headline CPI MoM | 64 | Jun 2021 |
| `KXCPICORE` | Core CPI MoM | ~51 | Jul 2022 |
| `KXCPIYOY` | Headline CPI YoY | ~45 | Dec 2022 |

### KXCPI sample size
- There are 67 events in total: 64 settled (Jun 2021 → Sep 2026, every month, no gaps) and 3 still open (Oct, Nov, Dec 2026).
  - 62 of the settled events are in the historical tier and 2 (Aug and Sep 2026) are in the live tier.
- Strikes per event: median 8 in the historical tier (range 1–15).
  - Jun 2021 to Feb 2022 had only 1–4 strikes, which is too few to form a distribution.
  - 55 settled events have ≥5 strikes (Mar 2022 onward).
- Liquidity varies a lot over time:
  - 2022: about 150k–480k contracts per event.
  - 2023–24: about 12k–240k, so thin.
  - 2025–26: about 450k–2.3M.
- Kalshi markets usually open well before the nowcast starts, a median of 62 days before release (minimum 25).

### Contract structure (this matters for the method)
KXCPI contracts are not mutually exclusive brackets. Each one is a cumulative threshold, "CPI MoM strictly greater than X%" (`strike_type: greater`). So the prices of one event's strikes form a survival curve P(CPI > x), not a set of probabilities that sum to 1.
- Bracket probabilities come from differencing adjacent strikes.
- "Overround normalization" turns into monotonicity repair: the survival curve has to be non-increasing in x.
- The tails past the lowest and highest strikes need an explicit rule.

### Candlesticks
- `period_interval` accepts 1 (minute), 60 (hour) and 1440 (day) in both tiers.
- The max is 5,000 candles per request. A full event at 1-minute resolution got rejected (83,520 candles), though 1-minute works if requests are chunked.
- Each candle has yes_bid and yes_ask OHLC, last-trade price OHLC/mean, volume, and open interest. That's enough for a bid/ask-based backtest.
- Candles are sparse in quiet periods. For example, CPI-23MAR-T0.4 had 93 hourly candles over about 53 days. Gaps have to be forward-filled carefully: carry the last quote, never interpolate.
- Plan: hourly candles for CPI markets (about 1,300 per market-event, 1 request each). That's fine enough to line up with the nowcast's ~10:00 ET update.

### Rate limits
- Kalshi documents token budgets for authenticated traffic. The Basic tier gets 200 read tokens/s at about 10 tokens/request, which works out to about 20 requests/s.
- The docs don't give a limit for unauthenticated requests. My probes ran at about 5–7 requests/s and never hit an HTTP 429.
- Plan: about 5 requests/s, with exponential backoff on 429.
- Scale: about 600 CPI markets × 1 hourly request each, so a few minutes in total.

## 2. CPI truth (settlement source)

### What Kalshi settles on (from the KXCPI contract terms PDF, `assets.kalshi.com/contract_terms/CPI.pdf`)
> The Underlying is "the signed one-month, one-decimal percent change in the seasonally adjusted Consumer Price Index for All Urban Consumers (CPI-U)", and revisions after expiration are not counted.

- Measure: CPI-U, all items, seasonally adjusted, month-over-month % change.
- Rounding: one decimal, as printed by BLS.
- Vintage: first release. Revisions are ignored.
- Payout: Yes if the value is strictly greater than the strike.
- If no data is available, the terms give a fallback formula (a 12-month geometric average of the index). This matters for Oct 2025 (see blockers).

### FRED / ALFRED
- The FRED API needs a free key (a request without one returns "api_key is not set").
- The CSV endpoints don't need a key:
  - Current series: `fred.stlouisfed.org/graph/fredgraph.csv?id=CPIAUCSL`, 958 rows, 1947 → Sep 2026.
  - Point-in-time vintage: `alfred.stlouisfed.org/graph/alfredgraph.csv?id=CPIAUCSL&vintage_date=YYYY-MM-DD`.
- In a spot check, first-release MoM from ALFRED matched Kalshi's settled value in 3/3 cases:

| Month | ALFRED first release (unrounded) | Rounded | Kalshi settlement |
|---|---|---|---|
| Jun 2022 | 1.322% | 1.3 | 1.3 |
| Jun 2025 | 0.287% | 0.3 | 0.3 (Yes on >0.2, No on >0.3) |
| Jul 2026 | 0.074% | 0.1 | 0.1 |

- Plan: use Kalshi's own `expiration_value` as the primary truth, since it's literally what settled, and cross-check it against ALFRED first-release for every event in Phase 1. Any mismatch gets flagged rather than silently fixed.

## 3. Cleveland Fed nowcast history: AVAILABLE

- The nowcasting page's chart loads `https://www.clevelandfed.org/-/media/files/webcharts/inflationnowcasting/nowcast_month.json`, a public 7.7 MB file with no auth. There are sibling files for the quarterly and annual nowcasts.
- It's a list of 160 target months (Jul 2013 → Oct 2026). Each one has dated daily series for:
  - Nowcasts: CPI, Core CPI, PCE, Core PCE (MoM %, SA, unrounded).
  - "Actual" values for the same four measures.
- This is the full daily sequence for each past month, not just the final values. There's a median of 28 business-day nowcasts per target month (range 26–53).
- Horizon coverage relative to the release date (55 events with ≥5 strikes):
  - First nowcast: median 42 days before release (range 37–45).
  - Last nowcast: median 5 days before release (range 2–6).
  - So the 1-day and 3-day horizons I planned aren't directly in the file. I can either (a) carry the last available nowcast forward, which is legitimate because it's what was knowable at the time, or (b) drop those horizons. I still have to decide which, and I also plan to verify the cut-off by snapshotting the file daily around the Nov 10, 2026 release.
- Real-time vs. revised:
  - The Cleveland Fed page and User's Guide don't say whether the chart history is real-time or re-estimated.
  - To test it I compared today's file with the Internet Archive's copy from 2026-03-25. All values for all 153 overlapping months are byte-identical, including the partial Mar 2026 month.
  - So history isn't being rewritten. That's strong evidence the archive is real-time, but it doesn't prove the 2021–22 values were never re-estimated before Mar 2026.
  - There's also an older snapshot (2022–23), but the archive rate-limited my requests (HTTP 429), so that check is pushed to Phase 1.
- Quirks: date labels are `MM/DD` with no year (they wrap into the next year for Nov/Dec targets). According to the page, it publishes at about 10:00 a.m. ET each business day.

## 4. Kalshi fees

- From the API: series `KXCPI` has `fee_type: quadratic_with_maker_fees`, `fee_multiplier: 1`. `GET /series/fee_changes?series_ticker=KXCPI` returned an empty list (no scheduled changes).
- The published formula, as reported by multiple secondary sources:
  - Taker: fee = ceil-to-cent( 0.07 × C × P × (1 − P) ).
  - Maker: 0.0175 × C × P × (1 − P), only on series with maker fees (KXCPI is one), and only charged when the resting order fills.
  - No settlement fee.
  - Example: 100 contracts at $0.50 → $1.75 taker fee.
- I haven't verified this against the primary source yet. `kalshi.com/docs/kalshi-fee-schedule.pdf` returned HTTP 429 (bot protection) to both automated fetches, and the sources disagree on its effective date (Feb 5 2026 vs Jul 7 2026).
- Limitation: the backtest will apply today's fee schedule to 2022–2024 trades, and Kalshi's fees in those years may have been different. This goes in the limitations section.

## Blockers and open items

1. Python isn't installed yet (only the Windows Store stub), and neither is PostgreSQL. Both are free and have to be set up before Phase 1. The Phase 0 probes used curl and Node.js, and Git is already there.
2. Fee PDF: `https://kalshi.com/docs/kalshi-fee-schedule.pdf` blocks automated downloads, so I need to open it in a browser to confirm the formula and effective date.
3. Oct 2025 CPI was never published (government shutdown; FRED shows a blank). Kalshi still settled `KXCPI-25OCT` at 0.2 on 2025-11-22, apparently through its fallback rule. I plan to exclude it from the head-to-head, since there's no real CPI print to forecast. Nov 2025 (released late on 2025-12-18, with no October base) needs checking too.
4. The 1-day / 3-day horizons aren't covered by the nowcast file (see §3), so I need to decide how to handle them.
5. Liquidity was thin in 2023–24, so expect stale quotes and wide spreads in those years. That affects calibration and how realistic the backtest is. It isn't a blocker, but those results will be noisier.

## Nothing found that costs money.

## Reproduce
- `node phase0/probe_kalshi.mjs KXCPI KXCPIYOY KXCPICORE` re-runs the Kalshi counts from the cache in `data/raw/phase0/kalshi/`.
- The other checks were one-off curl calls, and their raw outputs are in `data/raw/phase0/{fred,cleveland}/`.
