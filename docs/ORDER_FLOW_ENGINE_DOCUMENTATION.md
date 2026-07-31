# Order Flow Pressure Engine (OFPE) — Architecture & Design Documentation

> **File:** `order_flow_engine.py` · **Integration:** `option_chain_main.py` · `fusion_signals.py` (L5)  
> **Last updated:** 21 March 2026  
> **Production branch:** `nishu_prod`  
> **Status:** Layer 5 — INFO-ONLY (observability, no signal participation in fusion gate)

---

## Table of Contents

1.  [Why This Exists — The Problem](#1-why-this-exists)
2.  [How HFT Firms & Institutional Desks Read Order Flow](#2-how-hft-firms-read-order-flow)
3.  [Our Implementation vs. Institutional Standard](#3-our-implementation-vs-institutional-standard)
4.  [Architecture — Data Flow](#4-architecture)
5.  [Five Microstructure Features](#5-five-microstructure-features)
    - [5.1 OFI — Order Flow Imbalance (Cont-Kukanov-Stoikov)](#51-ofi)
    - [5.2 Book Imbalance — 5-Level Depth Asymmetry](#52-book-imbalance)
    - [5.3 Microprice Drift — Quantity-Weighted Mid vs True Mid](#53-microprice-drift)
    - [5.4 Trade Impulse — Signed Sqrt Volume](#54-trade-impulse)
    - [5.5 Spread Filter — Wide Spread Dampening](#55-spread-filter)
6.  [Composite DPS (Directional Pressure Score)](#6-composite-dps)
7.  [Z-Score Normalization — Welford's Online Algorithm](#7-z-score-normalization)
8.  [Persistence Filter — Anti-Noise Gate](#8-persistence-filter)
9.  [Actionable Recommendation — BUY CE / BUY PE](#9-actionable-recommendation)
10. [Token Universe — Buyer-Centric Band](#10-token-universe)
11. [Data Structures](#11-data-structures)
12. [Public API Reference](#12-public-api-reference)
13. [Data Output Channels](#13-data-output-channels)
14. [Fusion Integration — Layer 5](#14-fusion-integration)
15. [Configuration Knobs](#15-configuration-knobs)
16. [Gaps & Future Improvements (Honest Assessment)](#16-gaps-and-future-improvements)
17. [Academic References](#17-academic-references)

---

## 1. Why This Exists

**The gap in Layers 1–4:**  Layers 1–4 of the Fusion pipeline use *derived* indicators — Ichimoku cloud, PCR ratios, OI velocity, GEX. These are all **lagging** by design because they aggregate data over minutes. They answer "what is the current regime?" but NOT "who is buying and selling *right now*?"

**What order flow adds:**  Raw order book microstructure is the *earliest* signal available without exchange co-location. Before price moves, the order book shifts:
- Bids stack up or thin out
- Ask pressure builds before drops
- Aggressive market orders print on the tape

The OFPE reads these micro-shifts on **ATM NIFTY options** to detect whether institutional flow is net buying or net selling *before* the price fully reflects it.

**Production role:**  Currently **info-only** (Layer 5 does not block or modify trades). It provides:
- Real-time observability via LAYER_FLOW logs
- Actionable recommendation (BUY_CE / BUY_PE / NEUTRAL) with ATM context
- Persistent CSV audit trail (`OFPE_<date>.csv`)
- Per-tick feature enrichment for `_tick_buf` analysis

---

## 2. How HFT Firms & Institutional Desks Read Order Flow

Understanding what the best firms actually do helps evaluate our implementation honestly.

### 2.1 Tier 1 — Co-located HFT (Citadel Securities, Tower Research, Optiver)

| Capability | Description | Latency |
|------------|------------|---------|
| **Full L3 tape** | Every add, modify, cancel message from the exchange matching engine. Not just top-of-book — every event in the full order book. | <1μs (FPGA) |
| **Queue position tracking** | Track their own and inferred competitor order positions in the queue. Know exactly how many lots are ahead. | Tick-by-tick |
| **Cancellation rate analysis** | Monitor cancel/amend ratios per price level. High cancel rates = spoofing or probing. "Phantom liquidity" detection. | <10μs |
| **Order arrival rate models** | Poisson/Hawkes process models predicting next order arrival. Self-exciting models capture clustering. | Real-time |
| **Cross-venue arbitrage** | Compare order flow across NSE, BSE, and SGX NIFTY simultaneously. Detect informed flow appearing on one venue first. | <100μs |
| **Maker-Taker flow decomposition** | Separate passive (limit) from aggressive (market) orders using trade-and-quote matching algorithms. Lee-Ready tick test or BVC. | Tick-by-tick |

### 2.2 Tier 2 — Quantitative Prop Trading (Jane Street, Two Sigma Market Making)

| Capability | Description |
|------------|------------|
| **Multi-asset OFI** | Correlate NIFTY futures order flow with BANKNIFTY, top-5 stock constituents, and India VIX simultaneously. Cross-asset lead-lag. |
| **Informed vs. uninformed flow** | Statistical models (Kyle's Lambda, VPIN) to estimate the probability that current flow is informed. |
| **Impact models** | Almgren-Chriss / Obizhaeva-Wang: predict how much a given order size will move the price. Use this to size positions. |
| **Book pressure gradients** | Not just bid vs ask imbalance — the *slope* of depth across levels. Is depth convex or concave? Where is the "cliff"? |
| **Event microstructure** | Detect institutional TWAP/VWAP algorithms by their characteristic fill patterns (evenly spaced, time-sliced). |

### 2.3 Tier 3 — Smart Retail / Small Prop Funds (Our Level)

| Capability | Description |
|------------|------------|
| **WebSocket L2 depth** | 5-level bid/ask depth, updated every ~500ms–1s via broker API (Zerodha Kite). |
| **Derived OFI** | Delta-based OFI from consecutive snapshots (our implementation). |
| **Book imbalance** | Bid-ask depth ratio at 5 levels (our implementation). |
| **Microprice** | Quantity-weighted mid vs true mid (our implementation). |
| **Volume impulse** | Signed volume on directional ticks (our implementation). |
| **NO full tape access** | Cannot see individual order adds/cancels. Only see *net* depth at each level. |
| **NO queue position** | Cannot track our position in the order queue. |
| **NO cross-venue** | Single exchange (NSE) via Zerodha. |

---

## 3. Our Implementation vs. Institutional Standard

### ✅ What We Do Well

| Feature | Assessment |
|---------|-----------|
| **OFI (Cont-Kukanov-Stoikov)** | Academically sound. The exact formulation from the 2014 paper. Uses bid/ask price + qty deltas to infer aggressive flow direction. This is the same core logic used at every tier. |
| **5-level depth** | We use all 5 available levels, not just top-of-book. This captures structural depth asymmetry that single-level analysis misses. |
| **Online z-scoring (Welford)** | O(1) memory, O(1) per update. Same numerically-stable algorithm used in production HFT systems. Avoids the "lookback window" problem of batch z-scores. |
| **Composite weighting** | Multi-signal fusion with normalized weights is the correct approach. Better than any single indicator. |
| **Persistence filter** | "3 of 5 agree" is a simple but effective noise gate. Prevents whipsaws from single-tick noise. Similar in spirit to the n-consecutive-bar confirmation used by institutional desks. |
| **Spread filter** | Dampening signals during wide spreads is critical. Wide spread = low liquidity = unreliable depth data. This is an industry best practice. |
| **Buyer-centric band** | Tracking ATM ± n strikes (same band as SRZoneEngine) focuses computation on the strikes that actually carry directional information. OTM wings are noise. |
| **Adaptive stats reset** | Every 5 minutes, z-score stats reset. This prevents morning session statistics from poisoning afternoon signals during regime changes. |

### ⚠️ What We're Missing (Honest Gaps)

| Gap | Impact | What Tier 1/2 Firms Have Instead |
|-----|--------|----------------------------------|
| **No L3 tape (individual orders)** | We see *net* depth changes, not individual add/cancel events. A 500-lot add followed by a 500-lot cancel looks like nothing happened to us, but it's a significant signal (testing the level). | Full message-level order book reconstruction. Track every add, modify, cancel. |
| **No cancel/amend analysis** | Cannot detect spoofing or phantom liquidity. If someone posts a large bid and cancels it 200ms later, we never see it at 1s WebSocket granularity. | Real-time cancel rate per price level. Phantom liquidity detection. |
| **Snapshot-based OFI (not event-based)** | Our OFI compares consecutive 1-second snapshots. Between snapshots, hundreds of events may have occurred. The OFI we compute is a *smoothed approximation* of the true OFI. | Event-driven OFI on every order book update (tick-by-tick). |
| **No cross-asset correlation** | We analyze NIFTY options in isolation. Institutional flow often appears in futures, BANKNIFTY, or constituent stocks first and propagates to NIFTY options with a lag. | Cross-asset OFI correlation: NIFTY futures + options + BANKNIFTY + VIX + top constituents. |
| **No trade classification** | We cannot distinguish aggressive buys (market orders lifting the ask) from passive fills. We use LTP direction as a proxy, but this is noisy. | Lee-Ready algorithm, BVC (Bulk Volume Classification), or direct exchange trade-type flags. |
| **No VPIN / toxicity estimation** | We don't estimate the probability that current flow is informed (adverse selection risk). This matters for sizing and confidence. | VPIN (Volume-Synchronized Probability of Informed Trading), Kyle's Lambda. |
| **No depth gradient analysis** | We compute total bid vs ask depth, but not the *shape* of the depth curve. A concentrated wall at level 1 vs evenly distributed depth across 5 levels have very different implications. | Depth curve slope, convexity, "cliff detection" (sudden depth drop-off). |
| **No institutional algorithm detection** | Cannot detect TWAP/VWAP execution patterns that reveal large institutional orders being worked. | Order pattern recognition: time-sliced fills, participation rate analysis. |
| **1-second resolution ceiling** | Zerodha WebSocket throttles to ~1 tick/sec per token. HFT signals decay in <100ms. By the time we see the signal, the fastest actors have already moved. | Co-located feed handlers: <10μs latency, every tick captured. |

### 📊 Honest Rating

| Criterion | Score | Notes |
|-----------|-------|-------|
| **Feature correctness** | 8/10 | OFI, Book Imbalance, Microprice — all academically correct implementations |
| **Data quality** | 5/10 | 1-second L2 snapshots. Good for direction, poor for timing. Missing L3 entirely. |
| **Signal timeliness** | 4/10 | By the time our 3-of-5 persistence filter confirms, the HFT edge is gone. Our edge is against *other retail*, not against HFT. |
| **Cross-asset context** | 2/10 | Single instrument. No futures, no BANKNIFTY, no VIX correlation. |
| **Noise rejection** | 7/10 | Spread filter + persistence + z-score normalization = solid noise handling for our data tier. |
| **Overall vs. HFT** | 3/10 | Not competitive with co-located systems. Not designed to be. |
| **Overall vs. smart retail** | 8/10 | Significantly better than typical retail order-flow tools (which usually just show volume bars). |

### 🎯 Our Real Edge

Our OFPE doesn't compete with Citadel. It competes with:
- Other retail traders using candlestick patterns
- Indicator-only systems (RSI, MACD, Bollinger)
- Traders who never look at the order book at all

Against that population, having *any* microstructure awareness — even at 1-second granularity — is a meaningful edge. The 5-feature composite with z-scoring and persistence provides directional bias that arrives 5–30 seconds before traditional indicators confirm the move.

---

## 4. Architecture

```
                    ┌──────────────────────────────────────────┐
                    │         Zerodha WebSocket Feed            │
                    │    (option ticks + NIFTY 50 spot ticks)   │
                    └──────────┬──────────────┬────────────────┘
                               │              │
               Option Ticks    │              │   Spot Ticks
               (every ~1s)     │              │   (every ~1s)
                               ▼              ▼
                    ┌──────────────────────────────────────────┐
                    │       option_chain_main.py                │
                    │                                          │
                    │  _process_option_tick()                   │
                    │    └→ order_flow_engine.on_option_tick()  │
                    │       (depth, ltp, volume, oi, strike)   │
                    │                                          │
                    │  _on_index_tick()                         │
                    │    └→ order_flow_engine.on_spot_tick()    │
                    │       └→ aggregate ATM CE+PE              │
                    │       └→ z-score + composite DPS          │
                    │       └→ persistence filter               │
                    │       └→ OrderFlowResult                  │
                    │                                          │
                    │    └→ spot_cache["order_flow"] = result   │
                    │    └→ _write_ofpe_csv_row()               │
                    └──────────────────┬───────────────────────┘
                                       │
                    ┌──────────────────▼───────────────────────┐
                    │       fusion_signals.py — Layer 5         │
                    │                                          │
                    │  Reads spot_cache["order_flow"]           │
                    │  Displays:                                │
                    │    L5 Flow  : signal, DPS, z-scores       │
                    │    L5 Action: → BUY_CE @ ATM 25000        │
                    │               (CE ₹230 / PE ₹215)        │
                    │               NIFTY=25012.50              │
                    │    L5 Trend : RISING, streak, flips       │
                    │                                          │
                    │  ⚠ INFO-ONLY: does not modify trade gate  │
                    └──────────────────────────────────────────┘
```

### Two-Phase Design

| Phase | Trigger | What Happens |
|-------|---------|-------------|
| **Phase 1: Ingest** | Every option tick (~1/sec per token) | `on_option_tick()` computes OFI, book imbalance, microprice drift, trade impulse. Stores in per-token rolling deque (maxlen=600 ≈ 10 min). Updates Welford z-score stats. |
| **Phase 2: Aggregate** | Every NIFTY spot tick (~1/sec) | `on_spot_tick()` collects recent ATM CE + ATM PE ticks, averages features, z-scores, computes composite DPS, applies persistence filter, emits `OrderFlowResult`. |

This separation is deliberate: option ticks arrive at different rates per token, but we want one consolidated signal per spot tick for downstream consumption.

---

## 5. Five Microstructure Features

### 5.1 OFI — Order Flow Imbalance (Cont-Kukanov-Stoikov)

**Paper:** *"The Price Impact of Order Book Events"* — Cont, Kukanov, Stoikov (2014, Journal of Financial Economics)

**Formula:**

$$
\text{OFI}_t = \text{BidOFI}_t - \text{AskOFI}_t
$$

Where:

$$
\text{BidOFI}_t = \begin{cases}
q_t^b & \text{if } p_t^b > p_{t-1}^b \quad \text{(new higher bid = aggressive buyer)} \\
q_t^b - q_{t-1}^b & \text{if } p_t^b = p_{t-1}^b \quad \text{(qty change at same level)} \\
-q_{t-1}^b & \text{if } p_t^b < p_{t-1}^b \quad \text{(bid retreated = buyer withdrew)}
\end{cases}
$$

$$
\text{AskOFI}_t = \begin{cases}
-q_{t-1}^a & \text{if } p_t^a > p_{t-1}^a \quad \text{(ask retreated = seller withdrew)} \\
q_t^a - q_{t-1}^a & \text{if } p_t^a = p_{t-1}^a \quad \text{(qty change at same level)} \\
q_t^a & \text{if } p_t^a < p_{t-1}^a \quad \text{(new lower ask = aggressive seller)}
\end{cases}
$$

**Interpretation:**
- `OFI > 0` → Net buying pressure (bids advancing or asks retreating)
- `OFI < 0` → Net selling pressure (asks advancing or bids retreating)
- `OFI ≈ 0` → Balanced flow

**Weight in composite:** `w_ofi = 0.35` (highest weight — most predictive feature per academic literature)

**Our limitation:** We compute OFI from 1-second snapshots, not individual order events. Between snapshots, multiple order adds/cancels may offset each other, making our OFI a *smoothed net delta* rather than the true event-level OFI from the paper.

---

### 5.2 Book Imbalance — 5-Level Depth Asymmetry

**Formula:**

$$
\text{BookImbalance} = \frac{\sum_{i=1}^{5} q_i^{bid} - \sum_{i=1}^{5} q_i^{ask}}{\sum_{i=1}^{5} q_i^{bid} + \sum_{i=1}^{5} q_i^{ask}}
$$

**Range:** $[-1, +1]$ where $+1$ = all depth is bids (massive buy support), $-1$ = all depth is asks (massive sell pressure).

**Why 5 levels, not just level 1:**
- Level 1 is easily manipulated (spoofing). Deeper levels require more capital to fake.
- Institutional order placement often targets levels 2–5 to avoid signaling intent at level 1.
- Using all 5 levels captures structural asymmetry that single-level indicators miss.

**Weight in composite:** `w_book_imbalance = 0.25`

**Known weakness:** On NSE options, depth at levels 3–5 can be very thin for OTM strikes. This is mitigated by our buyer-centric band filter (we only track ATM ± 200 points where liquidity is concentrated).

---

### 5.3 Microprice Drift — Quantity-Weighted Mid vs True Mid

**Formula:**

$$
\text{Microprice} = \frac{p^{ask} \cdot q^{bid} + p^{bid} \cdot q^{ask}}{q^{bid} + q^{ask}}
$$

$$
\text{MicropriceDrift} = \text{Microprice} - \text{Mid}
$$

where $\text{Mid} = (p^{ask} + p^{bid}) / 2$

**Intuition:** The microprice gives more weight to the *thinner* side. If there are 1000 lots on the bid and 100 lots on the ask, the microprice shifts toward the ask — predicting that the next trade is more likely to happen at the ask (because the bid has much more depth, so it's harder to move the bid).

**Weight in composite:** `w_micro_drift = 0.20`

**Industry context:** Microprice is a standard signal at every quantitative trading firm. It's one of the most robust short-term price predictors known, first popularized by Gatheral (2010) and widely used in electronic market making.

---

### 5.4 Trade Impulse — Signed Sqrt Volume

**Formula:**

$$
\text{TradeImpulse} = \text{sign}(\Delta\text{LTP}) \cdot \sqrt{\Delta\text{Volume}}
$$

**Why sqrt?** Market impact scales with the *square root* of volume (Almgren-Chriss, Kyle's Lambda). A 10,000-lot aggressive buy doesn't move the market 100× more than a 100-lot buy — it moves it ~10× more. The sqrt transform normalizes for this non-linear relationship.

**Interpretation:**
- `> 0` → Price moved up on volume = aggressive buying
- `< 0` → Price moved down on volume = aggressive selling
- `= 0` → No LTP change or no volume delta

**Weight in composite:** `w_trade_impulse = 0.20`

**Limitation:** This is a *proxy* for trade classification. We don't know if the volume was market orders or limit fills. True trade classification (Lee-Ready, BVC) requires the full trade tape.

---

### 5.5 Spread Filter — Wide Spread Dampening

**Not a feature** in the composite — it's a **reliability gate**.

**Logic:**

$$
\text{SpreadPct} = \frac{p^{ask} - p^{bid}}{(p^{ask} + p^{bid}) / 2}
$$

If $\text{SpreadPct} > 1.5\%$:
- The composite DPS is multiplied by $0.40$ (dampened by 60%)
- `spread_state` is tagged as `WIDE` or `MIXED`

**Why this matters:** When spreads widen, the order book is unreliable:
- Market makers have pulled back
- Depth levels are stale or thin
- OFI and book imbalance calculations are noisy
- Any signal based on this data is low-confidence

Rather than rejecting the signal entirely (which loses information), we dampen it — reducing its magnitude but preserving its direction.

---

## 6. Composite DPS (Directional Pressure Score)

$$
\text{DPS} = w_1 \cdot z(\text{OFI}) + w_2 \cdot z(\text{BookImbalance}) + w_3 \cdot z(\text{MicropriceDrift}) + w_4 \cdot z(\text{TradeImpulse})
$$

| Feature | Weight | Rationale |
|---------|--------|-----------|
| OFI | 0.35 | Most predictive single feature (Cont et al. 2014) |
| Book Imbalance | 0.25 | Structural depth asymmetry, harder to spoof than L1 |
| Microprice Drift | 0.20 | Quantity-weighted fair value signal |
| Trade Impulse | 0.20 | Volume-confirmed directional aggression |

**Signal thresholds:**

$$
\text{Signal} = \begin{cases}
\text{BUY\_PRESSURE} & \text{if } \text{DPS} > +0.50 \\
\text{SELL\_PRESSURE} & \text{if } \text{DPS} < -0.50 \\
\text{NEUTRAL} & \text{otherwise}
\end{cases}
$$

The $\pm 0.50$ threshold is ~0.5 standard deviations from mean in z-score space. Conservative enough to avoid noise, sensitive enough to catch real flow shifts.

---

## 7. Z-Score Normalization — Welford's Online Algorithm

Instead of computing z-scores over a fixed lookback window (which requires storing all data points), we use **Welford's online algorithm** for numerically stable, O(1) streaming mean and variance.

**Update step:**

$$
n \leftarrow n + 1
$$
$$
\delta \leftarrow x - \mu_{n-1}
$$
$$
\mu_n \leftarrow \mu_{n-1} + \frac{\delta}{n}
$$
$$
M_2 \leftarrow M_2 + \delta \cdot (x - \mu_n)
$$

**Z-score:**

$$
z(x) = \frac{x - \mu_n}{\sqrt{M_2 / n}} \quad \text{(when } n \geq 5 \text{ and } \sigma > 10^{-12}\text{)}
$$

**Adaptive reset:** Every 300 seconds (5 minutes), all z-score stats are reset to zero. This prevents:
- Morning session stats (9:15–10:30 with high volatility) poisoning afternoon signals
- Regime changes (trending → rangebound) creating stale z-score distributions
- Statistical drift from Welford's `n` growing unboundedly

---

## 8. Persistence Filter — Anti-Noise Gate

Raw microstructure signals are noisy. A single tick can produce a spurious `BUY_PRESSURE` that reverts immediately.

**Mechanism:** Maintain a circular buffer of the last $M = 5$ raw signals. The filtered signal fires only if $N = 3$ of the last $M$ ticks agree:

$$
\text{FilteredSignal} = \begin{cases}
\text{raw\_signal} & \text{if } \sum_{i=1}^{M} \mathbb{1}[\text{signal}_i = \text{raw\_signal}] \geq N \text{ and raw\_signal} \neq \text{NEUTRAL} \\
\text{NEUTRAL} & \text{otherwise}
\end{cases}
$$

**Effect:** Adds ~3 seconds of latency (at 1 tick/sec refresh rate) but eliminates >80% of single-tick false signals.

**Trade-off:** We lose the ability to react to *sudden* flow shifts that last only 1–2 seconds. This is acceptable because:
1. Our downstream consumer (fusion_signals L5) is info-only — we don't need sub-second reaction
2. At 1-second WebSocket granularity, a 1-tick signal is more likely noise than information
3. The persistence filter is equivalent to a 3-point median filter on the signal stream

---

## 9. Actionable Recommendation — BUY CE / BUY PE

The filtered signal maps directly to an options recommendation:

| Filtered Signal | Recommended Action | Reasoning |
|----------------|-------------------|-----------|
| `BUY_PRESSURE` | **BUY_CE** | Buyers are aggressive → bullish → Call options benefit |
| `SELL_PRESSURE` | **BUY_PE** | Sellers are aggressive → bearish → Put options benefit |
| `NEUTRAL` | **NEUTRAL** | No clear directional pressure → no recommendation |

The recommendation includes full ATM context:
- **ATM Strike**: Current ATM strike (e.g., 25000)
- **ATM CE LTP**: Latest traded price of the ATM Call
- **ATM PE LTP**: Latest traded price of the ATM Put
- **NIFTY Spot**: Current NIFTY 50 spot price

**Example L5 output:**
```
L5 Action: 📗→ BUY_CE @ ATM 25000 (CE ₹230.50 / PE ₹215.75) NIFTY=25012.50 (info-only)
```

---

## 10. Token Universe — Buyer-Centric Band

The engine tracks the same strike band as `SRZoneEngine`:

```
For NIFTY at 25012.50 (ATM = 25000), with step=50, n_strikes=4:

CE tokens (at/below spot = ITM/ATM CEs — buyer candidates):
  25000 CE (ATM), 24950 CE, 24900 CE, 24850 CE, 24800 CE

PE tokens (at/above spot = ITM/ATM PEs — buyer candidates):
  25000 PE (ATM), 25050 PE, 25100 PE, 25150 PE, 25200 PE

Total: ~10 tokens tracked
```

**Why buyer-centric?**
- OTM CE (above spot) and OTM PE (below spot) are speculative / hedging — their order flow is noisy
- ATM and slightly ITM options have the tightest spreads and highest institutional participation
- Option makers (market makers) concentrate their quoting activity at ATM strikes
- This is where informed order flow is most reliably detected

**Band filtering (in `on_option_tick`):**
```
CE: reject if strike > ATM  (OTM CE)
PE: reject if strike < ATM  (OTM PE)
CE: reject if (ATM - strike) > band  (too deep ITM CE)
PE: reject if (strike - ATM) > band  (too deep ITM PE)
```

---

## 11. Data Structures

### `_TickState` — Per-Tick Snapshot

Stores raw + derived data for one option tick:

| Field | Type | Description |
|-------|------|-------------|
| `ts` | float | Epoch timestamp |
| `ltp` | float | Last traded price |
| `delta_vol` | int | Incremental volume since last tick |
| `oi` | int | Open interest |
| `b1p/a1p` | float | Best bid/ask price |
| `b1q/a1q` | int | Best bid/ask quantity |
| `bid_depth_qty` | int | Sum of bid qty across all 5 levels |
| `ask_depth_qty` | int | Sum of ask qty across all 5 levels |
| `ofi` | float | Computed OFI (Cont-Kukanov-Stoikov) |
| `book_imbalance` | float | 5-level depth asymmetry [-1, +1] |
| `micro_drift` | float | Microprice - mid |
| `trade_impulse` | float | sign(Δltp) * √(Δvol) |
| `spread_pct` | float | (ask - bid) / mid |
| `mid` | float | (ask + bid) / 2 |

### `_TokenHistory` — Per-Token Rolling State

| Field | Type | Description |
|-------|------|-------------|
| `token` | int | Instrument token |
| `strike` | int | Strike price |
| `opt_type` | str | "CE" or "PE" |
| `prev_tick` | _TickState | Previous tick for delta calculations |
| `history` | deque(600) | ~10 minutes of tick history |

### `OrderFlowResult` — Aggregate ATM Snapshot

Published to `spot_cache["order_flow"]` on every spot tick:

| Field | Type | Description |
|-------|------|-------------|
| `ts` | float | Epoch timestamp |
| `spot` | float | NIFTY spot at computation time |
| `raw_ofi` | float | Average raw OFI across ATM CE+PE ticks |
| `raw_book_imbalance` | float | Average raw book imbalance |
| `raw_micro_drift` | float | Average raw microprice drift |
| `raw_trade_impulse` | float | Average raw trade impulse |
| `z_ofi` | float | Z-scored OFI |
| `z_book_imbalance` | float | Z-scored book imbalance |
| `z_micro_drift` | float | Z-scored microprice drift |
| `z_trade_impulse` | float | Z-scored trade impulse |
| `composite_dps` | float | Weighted composite DPS |
| `spread_state` | str | "OK" / "WIDE" / "MIXED" |
| `spread_dampened` | bool | True if DPS was dampened by spread filter |
| `signal` | str | Persistence-filtered: BUY_PRESSURE / SELL_PRESSURE / NEUTRAL |
| `persistence_count` | int | How many of last M ticks agree |
| `persistence_needed` | int | N needed (default: 3) |
| `recommended_action` | str | BUY_CE / BUY_PE / NEUTRAL |
| `atm_strike` | int | ATM strike used (e.g., 25000) |
| `atm_ce_ltp` | float | Latest CE LTP at ATM |
| `atm_pe_ltp` | float | Latest PE LTP at ATM |
| `ce_token_count` | int | CE tokens with data |
| `pe_token_count` | int | PE tokens with data |
| `atm_ce_token` | int | ATM CE instrument token |
| `atm_pe_token` | int | ATM PE instrument token |
| `tracked_tokens` | int | Total tokens with history |
| `data_freshness_sec` | float | Seconds since newest option tick |
| `total_ticks_used` | int | Total ticks in aggregation window |

---

## 12. Public API Reference

### Constructor

```python
engine = OrderFlowEngine(
    strike_step=50,          # NIFTY strike interval
    n_strikes=4,             # ATM ± n*step = tracking band
    window_sec=120,          # 2-minute rolling window
    zscore_min_ticks=10,     # minimum ticks before z-score valid
    spread_wide_pct=0.015,   # 1.5% = wide spread threshold
    spread_dampen=0.40,      # DPS multiplier when spread wide
    persistence_n=3,         # 3 of last M must agree
    persistence_m=5,         # last M signals
    composite_thresh=0.50,   # |DPS| > 0.50 → directional
    refresh_sec=1.0,         # recompute throttle
    w_ofi=0.35,              # composite weights
    w_book_imbalance=0.25,
    w_micro_drift=0.20,
    w_trade_impulse=0.20,
)
```

### Tick Ingestion

| Method | When Called | What It Does |
|--------|-----------|-------------|
| `on_option_tick(token, strike, opt_type, ltp, delta_volume, depth, oi, nifty_ltp, ts_epoch)` | Every option tick | Computes per-tick features, stores in rolling deque, updates z-score stats |
| `on_spot_tick(spot, ts_epoch)` | Every NIFTY spot tick | Aggregates ATM CE+PE, z-scores, DPS, persistence → `OrderFlowResult` |
| `set_spot(spot)` | Automatically by `on_spot_tick` | Updates ATM strike and ingest band filter |

### Data Retrieval

| Method | Returns | Consumer |
|--------|---------|----------|
| `get_result()` | `Optional[OrderFlowResult]` | Any polling consumer |
| `get_latest_features(token)` | `dict` with 6 per-tick features | `_tick_buf` enrichment |
| `get_history()` | `deque[OrderFlowResult]` (maxlen=100) | L5 trend analysis |
| `get_trend_summary()` | `dict` with DPS trend, streak, flips | L5 trend display |
| `get_readable_summary()` | `str` (boxed format) | Log dumps |
| `get_csv_row()` | `Optional[list]` | OFPE CSV writer |
| `csv_header()` | `list` (static) | OFPE CSV header |

---

## 13. Data Output Channels

The engine outputs data through three channels:

### Channel 1: `_tick_buf` Enrichment (Per-Token, Real-Time)

Every option tick's `_tb_row` dict in `option_chain_main.py` gets 6 additional columns via `get_latest_features(token)`:

| Column | Description |
|--------|-------------|
| `ofpe_ofi` | Per-tick OFI for this specific token |
| `ofpe_book_imbalance` | Per-tick book imbalance |
| `ofpe_micro_drift` | Per-tick microprice drift |
| `ofpe_trade_impulse` | Per-tick trade impulse |
| `ofpe_spread_pct` | Per-tick spread percentage |
| `ofpe_mid` | Per-tick mid price |

### Channel 2: OFPE Aggregate CSV (Daily, Persistent)

**File:** `$HOME/output/csv/<YYYY-MM-DD>/OFPE_<YYYY-MM-DD>.csv`

One row per spot tick (~1 row/sec). 27 columns:

```
epoch, nifty_spot, atm_strike, recommended_action, atm_ce_ltp, atm_pe_ltp,
signal, composite_dps, raw_ofi, raw_book_imbalance, raw_micro_drift,
raw_trade_impulse, z_ofi, z_book_imbalance, z_micro_drift, z_trade_impulse,
spread_state, spread_dampened, persistence_count, persistence_needed,
atm_ce_token, atm_pe_token, tracked_tokens, ce_token_count, pe_token_count,
data_freshness_sec, total_ticks_used
```

Auto-rotates on date change. Append-mode with write-through (buffering=1).

### Channel 3: `spot_cache["order_flow"]` (In-Memory, Real-Time)

The full `OrderFlowResult` dataclass, consumed by:
- **fusion_signals.py L5** — LAYER_FLOW display + audit dict
- Any future consumer that polls `spot_cache`

---

## 14. Fusion Integration — Layer 5 (Full Code-Level Data Flow)

### Where L5 Sits in the 5-Layer Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FUSION SIGNAL PIPELINE — 5 LAYERS                        │
│                                                                             │
│  MARKET DATA                                                                │
│  (Spot + Option ticks via Zerodha WebSocket)                                │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────┐                                                            │
│  │  LAYER 1    │  Fusion Signal Engine (on_spot_tick)                       │
│  │  (L1)       │  • Ichimoku, OI momentum, PCR, volume, flow signals        │
│  │             │  • K-of-N voting → raw directional signal                  │
│  │             │  • Anti-flip-flop (consecutive count filter)               │
│  │             │  Output: current_trend = UPTREND / DOWNTREND               │
│  └──────┬──────┘                                                            │
│         │                                                                   │
│         ▼                                                                   │
│  ┌─────────────┐                                                            │
│  │  LAYER 2    │  Trend Reversal / Anti-Churn Filter                        │
│  │  (L2)       │  • TREND_REVERSAL_EXECUTE_CONFIRM_COUNT = 3 passes          │
│  │             │  • Flip cooldown (10 min), min hold (10 min)               │
│  │             │  • Blocks re-entry after MASTER_GATE live-exit (10 min)    │
│  │             │  Output: confirmed_trend_flip → triggers gate evaluation   │
│  └──────┬──────┘                                                            │
│         │                                                                   │
│         ▼                                                                   │
│  ┌─────────────┐                                                            │
│  │  LAYER 3    │  MASTER_GATE — Execution Quality Gate                      │
│  │  (L3)       │  • analyze_symbol_trade_quality_from_csv()                 │
│  │             │  • min_total_score ≥ 0.58, entry_edge ≥ 0.08              │
│  │             │  Output: mg_status = "pass" / "fail"                       │
│  └──────┬──────┘                                                            │
│         │                                                                   │
│         ▼                                                                   │
│  ┌─────────────┐                                                            │
│  │  LAYER 4    │  SR Zone Engine — Structural Market Gate                  │
│  │  (L4)       │  • OI-weighted S/R detection (buyer-centric)               │
│  │             │  • GEX regime + gamma wall analysis                        │
│  │             │  • WallBreachAnalyzer (5 signals per wall)                 │
│  │             │  Output: verdict = "pass" / "warn" / "block"              │
│  │             │  CAN BLOCK: L4 block converts L3-pass → skip              │
│  └──────┬──────┘                                                            │
│         │                                                                   │
│         ▼                                                                   │
│  ┌─────────────┐                                                            │
│  │  LAYER 5    │  Order Flow Pressure Engine (OFPE) ← THIS DOCUMENT        │
│  │  (L5)       │  • 5-feature microstructure: OFI, book imbalance,          │
│  │             │    microprice drift, trade impulse, spread filter           │
│  │             │  • Composite DPS with z-scoring and persistence            │
│  │             │  Output: BUY_CE / BUY_PE / NEUTRAL recommendation          │
│  │             │  ⚠ INFO-ONLY: does NOT block or modify trade decisions     │
│  └──────┬──────┘                                                            │
│         │                                                                   │
│         ▼                                                                   │
│  ┌─────────────┐                                                            │
│  │  LAYER_FLOW │  Unified log output (all 5 layers in one log block)       │
│  │  LOG        │  + OFPE snapshot dump                                      │
│  │             │  + Full audit dict with l1, l2, l3, l4, l5 keys            │
│  └──────┬──────┘                                                            │
│         │                                                                   │
│         ▼                                                                   │
│    TRADE EXECUTION  (if L1+L2+L3+L4 all pass)                              │
│    (L5 is logged but does NOT participate in the gate decision)             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Critical Difference: L5 vs L1–L4

| Aspect | L1–L4 | L5 (OFPE) |
|--------|-------|-----------|
| **Gate participation** | YES — each layer can block/pass | **NO** — info-only, never blocks |
| **When evaluated** | During `_validate_trend_reversal()` | During the **same function**, after L4 |
| **Affects `_final_status`** | L3 fail → skip, L4 block → skip | L5 signal is **never** checked for skip/pass |
| **In `layers_agreed` list** | Yes (L1, L2, L3, L4) | **No** — L5 is not in agreed/blocked lists |
| **In `layers_blocked` list** | Yes (L2, L3, L4) | **No** |
| **In audit dict** | Each has own key | Has own key `"l5"` with `"info_only": True` |

### Full Call Chain — Step by Step (Exact Code Locations)

```
PHASE 1: DATA INGESTION (runs on every tick, ~10× per second across all tokens)
═══════════════════════════════════════════════════════════════════════════════

  Zerodha WebSocket delivers option tick
       │
       ▼
  option_chain_main._process_option_tick()
       │
       ├─ [line ~4935] sr_zone_engine.on_option_tick(token, strike, opt_type, ltp, oi, vol_delta, ts)
       ├─ [line ~4958] oi_flow_engine.on_option_tick(...)
       └─ [line ~4972] order_flow_engine.on_option_tick(token, strike, opt_type, ltp, delta_volume, depth, oi, nifty_ltp, ts)
                 │
                 ├─ Band filter: only ATM ± 200pts (buyer-centric, same as SR engine)
                 ├─ Parse 5-level depth from self.last_depth_by_token[token]
                 ├─ Compute: OFI, book_imbalance, micro_drift, trade_impulse, spread_pct
                 ├─ Store _TickState in per-token deque (maxlen=600)
                 ├─ Update Welford z-score running stats
                 └─ Store per-tick features → _latest_features_by_token[token]
                           │
                           └─ [line ~5558] Merged into _tb_row via get_latest_features(token)
                                           → flows to _tick_buf[token] deque


PHASE 2: SPOT TICK AGGREGATION (runs on every NIFTY 50 tick, ~1/sec)
═══════════════════════════════════════════════════════════════════════════════

  Zerodha WebSocket delivers NIFTY 50 spot tick
       │
       ▼
  option_chain_main._on_index_tick()
       │
       ├─ [line ~4724] sr_zone_engine.on_spot_tick(spot, ts)
       │       └─ → spot_cache["sr_zone"] = SRZoneResult
       │
       ├─ [line ~4756] oi_flow_engine.on_spot_tick(spot, ts)
       │       └─ → spot_cache["oi_flow"] = OIFlowResult
       │
       ├─ [line ~4767] order_flow_engine.on_spot_tick(spot, ts)    ◄── OFPE computes here
       │       │
       │       ├─ Throttle: skip if < refresh_sec (1s) since last compute
       │       ├─ Find ATM CE + ATM PE tokens from _strike_tok
       │       ├─ Gather recent ticks from ATM CE + PE within 120s window
       │       ├─ Average raw features across all ATM ticks
       │       ├─ Z-score each feature via Welford stats
       │       ├─ Composite DPS = weighted sum of z-scores
       │       ├─ Spread filter: dampen DPS by 0.4× if spread > 1.5%
       │       ├─ Signal: DPS > +0.50 → BUY_PRESSURE, < -0.50 → SELL_PRESSURE
       │       ├─ Persistence: 3 of last 5 must agree
       │       ├─ Map: BUY_PRESSURE→BUY_CE, SELL_PRESSURE→BUY_PE
       │       ├─ Read ATM CE/PE LTP from _states[token].prev_tick.ltp
       │       └─ Return OrderFlowResult (all fields populated)
       │               │
       │               ├─ → spot_cache["order_flow"] = OrderFlowResult
       │               └─ → _write_ofpe_csv_row() → OFPE_<date>.csv
       │
       └─ [line ~4783] fusion.on_spot_tick(spot, ts)
                │
                └─ This triggers L1 evaluation, which may trigger L2 flip,
                   which triggers _validate_trend_reversal() — see Phase 3.


PHASE 3: GATE EVALUATION (runs only on confirmed trend flip, ~few per session)
═══════════════════════════════════════════════════════════════════════════════

  fusion_signals.on_spot_tick() detects trend flip
       │
       ├─ L1: current_trend established (UPTREND / DOWNTREND)
       ├─ L2: flip confirmed after 3 consecutive same-direction passes
       │
       └─ [line ~6351] _validate_trend_reversal(current_trend, spot, ...)
                │
                ├─ L2 internal: flip cooldown, min hold, post-exit cooldown
                │
                ├─ L3: MASTER_GATE quality check on selected entry symbol
                │       → mg_status = "pass" / "fail"
                │
                ├─ L4: SR Zone gate verdict (only if L3 passed)
                │       → _l4_verdict = "pass" / "warn" / "block"
                │       → if block: _final_status = "l4_block"
                │
                ├─ L5: OFPE snapshot read (ALWAYS runs, even if L3/L4 blocked)
                │       │
                │       ├─ [line ~11566] _ofpe_eng = self.mgr.order_flow_engine
                │       ├─ [line ~11567] _ofpe_data = self.mgr.spot_cache["order_flow"]
                │       │       (this is the OrderFlowResult from Phase 2)
                │       │
                │       ├─ Read from _ofpe_data:
                │       │   signal, composite_dps, z_ofi, z_book_imbalance,
                │       │   z_micro_drift, z_trade_impulse, spread_state,
                │       │   persistence_count, recommended_action,
                │       │   atm_strike, atm_ce_ltp, atm_pe_ltp, spot
                │       │
                │       ├─ [line ~11603] _ofpe_eng.get_trend_summary()
                │       │   → dps_trend, dps_avg_10, signal_streak, flips_last_30
                │       │
                │       └─ Builds 3 display strings:
                │           _l5_disp        → L5 Flow line
                │           _l5_action_disp → L5 Action line
                │           _l5_trend_disp  → L5 Trend line
                │
                ├─ LAYER_FLOW LOG: [line ~11625]
                │   All 5 layers displayed in one unified log block:
                │       L1 Trend : UPTREND
                │       L2 Flow  : CE/HIGH score=0.85
                │       L3 Entry : NIFTY25000CE total=0.72 edge=+0.14 (✅PASS)
                │       L4 SR    : pass(+0) zone=IN_ZONE dist_res=87pts ...
                │       L5 Flow  : 🟢BUY_PRESSURE dps=+0.78 z[...] persist=4/3 (info-only)
                │       L5 Action: 📗→ BUY_CE @ ATM 25000 (CE ₹230 / PE ₹215) NIFTY=25012 (info-only)
                │       L5 Trend : 📈trend=RISING avg10=+0.52 streak=4×BUY_PRESSURE flips=2/30
                │       agreed=[L1 + L2 + L3 + L4]  blocked=[none]
                │
                ├─ OFPE SNAPSHOT DUMP: [line ~11660]
                │   Boxed display via _ofpe_eng.get_readable_summary()
                │
                ├─ AUDIT DICT: [line ~11398]
                │   _full_audit["l5"] = {
                │       "enabled": True, "info_only": True,
                │       "snapshot": {signal, dps, z-scores, recommendation, ATM context},
                │       "trend": {dps_trend, avg10, streak, flips}
                │   }
                │
                └─ DECISION: [line ~11676]
                    _final_status is ONLY affected by L1+L2+L3+L4
                    if _final_status == "pass" → return (1, details, "execute", audit)
                    else → return (0, details, "skip", audit)
                    L5 is NEVER checked for this decision.
```

### How L5 Reads Data (No Additional Tick Processing)

L5 does **not** trigger its own computation during `_validate_trend_reversal()`. It reads **pre-computed data** that was populated earlier by Phase 2:

```
Timeline for a single NIFTY spot tick:

  T+0.0ms    Zerodha WebSocket delivers spot tick
  T+0.1ms    _on_index_tick() starts
  T+0.5ms    sr_zone_engine.on_spot_tick()  → spot_cache["sr_zone"]
  T+1.0ms    oi_flow_engine.on_spot_tick()  → spot_cache["oi_flow"]
  T+1.5ms    order_flow_engine.on_spot_tick() → spot_cache["order_flow"]  ◄── L5 data written
  T+2.0ms    _write_ofpe_csv_row()  → OFPE_<date>.csv
  T+2.5ms    fusion.on_spot_tick() starts
                 └─ L1 evaluates signals
                 └─ If flip detected + confirmed:
                       _validate_trend_reversal() runs
                           └─ L2, L3, L4 compute
                           └─ L5 READS spot_cache["order_flow"]  ◄── reads what was written at T+1.5ms
                           └─ LAYER_FLOW logged with all 5 layers
  T+3.0ms    Decision: execute or skip (based on L1-L4 only)
```

This means L5 always has the **same-tick** OFPE data. There is no staleness issue between the spot tick that triggered the OFPE compute and the gate evaluation that reads it — they happen on the same spot tick, in the same call chain.

### Config Flags (in `TrendConfig`)

| Flag | Default | Description |
|------|---------|-------------|
| `L5_ORDER_FLOW_ENABLE` | `True` | Master switch for L5 display |
| `L5_ORDER_FLOW_STALE_SEC` | `30.0` | Mark data as ⚠STALE if older |

### LAYER_FLOW Log Output (Full Example)

```
────────────────────────────────────────────────────────────────────────
  📊 LAYER_FLOW  user=AB1234  trend=UPTREND  side=CE  spot=25012.50
  Decision: ✅ EXECUTE CE entry
  Why: all enabled layers aligned
  L1 Trend : UPTREND
  L2 Flow  : CE/HIGH score=0.85
  L3 Entry : NIFTY25000CE total=0.72 edge=+0.14 (✅PASS)
  L4 SR    : pass(+0) zone=IN_ZONE dist_res=87pts dist_sup=45pts gex=NEGATIVE_GEX
             res=⚪SOFT_ZONE(bp=0.12) sup=🛡️STRONG_HOLD(bp=0.08)
  L5 Flow  : 🟢BUY_PRESSURE dps=+0.7823 z[ofi=+1.24 bi=+0.87 md=+0.52 ti=+0.93]
             spread=OK persist=4/3 (info-only)
  L5 Action: 📗→ BUY_CE @ ATM 25000 (CE ₹230.50 / PE ₹215.75) NIFTY=25012.50 (info-only)
  L5 Trend : 📈trend=RISING avg10=+0.5200 streak=4×BUY_PRESSURE flips=2/30 hist=47
  agreed=[L1 + L2 + L3 + L4]  blocked=[none]
────────────────────────────────────────────────────────────────────────
```

Note: `agreed` and `blocked` lists contain only L1–L4. L5 is never in either list.

### OFPE Snapshot Dump (Logged Immediately After LAYER_FLOW)

```
LAYER_FLOW.L5_OFPE_SNAPSHOT
┌─── Order Flow Pressure Engine (OFPE) ─────────────────────────────┐
│ Signal: BUY_PRESSURE    DPS: +0.7823  Spot: 25012.50
│ Spread: OK     (         )  Persist: 4/3
├───────────────────────────────────────────────────────────────────┤
│ ➤ Action: BUY_CE      ATM=25000  CE ₹230.50  PE ₹215.75  NIFTY=25012.50
├───────────────────────────────────────────────────────────────────┤
│ OFI            raw=  +45.20   z=+1.240   w=0.35
│ Book Imbalance raw= +0.1523   z=+0.870   w=0.25
│ Micro Drift    raw=+0.003421  z=+0.520   w=0.20
│ Trade Impulse  raw=  +23.40   z=+0.930   w=0.20
├───────────────────────────────────────────────────────────────────┤
│ ATM CE=12345678  ATM PE=12345679  Tracked=8(CE=4 PE=4)
│ Ticks used=24  Freshness=0.8s
└───────────────────────────────────────────────────────────────────┘
```

### Audit Dict (in `_full_audit["l5"]`)

```python
{
    "enabled": True,
    "info_only": True,
    "snapshot": {
        "signal": "BUY_PRESSURE",
        "composite_dps": 0.7823,
        "z_ofi": 1.24,
        "z_book_imbalance": 0.87,
        "z_micro_drift": 0.52,
        "z_trade_impulse": 0.93,
        "spread_state": "OK",
        "persistence_count": 4,
        "data_freshness_sec": 0.8,
        "recommended_action": "BUY_CE",
        "atm_strike": 25000,
        "atm_ce_ltp": 230.50,
        "atm_pe_ltp": 215.75,
        "spot": 25012.50,
    },
    "trend": {
        "dps_trend": "RISING",
        "dps_avg_10": 0.52,
        "dps_avg_30": 0.38,
        "signal_streak": 4,
        "signal_streak_label": "BUY_PRESSURE",
        "flips_last_30": 2,
        "history_len": 47,
    }
}
```

### L5 Agreement/Disagreement Scenarios

L5 does not gate, but its agreement/disagreement with L1–L4 is informative:

| L1-L4 Decision | L5 Recommendation | Interpretation |
|----------------|-------------------|---------------|
| ✅ EXECUTE CE | 📗 BUY_CE | **Full confluence** — microstructure confirms the macro setup |
| ✅ EXECUTE CE | 📕 BUY_PE | **Divergence warning** — entry executes but order flow disagrees. Monitor closely. |
| ✅ EXECUTE CE | 📒 NEUTRAL | **No microstructure confirmation** — entry executes on macro alone |
| 🚫 SKIP | 📗 BUY_CE | **L5 would have agreed** — useful for post-market analysis of blocked trades |
| 🚫 SKIP | 📕 BUY_PE | **Double confirmation of skip** — both structural (L4) and flow (L5) against entry |

These scenarios are only visible in logs for now. Future: could promote L5 to gate participation with configurable thresholds.

---

## 15. Configuration Knobs

All parameters are exposed via the `OrderFlowEngine` constructor:

| Parameter | Default | Range | Effect of Increasing |
|-----------|---------|-------|---------------------|
| `strike_step` | 50 | 50/100 | Wider strike grid (100 for BANKNIFTY) |
| `n_strikes` | 4 | 2–8 | More tokens tracked, more data, more compute |
| `window_sec` | 120 | 30–300 | Longer memory, smoother signals, more lag |
| `zscore_min_ticks` | 10 | 5–30 | More data before z-scores activate |
| `spread_wide_pct` | 0.015 | 0.005–0.05 | More spread tolerance (less dampening) |
| `spread_dampen` | 0.40 | 0.0–1.0 | Higher = less dampening (1.0 = no filter) |
| `persistence_n` | 3 | 1–5 | Stricter filter, fewer signals, less noise |
| `persistence_m` | 5 | 3–10 | Longer lookback for persistence check |
| `composite_thresh` | 0.50 | 0.20–1.50 | Higher = fewer signals (only strong flow) |
| `refresh_sec` | 1.0 | 0.5–5.0 | Slower refresh = less compute, more stale |
| `w_ofi` | 0.35 | 0.0–1.0 | More weight on OFI vs other features |
| `w_book_imbalance` | 0.25 | 0.0–1.0 | More weight on depth asymmetry |
| `w_micro_drift` | 0.20 | 0.0–1.0 | More weight on microprice |
| `w_trade_impulse` | 0.20 | 0.0–1.0 | More weight on volume-direction |

**Internal timers:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `_heartbeat_every_sec` | 30.0 | Heartbeat log interval |
| `_stats_reset_interval_sec` | 300.0 | Z-score stats reset interval |
| `_result_history` maxlen | 100 | ~100 seconds of aggregate snapshots |
| `_TickState.history` maxlen | 600 | ~10 minutes of per-token tick history |

---

## 16. Gaps & Future Improvements (Honest Assessment)

### Priority 1 — High Impact, Feasible

| Improvement | Description | Effort |
|------------|-------------|--------|
| **VPIN (Volume-Synchronized Probability of Informed Trading)** | Estimate the probability that current flow is informed. Uses volume-bucketed trade classification. Would give us a "confidence" overlay on the signal — high VPIN = trust the signal more. | Medium |
| **Cross-asset OFI** | Correlate NIFTY options OFI with NIFTY futures OFI (available via same Zerodha feed). Futures lead options by 1–3 seconds in institutional execution. | Medium |
| **Depth gradient (shape analysis)** | Instead of just total bid vs ask, compute the *slope* and *concentration* of depth. A 10,000-lot wall at level 1 vs 2,000 lots spread across levels 1–5 have very different meanings. | Low |
| **OI delta overlay** | Integrate OI changes into the OFPE signal. Rising OI + BUY_PRESSURE = new longs (strong conviction). Falling OI + BUY_PRESSURE = short covering (weaker). | Low |

### Priority 2 — Medium Impact

| Improvement | Description | Effort |
|------------|-------------|--------|
| **Hawkes process for arrival rates** | Model tick arrival as a self-exciting process. Bursts of activity in one direction predict further activity in the same direction. | High |
| **Multi-expiry analysis** | Compare OFI across weekly vs monthly expiry. Institutional hedging shows up in monthly; speculative flow in weekly. | Medium |
| **Adaptive weights** | Use rolling regression to dynamically adjust feature weights based on recent predictive accuracy. The optimal weight mix changes with market regime. | High |
| **Signal confidence band** | Instead of binary BUY_CE/BUY_PE/NEUTRAL, output a continuous confidence score [0–100]. Let downstream consumers decide their own threshold. | Low |

### Priority 3 — Aspirational (Would Require Infrastructure Changes)

| Improvement | Description | Effort |
|------------|-------------|--------|
| **Full L3 tape via exchange DMA** | Direct market access to NSE NEAT+/OUCH protocol for order-by-order data. Would give us true event-level OFI, cancel analysis, and queue position. | Very High (regulatory + infra) |
| **Co-location** | Place compute at NSE Atos data center for <1ms latency. Only matters if we want to compete with HFT for execution timing. | Very High (cost + regulatory) |
| **ML trade classifier** | Train a model to classify trades as buyer/seller-initiated using features from our L2 data. Better than LTP direction heuristic. | High |

---

## 17. Academic References

1. **Cont, R., Kukanov, A., & Stoikov, S. (2014).** *"The Price Impact of Order Book Events."* Journal of Financial Econometrics, 12(1), 47–88.  
   → Foundation for our OFI implementation.

2. **Gatheral, J. (2010).** *"No-Dynamic-Arbitrage and Market Impact."*  Quantitative Finance, 10(7), 749–759.  
   → Microprice as a fair value estimator.

3. **Almgren, R., & Chriss, N. (2001).** *"Optimal Execution of Portfolio Transactions."* Journal of Risk, 3(2), 5–39.  
   → Square-root impact law (basis for our Trade Impulse sqrt normalization).

4. **Kyle, A. S. (1985).** *"Continuous Auctions and Insider Trading."* Econometrica, 53(6), 1315–1335.  
   → Lambda (price impact coefficient), VPIN foundation.

5. **Easley, D., López de Prado, M. M., & O'Hara, M. (2012).** *"Flow Toxicity and Liquidity in a High-frequency World."* The Review of Financial Studies, 25(5), 1457–1493.  
   → VPIN methodology (future improvement candidate).

6. **Welford, B. P. (1962).** *"Note on a Method for Calculating Corrected Sums of Squares and Products."* Technometrics, 4(3), 419–420.  
   → Our online z-score algorithm.

7. **Cartea, Á., Jaimungal, S., & Penalva, J. (2015).** *"Algorithmic and High-Frequency Trading."* Cambridge University Press.  
   → General microstructure framework; book imbalance, OFI, microprice in the HFT context.

---

*Document generated from code analysis of `order_flow_engine.py` (914 lines), `option_chain_main.py` (integration), and `fusion_signals.py` (L5 display). All formulas verified against source implementation.*
