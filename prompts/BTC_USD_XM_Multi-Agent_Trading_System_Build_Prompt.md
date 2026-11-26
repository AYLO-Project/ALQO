# BTC/USD Multi-Agent Trading System — Build Prompt (XM · MT5)

Nov 26, 2026 · adapted from the XAU/USD build prompt of Nov 25, 2026

---

## Role & parameters

**How to use:** fill every [bracket], then paste the whole document into your model. Leave out the two sections at the end, "Conversion notes" and "Reference links". Copy the XM values from the MT5 *Symbol Specification* window for your own account, not from this document or third-party sites.

Act as a Principal Quantitative Researcher, Multi-Agent Systems Architect and Algorithmic Simulation Engineer specializing in digital-asset market microstructure. We are building a production-grade BTC/USD trading system from scratch. It trades the BTCUSD CFD at XM through MetaTrader 5. Autonomous agents generate and filter signals. A deterministic Risk Router approves every order. A high-fidelity event-driven simulator validates each component before any capital is at risk.

Treat these parameters as binding. Ask about one only if it is missing and blocks the design.

- **Venue & instrument:** BTCUSD CFD at XM. Symbol name as shown in MT5: [BTCUSD / with suffix]. XM entity: [XM Global / Trading Point (CySEC) / other]. Account type: [Standard / Ultra Low / other that lists crypto].
- **Execution API:** MetaTrader 5, through [the official `MetaTrader5` Python package, which needs a running MT5 terminal on Windows logged in to XM] / [an MQL5 Expert Advisor bridge over sockets or ZeroMQ].
- **Account:** equity [USD __] · account currency [USD]. If the currency isn't USD, convert P&L at the broker's rate.
- **Contract size & lot step:** [1 lot = 1 BTC; minimum 0.01 lot = 0.01 BTC; step 0.01; maximum __ lots per order]. Also record the MT5 *stops level*, *freeze level*, price digits, margin rate and swap settings.
- **Account margin mode:** [hedging / netting], read from `ACCOUNT_MARGIN_MODE`.
- **Holding horizon:** [intraday, flat before Friday 21:00 UTC and before XM maintenance windows] / [swing, 1–10 days].
- **Bar stack:** [M1 execution · M5/M15 signals · H1/H4/D1 context]. Build D1 bars on the UTC day from M1 data. XM's own D1 bars open at server midnight, not at 00:00 UTC.
- **Data budget:**
  - [free: XM MT5 ticks recorded by you; Binance, Coinbase, Kraken and Bitstamp public market data; Deribit public API; FRED/ALFRED; CFTC; SEC EDGAR; GDELT; ETF issuer pages]
  - [paid: Tardis.dev / Kaiko / CoinAPI ticks and order books; CoinGlass / Laevitas derivatives data; Glassnode / Coin Metrics on-chain data; licensed crypto newswire; consensus forecasts]
- **Risk targets:** max drawdown ≤ [X]% · risk per trade ≤ [Y]% of equity at the stop.
- **Clock:** store all timestamps in UTC, convert with the IANA tz database, and display in [your timezone]. XM's MT5 server runs on [GMT+2 in winter / GMT+3 in summer — verify]. Derive the server-to-UTC offset from the data; never hard-code it.

---

## Core objectives & hard constraints

1. **Objective:** maximize out-of-sample CAGR subject to max drawdown ≤ [X]% and Calmar ≥ [Z]; break ties on Sortino. The system must also beat buy-and-hold BTC/USD on Calmar over the same windows. Raw CAGR is a misleading test for Bitcoin. BTC fell about 77% from its November 2021 high to its November 2022 low. It set a record near $126,200 on 6 October 2025 and had lost more than half of that by late November 2026.
2. **Zero added leverage:** gross notional (|position in BTC| × mid price) must stay ≤ 1.0 × account equity at all times, checked before every order and on every price update.
   - XM's BTCUSD is a margined CFD. Never use margin to exceed 1.0x.
   - In a hedging-mode account, long and short tickets both count toward gross notional. The router holds at most one net position per symbol.
   - Near $58,600 (25 November 2026), one 1-BTC lot is about $58,600 notional and the 0.01-lot minimum about $586. Size in BTC and round down to the lot step.
   - If the minimum lot breaches the cap or the risk budget, skip the trade and log it. Report the minimum equity needed to trade the intended sizes.
3. **No tabular gradient-boosted trees:** do not use XGBoost, LightGBM or CatBoost on OHLCV or technical-indicator features.
4. **Approved intelligence stack:**
   - structural market mechanics, including derivatives mechanics: perpetual funding, open interest, futures basis, liquidations and options skew;
   - transformer NLP/LLM news and macro intelligence (FinBERT, RoBERTa, LLM agents);
   - economic-calendar circuit breakers;
   - statistical regime models (HMM, GARCH family);
   - zero-shot foundation time-series models (Chronos, TimeGPT).

   Cross-asset, flow and on-chain state variables from the Bitcoin driver map may be used for regime conditioning and vetoes. They may never be inputs to a black-box fit.
5. **Scale-free parameters:** no fixed-dollar or "pip"/"point" thresholds. Express stops, targets, spread limits, breakout buffers and slippage as multiples of ATR or realized volatility, or in basis points of price.
   - BTC has traded from about $3,200 (December 2018) to about $126,200 (October 2025), a roughly 40× range.
   - An MT5 "point" depends on each broker's price digits, and brokers' BTCUSD specifications differ.
   - BTC trades 24/7. Annualize volatility and returns on 365 calendar days of 24 hours, not 252 trading days.
6. **No LLM in the order path:** LLM agents emit scores only. Deterministic code sizes, routes, amends and cancels orders. Every LLM call has a timeout and a neutral fallback (score 0, confidence 0).
7. **Honesty rule:** never state an expected CAGR, Sharpe ratio, win rate or drawdown. Every metric must come from running the engine on data; until then, write TBD.

---

## Bitcoin driver map

BTC/USD responds to several channels whose strength changes over time. All agents share this map. Estimate rolling, regime-conditional sensitivities; never hard-code a sign.

- **Global liquidity, the US dollar and the Fed path:** broad dollar index (DXY, or FRED DTWEXBGS), the 10-year TIPS yield (FRED DFII10), the 2-year Treasury yield, fed funds futures/OIS, and the yen-carry channel (BoJ policy, USD/JPY).
  - Model changes, not levels, and let the betas vary by regime.
  - The nomination of a hawkish Fed chair on 30 January 2026 hit BTC alongside equities and metals.
- **Risk appetite and equities:** Nasdaq-100 and S&P 500 returns, and credit spreads. BTC's correlation with tech equities rises and falls over time. Estimate it on a rolling window; never assume it.
- **Structural and institutional demand:**
  - US spot Bitcoin ETFs, trading since 11 January 2024: daily net creations and redemptions, published after the US close.
  - Corporate treasury buyers, disclosed in SEC 8-K filings.
  - Sovereign policy, e.g., the US Strategic Bitcoin Reserve order of 6 March 2025.
- **Supply and overhang:**
  - the halving schedule (block-height based: April 2024 at block 840,000; the next at block 1,050,000);
  - miner flows and hashprice;
  - government and bankruptcy-estate distributions (e.g., German government sales and Mt. Gox repayments in mid-2024);
  - long-term-holder distribution.
- **Leverage and positioning:**
  - perpetual funding rates, open interest and liquidation flows on the major venues;
  - CME futures basis and open interest;
  - CFTC Traders in Financial Futures (TFF) positioning for CME Bitcoin futures: leveraged funds versus asset managers.

  Crowded leverage unwinds violently. On 10–11 October 2025 about $19 billion of crypto positions were liquidated in roughly 24 hours, the largest such event on record.
- **Stablecoin liquidity and venue health:**
  - USDT/USDC supply growth and depeg events;
  - exchange solvency, hacks and withdrawal halts, e.g., FTX in November 2022, Bybit on 21 February 2025, and Binance's roughly $6.3 billion hot-wallet hack on 29 October 2026.
- **Regulation and policy:** SEC/CFTC actions, US legislation (e.g., the GENIUS Act, signed 18 July 2025), EU MiCA, and bans or crackdowns in major jurisdictions.
- **Risk aversion and geopolitics:** tariff shocks, wars and sanctions.
  - In a dash for cash, BTC is sold as a liquid risk asset: March 2020, 2022, and the February 2026 leverage unwind.
  - In a debasement narrative, it can rally with gold.
  - The sign is unstable; let the data decide.
- **Implied volatility:** Deribit DVOL, Deribit and CME options skew (25-delta risk reversals), and CME Bitcoin Volatility futures where available.

---

## Part 1 — Orchestrator & shared message rules

All agents publish typed messages to one asynchronous event bus. The deterministic Orchestrator / Risk Router is the only component allowed to create, amend or cancel orders.

- **Two timestamps on every message:** `event_time` (when it happened) and `available_time` (when this system could first know it). The router rejects any message whose `available_time` is later than the current clock.
- **One clock, two modes:** the same agent code runs on a simulated clock in backtests and on the wall clock live. No backtest-only code paths.
- **Message contracts:** MarketEvent, ReferencePriceEvent (composite spot), DerivativesEvent (funding, OI, liquidations, basis), CalendarEvent, NewsSignal, RegimeState, AlphaSignal, RiskDecision, OrderIntent, Fill, SymbolSpecEvent (XM spec, margin or trade-mode change), Heartbeat. Each carries `source`, `model_version`, `confidence` and `ttl`; messages past their time-to-live are ignored.
- **Router decision order:**
  1. **Hard gates:**
     - Agent 3 blackout;
     - spread breaker;
     - feed-integrity breaker: XM mid deviates from the composite spot mid by more than [k] × the rolling σ of their basis;
     - venue-stress flag;
     - XM symbol set to close-only or margin rate changed;
     - exposure cap;
     - equity breakers.
  2. News veto or boost (Agent 2).
  3. Regime routing (Agent 4).
  4. Sizing (Agent 5), then rounding down to the lot step and checking XM's min, max and step volume and stops level.
  5. Emit OrderIntent; log every rejected signal with a reason code.
- **Failure behavior:**
  - A missing heartbeat from Agent 3 or Agent 5 blocks new entries.
  - So does a missing composite reference feed, because the feed-integrity check can't run without it.
  - A missing Agent 2 or Agent 4 falls back to neutral scores and base size.
- **Broker-side protection:** every open position keeps a resting stop-loss at XM, attached to the MT5 position, so a disconnect never leaves it unprotected.
  - Place it at least the symbol's stops level from price.
  - Use two layers: the router manages the strategy stop, and a wider disaster stop rests at the broker. XM's stops trigger on XM's own price feed, which can spike or freeze when exchanges don't.
- **Reconciliation:** after every fill, reconnect and session start, compare MT5 positions, orders and balance against the router's state. On mismatch, block new entries and alert.

---

## Agent 1 — Alpha & Structure

Generates candidate trades from clean, low-parameter price structure. Each setup is its own module that can be tested and switched off alone.

- **Session structure on a 24/7 market:**
  - the Asia range (configurable, e.g., 00:00–07:00 UTC) and its breakout or failure at the European open;
  - continuation or reversal into the US cash-equity and spot-ETF open (09:30 ET);
  - the weekend range and how Monday resolves it.
- **Liquidity sweeps:**
  - Levels: the prior UTC-day high/low, the Asia range, the weekly open (Monday 00:00 UTC), the monthly open, and round-number handles ($1,000 / $5,000 / $10,000), where stops, options strikes and liquidation levels cluster.
  - Estimated liquidation clusters from open interest and leverage. These are model estimates, so use them only point-in-time.
  - For history before 29 May 2026, also the CME weekend-gap levels.
  - The level is in dollars; the sweep buffer is ATR-scaled.
- **Trend core:** multi-timeframe alignment (D1/H4/H1) with volatility-scaled entries.
- **Volatility expansion:** breakouts after measured compression (range or Bollinger-width percentile, or DVOL percentile).
- **Mean-reversion sleeve:** fades of session-VWAP or z-score extremes, active only in the chop regime, with a time stop.
- **Positioning-extreme fade (optional, off by default):** fades a failed sweep when funding and open-interest z-scores are extreme. Active only in chop or the leverage-euphoria regime, with a tight time stop.
- **Real volume:** XM's BTCUSD is a CFD, so MT5 tick volume is not market volume.
  - Use composite spot volume from reputable venues (e.g., Coinbase, Kraken, Bitstamp, Binance). Exclude venues with suspected wash trading.
  - Add CME BTC (5 BTC) / MBT (0.1 BTC) volume and open interest (roll-adjusted front month) and perpetual volume and open interest for confirmation.
  - Compute VWAP on the composite, then map it to XM's price through the rolling XM–composite basis.
- **Output (AlphaSignal):** direction, setup id, entry, ATR-scaled stop and target, expected holding time, raw confidence.

---

## Agent 2 — News & Macro Intelligence

Turns point-in-time news, regulatory and central-bank text into a Bitcoin-specific impact score.

- **Generic sentiment is the wrong label:** FinBERT's positive/negative is not Bitcoin's up/down.
  - A strong payrolls print reads "positive" and is often bearish for BTC through the rates channel.
  - "Fed signals no cuts" can read neutral and is usually bearish.
  - A hack or a withdrawal halt at a major exchange is bearish even when the story is framed as a recovery.
- **Pipeline:**
  1. Relevance filter.
  2. Channel tag: USD/liquidity/Fed; risk appetite and equities; regulation and policy; ETF and institutional flows; corporate and sovereign treasuries; venue security and solvency; stablecoins; supply overhang; network and protocol; geopolitics and tariffs.
  3. Signed BTC impact per channel, from an LLM with a fixed rubric or FinBERT plus a channel sign map.
  4. Decay-weighted aggregation.
- **Outputs (NewsSignal):**
  - `btc_impact_score` in [−1, +1];
  - `event_shock_velocity`: change in headline count and impact over 5–30 minutes;
  - `narrative_shift` alerts: Fed pivot, regulatory action or approval, exchange hack or insolvency, stablecoin depeg, sovereign or corporate adoption, large overhang distribution;
  - `confidence`;
  - source ids.
- **Veto and boost:** veto an opposing technical trade when |score| ≥ θ and confidence ≥ c. When aligned, apply a bounded size multiplier. Fit θ and c only in walk-forward.
- **Source verification:** crypto news carries a high rate of false or compromised headlines.
  - On 16 October 2023 a major crypto outlet falsely reported a spot-ETF approval.
  - On 9 January 2024 the SEC's X account was compromised and posted a false approval.
  - No headline may push |score| past θ until a primary source (regulator, issuer, exchange status page, SEC EDGAR) or [2] independent credible outlets confirm it. Weight sources by historical reliability.
  - Store social posts with retrieval time and edit/deletion status.
- **Leakage guard:** LLMs were trained on text from your backtest years and may "know" what BTC did next.
  - Prefer models whose training cutoff predates the test window.
  - Mask names, tickers and dates where possible.
  - Judge the agent on data after its cutoff.
- **Reproducibility:** cache every model output by headline id, model version and prompt hash. Backtests read the cache, never a live API.

---

## Agent 3 — Economic Calendar & Microstructure Guardrail

Enforces point-in-time event gates and liquidity filters, with absolute veto power over new entries.

### Event tiers

- **Tier 1 (full protocol):** NFP, US CPI and Core PCE (all 08:30 ET), and FOMC. Treat FOMC as two events: the 14:00 ET statement and the 14:30 ET press conference. Its blackout runs until 30 minutes after the press conference ends.
- **Tier 2 (no new entries T−15m to T+15m):**
  - PPI, retail sales, ISM manufacturing and services, JOLTS, weekly jobless claims, advance GDP, UMich inflation expectations;
  - FOMC minutes, Fed Chair testimony and Jackson Hole, large Treasury auctions;
  - BoJ policy decisions, for the yen-carry channel;
  - scheduled SEC/CFTC decision deadlines and congressional votes on crypto legislation.
- **Crypto-specific scheduled events:**
  - Deribit option expiries (Fridays 08:00 UTC; monthly and quarterly are the large ones);
  - CME Bitcoin futures and options expiry (last Friday of the month, settled on the CME CF Bitcoin Reference Rate at 16:00 London);
  - perpetual funding timestamps (typically 00:00 / 08:00 / 16:00 UTC; read each venue's schedule, since some contracts settle more often);
  - the US spot-ETF session open and close (09:30 / 16:00 ET) and the following publication of flow data;
  - corporate-treasury 8-K filing windows;
  - XM's swap rollover (server midnight) and announced maintenance windows;
  - US market holidays, when there are no ETF flows;
  - the halving, a block-height event whose time you estimate from block cadence and update as it approaches.

### Tier-1 windows

- **Pre-release buffer (T−60m to T−15m):** no new entries; tighten stops or trim open exposure.
- **Hard blackout (T−15m to T+30m):** complete signal freeze, plus the spread circuit breaker.
- **Post-event confirmation (T+30m to T+60m):** re-enable only when both hold:
  - the spread has stayed ≤ 1.5× its time-of-day median for N minutes;
  - an M5 bar has closed beyond the event range (high–low from T to T+30m) by ≥ k × ATR(M5).

### Spread circuit breaker

Trip when the spread exceeds 2.5× the rolling median for that 15-minute bucket and day type (weekday, weekend, holiday). Never use a global median: XM's BTCUSD spread is structurally wider on weekends, in thin Asia hours and around the swap rollover.

### Surprise feed

After each release, publish the standardized surprise, (first-print actual − consensus) ÷ historical σ, so Agents 1 and 2 can align post-event direction.

### Liquidity and venue filters

- **XM rollover and maintenance:** CFD spreads widen around server midnight, and XM can pause crypto for maintenance while the underlying keeps trading. Block entries 15–30 minutes either side, and treat any XM pause as a gap event on resumption.
- **Weekends:** order books are thinner, and weekend macro or geopolitical headlines hit crypto first because it's the only major market open. Apply [no new entries / reduced size] from [Saturday 00:00 to Sunday 21:00 UTC].
- **Sunday-evening reopen:** treat the reopen of FX and equity-index futures (Sunday 17:00–18:00 ET) as an event window. In history before 29 May 2026, the CME reopen also created the "CME gap". After that date CME crypto futures trade 24/7, apart from a weekly weekend maintenance window.
- **Thin holiday and off-hours sessions:** treat Christmas–New Year, US Thanksgiving and Lunar New Year as high risk. The record liquidation cascade began on a Friday evening (10 October 2025) after the US equity close, on a tariff headline. BTC fell from about $122,600 to about $104,800 within hours.
- **Liquidation-cascade detector:** open interest falls by more than [k]σ within minutes, liquidation prints spike and funding flips. While it fires, freeze new entries. Liquidation feeds are sampled (e.g., Binance publishes at most one liquidation order per second per symbol), so treat feed counts as a lower bound.
- **Venue-stress flag:** freeze new entries on any of these:
  - a major exchange halting withdrawals or reporting a hack;
  - a major stablecoin trading more than [x] bps from par;
  - the feed-integrity breaker tripping.
- **Prioritize** the US cash-equity and ETF session, especially its overlap with the London afternoon, and the European morning.

### Clock and calendar integrity

- **DST:** US and UK clocks change on different dates (second Sunday of March and first Sunday of November, versus last Sunday of March and last Sunday of October). The London–New York gap shifts for a few weeks each year.
- **XM server time:** it follows its own DST schedule, and XM D1 bars start at server midnight. MT5 tick and bar times are stamped in server time even when they look like Unix epochs. Convert everything to UTC and verify the offset against a known timestamped event.
- **Block timestamps** are set by miners and can be off by up to about two hours. Use your node's first-seen time as `available_time`.
- **Releases move or vanish.** During the 2025 US shutdown, BLS canceled the October jobs report and issued no October CPI release; September payrolls were rescheduled from 3 October to 20 November. Store actual publication timestamps and data vintages; never rebuild them from a static schedule.

---

## Agent 4 — Regime & Foundation Forecasting

Classifies the market state, forecasts forward volatility cones and decides which sleeve may trade.

- **HMM features:**
  - multi-horizon log returns and realized volatility (24/7 bars);
  - DVOL;
  - perpetual funding z-score and the change in open interest;
  - annualized CME basis;
  - spot-ETF net-flow z-score, lagged to its publication time;
  - stablecoin supply growth;
  - DXY and Nasdaq-100 returns;
  - change in the 10-year real yield.

  Output state probabilities rather than hard labels, with a minimum dwell time so states don't flip-flop.
- **Five regimes:**
  1. trending low-vol;
  2. trending high-vol;
  3. mean-reverting chop;
  4. crisis-deleveraging: forced selling and liquidation cascades, e.g., 12–13 March 2020, 19 May 2021, November 2022, 10–11 October 2025, 5 February 2026;
  5. leverage-euphoria squeeze: a blow-off driven by extreme funding and open interest, prone to violent reversal.
- **Routing:**
  - trend core in trending regimes;
  - mean-reversion sleeve only in chop;
  - both off, or at minimum size, in crisis-deleveraging;
  - trend core at reduced size with tighter trailing stops in leverage-euphoria.
- **GARCH:** fit GJR-GARCH or EGARCH with Student-t errors, Tier-1 event dummies and a weekend dummy. Don't assume the equity leverage effect; BTC has at times shown the inverted pattern, with volatility rising on rallies. Let the estimated sign decide.
- **Foundation models:** Chronos or TimeGPT, zero-shot, for 1-hour, 1-day and 5-day quantile cones. Chronos-2 accepts known-future covariates, so pass event-calendar, session, weekend, funding-time and expiry flags. The original Chronos and Chronos-Bolt don't support covariates natively.
- **Contamination check:** Bitcoin's price history is among the most widely published series, so assume it is in pretraining data. Score the cones on data after each model's release, against a GARCH baseline, before they influence sizing.

---

## Agent 5 — Capital Allocation & Drawdown Shield

Turns approved signals into unleveraged position sizes and protects the equity curve.

- **Sizing:**
  - Risk-budget size = (equity × risk per trade) ÷ (stop distance in $/BTC).
  - Inverse-ATR volatility parity targets a constant ex-ante daily P&L volatility.
  - Fractional Kelly (≤ 0.25×) uses shrunk, out-of-sample edge estimates.
  - Final size = min(risk budget, vol parity, Kelly, 1.0x notional cap, XM margin limit), rounded down to the lot step and clipped to XM's min and max volume.
- **Conviction multiplier:** bounded at 0.25×–1.5× of base risk when Agents 1, 2 and 4 agree. It can never breach the 1.0x cap.
- **Equity-curve breakers:** halve size after 3 consecutive losses or a rolling drawdown past a threshold, then apply a cooling period.
  - Calibrate the threshold from the strategy's own out-of-sample trades (e.g., the 95th percentile of rolling 10-trade drawdowns) instead of a fixed 3%.
  - Count cooling periods in liquidity-weighted sessions rather than clock hours, so a quiet weekend doesn't use them up. For example, use 8-hour Asia/Europe/US blocks, with a weekend block counting [0.5].
- **Stops:** adaptive GARCH/ATR trailing stops, plus a disaster stop always resting at XM.
- **Weekend and gap policy:**
  - Intraday mode is flat before Friday 21:00 UTC and before any announced XM maintenance.
  - Swing mode sizes each position assuming a gap of [k] × ATR straight through the stop, from a liquidation cascade, an XM feed freeze or an XM maintenance pause.
- **Carry:** include XM's swap in every swing trade's expected value.
  - Crypto CFD swaps are often large annualized rates. Depending on XM's settings, they may be charged every calendar night, including weekends, or tripled on one weekday.
  - At startup and daily, read `swap_long`, `swap_short`, `swap_mode` and `swap_rollover3days` from the MT5 symbol info, and log any change as a SymbolSpecEvent.
- **Margin:** track XM's BTCUSD margin rate separately from account leverage. Retail crypto-CFD leverage caps differ by XM entity and jurisdiction (e.g., 2:1 under ESMA rules), and brokers raise crypto margins in volatile markets.

---

## Part 2 — Simulation & Stress-Testing Engine

Build this first. No agent is trusted until it passes here.

### Event-driven engine (zero look-ahead)

- **One priority queue** ordered by `available_time` merges:
  - XM bid/ask ticks or M1 bid/ask bars;
  - composite spot trades and quotes;
  - funding, open-interest and liquidation events;
  - calendar events (scheduled and actual times);
  - news (publication time);
  - macro vintages;
  - COT (Friday release time);
  - ETF flows (publication time);
  - on-chain data (first-seen time);
  - XM symbol-spec changes.
- **Execution timing:** a signal computed on bar t executes no earlier than the first tick of bar t+1. With bars only, resolve stop-versus-target ambiguity conservatively: stop first.
- **Automated leakage tests:** perturb all data after time t and assert that decisions at t don't change; audit every feature's timestamps.

### XM BTCUSD friction model

- **Spread:** a function of the 15-minute time-of-day bucket × day type, realized volatility, event proximity, the swap-rollover window and the holiday flag.
  - Fit it on XM's own tick history. MT5 history depth is limited, so run a tick recorder from day one.
  - Fit weekends and the rollover window separately.
  - Before your XM history covers a period, model XM's spread as the composite spread times a fitted XM markup, and widen the cost-uncertainty bands for that period.
- **Price basis:** model the XM-mid-minus-composite-mid basis, including its widening and the stale-quote episodes during cascades. Stops trigger on XM's feed.
- **Slippage:** half-spread + size-dependent impact + latency drift.
  - Stops in fast markets fill at the worse of the stop and the next tick, with extra slippage inside Tier-1 windows and detected liquidation cascades.
  - After an XM pause, gaps fill at the first tick.
- **Costs:** commission per lot if your XM account type charges one, plus nightly swaps according to XM's swap mode and rollover days. Express all of them in bps of price.
- **Fills:** model partial fills (MT5 IOC/return fill policies), requotes, off-quotes, rejections for market closed, invalid stops or insufficient margin, and stops-level / freeze-level constraints. All of these are more likely during news, cascades and thin liquidity. Calibrate them from XM demo and live logs.

### Validation & stress suite

- **Walk-forward:** rolling windows (e.g., 2-year train, 3-month test, 3-month step) with purging and an embargo.
  - Prefer parameter plateaus, where results hold within ±20% of each value.
  - Report the Deflated Sharpe Ratio and the Probability of Backtest Overfitting (CSCV).
  - Report results separately on each side of the structural breaks: the CME futures launch (December 2017), the US spot-ETF launch (11 January 2024) and CME's move to 24/7 trading (29 May 2026).
- **Regime coverage:** report bull, bear and range years separately, plus long-only and short-only P&L. Bear tests must include 2018, 2022 and the decline since the 6 October 2025 all-time high.
- **Named stress replays:**
  - the 17 December 2017 top and the 2018 bear;
  - 12–13 March 2020 "Black Thursday" (BTC roughly halved in two days);
  - the 19 May 2021 crash;
  - the 10 November 2021 top;
  - the May 2022 Terra/LUNA–UST collapse;
  - June 2022 Celsius and Three Arrows;
  - the 8–21 November 2022 FTX collapse;
  - 10–13 March 2023: SVB failure and USDC depeg;
  - 16 October 2023: false ETF-approval headline;
  - 9–11 January 2024: compromised SEC account, then ETF approval and launch;
  - the April 2024 halving;
  - 5 August 2024: yen-carry unwind;
  - 21 February 2025: Bybit hack;
  - 7 April 2025: tariff shock;
  - 10–11 October 2025: record liquidation cascade;
  - 30 January – 6 February 2026: the Warsh-nomination sell-off and leverage unwind to about $60,000;
  - 29 October 2026: Binance hot-wallet hack and withdrawal halt;
  - the November 2026 slide below $60,000;
  - every Tier-1 release in the sample.

  XM CFD data won't cover the early episodes. Replay those on the composite with the fitted XM cost model and say so in the report.
- **Monte Carlo:** stationary block bootstrap of returns and trade sequences, which keeps volatility clustering.
  - Inject random 10× spread shocks at Tier-1 events and XM rollovers.
  - Inject liquidation-cascade jumps of [3–6] × ATR(H1) within minutes.
  - Inject XM feed freezes and maintenance gaps of 2–5 × ATR.
  - Inject latency spikes, feed outages, swap-rate and margin-rate shocks, the symbol going close-only, and venue-dislocation shocks where XM's price detaches from the composite.
  - Decide on 5th-percentile CAGR, max drawdown and time-to-recovery, not the median.
- **Ablation matrix:**
  - Agent 1 alone, then +2, +3, +4, +5 cumulatively;
  - leave-one-out runs;
  - costs at 1×, 1.5× and 2×.

  Bootstrap confidence intervals on ΔCAGR and Δmax drawdown, and drop any agent whose lift interval includes zero.
- **Paper trading:**
  1. Shadow-trade an XM demo account for [N] weeks. Demo fills are often better than live fills.
  2. Run a minimum-size live phase of [M] weeks.
  3. Compare real fills, spreads, swaps and stop executions with the cost model, and recalibrate before scaling up.

---

## Point-in-time data sources

Every input stores the time it became knowable, not just the time it describes.

| Data | Source options | Timestamp to store |
|---|---|---|
| XM BTCUSD bid/ask ticks | XM MT5 (`copy_ticks_range`), recorded continuously by your own collector | Tick time converted from server time to UTC; receipt time |
| Composite spot trades and quotes | Coinbase, Kraken, Bitstamp, Binance public data (free); Tardis.dev / Kaiko / CoinAPI (paid) | Exchange time and local receipt time, UTC |
| Reference rate | CME CF Bitcoin Reference Rate / Real-Time Index (check license terms) | Publication time |
| CME BTC/MBT volume, OI and basis | CME data via a vendor | Exchange time; OI at its preliminary and final publication times |
| Perpetual funding, OI, liquidations | Exchange APIs (Binance, Bybit, OKX); CoinGlass / Laevitas (paid) | Funding settlement time; snapshot time. Liquidation feeds are sampled |
| Options and implied vol | Deribit (DVOL, options chain); CME options | Snapshot time |
| Spot-ETF flows | Issuer websites (e.g., IBIT, FBTC); aggregators | Publication time, not the trade date |
| Real yields, dollar, inflation | FRED; ALFRED for vintages | Release time of each vintage |
| Equities and cross-asset | Index vendor or broker data | Close time / tick time |
| Calendar, consensus and actuals | Paid vendor or a validated scrape | Actual release time; first-print value |
| CFTC Commitments of Traders (TFF, CME Bitcoin) | CFTC website | Friday 15:30 ET release, not Tuesday |
| Corporate and sovereign treasury activity | SEC EDGAR 8-K and 10-Q; government announcements | EDGAR acceptance time; publication time |
| Stablecoin supply | Issuer transparency pages; on-chain data | Block first-seen time; retrieval time |
| On-chain metrics | Your own Bitcoin node; Glassnode / Coin Metrics | Node first-seen time. Vendor metrics are revised and back-filled, so store each vintage or flag it as look-ahead |
| News and social | GDELT (free), licensed crypto wire, regulator press releases, X | Publication, retrieval, edit and deletion times |
| XM symbol specification | MT5 `symbol_info` (volume limits, stops level, margin, swaps, trade mode) | Poll time; log every change |

---

## Deliverables

1. **Architecture & communication flow:** a Mermaid diagram, plus data contracts as Python dataclasses or pydantic v2 models for every message type, each with `event_time`, `available_time`, `source` and `version`.
2. **Directory & codebase blueprint:**

   ```text
   btc_system/
     configs/          instrument_btcusd_xm.yaml  broker_xm_mt5.yaml  sessions.yaml
                       events.yaml  costs.yaml  risk.yaml
     data_pipelines/   prices/xm_mt5/  prices/spot_composite/  derivatives/  calendar/
                       news/  macro/  flows/  onchain/  vol/  pit_store/
     agents/           alpha_structure/  news_macro/  calendar_guard/
                       regime_forecast/  allocator/
     orchestrator/     bus.py  clock.py  router.py  contracts.py
     risk/             sizing.py  exposure.py  breakers.py  stops.py  feed_integrity.py
     simulation/       engine.py  broker_sim.py  fills.py  cost_models/xm_btcusd.py
     validation/       walk_forward.py  monte_carlo.py  ablation.py
                       stress_replays.py  overfit_stats.py
     execution/        adapters/xm_mt5.py  symbol_spec_sync.py  tick_recorder.py
                       reconciliation.py  kill_switch.py
     monitoring/       metrics.py  drift.py  alerts.py
     tests/            test_lookahead.py  test_contracts.py  test_costs.py  test_exposure.py
                       test_server_time.py  test_symbol_spec.py  test_feed_integrity.py
   ```

3. **Simulation engine & Risk Router skeleton:** runnable Python 3.11+ (asyncio, fully typed). It covers:
   - the clock and the event queue;
   - a broker simulator with the XM BTCUSD cost model and MT5 symbol constraints (volume min, step and max; stops and freeze levels; swap schedule; trade mode);
   - the router, with the exposure cap, event gates, spread breaker, feed-integrity breaker and equity breakers.

   It must run end to end on a small sample dataset, with unit tests that include the look-ahead test and the server-time-to-UTC conversion test.
4. **Config schemas:** YAML for the instrument and XM broker spec, sessions (UTC with DST rules, plus XM server-time rules), event tiers and windows, cost parameters and risk limits.
5. **Validation report template:** walk-forward table, Monte Carlo distributions, ablation matrix, regime and long/short breakdowns, structural-break splits, buy-and-hold comparison.
6. **Phased build plan with exit criteria:**
   1. data, point-in-time store and XM tick recorder;
   2. engine and costs;
   3. Agent 1 baseline;
   4. Agents 3 and 5;
   5. Agent 4;
   6. Agent 2;
   7. XM demo, then minimum-size live.

---

## Output rules

- Work one deliverable at a time, and state every assumption.
- Ask only when a missing parameter blocks the design.
- Flag any component that cannot be backtested without look-ahead. That includes revised on-chain metrics, liquidation heatmaps and any data without a stored publication time.
- Report no performance number the engine has not produced.

---

## Conversion notes (not part of the prompt)

What changed from the XAU/USD version, and what to check on XM before you fill the brackets:

- **24/7 market:** the daily-break and Sunday-gap rules are replaced by XM rollover/maintenance gaps, weekend thinness and the Sunday-evening macro reopen. Volatility is annualized on 365 days. D1 bars are rebuilt on the UTC day.
- **Gold drivers replaced:** central-bank gold, LBMA auctions, GVZ and festival demand give way to ETF flows, funding/OI/basis, liquidations, stablecoins, venue risk, regulation and DVOL.
- **Regimes:** "crisis-liquidation" and "crisis-safe-haven" become "crisis-deleveraging" and "leverage-euphoria squeeze".
- **New gates:** a feed-integrity breaker (XM versus the composite), a venue-stress flag, a liquidation-cascade detector and XM symbol-spec monitoring.
- **Verify in XM MT5 → Symbol Specification for BTCUSD:** contract size, min/max/step volume, digits, stops and freeze levels, margin rate, swap long/short, swap mode, triple-swap day, trading sessions and maintenance windows. Third-party sources broadly agree that 1 lot = 1 BTC and that the minimum is 0.01 lot with 24/7 trading. They disagree on maximum lot size and leverage.
- **Verify XM server time** (commonly GMT+2 / GMT+3 with DST) and whether your XM entity offers crypto CFDs in your jurisdiction.
- **MT5 Python package:** the official `MetaTrader5` package needs a running MT5 terminal on Windows. On Linux you need Wine or a Windows VM.

---

## Reference links (not part of the prompt)

- Fortune: [Price of Bitcoin, 25 November 2026](https://fortune.com/article/price-of-bitcoin-11-25-2026/) · [20 November 2026](https://fortune.com/article/price-of-bitcoin-11-20-2026/)
- Business Standard: [Bitcoin tops $126k, record of 6 October 2025](https://www.business-standard.com/amp/markets/cryptocurrency/historic-high-bitcoin-tops-126k-on-institutional-inflows-debasement-bets-125100700286_1.html)
- Decrypt: [Record ~$19 billion crypto liquidations, 10–11 October 2025](https://decrypt.co/343941/bitcoin-ethereum-dogecoin-down-record-19-billion-crypto-liquidations)
- CoinGecko: [The 10 October crypto crash explained](https://www.coingecko.com/learn/october-10-crypto-crash-explained)
- The Block: [Bitcoin slides below $83K as ETFs bleed, 30 January 2026](https://www.theblock.co/news/markets/2026-01-30-bitcoin-slides-below-83k-as-wall-street-pulls-over-1b-from-crypto-etfs-after-fed-pause-387811)
- CNBC: [Bitcoin gets slashed in half, 6 February 2026](https://www.cnbc.com/2026/02/06/bitcoin-gets-slashed-in-half-whats-behind-the-cryptos-existential-crisis.html)
- Cointelegraph: [Binance $6.3B hack linked to TraderTraitor, October 2026](https://cointelegraph.com/news/live-updates-binance-63b-hack-linked-to-north-koreas-tradertraitor)
- CME Group: [Launch of 24/7 crypto futures and options trading](https://www.cmegroup.com/media-room/press-releases/2026/6/01/cme_group_announceslaunchof247cryptocurrencyfuturesandoptionstra.html)
- The Block: [CME 24/7 crypto trading to start 29 May 2026](https://www.theblock.co/news/markets/2026-02-19-cme-group-launch-24-7-crypto-futures-options-trading-may-29-390491)
- Fortune: October 2025 jobs report canceled (carried over from the XAU/USD prompt)
- BLS: 2025 shutdown impact on CPI (carried over from the XAU/USD prompt)
- Hugging Face: [Chronos-2 model card](https://huggingface.co/amazon/chronos-2)
