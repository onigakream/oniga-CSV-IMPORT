# Phase 1 — Specification: 24/7 Systematic Trading Agent

Status: **DRAFT — awaiting approval at Gate 1 (spec)**
Date: 2026-10-17

> No performance numbers appear in this document. Every Sharpe, drawdown, and
> trade count is `TBD` until the backtest harness (Phase 4) produces it from
> real data. Anything not verified from a primary source is tagged
> **[ASSUMPTION]** or **[UNVERIFIED]**.

---

## 1. System overview

| Item | Proposal |
|---|---|
| Asset class | Crypto. It is the only liquid market that trades 24/7, which is what a 24/7 agent needs. |
| Venue (default) | **Kraken** spot (US-accessible, documented REST + WebSocket API). Alternative: Coinbase Advanced. |
| Perps (optional) | Only if your jurisdiction allows them. Kraken Futures, or Coinbase International (non-US only). Every strategy that needs funding rates is **conditional** on this. |
| Universe | BTC/USD and ETH/USD to start. The universe for Strategy C is a top-N list by trailing liquidity, rebuilt point-in-time. |
| Bar size | 1h bars for signals, daily bars for slow strategies. Nothing below 1 minute: the edge we can credibly claim is not a latency edge. |
| Approach | Generate 5 candidates, then eliminate them through walk-forward tests, cost stress, and parameter-plateau tests. Deploy at most the 1–2 survivors, paper first. |

### 1.1 Venue facts (verify before Gate 2)

| Fact | Value | Source / status |
|---|---|---|
| Kraken spot fees | Range −0.02% maker to 0.80% taker. Depends on 30-day volume and pair. | [Kraken support](https://support.kraken.com/articles/201893638-fee-schedule) |
| Kraken base tier | **Conflicting sources**: 0.40% / 0.80% vs 0.25% / 0.40% | **[UNVERIFIED]**. Must be read from the live `GetTradeVolume` endpoint on your account. |
| Kraken fee restructure | Tier reachable by spot volume, futures volume, or assets held (Jul 2026) | Third-party review, **[UNVERIFIED]** |
| Kraken futures fee endpoint | Deprecated. Its values no longer match fees charged since 2026-06-22. | [Kraken API docs](https://docs.kraken.com/api-reference/fee-schedules/get-fee-schedules.md) |
| Kraken rate limits | ~20 req/s spot, ~50 req/s futures | Third-party, **[UNVERIFIED]**. Kraken's real limits are counter-based per tier. |
| Coinbase perps (non-US) | 0.00% maker / 0.03% taker (promotional), 10 USDC min notional, 10x max leverage | [Coinbase docs](https://docs.cdp.coinbase.com/coinbase-app/advanced-trade-apis/guides/perpetual) |
| Min order size | Per-pair. Read from the `AssetPairs` endpoint (`ordermin`, `costmin`). | Never hard-coded |

**Design rule:** fees, minimum sizes, and tick sizes are **fetched at startup
and stored with every backtest run**. They are never constants in code.

**Backtest cost assumption [ASSUMPTION]:** taker 0.40% per side, plus spread
from historical quotes, plus slippage of 0.5 × spread + an impact term. This is
deliberately worse than base maker fees. A strategy that only survives at maker
rates does not survive.

### 1.2 Regime

I could **not** determine the current regime. This environment blocks
exchange APIs, so I have no real data, and I will not describe the market
from memory. Instead, the regime is a **computed module** (`regime/`) that
reports these on each run:

- Trend: sign and t-stat of 30d/90d returns, price vs the 200-day MA
- Volatility: 30d realised vol and its percentile vs 3 years
- Liquidity: quoted spread and top-of-book depth percentile
- Funding / OI (perps only): 7-day average funding, change in open interest
- Macro calendar: FOMC, CPI, and NFP timestamps from a static calendar file

**Trade / stand-aside conditions (proposed):**

- Stand aside if spread is above the 95th percentile or depth is below the 5th percentile.
- Stand aside if realised vol is above the 99th percentile. Cascades blow through stops.
- Stand aside from 15 min before to 30 min after a tier-1 macro release.
- Stand aside if the venue status endpoint reports anything except `online`.
- Stand aside if data is stale (see §4).

---

## 2. Candidate strategies (signal only; execution is separate)

Five candidates. Each one is **expected to be killed** unless the data says otherwise.

### A. Time-series trend (BTC, ETH; daily)

- **Edge:** crypto returns show autocorrelation at 1–12 week horizons because information diffuses slowly and flows chase returns.
- **Other side:** late mean-reversion traders, and holders forced to rebalance.
- **Entry:** long if 20d EMA > 100d EMA *and* the 60d return is > 0. Flat otherwise. Spot is long/flat only.
- **Exit:** the signal flips, or a 3×ATR(20) trailing stop.
- **Holding period:** weeks. **Frequency:** ~5–15 round trips per asset per year [ASSUMPTION; to be measured].
- **Why it might persist:** behavioural, and capacity-limited only at large size. **Risk:** the trend premium has decayed as the market matured.

### B. Volatility-targeted buy-and-hold (benchmark, not alpha)

- **Edge:** none claimed. Just risk-scaled beta.
- **Purpose:** the **baseline** every other strategy must beat after costs. If A cannot beat B, A is dropped.

### C. Cross-sectional momentum rotation (top-N liquid coins; weekly)

- **Edge:** coins with strong 4-week relative strength keep outperforming over the next 1–2 weeks (attention and flows).
- **Other side:** holders of laggards, and dip buyers.
- **Entry:** each week, rank point-in-time liquid coins by 28-day return (skipping the last day). Hold the top 3, weighted equally by risk. Spot, long-only.
- **Exit:** weekly rebalance, or a coin drops out of the liquidity filter.
- **Main risk:** **survivorship bias.** The universe must include delisted coins. If no delisting-aware dataset can be found, this strategy is **rejected by default**.

### D. Funding carry / basis (conditional on perps access)

- **Edge:** leveraged longs pay persistent positive funding. Long spot plus short perp collects it, roughly delta-neutral.
- **Other side:** leveraged directional speculators.
- **Entry:** 7-day average annualised funding > (round-trip cost × 52 + 5% buffer).
- **Exit:** funding < 0 for 24h, or basis dislocation > X.
- **Risks:** exchange counterparty risk (assets sit on the venue), margin calls on the short leg in a squeeze, and the trade being crowded / arbitraged away.

### E. Post-shock mean reversion (BTC, ETH; 1h)

- **Edge:** liquidity provision after forced liquidations. Overshoot, then partial reversion.
- **Other side:** forced sellers (liquidation engines).
- **Entry:** a 1h return below −3σ (vol from the trailing 30 days) with volume above the 95th percentile. Buy a limit at mid − k·spread.
- **Exit:** +1σ target, a 12h time stop, or a −2σ stop.
- **Main risk:** **cost sensitivity** and catching falling knives in regime breaks. I expect this one to die in cost stress.

### Comparison table (all metrics TBD until Phase 4)

| Strategy | Edge | Market | Timeframe | Sharpe (OOS) | Max DD | Trades | Cost sensitivity | Verdict |
|---|---|---|---|---|---|---|---|---|
| A Trend | Slow diffusion | BTC, ETH spot | 1d | TBD | TBD | TBD | Low (expected) | Candidate |
| B Vol-target hold | None (baseline) | BTC, ETH spot | 1d | TBD | TBD | TBD | Very low | Benchmark |
| C XS momentum | Attention/flows | Top-N spot | 1w | TBD | TBD | TBD | Medium | Blocked on survivorship-free data |
| D Funding carry | Leverage demand | Spot + perp | 8h | TBD | TBD | TBD | Medium | Blocked on jurisdiction |
| E Shock reversion | Liquidity provision | BTC, ETH spot | 1h | TBD | TBD | TBD | **High** | Candidate, likely to fail |

---

## 3. Validation protocol (Phase 4 contract)

1. **Data:** OHLCV plus trades from the venue. Every row has `event_time` and `available_time`, and signals may use only `available_time ≤ decision_time`. A unit test injects future data and must fail on leakage.
2. **Split by time:** 60% train, 20% validation, and a **20% hold-out that is opened exactly once**, at Gate 4. Walk-forward with an expanding window on the train and validation portion.
3. **Costs:** fees from §1.1, historical spread, and a slippage model. Every result is reported at **1×, 2×, and 3× costs**. A strategy is killed if its Sharpe is ≤ 0 at 2×.
4. **Parameter plateau:** sweep ±50% around every parameter. A strategy is rejected if fewer than 70% of neighbours have a Sharpe above 50% of the best.
5. **Multiple testing:** a trial ledger (`results/trials.jsonl`) logs **every** variant ever run. Report the Deflated Sharpe Ratio (Bailey & López de Prado) using that count.
6. **Report:** Sharpe, max drawdown, win rate, profit factor, number of trades, turnover, exposure, and the trial count.
7. **Minimum evidence:** at least 100 OOS trades (or 3 years for daily strategies), or the result is labelled "insufficient sample".

---

## 4. Execution and risk (hard-coded; no model can override)

A `RiskGate` sits between signals and the broker. It is a pure function with no
external inputs except config and state. An LLM or strategy can only *propose*
orders.

| Limit | Default [ASSUMPTION, needs your numbers] |
|---|---|
| Max position per asset | 25% of equity notional |
| Max gross exposure | 100% (spot, no leverage) |
| Max daily loss | 2% of start-of-day equity → flatten and halt until next UTC day plus your approval |
| Max drawdown from peak | 10% → flatten and halt **permanently** until manual re-arm |
| Stale data | Last tick older than 10 s (1h strategies) or last bar older than 2 bars → no new orders |
| API errors | 5 errors in 60 s, or any auth error → halt |
| Order sanity | Reject if price is more than 2% from mid, or size is above per-order cap or below venue `ordermin` |
| Sizing | Volatility targeting (10% annualised per strategy) capped by ¼-Kelly. **Never full Kelly.** |
| Kill switch | CLI command plus file flag. Cancels all orders and market-flattens all positions. Tested in paper every week. |
| Money movement | **The agent has no withdrawal permission.** API keys are created with trade-only scope, and withdrawals are disabled at the key level. |

Every decision is logged as one JSONL record: inputs (data snapshot hash),
signal, risk-gate verdict and reason, order sent, and fill result.

---

## 5. Bear case and automatic stop rules

| Failure mode | Defence |
|---|---|
| Overfitting | Trial ledger, deflated Sharpe, plateau test, hold-out opened once |
| Look-ahead | `available_time` column plus a leakage unit test |
| Survivorship | Delisting-aware universe, or reject Strategy C |
| Cost sensitivity | 2×-cost kill rule |
| Regime dependence | Report results per regime bucket. A strategy must not depend on a single bucket for more than 70% of its PnL. |
| Crowding / arbitraged away | Rolling 1-year Sharpe monitored. Compare to the published decay of trend/carry. |
| Liquidity traps | Size ≤ 1% of median 1-min volume, and stand aside on spread/depth percentiles |
| Exchange failure | Halt on status ≠ online. Reconcile positions with the venue after reconnect. Keep only working capital on the venue. |
| Single host | Watchdog plus heartbeat alert. On restart, the process **reconciles first and trades second**. |

**Live auto-stop (per strategy):** halt if any of these happen:

- Live drawdown exceeds 1.5× the worst OOS drawdown.
- Rolling 60-trade live Sharpe is more than 2 standard errors below the OOS Sharpe.
- Realised slippage exceeds 2× the modelled slippage over 30 trades.

---

## 6. Phases and gates (AgenKit-style)

I could not reach agenkit.xyz (DNS fails from this environment), so I don't
have its exact conventions. I am using this phase structure until you share them:

| Gate | Phase | Deliverable | Needs your approval |
|---|---|---|---|
| **1** | **Spec** | This document | ✅ now |
| 2 | Architecture | Module diagram, interfaces, data schemas | ✅ |
| 3 | Plan | Ordered task list, test list per module | ✅ |
| 4 | Test-first build | Data, backtester, and risk gate with tests. Backtest results fill the table above. Hold-out opened. | ✅ |
| 5 | Review | Adversarial review plus a "how could this fail" pass | ✅ |
| 6 | Ship (paper) | Paper trading ≥ 4 weeks | ✅ |
| 7 | Live (smallest size) | Only with your explicit go | ✅ |

Proposed first module (Phase 4): `data/` ingestion with point-in-time
timestamps plus the leakage test. Nothing else is trustworthy until that exists.

---

## 7. Open decisions for you (blocking Gate 1)

1. **Jurisdiction:** US or non-US? This decides whether perps (Strategy D) are allowed.
2. **Venue:** Kraken (default) or Coinbase, or another exchange you already hold an account on.
3. **Capital and limits:** the paper notional to use, plus the max daily loss % and max drawdown % you want instead of the defaults.
4. **Data budget:** free venue OHLCV only, or a paid point-in-time dataset (needed for Strategy C)?
5. **AgenKit:** if it has a spec template or CLI, share it and I'll conform to it.

---

## HOW COULD THIS SYSTEM FAIL?

- **The weakest assumption is that any of A–E has a real edge after costs at retail fee tiers.** Base taker fees of 0.4–0.8% per side destroy most strategies with holding periods under a day. The most likely result of Phase 4 is: "only A (or nothing) beats the baseline B".
- **The edge may already be arbitraged away.** Trend and carry are widely known, and funding carry in particular is crowded by basis funds with far lower costs.
- **Higher costs.** If the real fee tier is 0.80% taker, E is dead on arrival and C is marginal.
- **The data feed fails:** the stale-data check blocks new orders, and existing positions keep their exchange-side stop orders, which are placed at entry and do not depend on our process.
- **The exchange fails** (outage, insolvency, frozen withdrawals): this cannot be defended in code. The only defence is keeping limited capital on the venue.
- **Operator risk:** the most common failure of retail bots is the human disabling the limits after a drawdown. Re-arming after a halt requires a deliberate manual step and a logged reason.
- **Survival beats a good backtest.** If the honest answer at Gate 4 is "nothing survives", the system will say so and not deploy.
