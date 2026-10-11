# Public Market Data Guide

Revolut X provides market data endpoints without requiring authentication.

---

## Tickers and Order Books

### Getting Tickers
You can fetch tickers for all active pairs or a single specific pair:

```python
from revolut_x import RevolutXClient

client = RevolutXClient()

# Single pair ticker
ticker = client.get_ticker("BTC-EUR")
print(f"Last Price: {ticker['last_price']} EUR")
print(f"Best Bid:   {ticker['bid']} EUR")
print(f"Best Ask:   {ticker['ask']} EUR")

# All tickers
all_tickers = client.get_tickers()
for t in all_tickers[:3]:
    print(f"{t['symbol']}: {t.get('last_price')}")
```

### Getting Order Books
Fetch the current bid and ask price levels. Each level is a **dictionary** —
the price lives under `p` and the quantity under `q`:

```python
book = client.get_order_book("BTC-EUR")

print("Bids (Buyers):")
for level in book["bids"]:
    print(f"  {level['p']} EUR x {level['q']} BTC")

print("\nAsks (Sellers):")
for level in book["asks"]:
    print(f"  {level['p']} EUR x {level['q']} BTC")
```

!!! note "Book depth is fixed"
    The exchange currently returns **5 levels per side** and ignores the
    `depth` argument, so don't rely on requesting a deeper book.

---

## Candlesticks (OHLCV)

!!! warning "The interval decides how far back you can go"
    Two independent limits apply to candles, and only the first one is something
    the library can work around:

    - **1000 candles per request** — a request-size limit. `iter_candles()`
      slices the window and handles this for you.
    - **History retention** — how long the exchange keeps a given resolution.
      Fine-grained candles are pruned, and no amount of pagination brings them
      back.

    An out-of-retention request is **not** an error. The API answers HTTP 200
    with an empty list, so `iter_candles()` faithfully yields nothing and a
    30-day 1-minute loop simply ends with `Streamed 0 candles.`

    Measured on `BTC-EUR` with `Interval.MIN_1`, asking for 500 candles at
    increasing age:

    | Window start | Candles returned |
    | --- | --- |
    | 2 days ago | 500 |
    | 7 days ago | 500 |
    | 14 days ago | 500 |
    | 30 days ago | 0 |

    So 1-minute data reaches past 14 days but not 30, while `Interval.HOUR_1`
    covers 30 days without trouble — the coarser the interval, the deeper the
    history. These are observed numbers, not a documented guarantee, so treat
    them as a ballpark. **If a backfill comes back empty, ask for a coarser
    interval before hunting for a bug in your own code.**

### Fetching a Single Batch
The exchange accepts interval resolutions from 1 minute up to 4 weeks:

```python
from revolut_x import RevolutXClient, Interval

client = RevolutXClient()

# 1-hour candles
candles = client.get_candles(
    "BTC-EUR",
    interval=Interval.HOUR_1,
    since=1700000000000
)

for c in candles[-5:]:
    print(f"Time: {c['start']} | Open: {c['open']} | High: {c['high']} | Close: {c['close']}")
```

#### Accepted time formats
Every time bound — `since` / `until` on the candle methods and
`start_date` / `end_date` on the trade methods — accepts any of these:

| Value | Meaning |
| --- | --- |
| `1700000000000` | Unix timestamp in **milliseconds**, passed through as-is. |
| `datetime(2024, 1, 1, tzinfo=timezone.utc)` | Converted using its own offset. |
| `datetime(2024, 1, 1)` | Naive — interpreted as **UTC**, never local time. |
| `date(2024, 1, 1)` | Midnight UTC on that day. |
| `"2024-01-01T00:00:00Z"` | ISO-8601 string (a trailing `Z` is fine). |
| `"1700000000000"` | Digits-only string of 12+ digits — milliseconds, for values arriving as text. |

```python
from datetime import datetime, timedelta, timezone

candles = client.get_candles(
    "BTC-EUR",
    interval=Interval.HOUR_1,
    since=datetime.now(timezone.utc) - timedelta(hours=500),
)
```

!!! warning "Numbers are milliseconds, not seconds"
    Raw numbers are never rescaled, so `since=int(time.time())` is not an error
    — it is a window starting in January 1970 and you get back an empty or
    useless range. Multiply by 1000, or just pass a `datetime` and let the
    library convert it. The same holds for response fields: `candle["start"]`
    and `trade["timestamp"]` are milliseconds too.

    Need the conversion elsewhere in your own code?
    `from revolut_x import to_timestamp_ms` exposes the same helper.

#### Values arriving from a request

Convert user input yourself instead of forwarding the raw string — the parser
here is a convenience, not an input validation layer:

```python
import re
from datetime import datetime
from zoneinfo import ZoneInfo

USER_TZ = ZoneInfo("Europe/Prague")

raw = request.args.get("from", "")          # e.g. '2024-01-01' from <input type="date">
if raw and not re.fullmatch(r"\d{4}-\d{2}-\d{2}", raw):
    abort(400, "from must be YYYY-MM-DD")

# Midnight in the *user's* zone — not UTC midnight.
since = datetime.fromisoformat(raw).replace(tzinfo=USER_TZ) if raw else None

candles = client.get_candles("BTC-EUR", Interval.HOUR_1, since=since)
```

Three things that bite when the value comes straight from a browser:

- **A string without an offset is UTC.** `"2024-01-01"` is UTC midnight, but a
  user in Prague who picked 1 January means 23:00 on 31 December UTC. Build the
  bound in their timezone and pass the aware `datetime`, as above. From
  JavaScript, `d.getTime()` (milliseconds) or `d.toISOString()` (carries `Z`)
  are both unambiguous; `"2024-01-01T13:30"` from `<input type="datetime-local">`
  is **not** — it has no offset, so it is read as UTC.
- **An empty field is a string, not a missing one.** HTML forms submit `?from=`
  for untouched inputs, and `""` raises `ValueError` — hence the `if raw` above.
- **Separatorless dates are refused, not guessed.** `"2024"` and `"20240101"`
  raise `ValueError` rather than resolving to 1970 or to a date, because a short
  run of digits is ambiguous between the two. Only 12+ digits count as
  milliseconds.

### Streaming Long Time Ranges with `iter_candles()`
The API limits single requests to 1000 candles. To download continuous ranges, use the generator `iter_candles()` which automatically handles time window slicing. Note that this widens how much you can *request*, not how far back the data exists — see the retention warning above:

```python
import time
from revolut_x import RevolutXClient, Interval

client = RevolutXClient()

# Download the last 7 days of 1-minute data seamlessly (10 080 candles,
# well past the 1000-per-request limit but still inside retention)
start_ms = int(time.time() * 1000) - (7 * 24 * 3600 * 1000)

count = 0
for candle in client.iter_candles("BTC-EUR", Interval.MIN_1, since=start_ms):
    count += 1
    # Process candle...

print(f"Streamed {count} candles.")
```

!!! warning "A candle is a dict, not an object"
    `iter_candles()` yields plain dictionaries. `print(candle())` raises
    `TypeError: 'dict' object is not callable`, and `candle.close` raises
    `AttributeError`. Use `print(candle)` to dump the whole record, or
    `candle["close"]` to read one field. All OHLCV values are **strings** so
    decimal precision is never lost — wrap them in `Decimal` before doing math,
    not `float`.

#### Reading the fields

```python
from datetime import datetime, timezone

for candle in client.iter_candles("BTC-EUR", Interval.MIN_1, since=start_ms):
    opened = datetime.fromtimestamp(candle["start"] / 1000, tz=timezone.utc)
    print(
        f"{opened:%Y-%m-%d %H:%M} "
        f"O={candle['open']} H={candle['high']} "
        f"L={candle['low']} C={candle['close']} V={candle['volume']}"
    )
```

#### Bounding both ends of the window
Pass `until` to stop at a fixed point instead of streaming up to *now*:

```python
from datetime import datetime, time as dtime, timedelta, timezone

midnight = datetime.combine(
    datetime.now(timezone.utc).date(), dtime.min, tzinfo=timezone.utc
)
yesterday = midnight - timedelta(days=1)

candles = list(client.iter_candles(
    "BTC-EUR",
    Interval.MIN_1,
    since=yesterday,
    until=midnight,
))
print(f"{len(candles)} one-minute candles for {yesterday:%Y-%m-%d}")  # ~1441, 2 requests
```

Keep both bounds relative to *now* like this rather than hard-coding calendar
dates — a fixed window drifts out of retention as time passes and starts
returning nothing.

The two optional bounds behave like this:

| Arguments | Behaviour |
| --- | --- |
| `since` + `until` | Paginates the whole window, nothing after `until` is yielded. |
| `since` only | Paginates from `since` up to the current time. |
| neither (or `until` only) | **No pagination** — a single request returning the most recent batch. |

#### Stopping early
The generator is lazy: it only issues the next request when you ask for the next
candle, so breaking out of the loop stops the download too.

```python
import itertools
from decimal import Decimal

# Peek at the first 10 candles only — one request, not the full backfill
head = itertools.islice(client.iter_candles("BTC-EUR", Interval.MIN_5, since=start_ms), 10)
for candle in head:
    print(candle["start"], candle["close"])

# Or stop on a condition
for candle in client.iter_candles("BTC-EUR", Interval.MIN_5, since=start_ms):
    if Decimal(candle["high"]) > Decimal("100000"):
        print("First breakout above 100k at", candle["start"])
        break
```

#### Aggregating without buffering everything
Ten thousand one-minute candles is a lot of dicts to hold at once. Fold them as
they arrive instead:

```python
from decimal import Decimal

count = 0
high = Decimal("-Infinity")
low = Decimal("Infinity")
volume = Decimal(0)

for candle in client.iter_candles("BTC-EUR", Interval.MIN_1, since=start_ms):
    count += 1
    high = max(high, Decimal(candle["high"]))
    low = min(low, Decimal(candle["low"]))
    volume += Decimal(candle["volume"])

print(f"{count} candles | 7d range {low}–{high} EUR | volume {volume} BTC")
```

#### Writing straight to CSV
`csv.DictWriter.writerows()` accepts the generator directly, so the file is
written incrementally:

```python
import csv

with open("btc_eur_1m.csv", "w", newline="", encoding="utf-8") as fh:
    writer = csv.DictWriter(
        fh, fieldnames=["start", "open", "high", "low", "close", "volume"]
    )
    writer.writeheader()
    writer.writerows(client.iter_candles("BTC-EUR", Interval.MIN_1, since=start_ms))
```

#### Loading into pandas

```python
import pandas as pd

df = pd.DataFrame(client.iter_candles("BTC-EUR", Interval.HOUR_1, since=start_ms))
df["start"] = pd.to_datetime(df["start"], unit="ms", utc=True)
df = df.set_index("start").astype(float)

df["sma_24"] = df["close"].rolling(24).mean()
print(df.tail())
```

!!! note "How many requests is that, and how long does it take?"
    Each request covers at most 1000 candles, and the client spaces requests
    `request_delay` seconds apart (0.85 s by default) while retrying HTTP 429
    for you. So 7 days of 1-minute data is 10 080 candles ≈ 11 requests ≈ 10 s
    of wall time. For long backfills prefer a coarser interval, or persist the
    result once and read it from disk afterwards.

!!! note "A generator is single-use"
    Once the loop finishes, the iterator is exhausted — a second `for` over the
    same object yields nothing. Materialise it with `list(...)` if you need to
    walk the data more than once.
