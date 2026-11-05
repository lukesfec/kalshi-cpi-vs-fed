# Decisions Log

The choices I made along the way and why, grouped by project phase.

## Phase 0

The headline series is `KXCPI` (CPI-U MoM, SA), because it's the Kalshi contract that exactly matches the Cleveland Fed's headline CPI nowcast measure. Legacy `CPI-*` events belong to series `KXCPI` too. The API files them there, so I didn't need a separate pull for a `CPI` series.

Truth is Kalshi's `expiration_value`, cross-checked against ALFRED first-release CPI. The contract settles on the first BLS print at one decimal and ignores revisions, so final revised FRED values would be the wrong target.

Sample rules:
- The head-to-head starts in Mar 2022, the first month with ≥5 strikes per event. Events with 1–4 strikes can't form a meaningful probability distribution.
- Oct 2025 (`KXCPI-25OCT`) is excluded. BLS never published that month's CPI, so it settled through Kalshi's fallback rule instead of a real print.

CPI markets use hourly candles. That's fine enough to line up with the ~10:00 ET nowcast update and costs 1 request per market; 1-minute candles need chunking and wouldn't add much.

The archive has no nowcasts for the final 2–3 business days before a release, so at the 1-day and 3-day horizons I carry forward the last nowcast published before that day, and flag it. Carrying forward only uses information that was available at the time, so there's no lookahead. It does make the Fed's input 2–4 days stale next to a live market price. That's why the headline comparison uses 7 days, where the nowcast is fresh. I still report 1 and 3 days, with that caveat.

## Phase 1

Ingestion writes raw API responses to `data/raw/` and small tidy CSVs to `data/processed/`, and loading into Postgres waits for Phase 2. Keeping "download" separate from "transform" means the database can always be rebuilt from the cache.

On the Kalshi side:
- Requests are throttled to ~5/s with exponential backoff on HTTP 429. Unauthenticated limits aren't documented. At 5/s I get a 429 about every 25 requests, and the backoff handles it.
- Candle requests are chunked at 4,800 candles, since the API rejects windows over 5,000.
- The calibration set uses daily candles. I only need prices at fixed times-to-close (e.g. 1, 7, 30 days), and one request covers a market's whole life.

The calibration universe is a stratified random sample: up to 40 series per Kalshi category (seed 42), with combo/parlay series (`KXMVE…`) left out. From each series I take every settled binary market with volume > 0, capped at 50 per series (seeded). Pulling the full universe isn't practical (15,438 series, and one unfiltered historical page covers <1 hour of closes). Picking whole series at random doesn't depend on outcomes, so it can't bias calibration, and the cap keeps a few huge series from dominating. The combo/parlay markets are out because they're auto-generated multi-leg products priced differently, and by count they'd swamp the sample.

For CPI truth I use ALFRED vintages, with a binary search for each month's release date. A month only shows up in vintages on/after its release, so this gives the exact release date and the index levels as first printed, without an API key. First-release MoM is computed within the release-day vintage and rounded half-up to 1 decimal, which is how BLS gets the printed number (from the index levels published that day, including any February seasonal revisions). CPI history starts in Jun 2013 to match the start of the Cleveland Fed nowcast archive. Phase 4 needs that long error history for the expanding-window error variance.

I store the Cleveland Fed's vertical-line release markers separately (`nowcast_release_markers.csv`) instead of treating them as observations. They're chart annotations, but they double as an independent record of release dates.

The truth check passed. Kalshi `expiration_value` equals ALFRED first-release MoM (1dp) in 62/62 comparable events, and all 64 events' yes/no results agree with their value. Oct 2025 and Nov 2025 have no official MoM print (shutdown), but Kalshi settled both at 0.2.

Release dates come from ALFRED and the Cleveland Fed's markers, which agree on every month. Kalshi's `close_time` isn't always the release date: in 2021 contracts closed the evening before release, and Sep 2025 closed on its originally scheduled date (Oct 15) while BLS released on Oct 24.

Two shutdown-related calls:
- Oct 2025 and Nov 2025 are out of the head-to-head but stay in the database with a footnote. BLS printed no MoM change for either month, so there's no real CPI value to score against; Kalshi's 0.2 came from its fallback rule.
- Sep 2025 stays in. For horizons that fall after Kalshi's last trade (Oct 15), both sources are cut off at that trade, with horizons still counted from the actual release (Oct 24). Without the cap the nowcast would get days of information the market never had.

## Phase 2

Most of this phase is about timing. A candle's `ts` is the end of its period, so a candle with `ts <= t` only contains information available at t, and "no lookahead" becomes a single `<=` condition everywhere. The CPI release is at 08:30 ET (BLS publishes at 8:30), and a nowcast counts as visible from 10:00 ET on its date. The Cleveland Fed says it updates "around 10:00 a.m. ET", and using the later time is conservative: the nowcast never gets credit for an update before it existed.

For the matched-horizon snapshot, as-of = release − h days (h = 30, 14, 7, 3, 1), capped at Kalshi's last trading time. Both sources use their latest value visible at as-of: for Kalshi the last finished hourly candle, for the Fed the last published nowcast (carried forward).

The outcome is whether the first-release value is strictly greater than the strike, which is the contract rule. It agrees with Kalshi's own settlement on all 489 head-to-head strikes.

Head-to-head inclusion is computed in SQL (`v_cpi_events.in_headtohead`): ≥5 strikes AND an official MoM print. Having one rule in one place keeps it consistent, and the reason for each exclusion is recorded. The result is 53 releases (Mar 2022 – Sep 2026).

Other choices:
- The nowcast table only holds nowcasts. I don't load the Fed's "actual" values, so truth comes from a single source (ALFRED first release, which matches Kalshi's settlements).
- The calibration view exposes bid, ask, spread, mid and last trade but doesn't filter on quote quality yet. 28% of calibration rows have a $0 bid (longshots) and 2–5% are empty books (bid 0, ask $1), so the mid is unreliable there. I left the filtering rule for Phase 3.
- The SQL is tested against a throwaway Postgres schema built from the real `schema.sql`/`views.sql`, so the tests exercise the actual SQL and not a Python copy of it.
- No browser-UA spoofing or bot-check workarounds. Where a source blocks automated access (the Kalshi fee PDF, BLS pages), I verified it by hand.
- No nowcast fallback model was needed, because the Cleveland Fed's own daily nowcast archive is public.

Kalshi strikes are a survival curve P(CPI > x), not brackets. The contracts are cumulative "strictly greater than" thresholds, so normalizing them means repairing monotonicity, not scaling out an overround.

## Phase 3

Quote filter: drop empty books (bid $0 and ask $1), and use the bid/ask midpoint as the primary price when the spread is ≤ 10¢. A midpoint of a wide or empty book isn't a probability anyone would actually trade at. For robustness I also run it on all non-empty quotes (any spread) and on last trade price. The primary sample is 5,673 prices, 3,340 markets, 1,123 events.

Time-to-close horizons are 1, 7 and 30 days. Headline numbers pool them, and I also report each one separately. Because pooling repeats some markets, all inference is clustered by event: every CI is an event-clustered bootstrap (500 draws, fixed seeds), and regression SEs are cluster-robust by event. Markets in one event are mechanically linked (one bracket wins), so treating them as independent would overstate precision.

Smaller choices:
- Prices are clipped to [0.005, 0.995] before any logit transform. This avoids infinite log-odds, and it's the smallest half-tick a midpoint can take.
- Categories with < 30 events are pooled into "Other", since per-category estimates from a handful of events are noise.

The calibration regression is logit P(yes) = a + b·logit(price). Perfect calibration is a = 0, b = 1. If b > 1, prices are pulled toward 50%, meaning longshots are over-priced and favourites under-priced.

The mispricing model is a logistic regression on logit(price), its interaction with standardized log volume, spread, horizon and category dummies (Sports = reference). The interaction term is a direct test of whether the calibration slope changes with liquidity.

For probability weighting the model is price = w(true probability), fit by maximum likelihood of the outcomes under p = w⁻¹(price). I use the one-parameter Tversky–Kahneman (1992) and one-parameter Prelec (1998) forms. This maps true frequencies to prices, and γ < 1 is the inverse-S (longshot) distortion. The likelihood is computed on (unique price, #yes, #total) counts. Midpoints sit on a half-cent grid, so this is mathematically identical and 14× faster.

Group differences are tested with a Wald test using bootstrap SEs (the groups are disjoint sets of events, so they're independent), plus a bootstrap CI for high-minus-low liquidity. A plain likelihood-ratio test would assume independent observations.

Literature comparison values:
- TK 1992: γ = 0.61 (gains), 0.69 (losses), checked against reproductions of the paper's estimates.
- Camerer & Ho 1994: 0.56. Wu & Gonzalez 1996: 0.71. Both as reported in Gonzalez & Wu (1999).
- Prelec 1998: α ≈ 0.65 "rough estimate", as reported in Wilcox (2019).
- The last three are secondary-source citations, and the write-up says so.

The charts use three series colors (blue/orange/aqua) plus a gray reference line, and they resize with the page. Light/dark theming waits for the Phase 7 dashboard.

I also found and fixed a Phase 2 bug here. Pre-Dec-2022 CPI markets had no strike in the API (121 markets, 18 events). The strike is now parsed from the ticker (`TN0.4` = −0.4) and has to match the title, or the load fails. The Phase 2 outcome check had been silently skipping those NULL rows; it now uses `IS DISTINCT FROM` and also counts NULL strikes. All 489 head-to-head strikes are now verified against Kalshi's settlement.

## Phase 4

Turning the nowcast into a strike probability: P(YES at k) = P(nowcast + error ≥ k + 0.05). Kalshi pays on the printed one-decimal value, and for strikes on the 0.1 grid, printed > k ⇔ unrounded ≥ k + 0.05.

The error distribution is estimated with an expanding window at the same horizon, using only months released strictly before the forecast's as-of time (minimum 24 past months). This is the no-lookahead rule. The history starts in Aug 2013, so even the first head-to-head release has ~100 past errors behind it. When the as-of time is capped at Kalshi's last trade (Sep 2025), I take the error spread from the horizon that matches the real distance to release, because that forecast really was made 9 days out.

The error model is Student-t with zero location (`t0`), chosen on pre-sample data only (months released before Mar 2022). Mean out-of-sample log predictive density was 0.49 for t vs 0.23 for normal and 0.25 for normal-with-bias. t wins at every horizon, but the month-level edge is not decisive (diff 0.27, SE 0.20). The fitted df is ≈ 2.3, which is very fat-tailed and driven by 2020. Picking the model before looking at the test period avoids tuning it to the result, and a normal-error sensitivity check gives the same conclusions.

Kalshi normalization is isotonic (pool-adjacent-violators) regression to a non-increasing survival curve over strikes. Each strike is a separate binary market with no overround to divide out. The only coherence condition is that P(CPI > k) is non-increasing in k, and isotonic regression is the least-squares projection onto that set.

Scoring setup:
- Both sources are scored on exactly the same strikes, namely the ones listed with a bid/ask at the as-of time.
- The unit of analysis is one release × horizon, and each source's score is averaged over that release's strikes. That gives 53 observations per horizon (52 at 30 days) instead of ~490 strike rows, so brackets are never treated as independent.
- Scores are mean Brier per strike (primary), RPS, and log score. For cumulative threshold contracts, RPS over the listed thresholds equals the sum of the per-strike Brier scores. The only difference from mean Brier is that releases with more strikes get more weight, so I report it as a robustness check, not as new evidence.

The Diebold–Mariano test runs on per-release loss differences in time order, with Newey–West variance (lag = ⌊T^⅓⌋ = 3), the Harvey–Leybourne–Newbold small-sample correction, and a t(T−1) reference. I also report a bootstrap CI of the mean difference and the share of releases each source wins.

Encompassing is a logistic regression of outcomes on logit(p_Kalshi) and logit(p_Fed), with SEs clustered by release. If one source's coefficient is significant, that source carries information the other lacks. The combination forecast is a simple 50/50 average of the two probabilities. It's the simplest benchmark and needs no estimated weights, so it gets no in-sample fitting advantage.

On multiple comparisons: I report 5 horizons × 3 metrics, and they're highly correlated. A single p-value near 0.05 is presented as borderline, not as a discovery.

## Phase 5

The daily calendar is the set of days the Fed published a nowcast for that release. Two days count as consecutive if they're ≤ 4 calendar days apart (weekend/holiday). Changes are only computed within a release, never across releases.

For the primary alignment, the Kalshi snapshot is taken at 09:30 ET from the last hourly candle finished by then (in practice the one ending at 09:00). That's strictly before the ~10:00 ET nowcast, so Kalshi's daily change can't include that day's nowcast. It does include the previous day's, which is exactly what the "Fed → Kalshi" test needs.

Kalshi's daily level is the median of its implied distribution of unrounded CPI (the isotonic-repaired survival curve, linearly interpolated between thresholds), which puts it on the same scale as the nowcast. It's NaN when 50% falls outside the listed strikes (3.4% of days). I use the median instead of the mean because the mean depends on how the tails are truncated and jumps when strikes get added.

The tests are Granger-style panel OLS on daily changes with 2 lags of both series, SEs clustered by release, and joint Wald tests on the cross lags, plus cross-correlations at lags −3…+3 with release-level bootstrap CIs.

Robustness checks:
- A 16:00 ET snapshot, after the nowcast. Same-day co-movement then includes Kalshi reacting to that morning's update and any data releases.
- The mean instead of the median (only between days with identical strike sets).
- Dropping CPI/PCE data-release days (taken from the Fed chart's release markers, now the `data_release_markers` table). On those days both series react to the same public print, which isn't a lead by either one.

I report 7 lags × 4 variants of cross-correlations. An isolated significant lag that doesn't survive the robustness checks isn't treated as a finding.

## Phase 6

The signal compares the Fed-implied probability (the Phase 4 Student-t model, no lookahead) with Kalshi's actual quote for each strike, at the 5 Phase-4 decision times. Reusing the probabilities already validated in the head-to-head means the backtest can't quietly switch to a different, better-tuned model.

The trading rule is: buy YES at the ask if p_fed − ask > θ, buy NO at (1 − bid) if bid − p_fed > θ, and hold to settlement. Crossing the spread as a taker is the realistic way to act on a signal right away, and holding to settlement means I don't need exit prices.

Fees are Kalshi's taker fee, ceil(0.07 × C × P × (1−P)), on 100-contract orders. That's the published formula, and the KXCPI series in the API shows fee type `quadratic_with_maker_fees` with multiplier 1. At 100 contracts the round-up to the cent is negligible.

Two assumptions that are listed as limitations:
- Today's fee schedule is applied to all years. Kalshi's historical fee schedules aren't in the API (`fee_changes` is empty), so earlier fees may have been different.
- Fills assume the full 100 contracts trade at the quoted best bid/ask. Hourly candles give no order-book depth, so this is optimistic for thin books.

Train is the Mar 2022 – Dec 2024 releases (34) and test is Jan 2025 – Sep 2026 (19, leaving out the two shutdown months). It's a time split, never shuffled, so the test really is out-of-sample. θ is chosen on train only, as the value with the highest total net P&L over θ ∈ {0.00, 0.01, …, 0.30} with ≥ 20 trades, and then applied once to the test period. Using total P&L (not the mean) avoids picking a θ that trades twice and gets lucky.

The chosen θ = 0.15 lost money in training (−0.5¢ per contract). I use it anyway and report it that way. The rule has to pick something, and the honest summary is that no training threshold was profitable after costs.

P&L inference uses a release-level bootstrap of mean P&L per contract and a t-stat on per-release P&L totals. Strikes in one release settle on the same CPI print, so they aren't independent bets. I break costs down into (1) midpoint fills, no fees; (2) bid/ask, no fees; (3) bid/ask + fees, to show where an edge, if any, gets lost.

The per-horizon results (longer horizons looking better) are labelled exploratory. I didn't choose that split before seeing the data, and confirming it would need new releases.

## Phase 7

The dashboard is a single static page (`docs/index.html`) generated by `src/viz/build_dashboard.py` from the phase outputs and the database. Every number in the narrative is inserted by code, so editing a result can't leave stale text behind.

Charts are drawn client-side with Plotly.js, bundled locally (`docs/assets/plotly.min.js`) instead of loaded from a CDN. The page works offline with no third-party dependency, and client-side rendering lets the charts re-theme for light/dark mode, which pre-rendered Plotly HTML can't do. The palette uses the same three colors with separate light and dark versions, and I checked contrast for every pair in both modes. Light aqua falls below 3:1 contrast, so every chart has a legend and every chart's data can be downloaded as CSV (the table view).

Legends are pinned to the bottom of each chart container (`yref: container`). With paper-relative positioning they collided with the x-axis titles at some widths.

Headline wording follows the statistics, so it says "borderline", not "significantly better". At 1 day the DM p-value is 0.07 (Brier), and 15 correlated tests are shown.

`src/run_all.py` rebuilds database → analyses → dashboard in one command (about 4 minutes). That's the reproducibility path. Downloads stay separate because they hit external APIs.

## Later changes

After a first full pass I tightened several things.

I fixed the primary test as Diebold–Mariano on the mean Brier score per release, with RPS and log score as secondary. Brier is the score readers look at first, and naming one test up front keeps the headline from picking whichever of the 15 related tests looks best.

I also moved the headline horizon from 1 day to 7 days. At 1–3 days the Fed's nowcast is stale (archive gap) while Kalshi is live, so part of Kalshi's short-horizon edge is mechanical. At 7 days both are current, and there's no significant difference (Brier 0.072 vs 0.089, p = 0.18). The 1–3 day gap is described as suggestive.

Intervals:
- Difference intervals now use the DM test's own variance: mean ± t(T−1) × HAC standard error (Newey–West, lag 3, HLN-corrected). My earlier intervals resampled releases one at a time, which ignored the serial correlation the test allows for, so at 1 day the interval excluded zero while p = 0.068. Now an interval excludes zero exactly when p < 0.05, and a unit test checks this.
- All the other bootstrap intervals (per-source error bars, backtest P&L) use a circular block bootstrap with blocks of 3 consecutive releases. That matches the Newey–West correlation window instead of treating months as independent.

For the backtest, no threshold made money in training (0 of 31), so I report the held-out result as noise and not as a strategy that held up. Picking the least-bad of a set of losing thresholds isn't evidence of anything. Power is reported as a minimum detectable edge: (1.96 + 0.84) × block-bootstrap SE ≈ 28¢ per contract on 14 traded test releases. This makes it explicit that the test can't distinguish "no edge" from "too little data".

The lead-lag text says that neither source leads (no cross-lag p < 0.12 in any of 4 versions), and I retitled the section "Does either one move first? No." The old title suggested a winner the data doesn't support.

Finally, the 30-day horizon has 52 releases. Dec 2025's first market opened Dec 18, 2025, only ~26 days before its Jan 13 release (because of the shutdown delays). The dashboard footnotes this, and the footnote is generated from the data.
