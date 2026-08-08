# Chapter 11: Time Series — Conclusion

Recap of everything covered in `Time_Series/`. Each section below is a quick
summary of concepts **plus the reference code worth remembering** — for full
examples and detailed nuances/gotchas, open the referenced notebook (each one
also ends with its own "Summary / Cheat Sheet" cell).

Common setup used throughout:

```python
import pandas as pd
import numpy as np
from datetime import datetime
```

---

## 11.1 — Date and Time Data Types and Tools

`Time_Series/11_01_Date_and_Time_Types_and_Tools.ipynb`

- Everything in this chapter builds on the standard library's `datetime`,
  `time`, `timedelta`, and `tzinfo` types.
- `datetime` = a single instant (date + time); `timedelta` = a duration
  (difference between two `datetime`s).
- Three ways to turn a string into a date: `datetime.strptime` (needs a format
  string), `dateutil.parser.parse` (flexible, one at a time),
  `pandas.to_datetime` (handles whole arrays — the one you'll actually use).
- Missing timestamps become **`NaT`** ("Not a Time") — the datetime equivalent
  of `NaN`, and it shows up everywhere for the rest of the chapter.
- **Gotcha:** `timedelta.seconds` is only the leftover seconds after whole days
  are removed, not the total — use `.total_seconds()`.

```python
now = datetime.now()
now.strftime("%Y-%m-%d")                       # datetime -> string
datetime.strptime("2011-01-03", "%Y-%m-%d")    # string -> datetime (exact format)

delta = datetime(2011, 1, 7) - datetime(2008, 6, 24, 8, 15)
delta.days, delta.seconds, delta.total_seconds()  # .seconds != total duration!

from dateutil.parser import parse
parse("Jan 31, 1997 10:45 PM")
parse("6/12/2011", dayfirst=True)              # non-US day-first dates

pd.to_datetime(["2011-07-06 12:00:00", None])  # whole array at once; None -> NaT
```

## 11.2 — Time Series Basics

`Time_Series/11_02_Time_Series_Basics.ipynb`

- A `Series` indexed by timestamps automatically gets a **`DatetimeIndex`**;
  scalars pulled from it are pandas `Timestamp` objects (a richer drop-in
  replacement for `datetime`).
- Indexing/selection works like any Series, plus **partial-string indexing** —
  `ts["2011"]` or `ts["2011-01"]` selects a whole slice at once.
- On a DataFrame, use `.loc["2011-01"]` for row selection — plain
  `df["2011-01"]` looks for a _column_ with that name instead.
- Label-based slices are **views**, not copies (call `.copy()` if you need an
  independent copy); slicing also requires a sorted index.
- Duplicate timestamps in the index are legal — indexing returns a scalar or a
  Series depending on whether that label repeats; aggregate them with
  `ts.groupby(level=0).mean()`.

```python
dates = pd.date_range("2011-01-02", periods=6)
ts = pd.Series(np.random.standard_normal(6), index=dates)

ts["2011-01-03"]                 # exact label
ts["2011"]                       # partial-string: whole year
ts["2011-01"]                    # partial-string: whole month
ts[datetime(2011, 1, 7):]        # slice by datetime object
ts.truncate(after="2011-01-04")  # slice via method

df.loc["2011-01"]                # DataFrame ROWS -- df["2011-01"] looks for a column!

dup_ts.index.is_unique           # False if timestamps repeat
dup_ts.groupby(level=0).mean()   # collapse duplicate timestamps
```

## 11.3 — Date Ranges, Frequencies, and Shifting

`Time_Series/11_03_Date_Ranges_Frequencies_and_Shifting.ipynb`

- `pandas.date_range(start, end, freq=...)` generates a regular `DatetimeIndex`;
  pass `periods=` instead of one endpoint to count forward/backward a fixed
  number of steps.
- **Frequency aliases were overhauled in modern pandas** — old single-letter
  aliases (`M`, `A`, `Q`, `T`, `H`, `L`, `U`, ...) are now hard errors as of
  pandas 2.2+. Use the current aliases (`ME`, `YE`, `QE`, `min`, `h`, `ms`,
  `us`, ...) — full table in the notebook.
- `shift()` has two distinct modes, easy to mix up: **naive shift** (default)
  moves the _data_, leaving the index unchanged and introducing `NaN`s;
  **frequency-aware shift** (`freq=`) moves the _index labels_ instead,
  introducing no `NaN`s. `ts.pct_change()` is shorthand for the common
  naive-shift percent-change pattern.
- `Resampler` objects (from `.resample(...)`) are **lazy**, just like `GroupBy`
  — nothing computes until you call an aggregation.

```python
pd.date_range("2012-04-01", "2012-06-01")               # daily, inclusive of both ends
pd.date_range(start="2012-04-01", periods=20)            # N days forward from start
pd.date_range(end="2012-06-01", periods=20)              # N days backward from end
pd.date_range("2000-01-01", "2000-12-01", freq="BME")    # business month end
pd.date_range("2012-05-02 12:00", periods=5, normalize=True)  # snap to midnight
pd.date_range("2012-01-01", "2012-09-01", freq="WOM-3FRI")    # 3rd Friday each month

ts.shift(2)               # naive: data moves, index unchanged, NaNs appear
ts.shift(2, freq="ME")    # freq-aware: index labels move, no NaNs
ts / ts.shift(1) - 1      # manual percent change ...
ts.pct_change()           # ... same thing, built in

from pandas.tseries.offsets import Hour, Day, MonthEnd
now + 3 * Day()
now + Hour(2)
MonthEnd().rollforward(ts_index)          # round each date up to its month end
ts.groupby(MonthEnd().rollforward).mean() # manual precursor to resample("ME")
```

## 11.4 — Time Zone Handling

`Time_Series/11_04_Time_Zone_Handling.ipynb`

- Modern pandas resolves time zone strings (e.g. `"America/New_York"`) via the
  standard library's **`zoneinfo`**, not `pytz` — `pytz` is now optional,
  despite what older material says.
- **`tz_localize`** attaches a time zone label without changing any clock values
  ("this `09:30` was always New York time"). **`tz_convert`** changes the
  displayed wall-clock time to show the same instant in a different zone. Easy
  to confuse — they do opposite things.
- Time zone-aware `Timestamp`s are stored internally as a single UTC instant;
  the zone is just a display lens. That's why combining series across different
  zones "just works," and why combining a naive series with an aware one raises
  an exception (there's no shared instant to align on).
- `DateOffset` arithmetic on an aware `Timestamp` operates on **real elapsed
  time**, not the wall-clock hour field — this is what makes it correctly handle
  DST "spring forward" / "fall back" transitions.
- Directly localizing a naive time that falls in a DST gap or repeat raises
  `ValueError` — pass `nonexistent=` / `ambiguous=` to resolve it.

```python
ts.index.tz is None                                  # check naive vs aware
pd.date_range("2012-03-09", periods=10, tz="UTC")    # generate tz-aware range directly

ts_utc = ts.tz_localize("UTC")        # naive -> aware; clock values DON'T change
ts_ny  = ts_utc.tz_convert("America/New_York")  # aware -> aware; clock values DO change

pd.Timestamp("2011-03-12 04:00", tz="America/New_York")

stamp = pd.Timestamp("2012-03-11 01:30", tz="US/Eastern")
stamp + Hour()   # arithmetic respects real elapsed time across DST transitions

# resolving DST edge cases when localizing a naive time
ts.tz_localize("US/Eastern", nonexistent="shift_forward")  # spring-forward gap
ts.tz_localize("US/Eastern", ambiguous="NaT")               # fall-back repeat
```

## 11.5 — Periods and Period Arithmetic

`Time_Series/11_05_Periods_and_Period_Arithmetic.ipynb`

- A **`Period`** represents a time _span_ (e.g., all of March 2012), unlike a
  `Timestamp`, which pins down a single _instant_.
- `asfreq(freq, how="start"/"end")` converts a period between frequencies
  (defaults to `"end"`); `to_period()` / `to_timestamp()` convert between
  `Timestamp` and `Period` entirely (also lossy — you get the start/end of a
  period back, not your original timestamp).
- A fiscal year's label always names the year it **ends** in, not the one it
  starts in (e.g., under `Y-JUN`, period `"2011"` = July 2010–June 2011) — a
  frequent source of "which year does this belong to" confusion.
- **Out of date on current pandas:** giving a `Period`/`PeriodIndex` a `"B"`
  (business-day) frequency now raises `FutureWarning` (use a `Timestamp` +
  `BDay` offset instead); `pd.PeriodIndex(year=..., quarter=..., freq=...)` is
  gone entirely — use `PeriodIndex.from_fields(...)`.

```python
p = pd.Period("2011", freq="Y-DEC")     # a whole calendar year, as one object
p + 5                                   # Period('2016', 'Y-DEC')
p.asfreq("M", how="start")              # Period('2011-01', 'M')
p.asfreq("M", how="end")                # Period('2011-12', 'M')

pd.period_range("2000-01-01", "2000-06-30", freq="M")
pd.Period("2012Q4", freq="Q-JAN")       # fiscal-year-end-aware quarter

ts.to_period("M")                       # timestamps -> periods
pts.to_timestamp(how="end")             # periods -> timestamps (lossy; picks a boundary)

pd.PeriodIndex.from_fields(year=data["year"], quarter=data["quarter"], freq="Q-DEC")

# business-day math on a Period's "B" freq is deprecated -- convert to Timestamp first,
# then use the BDay offset (e.g. "4pm on the 2nd-to-last business day of the quarter"):
from pandas.tseries.offsets import BDay
quarter_end = p.asfreq("D", how="end").to_timestamp()
(quarter_end - BDay(1)) + pd.Timedelta(hours=16)
```

## 11.6 — Resampling and Frequency Conversion

`Time_Series/11_06_Resampling_and_Frequency_Conversion.ipynb`

- `resample(rule).agg_func()` behaves like `groupby`, except the "groups" are
  time intervals. **Downsampling** = aggregating to a lower frequency;
  **upsampling** = converting to a higher one (no aggregation, just more/emptier
  bins).
- `closed` (which bin edge is inclusive) and `label` (which edge names the bin)
  default to `"right"` for `M`/`Y`/`Q`/`W`-family frequencies and `"left"` for
  everything else — the most common source of "my bucket looks off by one"
  confusion.
- Upsampling doesn't fill anything by default — `.asfreq()` just creates new,
  empty (`NaN`) rows; use `.ffill()`/`.bfill()`/`.interpolate()` to actually
  fill them.
- Resampling a `PeriodIndex` is stricter than resampling timestamps: frequencies
  must nest cleanly (subperiod/superperiod), or pandas raises — mostly relevant
  for fiscal-year-anchored frequencies.
- `pandas.Grouper(freq=...)` lets you combine a time-based resample with a
  normal `groupby` column key, for resampling multiple stacked ("long format")
  time series at once.

```python
ts.resample("ME").mean()                                    # downsample to month-end
ts.resample("5min").sum()                                   # downsample to 5-minute bars
ts.resample("5min", closed="right", label="right").sum()    # explicit bin-edge control
ts.resample("5min").ohlc()                                  # open/high/low/close per bucket

frame.resample("D").asfreq()          # upsample, leave gaps as NaN
frame.resample("D").ffill()           # upsample, forward-fill
frame.resample("D").ffill(limit=2)    # ... but only fill up to 2 periods forward

annual_frame.resample("Q-DEC").ffill()                      # upsample periods, convention="start"
annual_frame.resample("Q-DEC", convention="end").asfreq()   # ... or convention="end"

# grouped/long-format resampling
df.set_index("time").groupby(["key", pd.Grouper(freq="5min")]).sum()
```

## 11.7 — Moving Window Functions

`Time_Series/11_07_Moving_Window_Functions.ipynb`

- Three related tools: **`.rolling(window)`** (fixed-size sliding window, in
  row-count or a time-offset string like `"20D"`), **`.expanding()`** (window
  grows from the start of the series), **`.ewm(span=...)`** (no fixed window —
  all history contributes, weighted by recency).
- By default, `rolling` requires a full, non-`NA` window to produce a value — so
  the first `window - 1` rows are always `NaN`; `min_periods` relaxes this to
  get earlier (noisier) results.
- Binary window functions (`.rolling(...).corr(other)`, `.cov(other)`) compute a
  rolling statistic _between two series_ — e.g. a stock's rolling correlation to
  a benchmark index.
- `.rolling(...).apply(func)` runs your own function per window, but `func` must
  reduce each window to a single scalar, and it's a Python-level loop (slower
  than the built-in vectorized aggregations).

```python
close_px.rolling(250).mean()                    # 250-day simple moving average
close_px.rolling(250, min_periods=10).std()     # allow a partial window early on
close_px.rolling("20D").mean()                  # time-offset window (calendar days, not rows)

std250.expanding().mean()                       # running/cumulative mean from the start

aapl_px.ewm(span=30).mean()                     # exponentially weighted moving average

returns["AAPL"].rolling(125, min_periods=100).corr(spx_rets)  # rolling corr, one series
returns.rolling(125, min_periods=100).corr(spx_rets)           # rolling corr, whole DataFrame at once

def score_at_2percent(x):
    return percentileofscore(x, 0.02)
returns["AAPL"].rolling(250).apply(score_at_2percent)          # user-defined window function
```

---

## Recurring Themes Across the Chapter

A few patterns show up repeatedly and are worth internalizing as a set:

- **The lazy-object pattern:** `resample(...)`, `rolling(...)`, `groupby(...)`
  all return an intermediate object that computes nothing until you call an
  aggregation (`.mean()`, `.sum()`, ...).
- **Instant vs. span:** `Timestamp`/`DatetimeIndex` represent single instants;
  `Period`/`PeriodIndex` represent spans. A lot of "why doesn't this line up"
  confusion (11.5, 11.6) traces back to mixing these up.
- **Modern vs. old aliases:** single-letter frequency aliases (`M`, `A`, `Q`,
  `H`, `T`, `L`, `U`) are dead — always use the current two-letter/lowercase
  forms (`ME`, `YE`, `QE`, `h`, `min`, `ms`, `us`). This bites in 11.3, 11.5,
  and 11.6 alike.
- **`NaN`/`NaT` as the universal "missing" marker:** shifting, upsampling,
  resampling with gaps, and reindexing periods all produce these rather than
  erroring — pandas assumes you'll filter/fill explicitly.
- **UTC as the ground truth:** time zone-aware timestamps are always a UTC
  instant under the hood (11.4) — every zone-related operation is really just a
  different _view_ onto that one number.

## Quick Reference: "I want to..."

| Goal                                          | Code                                                 | Notebook |
| --------------------------------------------- | ---------------------------------------------------- | -------- |
| Parse date strings into a `DatetimeIndex`     | `pd.to_datetime(arr)`                                | 11.1     |
| Select a whole year/month from a time series  | `ts["2011"]` / `df.loc["2011-01"]`                   | 11.2     |
| Generate a regular sequence of dates          | `pd.date_range(start, end, freq=...)`                | 11.3     |
| Shift data forward/backward, or relabel dates | `ts.shift(n)` / `ts.shift(n, freq=...)`              | 11.3     |
| Compute percent change                        | `ts.pct_change()`                                    | 11.3     |
| Attach or convert a time zone                 | `ts.tz_localize(zone)` / `ts.tz_convert(zone)`       | 11.4     |
| Represent a month/quarter/year as a span      | `pd.Period(...)` / `pd.period_range(...)`            | 11.5     |
| Convert timestamps ↔ periods                  | `ts.to_period(freq)` / `pts.to_timestamp()`          | 11.5     |
| Aggregate to a lower/higher frequency         | `ts.resample(rule).mean()` / `.ffill()`              | 11.6     |
| Resample per group in a long-format table     | `df.set_index(t).groupby([key, pd.Grouper(freq=r)])` | 11.6     |
| Compute a moving average / rolling stat       | `ts.rolling(window).mean()`                          | 11.7     |
| Compute a rolling correlation to a benchmark  | `ts.rolling(window).corr(other)`                     | 11.7     |
