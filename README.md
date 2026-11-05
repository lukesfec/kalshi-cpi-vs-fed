# Crowd vs. the Fed: Who Predicts Inflation Better?

Kalshi lets anyone trade on next month's CPI print, and the Cleveland Fed publishes a daily model-based nowcast of the same number. I wanted to know which one is the better forecast. I compare them like-for-like across 53 monthly CPI releases (Mar 2022 – Sep 2026), using the same strikes at the same moment with no hindsight. I also look at how well-calibrated Kalshi is in general, and whether the gap between the two can be traded after costs.

Dashboard: `docs/index.html` (a static page; GitHub Pages instructions are below).

## Findings

1. **When both forecasts are current, neither one clearly wins.**
   - I fixed the primary test in advance: Diebold–Mariano on the mean Brier score per release.
   - Seven days before release, when the Fed nowcast is fresh, Kalshi's Brier score is 0.072 vs 0.089 for the Fed. Kalshi is lower, but the difference is not significant (p = 0.18).
2. **Kalshi looks better closer to release, though the evidence is only suggestive.**
   - At 1 day it's 0.068 vs 0.091 (p = 0.07), which isn't significant on the primary test. The secondary scores only just clear 5% (RPS p = 0.04, log p = 0.05), and that's out of 15 related tests.
   - Some of the gap is mechanical. The Fed's archive skips the last 2–3 business days before each release, so at 1–3 days I'm comparing a ~4-day-old nowcast with a live market price.
3. **A month out, the Fed holds its own and knows things the market doesn't.**
   - At 30 days the Fed is slightly ahead (0.117 vs 0.131, p = 0.43), and a 50/50 average of the two beats Kalshi alone (p = 0.036).
   - In an encompassing regression the nowcast adds information beyond Kalshi's price at 30 days (coefficient 0.44, p = 0.006). It doesn't at 7 days (p = 0.33) or 1 day (p = 0.62).
4. **Day to day, neither source leads the other.** This is a null result.
   - None of the cross-lag effects are significant in any of the 4 versions of the test I ran (smallest p = 0.12).
   - The nowcast doesn't change at all on 72% of days, and what little co-movement there is comes from both reacting to the same CPI/PCE data releases.
5. **Kalshi is well calibrated overall, with a mild favorite-longshot bias.** I measured this on 5,673 prices from 1,123 events across the whole exchange.
   - Contracts priced ≤5¢ averaged 2.0¢ but only happened 0.9% of the time.
   - The calibration slope is 1.11 (SE 0.05).
   - The fitted Tversky–Kahneman probability-weighting parameter is γ = 0.91 [0.84, 0.97]. So there is real distortion, but it's much milder than the classic lab estimate of 0.61.
6. **I couldn't find a way to trade the disagreement profitably after costs, and the test is too small to say much more.**
   - The rule: buy when the Fed's probability is more than θ away from Kalshi's bid/ask, hold to settlement, and pay Kalshi's taker fee.
   - In training (34 releases), 0 of 31 thresholds made money. With no profitable strategy to carry forward, the rule just picked the least-bad threshold (θ = 0.15).
   - On the held-out period that θ made +4.9¢ per contract over 14 traded releases (95% CI −10.8¢ to +27.0¢). I read that as noise, not a strategy that held up.
   - With 80% power the test can only detect an edge of about 28¢ per contract, so it can't tell "no edge" from "too little data".
   - At midpoint prices with no fees the same signal earned +11.4¢, which means the bid-ask spread is what eats it.

All of these numbers come out of the code in this repo, run on the project database (`outputs/phase3` … `outputs/phase6`).

## Data (all free)

| Source | What | Access |
|---|---|---|
| [Kalshi API](https://docs.kalshi.com) | Every settled `KXCPI` market (headline CPI MoM) with hourly bid/ask candles. A seeded random sample of 6,020 markets across all categories for calibration. | Public endpoints, no key. Live and historical tiers are handled separately. |
| [Cleveland Fed inflation nowcasting](https://www.clevelandfed.org/indicators-and-data/inflation-nowcasting) | Daily nowcast sequence for every target month since Jul 2013 (the JSON file behind the site's chart) | Public. I checked it against an Internet Archive copy from 7 months earlier and nothing had changed. |
| [ALFRED](https://alfred.stlouisfed.org) (St. Louis Fed) | First-release CPI-U (SA), with exact release dates | Vintage CSVs, no key needed |

The "truth" series matches Kalshi's settlement rule exactly. Kalshi pays on the first-published, one-decimal, seasonally adjusted CPI-U monthly change and ignores revisions. ALFRED's release-day vintage matched Kalshi's settled value in 62 of 62 releases I could check.

## Method

- **Fed probabilities.**
  - Kalshi strike *k* pays if CPI prints above *k*, i.e. if the unrounded change is ≥ *k* + 0.05.
  - P(YES) = P(nowcast + error ≥ *k* + 0.05), with a Student-t error distribution.
  - I estimate the error distribution with an expanding window, using only months released before the forecast moment and at the same horizon.
  - I picked Student-t over the normal using pre-sample data (before Mar 2022) only.
- **Kalshi probabilities.** Bid/ask midpoints, repaired with isotonic regression so P(CPI > *k*) never rises with *k*. Each strike is its own binary market, so there's no overround to remove.
- **Matched horizons.** Both sources use their latest value published at or before release − *h* days (*h* = 30, 14, 7, 3, 1), and never anything after Kalshi's last trade.
- **Statistics.**
  - Each source gets one score per release, so strikes aren't treated as independent.
  - The primary test is Diebold–Mariano on the Brier score, with Newey–West variance (lag 3) and the Harvey–Leybourne–Newbold correction. The difference intervals use that same variance, so they agree with the test.
  - Encompassing and calibration regressions have SEs clustered by release/event.
  - The other intervals come from a circular block bootstrap (blocks of 3 consecutive releases).
  - The prospect-theory weighting functions (Tversky–Kahneman 1992; Prelec 1998) are fit by maximum likelihood.
- **No lookahead.** Kalshi candles are timestamped at the end of their period, and a value only gets used if it was published at or before the as-of time. The tests check this directly.

[DECISIONS.md](DECISIONS.md) lists each methodological choice I made and why. [FEASIBILITY.md](FEASIBILITY.md) has the data feasibility check I did before starting.

## Limitations

- **Small sample.** There are 53 releases (52 at 30 days, because the Dec 2025 market opened only ~26 days before its release). Effects this size are hard to confirm, so I call p-values near 0.05 suggestive. I show a lot of correlated tests, and one significant result among them shouldn't be over-read. The backtest's 14 traded test releases can only detect an edge of ~28¢ per contract.
- **Nowcast archive gap.** The Fed archive has no values for the last 2–3 business days before each release, so the 1–3 day comparisons carry forward a nowcast about 4 days old. I didn't need a fallback model and didn't use one. This gap is the reason the headline uses 7 days, where the nowcast is fresh and the two sources are statistically tied.
- **Kalshi liquidity.** Some 2023–24 CPI events were thin. Calibration uses midpoints of books with spreads up to 10¢. The backtest assumes full fills at the quoted bid/ask since I have no order-book depth data.
- **Settlement definition.** Oct and Nov 2025 have no official monthly print because of the federal shutdown, so I excluded them. Sep 2025 came out 9 days late, so both sources are cut off at Kalshi's last trade.
- **Fees.** I apply today's published taker fee (0.07 × C × P × (1−P), rounded up) to every year, because the API doesn't provide historical schedules.
- **Error model.** The Fed's probabilities depend on which error distribution I chose. A normal-error version leads to the same conclusions.

## Reproduce

You need Python 3.11+ and a local PostgreSQL 16. Everything is free. The raw cache is about 5 GB.

```bash
python -m venv .venv
.venv/Scripts/python -m pip install -r requirements.txt     # on macOS/Linux: .venv/bin/python
cp .env.example .env                                         # then set PGPASSWORD; create database kalshi_vs_fed
```

Download the data. The scripts are resumable and idempotent, and they cache every raw response in `data/raw/`:

```bash
.venv/Scripts/python -m src.ingest.kalshi cpi
.venv/Scripts/python -m src.ingest.kalshi calibration        # ~30 min, rate-limited
.venv/Scripts/python -m src.ingest.fred
.venv/Scripts/python -m src.ingest.cleveland_nowcast
.venv/Scripts/python -m src.ingest.validate                  # cross-source checks
```

Build the database, run all the analyses and build the dashboard (takes about 4 minutes):

```bash
.venv/Scripts/python -m src.run_all
.venv/Scripts/python -m pytest
```

To view the dashboard locally:

```bash
.venv/Scripts/python -m http.server 8765 --directory docs
```

To deploy, push to GitHub and set *Settings → Pages → Deploy from a branch → `main` / `/docs`*.

## Repo layout

```
sql/            schema.sql (tables), views.sql (analysis views), checks.sql (post-load sanity checks)
src/ingest/     kalshi.py, fred.py, cleveland_nowcast.py, validate.py, http_cache.py
src/load/       load_db.py (raw cache -> Postgres)
src/analysis/   calibration.py, cpi_headtohead.py, leadlag.py, backtest.py + scoring, weighting, fees, nowcast_probs
src/viz/        build_dashboard.py, template.html
outputs/        result tables and charts per phase
docs/           the static dashboard (GitHub Pages)
tests/          pytest suite (scoring, probability conversion, normalization, horizon alignment, backtest mechanics)
```

The schema has a `venue` column, so data from other venues (e.g. Polymarket) can go into the same tables.
