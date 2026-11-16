# BTC Settlement Market-Making Bot — IMC Hackathon 2026 Winner

I won the IMC Hackathon 2026 overall as a solo entrant. I built this Python bot to make markets in two contracts settled from Coinbase BTC-USD trade data over a 15-minute window. It finished first with **41,900 P&L**, compared with 40,440 for second place, ending flat in Range and 12 lots short in Trades.

- **Trades** settles to the number of Coinbase BTC-USD trades in the window.
- **Range** settles to the highest Coinbase BTC-USD trade price minus the lowest.

The bot estimates each settlement value as the window unfolds, quotes on both sides while controlling inventory, and separately takes sufficiently mispriced orders from the book. The design and reported results are from one competition run; they should not be read as evidence of performance across many windows.

## Results from the competition run

I finished first overall with **41,900 P&L**, compared with 40,440 for second place. The bot finished flat in Range and 12 lots short in Trades.

| Product | Bought / sold | Average buy / sell | Final position | P&L |
|---|---:|---:|---:|---:|
| Trades | 114 / 126 | 2,969.39 / 3,121.91 | -12 | 20,710 |
| Range | 257 / 257 | 1,026.30 / 1,108.76 | 0 | 21,190 |

Range exchange prices include a contract multiplier of 10, so the 82-point average buy/sell difference corresponds to about $8.20 per lot in Range value. The README's original attribution estimated that roughly 92% of total P&L came from matched round trips; the remainder came from the final short Trades position settling below its average sale price. The separate mispriced-order logic contributed little in this run.

This result came from a single 15-minute window, with a 3.6% margin over second place. The bot was dry-run and briefly live-tested before the event, but it was not backtested.

## How the bot works

```text
Coinbase WebSocket --+
                     +--> CoinbaseState --> Trades / Range estimates
Coinbase REST -------+    (seed and gap backfill)

CMI market stream --> BookStore --> Strategy loop (~200 ms)
        |                              +-- passive quotes
        |                              +-- inventory and endgame actions
        +-- latest book --> SnipeWorker +-- aggressive trades
                                               |
                                               v
                                  OrderManager + position ledger
                                               |
                                               v
                                        CMI order API
```

[`bot/main_bot.py`](bot/main_bot.py) wires together the market data, fair-value models, strategy, ledger, and order manager. The key responsibilities are:

| Component | Responsibility |
|---|---|
| [`bot/coinbase_feed.py`](bot/coinbase_feed.py) | Maintains Coinbase trade IDs, prices, observed window statistics, and volatility/rate inputs. Seeds history and repairs trade-ID gaps through REST. |
| [`bot/models_fv.py`](bot/models_fv.py) | Pure calculations for Trades and Range fair value, uncertainty, and operational lower-bound values. |
| [`bot/strategy.py`](bot/strategy.py) | Opening inventory target, passive quote construction, snipe decisions, endgame, and product-level risk controls. |
| [`bot/execution.py`](bot/execution.py) | API rate limiting, order submissions and cancellations, position reservations, fill accounting, and order reconciliation. |
| [`bot/config.py`](bot/config.py) | Model, quote, risk, and window parameters. |
| [`bot/external_feeds.py`](bot/external_feeds.py) | Optional Deribit volatility and Binance activity feeds. Both are disabled in the default configuration. |

CMI book events replace the stored book snapshot. They also feed a dedicated snipe worker, so aggressive opportunities do not have to wait for the regular strategy iteration. The strategy loop runs approximately every 200 ms. Position polling runs about once per second; order reconciliation runs about every five seconds.

## Fair value and uncertainty

Both contracts accumulate known settlement value during the window. The model therefore separates the **observed amount so far** from the **expected amount still to come**. The remaining-time component shrinks toward zero as the window closes. The strategy uses the fair value to centre quotes and decide whether external prices are attractive enough to trade. Its uncertainty estimate scales quote widths and trading thresholds.

### Trades: observed count plus expected remaining trades

The count is inferred from Coinbase trade IDs rather than by counting WebSocket messages:

```text
n_obs = trade_id_end - trade_id_start
```

Coinbase heartbeats report the latest trade ID, so the count can advance even when individual match messages are missed. The feed detects gaps in received match IDs and queues REST backfill. The window start and end IDs, and the current observed count, are assembled in [`CoinbaseState`](bot/coinbase_feed.py).

The fair-value calculation is:

```text
F_Trades = n_obs + lambda_hat * tau
```

Here `tau` is the time remaining and `lambda_hat` is the forecast trade rate. The rate estimator blends:

- A **seasonal prior** of 5,280 trades per 15 minutes.
- A **30-minute regime estimate** derived from Coinbase trade IDs. Once the target window starts, the regime reference is frozen at that start.
- A **fast EWMA** of heartbeat trade rates with a 30-second half-life.
- A **slow in-window rate**, blended from the prior toward the observed count-per-second once at least 10 seconds have elapsed. The prior weight decays on a 300-second scale.

When both seasonal and regime estimates are available and reasonably aligned, the default prior is a geometric blend with 75% seasonal weight and 25% regime weight. A large disagreement shifts more weight to the regime estimate and widens the model's uncertainty. The fast rate receives more weight near the window end; the slow rate matters more earlier. See [`RateEstimator`](bot/models_fv.py).

The remaining-count variance is modelled as:

```text
Var = phi * mu + (c * mu)^2 + sigma_conv^2
mu  = lambda_hat * tau
```

The terms allow for noisy trade arrivals, uncertainty in the forecast rate, and a small count-conversion floor. With the defaults, `phi = 3`, the prior-error coefficient is 0.10 when both prior sources are available, and the conversion floor is 20 trades. Fair value is clamped to at least the count already observed.

### Range: observed range plus expected extension

The observed range is:

```text
R_now = M - m
```

Here `M` and `m` are the highest and lowest Coinbase trade prices in the window. The model also tracks the latest price `S` and estimates price volatility `sigma_hat`.

Volatility is estimated from Parkinson high-low variance. A fast estimate uses 10-second bars; slower estimates use one-minute bars and blend a 30-minute regime estimate with the seasonal prior. The seasonal prior is based on an expected 15-minute range of $166. As the window progresses, completed in-window one-minute bars influence the slow estimate. The fast estimate has increasing influence near the end of the window.

Under a driftless Brownian-motion approximation, the expected amount by which the running maximum and minimum extend over the remaining time has a closed form. Let `s = sigma_hat * sqrt(tau)`. The expected upward extension is:

```text
U    = 2 * s * g((M - S) / s)
g(x) = phi(x) - x * (1 - Phi(x))
```

There is a mirrored expression for the downward extension `D`. The fair value is:

```text
F_Range = R_now + kappa * (U + D)
```

The configured `kappa` is 1. The expected extension is generally larger when the current price is near a running high or low, and smaller when it is well inside the existing extremes. Uncertainty combines the variances of the two extensions, their configured negative correlation, an allowance for volatility-estimation error, and a small conversion floor. Fair value is clamped to at least the range already observed. See [`range_extension` and `range_fair_value`](bot/models_fv.py).

### Lower-bound values

The fair-value functions keep their estimates above the observed count or range. They also return an `lb` value used by the snipe logic to identify especially cheap buy offers. In the current code, `lb` is **margin-adjusted**: Trades uses a rate-dependent margin below the observed count, and Range uses a $2 margin below the observed range. It is an operational threshold, not the strict mathematical minimum settlement value.

## Trading logic

The bot has two complementary trading paths: passive market making and aggressive taking. Passive orders seek spread income while controlling position; snipes seek to trade against book prices that clear a model-based threshold.

### Passive market making

The default mode is `hybrid`. It can establish a temporary opening inventory target when the market and model are sufficiently far apart and the model passes data-quality checks. The target is gated on feed freshness, enough live data, agreement between prior estimates, and confirmation from the live rate or volatility. It ramps up beyond a 1.5-standardized-gap threshold, is capped at half the hard position limit, and decays back toward zero after the opening phase ends. The target calculation is in [`_update_target`](bot/strategy.py).

The default hard position limits are 20 lots for Trades and 60 lots for Range. Passive market making is scaled to half those limits, leaving headroom for aggressive trading. The bot computes skewed model quotes around its fair value. In simplified form:

```text
bid = F - h * (1 + beta * x)
ask = F + h * (1 - beta * x)
```

Here `h` is a half-spread based on settlement uncertainty, `x` is inventory relative to the target and market-making limit, and `beta` controls the skew. Long inventory lowers the bid and brings the ask closer; short inventory does the reverse. The reducing side may be allowed closer to or through fair value to work inventory down.

The half-spread is 0.15 times model uncertainty, clamped to product-specific tick limits. The bot places two levels per side by default, sizes them by product, tapers the inventory-increasing side as inventory grows, and caps reducing-side size so it does not cross its target. When the external book is available, quotes can join or improve it by one tick, subject to fair-value and adverse-selection limits. For passive quote placement only, the centre is blended 60% toward the external mid and 40% toward model fair value. Snipes remain centred on the model value. Quote construction is in [`compute_quotes`](bot/strategy.py).

### Aggressive trading and book snipes

A separate worker evaluates updated books as they arrive. Normal snipe thresholds are based on model uncertainty and are lower on the side that reduces inventory. A minimum tick-distance threshold also applies. Additional rules can target:

- Offers at or below the operational lower-bound threshold.
- Prices more than two adjusted standard deviations from fair value.
- Orders at exchange price limits.
- Cheap levels separated from the next level by a large book gap.

The bot caps ordinary snipes at 3 lots in Trades and 10 lots in Range. A larger fat-finger sweep is allowed only after checks for Coinbase freshness, fair-value stability, inactive spike/burst guards, and agreement between the model and the surrounding book. If those checks fail, the fat-finger trade is capped. See [`_evaluate_view` and `_large_ok`](bot/strategy.py).

Before sending a snipe, the strategy checks position capacity, cancels same-side resting orders when needed to make room, and cancels opposite-side orders that might otherwise cause a self-trade. It submits a limit order and cancels any unfilled remainder, emulating immediate-or-cancel execution. This path is handled by [`_launch`](bot/strategy.py) and [`OrderManager`](bot/execution.py).

### Position limits and execution safeguards

The order ledger tracks polled and estimated positions, resting orders, pending reservations, and fills. It reserves available capacity before an order is sent, so simultaneous requests cannot each assume the same unused headroom. Fill IDs are deduplicated; when a response already reports a fill, the matching later trade event is not counted twice. Uncertain order submissions retain their reservation until reconciliation can resolve the state.

Order traffic runs in worker pools, separate from the strategy and market-data callbacks. A priority token-bucket rate limiter favours snipes and urgent cancellations over normal quote changes and polling. HTTP 429 responses temporarily reduce the request rate. The order manager periodically reconciles the local order table with the exchange and polls positions.

Other controls include:

- **Range spike guard:** near a new running high or low, Range asks are moved farther away and reduced in size.
- **Trades burst guard:** a burst in recent Coinbase trade rate makes Trades asks less aggressive for a period.
- **Inventory ageing:** if inventory does not shrink, the reduce-side price becomes more aggressive over time; skew strength can also increase.
- **Endgame:** the passive position budget shrinks over the final 120 seconds. In the last 30 seconds, the bot may cross the book to reduce inventory within a fair-value-based reach. Passive quotes come off in the last five seconds.
- **Stale data:** passive quotes are pulled when the Coinbase feed is stale. At the configured window end, the bot cancels all orders and stops.
- **Drawdown and API failures:** drawdown can widen quotes or restrict quoting to the reducing side. Repeated order failures trigger cancellation, a pause, and resynchronization.

## Optional settlement override

`settle_override.json` can supply a manual settlement estimate and a minimum edge for either product. While a value is set, passive quotes and automatic endgame flattening stop; the snipe path instead compares orders with that value and may take qualifying levels up to the hard position limit. Setting the product value back to `null` restores normal behaviour. The checked-in file leaves both products unset. This is a deliberate directional mode, and the README-reported competition result did not use it.

## Running the bot

Use Python with the dependencies in [`requirements.txt`](requirements.txt). The bot reads its CMI connection credentials from environment variables and exits at startup if the required values are missing.

Required environment variables:

```text
CMI_URL
CMI_USERNAME
CMI_PASSWORD
```

Optional variables include `CMI_INVITE`, `TRADES_SYMBOL`, `RANGE_SYMBOL`, and `DRY_RUN`. Set `DRY_RUN=1` to run the feeds, strategy, and order planning without sending or cancelling orders through the API. The default is live order execution (`DRY_RUN=0`).

Install dependencies and launch:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
python run_bot.py
```

On macOS/Linux, activate with `source .venv/bin/activate`. The window is configured in [`window_config.json`](window_config.json); its default mode is `cmi_open`, with a 900-second duration. Set `WINDOW_START_UTC` in that file to provide an explicit start time.

Runtime logs and state CSV files are written to `logs/`. Do not commit credentials or live account data.

## Configuration and current defaults

Most strategy parameters are dataclasses in [`bot/config.py`](bot/config.py). Notable defaults include:

| Setting | Default |
|---|---:|
| Window length | 900 seconds |
| Strategy loop | 200 ms |
| Trades / Range hard position limits | 20 / 60 lots |
| Trades / Range passive limits | 10 / 30 lots |
| Trades / Range snipe caps | 3 / 10 lots |
| Passive quote centre | 60% external mid, 40% model fair value |
| Deribit / Binance feeds | Disabled |
| Manual settlement override | Unset |

The model, quote, risk, and feed defaults are mutable in code, not controlled by a general-purpose configuration file. The settlement window and manual override are the runtime JSON configuration files.

## Limitations and next steps

- **One competition window:** the reported P&L comes from one 15-minute period and a small lead over second place.
- **No backtest:** the implementation was validated with dry runs and a short live test, not a historical replay.
- **Trades prior error:** the seasonal prior was too high for the competition window. The in-window rate adjusted as data arrived, but early quotes were centred too high.
- **Settlement count mismatch:** the bot's final Coinbase count was about 2.4% below the exchange settlement count, suggesting a difference in counting conventions or boundaries that should be resolved before relying on the model.
- **Limited sweep usage:** resting orders and the position guard consumed capacity that could otherwise have been used for some mispriced-book opportunities.
- **Unoptimized parameters:** spreads, skew, and sizes were selected by reasoning rather than fitted on a replay dataset.

Useful next steps would be to confirm the exchange's exact settlement-count convention, record synchronized Coinbase and CMI data, replay complete windows, and tune the model and execution parameters on multiple periods. The optional external feeds should remain disabled unless they show measurable improvement under that evaluation.

## Repository map

```text
base_bot.py             CMI authentication, SSE stream, and base REST API methods
models.py               Exchange data structures
run_bot.py              Launcher
bot/config.py           Parameters, environment, and window configuration
bot/coinbase_feed.py    Coinbase WebSocket, REST seeding, and backfill
bot/models_fv.py        Trades and Range fair-value models
bot/strategy.py         Quoting, sniping, inventory, and risk logic
bot/execution.py        Order manager, rate limiter, and position ledger
bot/external_feeds.py   Optional Deribit and Binance inputs
bot/main_bot.py         Runtime wiring and lifecycle
docs/                   Strategy specification and design addenda
logs/                   Runtime logs and recorded state CSVs
```
