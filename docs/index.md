# Revolut X Python
<em>Python SDK for the Revolut X Crypto Exchange REST API (v1.0).</em>

---

!!! tip "Multi-Language Ecosystem"
    Looking for the **PHP SDK**? Check out the [Revolut X PHP Documentation](https://janprikryl1.github.io/revolut-x-php/) or the [PHP GitHub Repository](https://github.com/janprikryl1/revolut-x-php).

---

- **API Version 1.0 Support** — Targets Revolut X REST API `1.0` with configurable versioning.
- **Ed25519 Authentication** — Industry-standard elliptic curve signing for private endpoints.
- **Comprehensive Market Data** — Live order books, real-time tickers, trading pair rules, OHLCV candlesticks, and tick-by-tick public trade feeds.
- **Automated Streaming Iterators** — `iter_candles()` and `iter_trades()` transparently handle cursor pagination and rate-limit windows.
- **Zero-Fee Trading Support** — First-class support for `post_only` Maker orders (0.00% fee).
- **Built-in Helpers** — `FeeCalculator` and `OrderPayloadBuilder` provide pre-submission rule validation and fee projections.
- **100% Type Annotated** — Built with Python `TypedDict`, `Enum`, and modern type annotations for autocomplete in VS Code and PyCharm.
- **Structured Exceptions** — Hierarchical error handling (`AuthenticationError`, `RateLimitError`, `ApiError`, `NetworkError`).
- **AI Agent Skill** — Bundled Skill for AI assistants (Google Antigravity, Claude, Cursor) to automate exchange tasks. See [AI Agent Integration](guide/ai_agents.md).

---

## Installation & Downloads

### Package Installation
```bash
# Release from PyPI:
pip install revolut-x-python

# Direct install from GitHub (latest main):
pip install "git+https://github.com/janprikryl1/revolut-x-python.git"
```

!!! info "Pre-release builds"
    Release candidates land on **TestPyPI** before PyPI. Versions `0.0.1` – `0.1.0`
    exist on TestPyPI only; `1.0.0` is the first release on PyPI proper.

    ```bash
    pip install --extra-index-url https://test.pypi.org/simple/ revolut-x-python
    ```

    Use `--extra-index-url`, not `--index-url`: the `requests` and `cryptography`
    dependencies are not mirrored on TestPyPI and still have to come from PyPI.

### Source Code & Downloads
- **PyPI Package**: [pypi.org/project/revolut-x-python](https://pypi.org/project/revolut-x-python/)
- **GitHub Repository**: [github.com/janprikryl1/revolut-x-python](https://github.com/janprikryl1/revolut-x-python)
- **PHP SDK**: A standalone PHP library is available at [github.com/janprikryl1/revolut-x-php](https://github.com/janprikryl1/revolut-x-php).

---

## Quick Example

```python
from revolut_x import RevolutXClient, OrderSide, OrderType, Interval

# 1. Public Market Data (No API Key Required)
client = RevolutXClient()
ticker = client.get_ticker("BTC-EUR")
print(f"BTC Price: {ticker['last_price']} EUR")

# 2. Authenticated Trading with 0% Fee
client = RevolutXClient(
    api_key="your_api_key_here",
    private_key_path="keys/private.pem",
)

order = client.place_maker_order(
    symbol="BTC-EUR",
    side=OrderSide.BUY,
    quote_size="50.00",
    offset="0.10",
)
print(f"Order ID: {order['venue_order_id']}")
```

---

## Reference

This library was created as part of a **master's thesis** at **VSB – Technical University of Ostrava** (Faculty of Electrical Engineering and Computer Science).

- VSB – Technical University of Ostrava, Faculty of Electrical Engineering and Computer Science
- **Work type**: Master's Thesis
- **Author**: Bc. Jan Přikryl
