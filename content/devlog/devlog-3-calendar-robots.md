+++
title = "Devlog #3: Calendar Robots vs. Macro Shockwaves"
date = 2025-12-17
description = "Building an event-impact engine: FRED-powered calendars, ICS/CSV ingestion, Polars analytics, and a CLI that ranks which markets price news fastest."
[taxonomies]
tags = ["macro", "events", "yfinance", "polars", "cli", "python", "devlog"]
+++

**Catchy start:** Spent the week teaching robots to read calendars (CPI/FOMC) without getting 403’d, so the desk can see who prices news fastest.

## What I built (event-impact)
- FRED release ingester: `--events-fred-release-ids "cpi=9,fomc=10"` pulls real release dates (bounded by `--fred-start/--fred-end`) and stamps events at 08:30 ET by default.
- Extensible calendars: CSV/JSON (`--events-file`), ICS (local or URL with friendly UA), plus built-in CPI/FOMC/earnings for 2024–2025.
- Analytics: pre/post returns, realized vol deltas, post-event max drawdown, reaction time to peak move (minutes).
- CLI polish: logger instead of print spam, CSV export (`--output-csv` auto-creates folders), type-sound Polars/Yahoo fetches.
- Licensing: added BSD 3-Clause so others can fork/use freely.

## How to run
```bash
uv run event-impact \
  --assets "SPY,QQQ,GLD,TLT,EURUSD=X,CL=F" \
  --interval 1h \
  --categories "cpi,fomc" \
  --year 2025 \
  --events-fred-release-ids "cpi=9,fomc=10" \
  --fred-start 2025-01-01 --fred-end 2025-12-31 \
  --pre-hours 24 --post-hours 24 \
  --output-csv data/impacts_2025.csv
```
_Set `FRED_API_KEY` in `.env` for the FRED path. ICS is optional; local files avoid BLS bot blocks._

## What worked / what bit me
- **Worked:** FRED dates plus built-in calendars give a complete 2025 macro diary; reaction_minutes surfaces which asset spikes first.
- **Bit me:** BLS ICS responds with “Access Denied” HTML; downloading via browser and pointing to the local `.ics` is the workaround. Intraday Yahoo history rejects very long ranges—added period fallbacks and local window filtering.
- **Type gremlins:** yfinance-pl period types and Polars std/null handling needed explicit casting and NumPy std to keep linters happy.

## Event types & what they mean
- **CPI (inflation prints):** Price levels; upside surprises usually push rates up, equities mixed, USD stronger. We stamp at 08:30 ET.
- **FOMC (policy statements):** Policy rate + guidance; rates/FX react fastest, equities follow. Default 14:00 ET overrideable.
- **Earnings (single-name micro):** Post-close or pre-open; impacts stock + sector ETFs.
- **FRED releases (generic):** Any release id (e.g., Employment Situation, GDP, Retail Sales). We map each date to a timestamp and category label.
- **Custom ICS/CSV/JSON:** Anything you import; category drives grouping and dedupe.

## Math behind the metrics
- **Pre/Post returns:** simple window returns (not annualized):
  $$
  r_{\text{pre}} = \frac{P_{\text{event}}}{P_{\text{pre start}}} - 1,\qquad
  r_{\text{post}} = \frac{P_{\text{post end}}}{P_{\text{event}}} - 1
  $$
- **Realized vol change:** log returns $\ell_t = \ln\!\frac{P_t}{P_{t-1}}$, $\sigma = \operatorname{std}(\ell_t)$, report $\Delta\sigma = \sigma_{\text{post}} - \sigma_{\text{pre}}$.
- **Reaction_minutes:** peak absolute cumulative move after the event:
  $$
  c_t = \frac{P_t}{P_{\text{event}}} - 1,\quad
  t^* = \arg\max_t |c_t|,\quad
  \text{reaction} = \frac{t^* - t_{\text{event}}}{60\ \text{seconds}}
  $$
  Lower reaction time ⇒ faster pricing.
- **Dedupe logic:** bucket by `(category, calendar date)`; prefer FRED-sourced rows when they overlap built-ins.

### Tiny code slice (metrics core)
```python
def analyze_event(asset, df, event, window):
    event_ts = event.utc_timestamp()
    pre_df = df.filter((pl.col("timestamp") >= event_ts - window.pre) & (pl.col("timestamp") <= event_ts))
    post_df = df.filter((pl.col("timestamp") >= event_ts) & (pl.col("timestamp") <= event_ts + window.post))

    ref_price = float(pre_df.select(pl.col("close").last()).item())
    first_price = float(pre_df.select(pl.col("close").first()).item())
    last_price = float(post_df.select(pl.col("close").last()).item())

    pre_ret = ref_price / first_price - 1
    post_ret = last_price / ref_price - 1

    pre_vol = pre_df.select(pl.col("close").log().diff()).to_series().drop_nulls().std()
    post_vol = post_df.select(pl.col("close").log().diff()).to_series().drop_nulls().std()

    # reaction_minutes: find first max of |cum_ret|
    cum = (post_df["close"] / ref_price) - 1
    idx = cum.abs().arg_max()
    reaction_minutes = ((post_df["timestamp"][idx] - event_ts).total_seconds()) / 60
    # max drawdown from ref
    max_dd = ((post_df["close"] / ref_price) - 1).cummax().combine(
        (post_df["close"] / ref_price) - 1, lambda peak, cur: cur - peak
    ).min()
```

## Next knobs to turn
- Bundle a small sample ICS/CSV in the repo for zero-network demos.
- Add support for multiple calendars (e.g., US, UK, EU).
- Implement a feature to allow users to customize the event types and their meanings.

### Interesting tooling and the whys
- **Polars**: Columnar, fast expressions, and friendly casting make windowed metrics and joins straightforward; avoiding pandas keeps memory/time in check on intraday pulls.
- **yfinance-pl**: Native Polars frames from Yahoo Finance with interval control; faster than wrapping pandas and re-parsing.
- **httpx**: Simple async-ready HTTP client for FRED/ICS; cleaner than `requests` and already in the stack.
- **uv**: Lightweight project/run tool; keeps virtualenv and scripts tidy.

## Links
- Code: [event-impact](https://github.com/Sachin-Bhat/event-impact) (BSD-3)
