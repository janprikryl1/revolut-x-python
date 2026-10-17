# Revolut X Python SDK

Python client library for the **Revolut X Crypto Exchange REST API (v1.0)**.

📖 **Documentation**: [janprikryl1.github.io/revolut-x-python](https://janprikryl1.github.io/revolut-x-python/)  
🐘 **Looking for PHP?** See [revolut-x-php](https://github.com/janprikryl1/revolut-x-php).

---

## Features

- **Revolut X API v1.0** — Complete coverage of market data, order placement, balances, and transaction history.
- **Ed25519 Authentication** — Cryptographic request signing using `cryptography` (PKCS#8 PEM).
- **Smart Maker Strategy (0.00% fee)** — Automatic offset pricing and `post_only` execution to prevent taker fees (0.09%).
- **Streaming Iterators** — `iter_candles()` and `iter_trades()` automatically handle cursor pagination and rate-limit delays.
- **100% Type Annotated** — Built with Python `TypedDict`, `Enum`, and type annotations.
- **AI Agent Skill** — Pre-packaged skill definition for AI coding assistants (Google Antigravity, Claude, Cursor).

---

## Installation

```bash
pip install revolut-x-python
```

In a `requirements.txt`:

```
revolut-x-python>=1.0.0
```

<details>
<summary>Pre-release builds (TestPyPI)</summary>

Release candidates go to **TestPyPI** first. Versions `0.0.1` – `0.1.0` were
published there only; `1.0.0` is the first release on PyPI proper. To install
a pre-release build:

```bash
pip install --extra-index-url https://test.pypi.org/simple/ revolut-x-python
```

Use `--extra-index-url` (rather than `--index-url`) so PyPI stays available
for the `requests` and `cryptography` dependencies, which are not mirrored on
TestPyPI.

</details>

---

## Quickstart

### 1. Public Market Data (No API Key Required)

```python
from revolut_x import RevolutXClient, Interval

client = RevolutXClient()

# Get BTC-EUR ticker
ticker = client.get_ticker("BTC-EUR")
print(f"BTC Price: {ticker['last_price']} EUR")

# Get 1-hour OHLCV candles
candles = client.get_candles("BTC-EUR", Interval.HOUR_1)
```

### 2. Authenticated Trading & 0% Fee Maker Orders

```python
from revolut_x import RevolutXClient, OrderSide

client = RevolutXClient(
    api_key="your-api-key",
    private_key_path="keys/private.pem",
)

# Submit zero-fee Maker limit order (post_only=True)
order = client.place_maker_order(
    symbol="BTC-EUR",
    side=OrderSide.BUY,
    quote_size="50.00",
    offset="0.10",
)
print(f"Order ID: {order['venue_order_id']}")
```

---

## Running Tests

```bash
pytest tests
```

---

## Releasing & Publishing

Publishing to **PyPI** and **TestPyPI** is fully automated via GitHub Actions ([`.github/workflows/release.yml`](.github/workflows/release.yml)).

Whenever commits are pushed to `main`, the workflow runs the test suite and checks [`src/revolut_x/_version.py`](src/revolut_x/_version.py). If a new version is detected (no matching Git tag exists yet), it automatically builds the package, creates a Git tag and GitHub Release, and publishes to the corresponding registry.

### 1. Stable Release (PyPI)

To publish a standard production release to [PyPI](https://pypi.org/project/revolut-x-python/):

1. Bump `__version__` in `src/revolut_x/_version.py` (e.g. `__version__ = "1.2.0"`).
2. Document changes under a matching `## [1.2.0]` header in `CHANGELOG.md`.
3. Commit and push to `main`:
   ```bash
   git commit -am "chore: release 1.2.0"
   git push origin main
   ```
4. The workflow will:
   - Run the test suite (`pytest`).
   - Create and push tag `v1.2.0`.
   - Create a GitHub Release with notes extracted from `CHANGELOG.md`.
   - Publish the package to **production PyPI**.

### 2. Experimental / Pre-release (TestPyPI)

To publish an experimental or pre-release build to [TestPyPI](https://test.pypi.org/project/revolut-x-python/):

1. Set a pre-release version in `src/revolut_x/_version.py` (e.g. `"1.2.0rc1"`, `"1.2.0a1"`, `"1.2.0b1"`, `"1.2.0.dev1"`, or `"1.2.0-exp1"`).
2. Commit and push to `main`:
   ```bash
   git commit -am "chore: pre-release 1.2.0rc1"
   git push origin main
   ```
3. The workflow automatically recognizes the pre-release suffix and routes the upload to **TestPyPI** (and marks the GitHub Release as *Pre-release*).

> **Manual Dispatch**: You can also trigger a release or test run manually anytime from the GitHub web UI via **Actions → Release & Publish → Run workflow**, where you can optionally override the deployment target (`auto`, `testpypi`, or `pypi`).

---

## Academic Reference

This library was written as part of a **master's thesis** at **VŠB – Technical
University of Ostrava**, Faculty of Electrical Engineering and Computer Science.

- **Author**: Bc. Jan Přikryl
- **University**: VŠB – Technical University of Ostrava
- **Faculty**: Faculty of Electrical Engineering and Computer Science (FEI)

---

## Changelog

See [CHANGELOG.md](https://github.com/janprikryl1/revolut-x-python/blob/main/CHANGELOG.md).

---

## License

This project is licensed under the MIT License — see the
[LICENSE](https://github.com/janprikryl1/revolut-x-python/blob/main/LICENSE)
file for details.
