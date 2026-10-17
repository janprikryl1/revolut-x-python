# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

## [1.1.1] - 2026-10-17

### Added
- **`datetime` support for time ranges**: `since` / `until` on `get_candles()`
  and `iter_candles()`, and `start_date` / `end_date` on `get_trades()` and
  `iter_trades()`, now accept a `datetime`, a `date`, an ISO-8601 string
  (trailing `Z` included), a digits-only string of 12+ digits (milliseconds, for
  values arriving from JSON or query parameters) or the millisecond `int` they
  always took. A shorter run of digits — `'2024'`, `'20240101'` — raises
  `ValueError` instead of silently resolving to 1970 or to a date, because it is
  ambiguous between the two; note this also makes `'20240101'` behave the same
  on every supported interpreter, where `datetime.fromisoformat` accepted it as
  a basic-format date on 3.11+ but rejected it on 3.10. Naive
  `datetime`s and strings are interpreted as **UTC** rather than the machine's
  local timezone, so the same code selects the same window everywhere. Raw
  numbers are still passed through untouched — seconds are deliberately *not*
  auto-detected, because inferring the unit from the magnitude would silently
  change the meaning of the caller's input.
- **`to_timestamp_ms()`** (exported from `revolut_x`, together with the
  `TimeLike` type alias) performs that conversion and is reusable in
  application code.

Existing millisecond integers keep working unchanged.

## [1.1.0] - 2026-10-12
### Fixed
- **`get_ticker()` returned a random region's price.** `/public/tickers` serves
  one row per pair *and* region — 777 rows for 455 pairs, because 322 symbols
  are quoted in both `EEA` and `UK`, in every case at a different price and
  volume. `get_ticker()` returned the first match, and the exchange reorders
  the list between calls: across three consecutive requests the leading region
  flipped for 251 of those 322 symbols. Callers therefore saw prices jump
  (`RAD/USD` alternating between `0.3166` at volume 63 403 and `0.3193` at
  volume 1 015) with nothing having happened on the market, which is enough to
  trip any logic that compares quotes over time. It now returns the most liquid
  quote — highest `volume_24h`, region name breaking an exact tie — so repeated
  calls agree. Single-region pairs, including the documented `BTC/EUR`, are
  unaffected. To target one region, filter `get_tickers()` on `['region']`.

### Documentation
- **`get_tickers()`** no longer describes itself as returning tickers "for all
  pairs", which implied one row per pair; it documents the per-region
  duplication, the unstable ordering, and the `region` field.

## [1.0.0] - 2026-10-11

First release published to **PyPI**. Versions `0.0.1` – `0.1.0` were only ever
uploaded to TestPyPI, so `pip install revolut-x-python` works without an extra
index from this release onwards. No behavioural changes to the API surface —
the jump to `1.0.0` marks the public release and the stability promise, and
everything in `0.1.0` (verified against the live exchange) carries over
unchanged.

### Added
- **PyPI metadata**: trove `classifiers` (development status, supported Python
  3.10 – 3.14, audience, topic, `Typing :: Typed`) plus `Documentation` and
  `Changelog` entries in `[project.urls]`, so the project page renders the
  sidebar links and the version/compatibility filters work.
- **`RELEASING.md`**: the manual release procedure (version bump → build →
  `twine check` → TestPyPI staging → PyPI upload → tag), including the
  `rm -rf dist` step that keeps `twine upload dist/*` from re-uploading stale
  artifacts from an earlier build.
- **`MANIFEST.in`**: the sdist now ships `CHANGELOG.md`, `requirements.txt`,
  the `examples/` directory and the AI agent skill (`.agents/`), which is an
  advertised feature of the package but used to be reachable only by cloning
  the repository. `docs/`, `site/`, `dist/` and `venv/` are pruned.

### Changed
- **PEP 639 licensing**: added `license-files = ["LICENSE"]` and raised the
  build requirement to `setuptools>=77`, which is the first version that
  understands the `license = "MIT"` SPDX expression already declared here.
  The dev extra now requires `twine>=6.1` for the same reason — earlier twine
  rejects Metadata 2.4 artifacts. The `LICENSE` file is included in the wheel
  instead of being dropped silently.
- **`[tool.setuptools.packages.find]`**: constrained to `include = ["revolut_x*"]`
  so nothing else under `src/` can be picked up as a top-level package.
- **Installation docs rewritten** for the PyPI release (`README.md`,
  `docs/index.md`, `docs/index.cs.md`, `docs/guide/ai_agents.md`,
  `docs/guide/ai_agents.cs.md`, agent `SKILL.md`): plain
  `pip install revolut-x-python` is now the primary path, with TestPyPI kept as
  a documented pre-release channel. The TestPyPI command was corrected from
  `-i` to `--extra-index-url` — with `--index-url` the install fails outright,
  because `requests` and `cryptography` are not mirrored there.
- **Fixed stale install commands**: the GitHub install line carried a
  `#subdirectory=python` fragment from the old monorepo layout (the package
  lives at the repository root), and several commands and links still pointed
  at the pre-rename `revolut-x-library` repository.
- **README**: translated the remaining Czech section (`Academic Reference`),
  since the README is the PyPI project description, and made the `LICENSE` and
  `CHANGELOG` links absolute so they resolve on the PyPI page.

---

## [0.1.0] - 2026-10-10

Correctness release. Two public methods were unusable as shipped, and several
declared types did not match what the exchange actually returns. Every item
below was verified against the live Revolut X API.

### Fixed
- **`iter_candles()` crashed on page-aligned ranges**: any window that was an
  exact multiple of 1000 candles ended with a request where `since == until`,
  which the exchange rejects with HTTP 400 (`Invalid interval: 'until' is
  before 'since'`). Windows of 1000/2000/3000 candles failed; 999/1500 worked.
  Pagination now walks fixed windows forward instead of chaining off the last
  returned candle, which also makes it immune to gaps in the series (an
  illiquid pair with no trades in an interval used to end iteration early).
  Guaranteed: no candle yielded twice, none past `until`.
- **`get_trades()` / `iter_trades()` failed out of the box**: the default
  `limit` was 1900, but `/public/trades/all` accepts only **1–100** and rejects
  anything larger with HTTP 400 (`Limit must be between 1 and 100`). The
  default is now 100 and out-of-range values are clamped into the accepted
  range.
- **Query parameters are now sorted before being sent**: the canonical
  signature message sorts them alphabetically, while `requests` serialised them
  in insertion order, so the signed string and the string on the wire could
  differ for authenticated paginated calls (`get_historical_orders`,
  `get_transactions`, `get_account_trades` when a `cursor` was present).

### Changed
- **`OrderBook` type corrected**: levels are dictionaries, **not**
  `[price, quantity]` pairs. Added `OrderBookLevel` (exported from
  `revolut_x`) documenting all 13 fields the venue returns; read the price from
  `['p']` and the quantity from `['q']`. The previous shape made every
  documented order-book example raise `KeyError` or `ValueError`.
- **`Ticker` type corrected**: `high`, `low` and `volume` do not exist in the
  response — the real fields are `high_24h`, `low_24h` and `volume_24h`. Added
  the missing `mid`, `index_price`, `price_change_24h`, `quote_volume_24h` and
  `region`.
- **`Trade` type**: added the `region` field returned by the API.
- **`get_order_book(depth=...)`**: documented that the exchange currently
  ignores this argument and always returns 5 levels per side. The parameter is
  kept so deeper books are picked up automatically if that changes.
- **`get_candles()` docstring**: a window wider than 1000 candles is a hard
  HTTP 400, not a silent truncation. The previous example (`since` three years
  in the past) could not succeed.
- **Single source of truth for the version**: `pyproject.toml`,
  `revolut_x.__version__` and the default `User-Agent` now all resolve to
  `revolut_x._version.__version__`. They had drifted to three different values
  (0.0.8 / 0.0.6 / 0.1.0 respectively).
- **Documentation**: translated the remaining Czech text in the English guides
  (`index.md`, `concepts/auth_ed25519.md`, `concepts/maker_strategy.md`,
  `guide/market_data.md`) left over from the i18n migration.
- **AI Agent Skill**: corrected the order-book and ticker shapes in
  `.agents/skills/revolut-x-library/`, which had been teaching assistants the
  wrong field names.

### Added
- **Tests**: `tests/test_market.py` — 7 tests covering candle pagination
  (page-aligned windows, exact range coverage, no overshoot past `until`, gap
  tolerance), the trades `limit` bound, and order-book pass-through. The suite
  is now 43 tests.

---

## [0.0.8] - 2026-10-05

### Added
- **Timestamp synchronisation**: `sync_time()` derives a clock offset from the
  server `Date` header, with automatic drift correction on HTTP 409 and a
  `timestamp_offset_ms` client option, so request signing survives a skewed
  local clock ([`d9c5127`](https://github.com/janprikryl1/revolut-x-library/commit/d9c5127)).

---

## [0.0.7] - 2026-10-05

### Changed
- **Repository restructured**: the Python SDK moved from `python/` to the
  repository root (`src/`, `tests/`, `examples/`), so the package builds from
  the top level ([`52af8e3`](https://github.com/janprikryl1/revolut-x-library/commit/52af8e3)).
- **PHP SDK removed** from this repository; it now lives in
  [revolut-x-php](https://github.com/janprikryl1/revolut-x-php).

### Added
- **Unified multi-language documentation**: MkDocs site with English and Czech
  (`mkdocs-static-i18n`, Material theme), shared protocol concepts
  (`docs/concepts/auth_ed25519.md`, `docs/concepts/maker_strategy.md`), and a
  GitHub Actions workflow deploying to GitHub Pages.
- **AI Agent Skill**: skill definition and types reference
  (`.agents/skills/revolut-x-library/`) for AI coding assistants
  ([`c586fef`](https://github.com/janprikryl1/revolut-x-library/commit/c586feffa491bfde7c63b891893746c9571cd68d)).

## [0.0.6] - 2026-09-29

### Added
- **Smart Maker Order Strategy**: Added support for zero-fee Maker orders with dynamic offset pricing (`MakerOrderStrategy`) to prevent unintentional taker fees ([`f65686c`](https://github.com/janprikryl1/revolut-x-library/commit/f65686c9553cde642a3fdafc3bb9f05dcd4a092b)).
- **Example**: Added `python/examples/06_smart_maker_orders.py` showcasing zero-fee maker order placement and offset calculation.
- **Tests**: Added unit test suite `python/tests/test_maker_strategy.py`.
- **Documentation**: Updated order documentation in `python/docs/guide/orders.md` with maker strategy guidelines.

---

## [0.0.5] - 2026-09-29

### Fixed
- **Documentation**: Fixed and refined documentation across `python/README.md` and `python/docs/index.md` ([`7fd5100`](https://github.com/janprikryl1/revolut-x-library/commit/7fd5100b8826ce4e126f17a251e76165e2939930)).

---

## [0.0.4] - 2026-09-29

### Changed
- **API Alignment**: Renamed client methods for full consistency with the Revolut X Crypto Exchange API v1.0 specifications and Python conventions ([`37d1e03`](https://github.com/janprikryl1/revolut-x-library/commit/37d1e032cb6a5379a181e52432b61b87f384244e)).
- **Documentation**: Added explicit references to Revolut X API v1.0 throughout docstrings and documentation.

### Added
- **Tests**: Added dedicated orders unit test suite `python/tests/test_orders.py`.

---

## [0.0.3] - 2026-09-28

### Added
- **Packaging**: First release uploaded to TestPyPI ([`4471e7e`](https://github.com/janprikryl1/revolut-x-library/commit/4471e7efb6f964ea571e7a6d9735e9e66cd1cc21)).
- **Monorepo Structure**: Added root `README.md` describing multi-language SDK layout.
- **Type Information**: Added PEP 561 `py.typed` marker file for strict type checking support.

### Changed
- **Package Layout**: Migrated Python package to modern `src/` layout (`python/src/revolut_x/`).
- **Dependencies**: Configured optional dependencies (`dev`, `docs`) in `pyproject.toml`.

---

## [0.0.2] - 2026-09-28

### Added
- **CI/CD**: Added GitHub Actions workflow (`.github/workflows/docs.yml`) for automated MkDocs documentation build and GitHub Pages deployment ([`ae145d0`](https://github.com/janprikryl1/revolut-x-library/commit/ae145d0b7b2448e29280e904c27ae0c96eccf20d)).

### Fixed
- **CI/CD**: Fixed Python directory paths in the GitHub Actions documentation deployment workflow ([`a220efd`](https://github.com/janprikryl1/revolut-x-library/commit/a220efd650652765f30e26d146a344a4736b41d0)).
- **Repository Metadata**: Corrected repository display name in `mkdocs.yml` ([`c33a065`](https://github.com/janprikryl1/revolut-x-library/commit/c33a0655d4584cb534e758f3d380b110333f5c33)).

---

## [0.0.1] - 2026-09-28

### Added
- **Initial Implementation**: Initial commit of the Revolut X Crypto Exchange REST API Python SDK ([`c6857a5`](https://github.com/janprikryl1/revolut-x-library/commit/c6857a5e2ddc1f5da5d3cb02092bbf339ac7579e)).
  - **Client (`RevolutXClient`)**: Main client supporting both public and authenticated endpoints.
  - **Authentication (`_auth.py`)**: Ed25519 cryptographic request signing using private keys.
  - **HTTP Transport (`_http.py`)**: Robust HTTP session layer with request signing and error handling.
  - **Market Data API (`market.py`)**: Endpoints for tickers, order books, OHLCV candles, and public trades.
  - **Account API (`account.py`)**: Balances retrieval and account configuration.
  - **Orders API (`orders.py`)**: Placing limit and market orders, order cancellation, and querying active/historical orders.
  - **Helpers (`helpers.py`)**: Fee calculation utilities (Maker vs Taker), precision helpers, and payload validation.
  - **Data Types (`types.py`)**: Type definitions and enums (`OrderSide`, `OrderType`, `TimeInForce`, `Interval`).
  - **Error Handling (`exceptions.py`)**: Structured exception hierarchy (`AuthenticationError`, `RateLimitError`, `APIError`, etc.).
  - **Examples**: Example scripts covering public data, candle streams, account history, orders, and fee calculation.
  - **Unit Tests**: Test suites for authentication and helper functions (`test_auth.py`, `test_helpers.py`).
  - **Documentation**: Complete MkDocs documentation site setup with guides and API references.
