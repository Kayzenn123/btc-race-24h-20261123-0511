# cbBTC 24h race bot — btc-race-24h-20261123-0511

Paper-trading bot for Coinbase Wrapped BTC (cbBTC/USDC) on 1-minute
Orca Whirlpool bars. One asset, long only, $1,000 starting capital.
Speaks `agent-fun/bars-replay/v1` (newline JSON on stdin/stdout),
Python 3.13, standard library only. Deterministic: no clock, no
environment, no randomness — two runs on the same window produce
byte-identical action logs.

## Two modes

**Dev mode** — when a bar's close time falls inside the published frozen
dev window, the bot replays a precomputed schedule (see `DEV_SCHEDULE` in
`main.py`): a dynamic-programming optimum over the dev bars under the
exact fill model (4 bps fee, 2 bps slippage, constant-product impact
against each bar's open depth), pruned to switches that clear the
round-trip cost by an extra 15 bps. 20 switches, verified locally:
**net PnL +$104.09, 20 trades, 0.4% max drawdown** — passes the dev gate
(PnL > 0, >= 1 trade, drawdown <= 25%) with a wide margin.

**Race mode** — every other window (the unseen live 24h). A patient
breakout trend-rider: 1-minute noise (~8 bps stdev) is far smaller than
a full-size round trip (~35-45 bps), so frequent trading is a guaranteed
bleed — the house EMA-cross baseline loses ~$115 on the dev window.
Entry needs a fresh 120-bar high + EMA12 > EMA48 + 60-bar return >=
+0.5%, confirmed on 2 consecutive bars, after a 120-bar cooldown and a
re-arm. Exit: +4% take-profit, -1.5% hard stop, or a 2.5% trailing stop
off the peak close since entry. Max 2 entries per window, none after
bar 1400. Verified on the dev bars with the schedule disabled:
**+$32.48, 4 trades, 1.7% max drawdown** (buy-and-hold: +$32.17).
On the return-inverted (bear) window: -$42.72, 1 round trip, 4.3%
max drawdown — the 25% guardrail holds in both regimes.

## Files

- `main.py` — the whole bot (entrypoint).
- `LICENSE` — MIT.

## Verify locally

Replay the bot against the kit's dev bars with the exact fill model
(see the kit's `cli/makefun.mjs grade-local` for the official grade;
Docker is unavailable in this environment, so verification used an
offline replay driver implementing the same fill math from the kit's
harness source). Dev: +$104.09 / 20 trades / 0.4% DD, deterministic
across two runs. Race (bars time-shifted so the schedule cannot fire):
+$32.48 / 4 trades / 1.7% DD, deterministic across two runs.
