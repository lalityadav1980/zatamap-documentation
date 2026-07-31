# SR Zone Engine — Complete Architecture & Decision Documentation

> **Purpose:** End-to-end reference for understanding how the 5-layer Fusion Signal pipeline works, how each layer contributes to trade decisions, and how all layers integrate. SR Zone Engine operates as Layer 4; Order Flow Pressure Engine as Layer 5.
>
> **Files covered:** `sr_zone_engine.py` · `order_flow_engine.py` · `fusion_signals.py` (L4 + L5 sections)
>
> **Date:** March 2026

---

## Table of Contents

1. [High-Level Architecture Overview](#1-high-level-architecture-overview)
2. [Data Flow — Tick to Trade Decision](#2-data-flow--tick-to-trade-decision)
3. [Layer 1 — OI Data Ingestion Engine](#3-layer-1--oi-data-ingestion-engine)
4. [Layer 2 — compute_zone (9-Step Analysis Pipeline)](#4-layer-2--compute_zone-9-step-analysis-pipeline)
5. [Layer 3 — WallBreachAnalyzer (5-Signal Per-Wall Assessment)](#5-layer-3--wallbreachanalyzer-5-signal-per-wall-assessment)
6. [Layer 4 — Gate Verdict (S9 Trade Entry Filter)](#6-layer-4--gate-verdict-s9-trade-entry-filter)
7. [Fusion Integration — How All 5 Layers Feed the Trade Engine](#7-fusion-integration--how-all-5-layers-feed-the-trade-engine)
8. [Layer 5 — Order Flow Pressure Engine (OFPE)](#8-layer-5--order-flow-pressure-engine-ofpe)
9. [Open-Trade Monitoring — Live Position Protection](#9-open-trade-monitoring--live-position-protection)
10. [Logging & Debugging Guide](#10-logging--debugging-guide)
11. [Configuration Reference (TrendConfig)](#11-configuration-reference-trendconfig)
12. [Key Data Structures](#12-key-data-structures)
13. [Common Decision Scenarios (Worked Examples)](#13-common-decision-scenarios-worked-examples)
14. [Final Trade Decisions — Open, Close, Hold](#14-final-trade-decisions--open-close-hold)
15. [L4 Tight-Range Block & Conviction Contradiction Block — Complete Reference](#15-l4-tight-range-block--conviction-contradiction-block--complete-reference)
16. [19-Mar-2026 Fusion Events Deep Analysis — 5 Recommendation Fixes](#16-19-mar-2026-fusion-events-deep-analysis--5-recommendation-fixes)

---

## 1. High-Level Architecture Overview

The SR Zone Engine is a **buyer-centric, OI-weighted support/resistance system** with real-time GEX (Gamma Exposure) analysis. It operates as **Layer 4** inside the broader Fusion Signal pipeline, which has 5 sequential layers:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FUSION SIGNAL PIPELINE — 5 LAYERS                        │
│                                                                             │
│  MARKET DATA                                                                │
│  (Spot + Option ticks via Zerodha WebSocket)                                │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────┐                                                            │
│  │  LAYER 1    │  Fusion Signal Engine                                      │
│  │  (L1)       │  • Ichimoku, OI momentum, PCR, volume, flow signals        │
│  │             │  • K-of-N voting → raw directional signal                  │
│  │             │  • Anti-flip-flop (consecutive count filter)               │
│  └──────┬──────┘                                                            │
│         │ signal = CE_BUY / PE_BUY / FLAT                                  │
│         ▼                                                                   │
│  ┌─────────────┐                                                            │
│  │  LAYER 2    │  Trend Reversal / Anti-Churn Filter                        │
│  │  (L2)       │  • TREND_REVERSAL_EXECUTE_CONFIRM_COUNT = 3 passes          │
│  │             │  • Flip cooldown (10 min), min hold (10 min)               │
│  │             │  • Blocks re-entry after MASTER_GATE live-exit (10 min)    │
│  └──────┬──────┘                                                            │
│         │ confirmed_trend_flip                                              │
│         ▼                                                                   │
│  ┌─────────────┐                                                            │
│  │  LAYER 3    │  MASTER_GATE — Execution Quality Gate                      │
│  │  (L3)       │  • analyze_symbol_trade_quality_from_csv()                 │
│  │             │  • min_total_score ≥ 0.58, entry_edge ≥ 0.08              │
│  │             │  • OPP dominance check (≤ 2 opposite signals)             │
│  └──────┬──────┘                                                            │
│         │ mg_status = "pass" / "fail"                                      │
│         ▼                                                                   │
│  ┌─────────────┐                                                            │
│  │  LAYER 4    │  SR Zone Engine — Structural Market Gate                  │
│  │  (L4)       │  ← THIS DOCUMENT ─────────────────────────                │
│  │             │  • OI-weighted S/R detection (buyer-centric)               │
│  │             │  • GEX regime + gamma wall analysis                        │
│  │             │  • WallBreachAnalyzer (5 signals per wall)                 │
│  │             │  • Zone-aware entry filter: pass / warn / block            │
│  └──────┬──────┘                                                            │
│         │ verdict = "pass" / "warn" / "block"                              │
│         ▼                                                                   │
│  ┌─────────────┐                                                            │
│  │  LAYER 5    │  Order Flow Pressure Engine (OFPE) — INFO-ONLY            │
│  │  (L5)       │  • 5-feature microstructure (OFI, book imbalance,          │
│  │             │    microprice drift, trade impulse, spread filter)          │
│  │             │  • Composite DPS with z-scoring + persistence              │
│  │             │  • Recommendation: BUY_CE / BUY_PE / NEUTRAL              │
│  │             │  • ⚠ Does NOT participate in gate decisions                │
│  └──────┬──────┘                                                            │
│         │ (info-only — logged in LAYER_FLOW, never blocks/passes)          │
│         ▼                                                                   │
│    TRADE EXECUTION                                                          │
│    (CE BUY or PE BUY placed — gated by L1+L2+L3+L4 only)                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

> **Key principle:** Layer 4 only runs when **L1 + L2 + L3 all pass**. It never fires on its own. Its job is to prevent entries that are structurally wrong — buying into a ceiling, shorting into a reinforced floor, or entering when a gamma wall is actively fighting the trade direction.
>
> **Layer 5** runs unconditionally on every gate evaluation (even when L3 or L4 blocked the trade). It is **info-only** — it displays an actionable recommendation (BUY_CE / BUY_PE) with ATM context and NIFTY spot, but does not modify the trade decision. See [Section 8](#8-layer-5--order-flow-pressure-engine-ofpe) for details.

---

## 2. Data Flow — Tick to Trade Decision

```
NSE WebSocket (option ticks)
        │
        ├──► on_option_tick(token, strike, opt_type, ltp, oi, vol_delta, ts, bid, ask)
        │              │
        │              ▼
        │    Band Filter (buyer-centric asymmetric):
        │      CE: only strikes AT or BELOW current ATM (calls you'd actually buy)
        │      PE: only strikes AT or ABOVE current ATM (puts you'd actually buy)
        │              │
        │              ▼
        │    _StrikeState.history (rolling deque, ~30 min buffer)
        │    → each _Tick stores: ts, ltp, oi, vol_delta, bid, ask
        │
NSE WebSocket (spot/index tick)
        │
        └──► on_spot_tick(spot, ts_epoch)
                       │
                       ▼
              ┌────────────────────┐
              │   compute_zone()   │  ← called at most every 1s (throttled)
              │   9-Step Pipeline  │
              └────────────────────┘
                       │
                       ▼
              SRZoneResult (full snapshot)
              • zone, nearest_resistance, nearest_support
              • breakout/breakdown probability
              • wall states (STRONG_HOLD / SOFT_ZONE / CONTESTED / LIKELY_BREACH)
              • WallBreachAnalysis (5-signal per wall)
              • GEX regime + gamma wall strike
              • directional_bias
                       │
                 ┌─────┴──────────────────────────────────────┐
                 │                                             │
                 ▼                                             ▼
        spot_cache["sr_zone"]                 Change-detection:
        (shared with fusion signals)          • SR_WALL.CHANGE (INFO)
                 │                            • SR_WALL.HEARTBEAT (30s)
                 ▼                            • SR_WALL.CONVICTION_CHANGE (INFO)
        fusion_signals._validate_trend_reversal()
                 │
                 ▼
        get_gate_verdict(spot, direction)
        → ("pass" | "warn" | "block", reason, score_delta)
```

---

## 3. Layer 1 — OI Data Ingestion Engine

### What It Does

The ingestion layer maintains a **15-minute rolling per-strike cache** of all option ticks. This is the raw material every upstream layer depends on.

### Buyer-Centric Token Universe

NIFTY weekly options tracked = **10 tokens** (5 CE + 5 PE):

```
Spot = 25000 | ATM = 25000 | step = 50 pts | n_strikes = 4

  ┌──────────────────────────────────────────────────────┐
  │   CE (support floor — call BUYERS accumulate here)   │
  │   25000  24950  24900  24850  24800                   │
  │     ATM  -50   -100   -150   -200                    │
  │     ↑    ↑     ↑      ↑      ↑                       │
  │     |    |     | SPOT |     |                         │
  │                  25000                                │
  │     ↓    ↓     ↓      ↓      ↓                       │
  │   25000  25050  25100  25150  25200                   │
  │   PE (resistance ceiling — put BUYERS accumulate here)│
  └──────────────────────────────────────────────────────┘
```

**Why buyer-centric?**

| OI Type | Location | Rising OI means... | Falling OI means... |
|---------|----------|-------------------|-------------------|
| CE OI at/below spot | Support | Call buyers accumulating → floor getting stronger | Call buyers exiting → floor weakening |
| PE OI at/above spot | Resistance | Put buyers accumulating → ceiling getting stronger | Put buyers exiting → ceiling weakening (breakout?) |

### Band Filter Logic

```
Every option tick → check:
  if CE: strike must be in [ATM - 200pts, ATM]      → otherwise REJECTED
  if PE: strike must be in [ATM, ATM + 200pts]      → otherwise REJECTED

Why? Tracks only the 10 tokens that are meaningful for S/R.
Deep-OTM strikes have low OI and distort the average.
```

### Per-Tick Storage (`_Tick`)

```python
_Tick:
  ts        : float   # epoch seconds
  ltp       : float   # last traded price (premium)
  oi        : int     # open interest
  vol_delta : int     # new volume this tick (always ≥ 0)
  bid       : float   # best bid (optional; 0 if not provided)
  ask       : float   # best ask (optional; 0 if not provided)
```

The history deque holds up to **1800 ticks** (~30 minutes at 1 tick/sec) and is pruned to the **15-minute window** at analysis time.

---

## 4. Layer 2 — `compute_zone` (9-Step Analysis Pipeline)

Called every spot tick (≤ 1/sec). Produces the full `SRZoneResult`.

```
compute_zone(spot, ts_epoch)
│
├── Step 1: ATM snap + window strikes
│     nearest_strike(spot) → atm
│     window_strikes(atm) → 10 (strike, opt_type) pairs
│
├── Step 2: Per-token trend computation
│     For each of the 10 tokens:
│       ticks = history filtered to [ts − 15min, ts]
│       _compute_trends() → (oi_trend %/min, prem_trend %/min, vol_mult)
│       _compute_oi_acceleration() → 2nd derivative (%/min²)
│       _compute_prem_velocity() → 5-tick LTP slope (pts/sec)
│       _is_oi_wall() → compare OI vs 4 neighbors (is_wall, wall_mult)
│       _level_strength() → composite score [0..1]
│       → SRLevel built for each token
│
├── Step 3: Rank S/R candidates
│     sup_candidates = CE levels at/below spot (sorted by strength desc)
│     res_candidates = PE levels at/above spot (sorted by strength desc)
│     nearest_resistance = res_candidates[0]
│     nearest_support    = sup_candidates[0]
│
├── Step 3b–3c: Freshness + PCR
│     data_freshness_sec = ts - newest_option_tick_ts
│     window_pcr = total_PE_OI / total_CE_OI
│
├── Step 3d: GEX (Gamma Exposure) computation
│     For each level:
│       gamma = _bs_gamma(spot, strike, t_yrs, atm_iv)
│       gex   = OI × lot_size × gamma × spot²  / 1e11
│     CE levels → +GEX (dealers long gamma → buy dips)
│     PE levels → -GEX (dealers short gamma → sell rallies)
│     net_gex = sum(CE_GEX) - sum(PE_GEX)
│     gamma_wall_strike = strike with highest |GEX|
│     gex_flip_zone (GNL) = spot where net_GEX ≈ 0
│     gex_regime = POSITIVE_GEX | NEGATIVE_GEX | NEUTRAL_GEX
│
├── Step 4: Zone classification
│     spot >= res.strike      → BREAKOUT_ABOVE
│     spot <= sup.strike      → BREAKDOWN_BELOW
│     dist_to_res ≤ 25pts     → NEAR_RESISTANCE
│     dist_to_sup ≤ 25pts     → NEAR_SUPPORT
│     otherwise               → IN_ZONE
│
├── Step 5: Zone quality
│     EXCELLENT | GOOD | MARGINAL | POOR  (based on avg level strength)
│
├── Step 6: Breakout / Breakdown probability
│     _breakout_prob()  → OI flow signals + GEX adjustment [0..1]
│     _breakdown_prob() → mirror logic for SUP [0..1]
│
├── Step 6b: WallBreachAnalyzer  ← FULL DETAIL IN SECTION 5
│     _analyze_wall_breach(state, level, "RES", spot, t_yrs, atm_iv)
│     _analyze_wall_breach(state, level, "SUP", spot, t_yrs, atm_iv)
│     → res_breach_analysis, sup_breach_analysis (5-signal WallBreachAnalysis)
│     → res_wall_state, sup_wall_state (STRONG_HOLD | SOFT_ZONE | CONTESTED | LIKELY_BREACH)
│
├── Step 7: Tight-range detection
│     is_tight_range = zone_width_pts ≤ 2 × strike_step (≤ 100 pts for NIFTY)
│     min_entry_dist_pts = max(15, zone_width / 2)
│
├── Step 8: Breakout / Breakdown confirmation
│     _confirm_breakout() → 3 signals: spot > res, OI unwind, prem velocity
│     _confirm_breakdown() → mirror for support
│     Score ≥ 0.55 → confirmed = True
│
└── Step 9: Directional bias
      Priority order:
        1. breakout_confirmed  → UPTREND
        2. breakdown_confirmed → DOWNTREND
        3. is_tight_range      → SIDEWAYS
        4. breakout_prob ≥ 0.5 → UPTREND
        5. breakdown_prob ≥ 0.5 → DOWNTREND
        6. window_pcr > 1.1    → UPTREND
        7. window_pcr < 0.9    → DOWNTREND
        8. default             → SIDEWAYS
```

### Wall State Matrix (2×2 Classification)

Every wall is classified into one of four states based on **structural strength × breach probability**:

```
                    ┌───────────────────────────────────────────┐
                    │         BREACH PROBABILITY                │
                    │    LOW (< 0.35)       HIGH (≥ 0.35)       │
         ┌──────────┼───────────────────────────────────────────┤
  WALL   │  HIGH    │  STRONG_HOLD          CONTESTED           │
STRENGTH │  (OI     │  • Block CE/PE        • Downgrade         │
         │  wall or │    near wall          • block → warn      │
         │  str≥.40)│  • Gate vetoes        • Conviction needed  │
         ├──────────┼───────────────────────────────────────────┤
         │  LOW     │  SOFT_ZONE            LIKELY_BREACH       │
         │  (thin   │  • Wall irrelevant    • Flip block → pass  │
         │  wall)   │  • Pass freely        • BREAKOUT SETUP    │
         └──────────┴───────────────────────────────────────────┘

  Conviction modifiers:
    conviction = BREACH  → lower bar for LIKELY_BREACH (bp threshold: 0.65 → 0.50)
    conviction = RESPECT → raise bar    (contested threshold: 0.35 → 0.45)
```

### GEX Regime Impact on Decisions

```
                        GEX REGIME DECISION MAP

  POSITIVE_GEX (net_gex > 0)                NEGATIVE_GEX (net_gex < 0)
  Dealers NET LONG gamma                     Dealers NET SHORT gamma
  → Buy dips, mean-revert                    → Sell rallies, momentum
  ┌──────────────────────────┐               ┌──────────────────────────┐
  │ CE entry near support:   │               │ CE entry near resistance:│
  │   GAMMA_REINFORCED bonus │               │   GAMMA_WALL penalty     │
  │ PE entry near support:   │               │ CE entry below GNL:      │
  │   GAMMA_TRAP block       │               │   GNL_TRAP block         │
  │ Breakdown_prob reduced   │               │ Breakout_prob reduced    │
  └──────────────────────────┘               └──────────────────────────┘

  GNL = Gamma Neutral Level = estimated spot where net_GEX ≈ 0
  Below GNL in NEGATIVE_GEX → dealers MUST sell every rally (structural headwind for CE)
  Above GNL in POSITIVE_GEX → dealers MUST buy every dip  (structural tailwind for CE)
```

---

## 5. Layer 3 — WallBreachAnalyzer (5-Signal Per-Wall Assessment)

This is the deepest analytical layer. For each of the **nearest resistance** and **nearest support** walls, it runs a 5-signal pipeline to produce:
- `breach_probability` ∈ [0.0, 1.0]
- `conviction_direction` = "BREACH" | "RESPECT" | "UNCERTAIN"

### Signal Pipeline Flow Diagram

```
Per-wall (run twice: once for RES, once for SUP)
│
├── INPUT
│     _StrikeState.history (15-min tick window)
│     SRLevel (oi, oi_trend, prem_velocity, vol_mult, etc.)
│     spot, t_yrs (DTE), atm_iv (BSM ATM implied vol)
│
├── S1: OI Base Signal  [range: -0.18 → +0.30]
│     ─────────────────────────────────────────
│     _compute_wall_breach_prob(level, wall_direction)
│     Inputs: oi_trend_pct_min, oi_acceleration, prem_velocity,
│             vol_trend_mult, is_oi_wall, oi_wall_mult
│
│     OI trend contribution:
│       oi_trend ≤ -1.5%/m  → +0.30  (free fall — wall collapsing)
│       oi_trend ≤ -0.8%/m  → +0.22
│       oi_trend ≤ -0.3%/m  → +0.12
│       oi_trend ≥ +1.0%/m  → -0.18  (actively building — very strong)
│
│     Premium velocity (OI-stale safe):
│       RES wall: prem_vel ≥ +0.20pts/s → +0.28  (buyers paying through ceiling)
│       SUP wall: prem_vel ≤ -0.20pts/s → +0.28  (holders fleeing floor)
│
│     Vol surge + OI context:
│       vol_mult ≥ 2.5 + OI_unwinding → +0.15  (aggressive breach)
│       vol_mult ≥ 2.5 + OI_building  → -0.05  (wall reinforcement)
│
├── S2: Volume/OI Intensity  [range: -0.08 → +0.25]
│     ─────────────────────────────────────────────
│     vol_oi = sum(vol_delta in 15min window) / latest_oi
│
│     Normal NIFTY range: 0.05 – 0.30 per 15-min window
│
│     vol_oi ≥ 0.40 + OI unwinding  → +0.25  (institutional breach attempt)
│     vol_oi ≥ 0.25 + OI unwinding  → +0.18
│     vol_oi ≥ 0.15 + OI unwinding  → +0.12
│     vol_oi ≤ 0.005                → -0.08  (ghost volume = no energy)
│
├── S3: IV Trend (per-tick BSM)  [range: -0.15 → +0.20]
│     ────────────────────────────────────────────────
│     For each tick: _bs_iv(ltp, spot, strike, t_yrs) → IV
│     Collect last 40 ticks; run linear regression on IVs vs time
│     → iv_slope (change in IV per minute)
│
│     Expanding IV = option buyers paying more → breach energy building:
│       slope > 0.015/m  → +0.20
│       slope > 0.008/m  → +0.13
│       slope > 0.003/m  → +0.07
│
│     Collapsing IV = wall solidifying, pin expected:
│       slope < -0.015/m → -0.15
│       slope < -0.008/m → -0.10
│       slope < -0.003/m → -0.05
│
│     SKIP if: < 6 valid IV points OR time span < 0.1 min
│     EXCEPTION → logger.warning (was silent before!)
│
├── S4: Bid-Ask Spread Behaviour  [range: -0.10 → +0.14]
│     ────────────────────────────────────────────────
│     Uses last 15 ticks with bid/ask data
│     spread_pct = (ask - bid) / mid_price
│     spread_trend = (latest_spread - baseline_spread) / baseline_spread
│
│     Widening spread = MMs pulling quotes = wall structurally softening:
│       spread_trend > 35%  → +0.14
│       spread_trend > 20%  → +0.09
│       spread_trend > 10%  → +0.05
│
│     Tightening spread = MMs confident = wall holding:
│       spread_trend < -20% → -0.10
│       spread_trend < -10% → -0.06
│
│     SKIP if: no bid/ask data in ticks OR < 4 valid ticks
│
└── S5: Delta Proximity (gamma zone)  [range: -0.16 → +0.18]
      ────────────────────────────────────────────────────
      delta = _bs_delta(spot, strike, t_yrs, atm_iv, is_call)
      |delta| = 0 → deep OTM;  |delta| = 0.50 → ATM (max gamma)

      deep_gamma = (0.42 ≤ |delta| ≤ 0.58)  ← within 8% of ATM
      gamma_zone = (0.35 ≤ |delta| ≤ 0.65)  ← within 15% of ATM

      deep_gamma + OI_unwinding → +0.18  (gamma flip at ATM = violent move)
      deep_gamma + OI_building  → -0.16  (ATM wall holding = max resistance)
      gamma_zone + OI_unwinding → +0.10
      gamma_zone + OI_building  → -0.10

      SKIP if: t_yrs < 1e-6 or atm_iv < 0.01
```

### Signal Combination & Conviction

```
breach_probability = clamp(S1 + S2 + S3 + S4 + S5, 0.0, 1.0)

Fast signals (for conviction test):
  fast_breach = count of:
    • prem_vel > +0.08 pts/s (RES) or < -0.08 pts/s (SUP)
    • vol_oi_ratio_sig ≥ 0.12
    • iv_trend_sig ≥ 0.07

  fast_hold = count of:
    • prem_vel < -0.05 pts/s (RES) or > +0.05 pts/s (SUP)
    • vol_oi_ratio_sig ≤ -0.04
    • iv_trend_sig ≤ -0.05

Conviction assignment:
  bp ≥ 0.60 AND fast_breach ≥ 2  →  "BREACH"
  bp ≤ 0.28 AND fast_hold ≥ 1    →  "RESPECT"
  otherwise                       →  "UNCERTAIN"
```

### Worked Example — WBA Signal Combination

```
Scenario: CE BUY, spot=24980, resistance wall at 25000
  Options: PE 25000, t_yrs=0.007 (2 days), atm_iv=0.22, 28 ticks available

  S1: oi_trend=-1.2%/m → +0.22  prem_vel=+0.15pts/s → +0.18  vol_mult=2.1 → +0.10
      oi_base = 0.50

  S2: total_vol=1,200, latest_oi=85,000
      vol_oi = 1200/85000 = 0.0141 → +0.03 (below 0.08 threshold)

  S3: IV ticks: [0.21, 0.215, 0.22, 0.228, 0.235] over 5 minutes
      iv_slope = +0.005/min → +0.07

  S4: spread_pct=0.82%, spread_trend=+0.25 → +0.09

  S5: delta = _bs_delta(24980, 25000, 0.007, 0.22) ≈ 0.487
      deep_gamma(True) + OI_unwinding(True) → +0.18

  COMBINED: 0.50 + 0.03 + 0.07 + 0.09 + 0.18 = 0.87 → clamped = 0.87

  fast_breach: prem_vel=+0.15>0.08 ✓  vol_oi_sig=0.03<0.12 ✗  iv_sig=0.07=0.07 ✓
               fast_breach = 2

  conviction = bp(0.87) ≥ 0.60 AND fast_breach(2) ≥ 2 → "BREACH"

  → get_gate_verdict flips "block" to "pass+1" (LIKELY_BREACH override)
```

---

## 6. Layer 4 — Gate Verdict (S9 Trade Entry Filter)

`get_gate_verdict(spot, direction)` returns `("pass"|"warn"|"block", reason, score_delta)`.

### Decision Tree — CE BUY (direction = "UP")

```
get_gate_verdict(spot, "UP")
│
├── Data checks
│   ├── result is None       → warn "warming up"
│   └── age > 60s            → warn "stale"
│
├── TIGHT_RANGE path (zone_width ≤ 100pts)  ─── takes priority over all below
│   │
│   ├── breakout_confirmed → PASS+1  "TIGHT_RANGE_BREAKOUT_CONFIRMED"
│   │
│   ├── dist_to_res < min_entry_dist_pts:
│   │   ├── breakout_prob ≥ 0.85 → WARN "TOO_CLOSE_BUT_BRK_HIGH"
│   │   ├── res_wall_state == LIKELY_BREACH → PASS+1 "LIKELY_BREACH override"
│   │   ├── res_wall_state == CONTESTED → WARN "CONTESTED_RES"
│   │   └── default → BLOCK "TIGHT_RANGE_TOO_CLOSE_TO_RES"
│   │
│   ├── breakout_prob ≥ 0.5 → WARN "PRE_BREAKOUT"
│   │
│   └── breakout_prob < 0.5 + dist_to_res ≥ min_dist:
│       ├── GAMMA_TRAP_CE (res OI building + gamma wall + NEGATIVE_GEX) → BLOCK
│       ├── GNL_TRAP_CE (spot below GNL + NEGATIVE_GEX + deep) → BLOCK
│       └── default → PASS+1 "TIGHT_RANGE_SAFE_DIST"
│
└── NORMAL RANGE path (zone_width > 100pts)
    │
    ├── BREAKOUT_ABOVE zone:
    │   ├── breakout_confirmed → PASS+1 "BREAKOUT_CONFIRMED"
    │   └── unconfirmed        → WARN  "BREAKOUT_UNCONFIRMED"
    │
    ├── NEAR_RESISTANCE zone:
    │   │
    │   ├── Near-wall hard veto (dist ≤ near_zone+5pts AND brk_prob < 0.45):
    │   │   ├── LIKELY_BREACH → PASS+1  "CE_NEAR_RES_LIKELY_BREACH"  ← OVERRIDE
    │   │   ├── CONTESTED → WARN        "CE_NEAR_RES_CONTESTED"      ← DOWNGRADE
    │   │   └── default → BLOCK         "CE_NEAR_RES_WALL_HARD_BLOCK"
    │   │
    │   ├── oi_trend ≥ 0.3%/m (building):
    │   │   ├── GAMMA_WALL + NEGATIVE_GEX → BLOCK [GAMMA_WALL]
    │   │   ├── is_oi_wall or prem_vel < -0.05 → BLOCK "NEAR_RES"
    │   │   └── otherwise → WARN "NEAR_RES"
    │   │
    │   ├── oi_trend ≤ -0.3%/m (unwinding):
    │   │   ├── GAMMA_WALL + NEGATIVE_GEX → WARN (gamma persists despite OI unwind)
    │   │   └── default → PASS+1 "RES_UNWIND"
    │   │
    │   └── neutral oi_trend → WARN
    │
    └── IN_ZONE / NEAR_SUPPORT / BREAKDOWN_BELOW:
        ├── dist_to_res ≤ 2×min_dist AND brk_prob = 0.0 → WARN "CE_NO_BREAKOUT_ENERGY"
        ├── dist_to_res ≤ near_zone+5 AND brk_prob < 0.45:
        │   ├── LIKELY_BREACH → PASS+1  "CE_NEAR_RES_LIKELY_BREACH"
        │   ├── CONTESTED → WARN
        │   └── default → BLOCK "CE_NEAR_RES_WALL_HARD_BLOCK"
        ├── borderline_near_res + weak_brk_prob → WARN "BORDERLINE_NEAR_RES_WEAK"
        └── GAMMA_REINFORCED if gamma_wall == support + POSITIVE_GEX → PASS+1 bonus tag
```

### Decision Tree — PE BUY (direction = "DOWN")

Mirror of CE BUY with these key differences:

| CE BUY | PE BUY |
|--------|--------|
| Near RESISTANCE = danger | Near SUPPORT = danger |
| OI unwind at res = bullish | OI unwind at sup = bearish |
| GAMMA_TRAP at res ceiling | GAMMA_TRAP at sup floor |
| GNL below spot = headwind | GNL above spot = headwind |
| POSITIVE_GEX = reinforced support bonus | NEGATIVE_GEX = reinforced resistance bonus |
| NEAR_RESISTANCE zone → PE blocked (strategy: prefer CE breakout) | — |

### Verdict Summary Table

| Situation | Verdict | Score Delta |
|-----------|---------|-------------|
| Zone confirmed breakout/breakdown | pass | +1 |
| IN_ZONE, adequate distance | pass | +1 |
| OI unwinding at near wall | pass | +1 |
| LIKELY_BREACH override | pass | +1 |
| Gamma wall reinforcing support (CE) | pass | +1 |
| Tight range, safe distance, brk prob low | pass | +1 |
| Near wall, neutral OI | warn | 0 |
| Breakout/breakdown unconfirmed | warn | 0 |
| CONTESTED wall | warn | 0 |
| Zero breakout energy near ceiling | warn | 0 |
| Borderline distance, weak prob | warn | 0 |
| Near wall, OI building (no hard signals) | warn | 0 |
| Near wall, OI building + is_oi_wall | **block** | 0 |
| Near wall, OI building + prem falling | **block** | 0 |
| Near wall, OI building + GAMMA_WALL | **block** | 0 |
| Too close to wall, low brk_prob | **block** | 0 |
| GAMMA_TRAP (OI build + gamma + GEX) | **block** | 0 |
| GNL below spot in NEGATIVE_GEX | **block** | 0 |
| PE fade at resistance (strategy) | **block** | 0 |

---

## 7. Fusion Integration — How All 5 Layers Feed the Trade Engine

```
_validate_trend_reversal() in fusion_signals.py
│
├── L1: Fusion signals (Ichimoku, OI, PCR, flow) → raw signal
│
├── L2: Trend reversal filter
│   ├── TREND_REVERSAL_EXECUTE_CONFIRM_COUNT = 3  (3 consecutive passes)
│   ├── TREND_REVERSAL_FLIP_COOLDOWN_SEC = 600s   (10 min between flips)
│   └── MASTER_GATE_POST_EXIT_COOLDOWN_SEC = 600s  (10 min after live-exit)
│
├── L3: MASTER_GATE (analyze_symbol_trade_quality_from_csv)
│   ├── TREND_REVERSAL_MASTER_GATE_MIN_TOTAL_SCORE = 0.58
│   ├── TREND_REVERSAL_MASTER_GATE_MIN_ENTRY_EDGE = 0.08
│   ├── TREND_REVERSAL_MASTER_GATE_COMPONENT_EDGE = 0.10
│   └── TREND_REVERSAL_MASTER_GATE_OPP_DOMINANCE_MAX = 2
│
├── IF mg_status == "pass" → run L4:
│   │
│   ├── L4: _sr_eng.get_gate_verdict(spot, direction)
│   │     ↳ reads engine._last_result (SRZoneResult, refreshed each spot tick)
│   │
│   ├── _l4_verdict = "pass" / "warn" / "block" / "unavailable"
│   │
│   ├── Block logic:
│   │   L4_SR_ZONE_BLOCK_ON_BLOCK = True  → block verdict stops execution
│   │   L4_SR_ZONE_BLOCK_ON_WARN  = False → warn verdict still executes
│   │   L4_SR_ZONE_STALE_WARN_ONLY = True → stale data degrades block→warn
│   │
│   └── Execution outcome:
│       pass/warn (not blocking) → EXECUTE trade
│       block (and BLOCK_ON_BLOCK=True) → SKIP trade
│
└── ALWAYS (regardless of L3/L4 outcome) → run L5:
    │
    ├── L5: reads spot_cache["order_flow"] (OrderFlowResult)
    │     ↳ pre-computed by order_flow_engine.on_spot_tick() earlier in same tick
    │
    ├── L5 reads: signal, composite_dps, z_ofi, z_book_imbalance,
    │             z_micro_drift, z_trade_impulse, spread_state,
    │             persistence_count, recommended_action,
    │             atm_strike, atm_ce_ltp, atm_pe_ltp, spot
    │
    ├── L5 also calls: order_flow_engine.get_trend_summary()
    │     → dps_trend, dps_avg_10, signal_streak, flips_last_30
    │
    ├── ⚠ L5 does NOT affect _final_status — NEVER blocks or passes
    │
    └── L5 outputs:
        • 3 display lines in LAYER_FLOW log (Flow, Action, Trend)
        • Boxed OFPE snapshot dump (get_readable_summary())
        • "l5" key in _full_audit dict
        • NOT in layers_agreed or layers_blocked lists

Zone snapshot attached to trade log (from L4):
  zone, dist_to_resistance, dist_to_support,
  breakout_probability, breakdown_probability,
  gex_regime, gamma_wall_strike, gex_flip_zone,
  res_wall_state, res_breach_prob, res_conviction,
  sup_wall_state, sup_breach_prob, sup_conviction,
  window_pcr, data_freshness_sec, tracked_tokens

Order flow snapshot attached to audit dict (from L5):
  signal, composite_dps, z_ofi, z_book_imbalance,
  z_micro_drift, z_trade_impulse, spread_state,
  persistence_count, recommended_action,
  atm_strike, atm_ce_ltp, atm_pe_ltp, spot,
  dps_trend, dps_avg_10, signal_streak, flips_last_30
```

### L4 Display String in Trade Logs

When a trade fires, fusion logs this L4 block in the execution record:

```
L4:  verdict=pass  zone=IN_ZONE  dist_res=87pts  dist_sup=45pts
     gex=NEGATIVE_GEX  res=SOFT_ZONE(bp=0.12)  sup=STRONG_HOLD(bp=0.08)
     brk_prob=0.14  bdn_prob=0.06  pcr=1.23  fresh=2s
```

### L5 Display Strings in Trade Logs

L5 always appears in LAYER_FLOW, even when L3 or L4 blocked the trade:

```
L5 Flow  : 🟢BUY_PRESSURE dps=+0.7823 z[ofi=+1.24 bi=+0.87 md=+0.52 ti=+0.93]
           spread=OK persist=4/3 (info-only)
L5 Action: 📗→ BUY_CE @ ATM 25000 (CE ₹230.50 / PE ₹215.75) NIFTY=25012.50 (info-only)
L5 Trend : 📈trend=RISING avg10=+0.5200 streak=4×BUY_PRESSURE flips=2/30 hist=47
```

### Full LAYER_FLOW Example (All 5 Layers)

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

Note: `agreed` and `blocked` lists only contain L1–L4. L5 is never in either list.

---

## 8. Layer 5 — Order Flow Pressure Engine (OFPE)

> **Source file:** `order_flow_engine.py` (914 lines)
> **Full documentation:** `docs/ORDER_FLOW_ENGINE_DOCUMENTATION.md`
> **Role:** Info-only microstructure overlay — runs on every gate evaluation, never blocks/passes.

### 8.1 What It Does

The OFPE computes **real-time order-flow microstructure** from Zerodha's 5-level market depth snapshots. It answers the question: *"Is institutional money pushing this market directionally right now?"*

It calculates 5 features every tick:

| # | Feature | Formula Basis | What It Measures |
|---|---------|---------------|------------------|
| 1 | **OFI** (Order Flow Imbalance) | Cont-Kukanov-Stoikov delta approach | Net buying/selling pressure at best bid/ask |
| 2 | **Book Imbalance** | Σbid_qty / Σ(bid+ask) across 5 levels | Standing order weight asymmetry |
| 3 | **Microprice Drift** | Weighted mid-price minus naive mid | Where the "true" price is drifting |
| 4 | **Trade Impulse** | Tick returns × √volume, exponentially smoothed | Momentum of executed trades |
| 5 | **Spread Filter** | Bid-ask spread vs historical mean | Liquidity quality gate |

These are z-scored over a rolling window (default 120s) and combined into a single **Composite DPS** (Directional Pressure Score):

$$DPS = 0.35 \cdot z(OFI) + 0.25 \cdot z(BI) + 0.20 \cdot z(MD) + 0.20 \cdot z(TI)$$

### 8.2 Signal Generation & Persistence

```
DPS > +0.3  →  BUY_PRESSURE
DPS < -0.3  →  SELL_PRESSURE
else        →  NEUTRAL

Persistence: 3 of last 5 ticks must agree → stable signal emitted
             else → NEUTRAL (prevents micro-noise)
```

### 8.3 Recommendation Mapping

```
BUY_PRESSURE   →  recommended_action = "BUY_CE"
SELL_PRESSURE  →  recommended_action = "BUY_PE"
NEUTRAL        →  recommended_action = "NEUTRAL"

Each recommendation includes ATM context:
  atm_strike  = nearest 50-rounded strike
  atm_ce_ltp  = last traded price of ATM CE
  atm_pe_ltp  = last traded price of ATM PE
  spot         = current NIFTY 50 spot
```

### 8.4 How L5 Is Called (Code-Level)

```
option_chain_main.py                       fusion_signals.py
┌─────────────────────────┐               ┌──────────────────────────────┐
│ _on_index_tick(tick)     │               │ _validate_trend_reversal()    │
│   ├── sr_zone_engine     │               │   ├── L1 (fusion signals)     │
│   │   .on_spot_tick()    │               │   ├── L2 (anti-churn)         │
│   ├── oi_flow_engine     │               │   ├── L3 (MASTER_GATE)        │
│   │   .on_spot_tick()    │               │   ├── L4 (SR Zone)            │
│   ├── order_flow_engine  │────────────→  │   └── L5: reads               │
│   │   .on_spot_tick()    │  writes to    │       spot_cache["order_flow"] │
│   │   → result stored in │  spot_cache   │       + order_flow_engine      │
│   │     spot_cache       │               │         .get_trend_summary()   │
│   │     ["order_flow"]   │               │                                │
│   └── _write_ofpe_csv_row│               │   L5 NEVER modifies            │
│       → CSV persistence  │               │   _final_status                │
└─────────────────────────┘               └──────────────────────────────┘
```

**Timing:** L5 data is always **pre-computed** before `_validate_trend_reversal()` runs because `_on_index_tick()` fires first in the tick pipeline (same tick cycle).

### 8.5 L5 Configuration

| Config Key | Default | Purpose |
|------------|---------|---------|
| `L5_ORDER_FLOW_ENABLE` | `True` | Master on/off for L5 display |
| `L5_ORDER_FLOW_STALE_SEC` | `30.0` | Max age before L5 data is considered stale |

### 8.6 What L5 Does NOT Do

- ❌ Does not appear in `layers_agreed` or `layers_blocked`
- ❌ Does not modify `_final_status` (EXECUTE/SKIP)
- ❌ Does not change the trade direction or entry price
- ❌ Has no kill switch on trade execution

### 8.7 Why It Exists As Info-Only

The OFPE provides **informational overlay** because:
1. **Data source limitation** — Zerodha provides L2 depth snapshots (not raw L3 order-by-order feed), which limits microstructure precision
2. **Latency** — retail API tick latency (~200-500ms) is too slow for order-flow to be a reliable gate
3. **Validation pending** — the engine needs months of production data to verify signal quality before promotion to gate status

> **Future:** If backtesting confirms DPS signals add edge, L5 can be promoted to a soft gate (like L4's warn mode) in a later release.

---

## 9. Open-Trade Monitoring — Live Position Protection

> ⚠️ **Production Status (19 March 2026):** Both monitors below are **DISABLED** in
> production (`TREND_POSITION_WALL_MONITOR_ENABLE=False`,
> `L4_OPEN_TRADE_WALL_CONV_MONITOR_ENABLE=False`). They were causing a buy→close
> loop where positions were being closed within 3 seconds of entry. The monitors
> remain in code for future re-enabling with adjusted thresholds.

Once a trade is open, two independent monitors protect it using SR data:

### 8a. Position Wall Monitor (POSITION_WALL_MONITOR)

Protects **open CE near resistance** and **open PE near support** from wall bounces.

```
On every spot tick (open position exists):
│
├── ARM condition: spot within ARM_DIST_PTS (40pts) of the opposing wall
│   OR conviction = "BREACH" (arm early even if far from wall)
│
├── State: ARMED (watching for breach or fade)
│
├── BREACH detected: spot crosses wall by BREACH_MIN_PTS (5pts)
│   → HOLDING (waiting for fade or continuation)
│
├── FADE detected: spot retraces back into buffer zone by FADE_RETRACE_PTS (6pts)
│   ├── conviction = "RESPECT" → CLOSE immediately  (wall confirmed holding)
│   └── otherwise → CLOSE with tag "WALL_FADE"
│
└── Wall holds (no breach): spot bounces from buffer zone
    → position held, monitor resets
```

### 8b. L4 Open Trade Wall Conviction Monitor

A **faster, conviction-only monitor** that acts purely on WBA signals — no need for spot to physically cross the wall.

```
On every spot tick (L4_OPEN_TRADE_WALL_CONV_MONITOR_ENABLE = True):
│
├── Find relevant wall: CE open → watch nearest_resistance
│                       PE open → watch nearest_support
│
├── Check: dist_to_wall ≤ L4_OPEN_TRADE_WALL_CONV_MAX_DIST_PTS (60pts)
│
├── Debounce: conviction must be stable for L4_OPEN_TRADE_WALL_CONV_MIN_TICKS (3) ticks
│
├── RESPECT conviction + bp ≤ L4_OPEN_TRADE_WALL_CONV_RESPECT_MAX_BP (0.25):
│   → CLOSE position early  (wall confirmed holding — exit before full reversal)
│   → tag: "L4_WALL_RESPECT"
│
└── BREACH conviction + bp ≥ L4_OPEN_TRADE_WALL_CONV_BREACH_MIN_BP (0.55):
    → HOLD / extend alert  (wall dissolving — position is working)
    → tag: "L4_WALL_BREACH_HOLD"
```

### Monitor State Diagram

```
   UNARMED
     │
     │ spot within ARM_DIST_PTS of wall
     │ OR conviction == "BREACH"
     ▼
   ARMED ◄─────────────────────────────────────────────────────┐
     │                                                         │
     │ spot crosses wall + prem_vel + brk_prob signals         │
     ▼                                                         │
   BREACH_HOLDING                                              │
     │                                                         │
     ├── spot retraces into buffer zone:                       │
     │     conviction == "RESPECT" → CLOSE (fast exit)        │
     │     no conviction → CLOSE (fade exit)                   │
     │                                                         │
     └── spot continues through wall + OI collapses:          │
           → position runs (no close) ─────────────────────────┘
           monitor resets; TTL = 300s
```

---

## 10. Logging & Debugging Guide

### Log Levels Used

| Level | When | What |
|-------|------|------|
| `DEBUG` | Every analysis step | S1–S5 signal inputs/outputs, per-tick data |
| `INFO` | Notable events | BREACH/RESPECT conviction, wall CHANGE/HEARTBEAT, conviction flips |
| `WARNING` | Silent errors exposed | S3 IV / S4 Spread / S5 Delta exceptions (were silent `pass` before) |

### Log Tag Convention

All WBA logs use `f"WBA.{wall_direction}@{strike}"`:

```
WBA.RES@25000 ENTER spot=24980.0 ...
WBA.RES@25000 S1_OI_BASE ...
WBA.RES@25000 S3_IV_TREND ...
WBA.RES@25000 RESULT conviction=BREACH bp=0.872 ...
```

This makes grep-filtering trivial in production logs.

### Key Log Entries to Search For

```bash
# See every conviction result for a specific wall
grep "WBA.RES@25000 RESULT" app.log

# See all BREACH convictions today
grep "conviction=BREACH" app.log

# See all wall structure changes
grep "SR_WALL.CHANGE" app.log

# See all conviction flips (most useful for debugging wrong closes/holds)
grep "SR_WALL.CONVICTION_CHANGE" app.log

# See the full WBA diagnostic dump after any conviction flip
grep -A 50 "SR_WALL.CONVICTION_DUMP" app.log

# See why a gate blocked a trade
grep "S9_SR=🛑" app.log

# See all S3 IV exceptions (silent before this session!)
grep "S3_IV_TREND exception" app.log

# See WBA entry parameters
grep "WBA.*ENTER spot=" app.log

# ──── Layer 5 (OFPE) log searches ────

# See L5 flow display in every gate evaluation
grep "L5 Flow" app.log

# See L5 actionable recommendations (BUY_CE/BUY_PE)
grep "L5 Action" app.log

# See L5 trend summary (RISING/FALLING/FLAT)
grep "L5 Trend" app.log

# See full OFPE snapshot dumps (boxed diagnostic output)
grep "OFPE_SNAPSHOT\|L5_OFPE_SNAPSHOT" app.log

# See OFPE tick processing in option_chain_main
grep "OFPE" app.log

# See all L5 data in audit dict JSON dumps
grep '"l5"' app.log
```

### Enable Full DEBUG Logging (production investigation)

```python
import logging
logging.getLogger("sr_zone_engine").setLevel(logging.DEBUG)
```

At INFO level (normal production): only conviction flips and heartbeats fire (no log flood).
At DEBUG level: every tick's signal decomposition is visible.

### On-Demand Diagnostic Dump

```python
# Full box-formatted WBA analysis dump for current state:
print(engine.get_breach_analysis_str())

# Or in logs:
import logging
logger = logging.getLogger("sr_zone_engine")
logger.info("WBA_DUMP\n%s", engine.get_breach_analysis_str())
```

Output looks like:
```
┌─── WallBreachAnalyzer Diagnostic Dump ──────────────────────────────────────┐
│  spot=24980  zone=IN_ZONE  gex=NEGATIVE_GEX  dte=2.0d  atm_iv=0.220  fresh=1s  │
├─── RES WALL ─────────────────────────────────────────────────────────────────┤
│  RES@25000  🚀LIKELY_BREACH  🔥conviction=BREACH  breach_prob=0.872  ticks=28  │
│    S1 OI_BASE:    oi_sig=+0.50  prem_vel=+0.15                               │
│    S2 VOL_OI:     vol_oi=0.014 → sig=+0.03                                  │
│    S3 IV_TREND:   slope=+0.005/min → sig=+0.07                               │
│    S4 SPREAD:     spread=0.82%  trend=+0.25 → sig=+0.09                     │
│    S5 DELTA_PROX: delta=0.487  prox=0.974 → sig=+0.18                       │
│    COMBINED: 0.50+0.03+0.07+0.09+0.18 = 0.87 → clamped=0.872               │
├─── SUP WALL ─────────────────────────────────────────────────────────────────┤
│  SUP@24950  🛡️STRONG_HOLD  🧱conviction=RESPECT  breach_prob=0.091  ticks=31  │
│    ...                                                                        │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Configuration Reference (TrendConfig)

### Layer 4 Gate Config

| Key | Default | Description |
|-----|---------|-------------|
| `L4_SR_ZONE_GATE_ENABLE` | `True` | Master switch — False = completely bypass L4 |
| `L4_SR_ZONE_BLOCK_ON_BLOCK` | `True` | SR "block" → SKIP the trade |
| `L4_SR_ZONE_BLOCK_ON_WARN` | `False` | SR "warn" → SKIP too (conservative mode) |
| `L4_SR_ZONE_STALE_WARN_ONLY` | `True` | Stale data → degrade block to warn |

### SR Zone Engine Config

| Key | Default | Description |
|-----|---------|-------------|
| `strike_step` | 50 | NIFTY strike spacing (pts) |
| `n_strikes` | 4 | Strikes above ATM (PE) and below ATM (CE) |
| `window_sec` | 900 | 15-min rolling cache |
| `near_zone_mult` | 0.5 | near_zone_pts = 0.5 × 50 = 25 pts |
| `oi_build_thresh` | 0.003 | 0.3%/min = OI actively building |
| `oi_unwind_thresh` | -0.003 | -0.3%/min = OI actively unwinding |
| `strong_oi_mult` | 1.8 | OI ≥ 1.8× avg neighbors = OI wall |
| `near_wall_block_extra_pts` | 5.0 | Buffer above near_zone for hard veto |
| `near_wall_block_max_prob` | 0.45 | Hard veto only if brk_prob < this |
| `borderline_dist_mult` | 2.0 | "barely above min dist" zone multiplier |
| `borderline_weak_prob` | 0.35 | warn (not pass) if brk_prob below this |
| `prefer_ce_breakout_near_res` | `True` | Block PE fade near resistance |

### Position Wall Monitor Config

| Key | Default | Description |
|-----|---------|-------------|
| `TREND_POSITION_WALL_MONITOR_ENABLE` | `False` ⚠️ | Enable traditional wall monitor (DISABLED — buy→close loop) |
| `TREND_POSITION_WALL_MONITOR_ARM_DIST_PTS` | 40.0 | Arm when spot within N pts of wall |
| `TREND_POSITION_WALL_MONITOR_BUFFER_FRAC` | 0.20 | Buffer = 20% of dist to wall |
| `TREND_POSITION_WALL_MONITOR_BREACH_MIN_PTS` | 5.0 | Min pts above wall to count as breach |
| `TREND_POSITION_WALL_MONITOR_FADE_RETRACE_PTS` | 6.0 | Retrace N pts → FADE close |
| `TREND_POSITION_WALL_MONITOR_BREACH_CONV_ARM` | `True` | Arm early on BREACH conviction |
| `TREND_POSITION_WALL_MONITOR_RESPECT_CONV_CLOSE` | `True` | Fast close on RESPECT + fade |

### L4 Open Trade Conviction Monitor Config

| Key | Default | Description |
|-----|---------|-------------|
| `L4_OPEN_TRADE_WALL_CONV_MONITOR_ENABLE` | `False` ⚠️ | Enable conviction-only monitor (DISABLED — buy→close loop) |
| `L4_OPEN_TRADE_WALL_CONV_MIN_TICKS` | 3 | Debounce: stable for N ticks |
| `L4_OPEN_TRADE_WALL_CONV_BREACH_MIN_BP` | 0.55 | Min bp for BREACH hold alert |
| `L4_OPEN_TRADE_WALL_CONV_RESPECT_MAX_BP` | 0.25 | Max bp for RESPECT close |
| `L4_OPEN_TRADE_WALL_CONV_MAX_DIST_PTS` | 60.0 | Only act within N pts of wall |

---

## 12. Key Data Structures

### `SRZoneResult` — Full Zone Snapshot

```
SRZoneResult
├── ts, spot
├── nearest_resistance   : SRLevel   (strongest PE OI above spot)
├── nearest_support      : SRLevel   (strongest CE OI below spot)
├── mid_resistance       : SRLevel   (2nd strongest)
├── mid_support          : SRLevel   (2nd strongest)
├── far_resistance       : SRLevel   (3rd strongest)
├── far_support          : SRLevel   (3rd strongest)
│
├── zone                 : str       BREAKOUT_ABOVE | NEAR_RESISTANCE | IN_ZONE
│                                    | NEAR_SUPPORT | BREAKDOWN_BELOW
├── dist_to_resistance   : float     (pts from spot to nearest resistance)
├── dist_to_support      : float     (pts from spot to nearest support)
├── zone_width_pts       : float     (dist_res + dist_sup)
├── zone_quality         : str       EXCELLENT | GOOD | MARGINAL | POOR
│
├── breakout_probability : float     [0..1]
├── breakdown_probability: float     [0..1]
├── breakout_confirmed   : bool
├── breakdown_confirmed  : bool
├── breakout_conf_score  : float
├── breakdown_conf_score : float
│
├── res_breach_prob      : float     [0..1]
├── res_wall_state       : str       STRONG_HOLD | SOFT_ZONE | CONTESTED | LIKELY_BREACH
├── sup_breach_prob      : float     [0..1]
├── sup_wall_state       : str       STRONG_HOLD | SOFT_ZONE | CONTESTED | LIKELY_BREACH
├── res_breach_analysis  : WallBreachAnalysis   (5-signal full output)
├── sup_breach_analysis  : WallBreachAnalysis   (5-signal full output)
│
├── is_tight_range       : bool
├── tight_range_width_pts: float
├── min_entry_dist_pts   : float
│
├── gamma_wall_strike    : int
├── gamma_wall_gex       : float
├── net_gex              : float
├── gex_flip_zone        : float     (Gamma Neutral Level)
├── gex_regime           : str       POSITIVE_GEX | NEGATIVE_GEX | NEUTRAL_GEX
├── atm_iv_est           : float
├── dte_days             : float
│
├── window_pcr           : float     (total PE OI / total CE OI)
├── window_pcr_trend     : float     (change vs previous snapshot)
├── data_freshness_sec   : float     (seconds since freshest option tick)
├── tracked_tokens_count : int
└── directional_bias     : str       UPTREND | DOWNTREND | SIDEWAYS
```

### `SRLevel` — Per-Strike S/R Level

```
SRLevel
├── strike, opt_type, oi, ltp
├── oi_trend_pct_min     : float  (%/min; + = building, - = unwinding)
├── prem_trend_pct_min   : float  (%/min premium slope)
├── vol_trend_mult       : float  (recent vol / rolling baseline; >1.5 = elevated)
├── distance_pts         : float  (abs distance from current spot)
├── is_oi_wall           : bool   (OI ≥ 1.8× avg of ±2 neighbors)
├── oi_wall_mult         : float  (OI / avg_neighbors)
├── strength             : float  [0..1] composite score
├── oi_acceleration      : float  (%/min² — 2nd derivative; + = wall accelerating)
├── prem_velocity        : float  (pts/sec — 5-tick real-time LTP rate)
├── gex                  : float  (GEX contribution; +CE=support, -PE=resistance)
└── oi_velocity_abs      : float  (lots/min magnitude, sign is on oi_trend_pct_min)
```

### `WallBreachAnalysis` — 5-Signal Output

```
WallBreachAnalysis
├── breach_probability   : float   [0..1] combined score
├── conviction_direction : str     "BREACH" | "RESPECT" | "UNCERTAIN"
│
├── Signal contributions (signed: + = towards breach, - = towards hold)
│   ├── oi_signal         : float  (S1 OI base)
│   ├── prem_velocity_sig : float  (S1 premium velocity component)
│   ├── vol_oi_ratio_sig  : float  (S2 vol/OI intensity)
│   ├── iv_trend_sig      : float  (S3 IV slope)
│   ├── spread_sig        : float  (S4 bid-ask spread)
│   └── delta_prox_sig    : float  (S5 gamma zone proximity)
│
└── Raw values (for audit)
    ├── vol_oi_ratio      : float  (cumulative vol / latest OI)
    ├── iv_trend_pct_min  : float  (IV slope per minute)
    ├── spread_pct        : float  (current bid-ask as % of mid)
    ├── spread_trend      : float  (spread change ratio vs baseline)
    ├── delta_abs         : float  (|BSM delta|)
    ├── delta_proximity   : float  (0=deep OTM, 1=ATM)
    └── ticks_used        : int    (ticks available in 15-min window)
```

---

## 13. Common Decision Scenarios (Worked Examples)

### Scenario 1: CE BUY Blocked — Buying the Ceiling

```
Spot = 24990  |  Resistance = 25000  |  CE BUY direction = UP

Engine reads:
  dist_to_resistance = 10pts  (< near_zone+5 = 30pts)
  breakout_probability = 0.12  (< 0.45 threshold)
  res_wall_state = STRONG_HOLD
  res_breach_prob = 0.18

Decision: "CE_NEAR_RES_WALL_HARD_BLOCK"
Verdict: block  →  SKIP trade

Why: Spot is 10pts from resistance ceiling. Wall is strong.
     Breakout probability is low — no energy to push through.
     Entering here = buying the ceiling = premium absorbed by writers.
```

### Scenario 2: CE BUY ALLOWED — LIKELY_BREACH Override

```
Spot = 24990  |  Resistance = 25000  |  CE BUY direction = UP

Engine reads:
  dist_to_resistance = 10pts  (normally blocked!)
  res_wall_state = LIKELY_BREACH
  res_breach_analysis.conviction = BREACH
  res_breach_analysis.breach_probability = 0.87
  res_breach_analysis.iv_trend_pct_min = +0.012/min  (expanding IV)
  res_breach_analysis.vol_oi_ratio = 0.38 (high vol)

Decision: "CE_NEAR_RES_LIKELY_BREACH@25000"
Verdict: pass+1  →  EXECUTE trade with bonus

Why: Despite being only 10pts from resistance, the WBA shows
     all 5 signals pointing to wall dissolution.
     This is a BREAKOUT SETUP — the wall is the TARGET, not the ceiling.
```

### Scenario 3: CE BUY Blocked by GEX — GNL Trap

```
Spot = 24850  |  Resistance = 25000  |  GNL = 24900

Engine reads:
  gex_regime = NEGATIVE_GEX
  gamma_wall_strike = 25000  (same as resistance = dealers concentrated here)
  gex_flip_zone = 24900  (GNL)
  spot - GNL = 24850 - 24900 = -50pts  (spot is 50pts BELOW GNL)
  breakout_probability = 0.22  (low)

Decision: "TIGHT_RANGE_CE_BELOW_GNL"
Verdict: block

Why: In NEGATIVE_GEX regime, dealers carry massive short gamma at 25000.
     Below the GNL (24900), dealers MUST sell every rally toward 25000
     to delta-hedge. Even if OI looks neutral, the structural sell-pressure
     from dealer hedging makes this a losing entry. The system blocks it.
```

### Scenario 4: Open CE Position Near Wall — Early Close on RESPECT

```
Open CE position, entry at 24850
Spot = 24960  |  Resistance = 25000

Wall Monitor fires:
  spot within 40pts of resistance (ARM_DIST_PTS = 40)
  → state = ARMED

Next tick WBA:
  res_conviction = RESPECT
  res_breach_prob = 0.18  (< RESPECT_MAX_BP = 0.25)
  Stable for 3 ticks (debounce satisfied)

Decision: L4_OPEN_TRADE_WALL_CONV_MONITOR fires
Action: close_open_positions(tag="L4_WALL_RESPECT")

Why: WBA's fast signals (prem falling, IV collapsing, low vol) show
     the resistance wall is HOLDING. The CE position is approaching a
     confirmed ceiling. Rather than wait for a full reversal, the system
     closes early with retained profit.
```

### Scenario 5: Tight-Range Gamma Trap Blocked

```
Spot = 24975  |  Resistance = 25000  |  Support = 24950  (100pt range)
is_tight_range = True  |  min_entry_dist_pts = 50

CE BUY, dist_to_res = 25pts ≥ 50? NO → distance check OK
BUT:
  res.oi_trend_pct_min = +0.8%/m  (> 0.5%/m active short-build)
  gamma_wall_strike = 25000  (gamma wall IS resistance)
  gex_regime = NEGATIVE_GEX

Decision: "TIGHT_RANGE_GAMMA_TRAP_CE"
Verdict: block

Why: Despite adequate distance, the resistance strike is being actively
     reinforced by writers AND carries the maximum dealer gamma.
     In NEGATIVE_GEX, dealers SELL into every rally toward 25000.
     Entering CE here means fighting both OI writers and dealer hedges.
```

---

## Architecture Decision Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  WHAT EACH LAYER PROTECTS AGAINST                                           │
│                                                                             │
│  L1 (Fusion)    : Wrong directional bias (entering against trend)           │
│  L2 (Filter)    : Churn (rapid reversals, minimum hold violations)          │
│  L3 (MasterGate): Poor execution quality (weak CSV score, bad timing)       │
│                                                                             │
│  L4 (SR Engine) : Structural market mistakes:                               │
│    ZONE gate    → Buying a ceiling / Shorting a floor                       │
│    GEX gate     → Trading against dealer hedging flow                       │
│    WBA gate     → Entering when wall is confirmed HOLDING                   │
│    WBA override → NOT blocking when wall is actually BREACHING              │
│    Tight-range  → Entering when S/R are too close together                  │
│    Gamma trap   → OI building at gamma wall + adverse GEX regime            │
│    GNL trap     → Trading below (CE) or above (PE) gamma neutral level      │
│                                                                             │
│  Post-trade monitors:                                                       │
│    WALL_MONITOR → Protect profits when spot reaches opposing wall           │
│    CONV_MONITOR → Exit early when wall conviction confirms RESPECT           │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 14. Final Trade Decisions — Open, Close, Hold

This section explains how all layers combine at the actual moment of decision: whether to **open a new trade**, **close an existing trade**, or **hold the current position**.

---

### 13a. Opening a New Trade — Full Gate Sequence

```
Every spot tick (~1s) → on_spot_tick() → checks for reversal

┌─────────────────────────────────────────────────────────────────────────────┐
│                        NEW TRADE OPEN DECISION                              │
└─────────────────────────────────────────────────────────────────────────────┘

STEP 1 — Trend reversal detected?
  NiftyTrendAnalyzer.on_tick() → current_trend = UPTREND / DOWNTREND / SIDEWAYS
  latch_flipped = (prev_trend != current_trend)
  ↓ No flip → no trade considered. Loop ends.
  ↓ Flip detected → proceed

STEP 2 — Reversal exit cooldown (anti-churn)
  now_ts - _trend_reversal_last_exit_ts >= TREND_REVERSAL_EXIT_COOLDOWN_SEC (300s)
  ↓ Too soon → SUPPRESSED. Loop ends.
  ↓ OK → proceed

STEP 3 — Minimum hold protection
  Current position age < TREND_REVERSAL_MIN_HOLD_SEC (600s) → block close+flip
  ↓ Too young → MIN_HOLD_SUPPRESS. Loop ends.
  ↓ OK → proceed to _validate_trend_reversal()

STEP 4 — MASTER_GATE L3 (per-symbol CSV quality)
  analyze_symbol_trade_quality_from_csv():
    total_score >= MASTER_GATE_MIN_TOTAL_SCORE (0.58)     → pass
    entry_edge  >= MASTER_GATE_MIN_ENTRY_EDGE  (0.08)     → pass
    component_edge per signal >= COMPONENT_EDGE (0.10)    → pass
    opp_dominance <= OPP_DOMINANCE_MAX (2)                 → pass
  mg_status = "pass" / "fail"
  ↓ fail → SKIP. No trade.
  ↓ pass → proceed to L4

STEP 5 — L4 SR Zone Gate
  _sr_eng.get_gate_verdict(spot, direction)
  → "pass"  : SR structure supports trade
  → "warn"  : cautionary (blocked if L4_SR_ZONE_BLOCK_ON_WARN=True, else allow)
  → "block" : hard structural veto (blocked if L4_SR_ZONE_BLOCK_ON_BLOCK=True)
  _l4_block_entry = True → _final_status = "l4_block" → SKIP
  _l4_block_entry = False → _final_status = "pass" → proceed

STEP 6 — Execute confirm counter (debounce)
  exec_count += 1 on each EXECUTE_PENDING pass
  exec_count >= TREND_REVERSAL_EXECUTE_CONFIRM_COUNT (3) → confirmed
  exec_count < 3 → EXECUTE_PENDING, suppressed for this tick

STEP 7 — Peak drawdown / daily loss guard
  peak_pnl - current_pnl >= PEAK_DRAWDOWN_BLOCK_PCT (10%) → PEAK_DRAWDOWN_BLOCK
  current_pnl <= -PEAK_DRAWDOWN_BLOCK_PCT → DAILY_LOSS_BLOCK (if enabled)
  ↓ blocked → should_flip = False
  ↓ OK → proceed

STEP 8 — Auto-flip cooldown
  now_ts - _trend_reversal_last_flip_ts >= TREND_REVERSAL_FLIP_COOLDOWN_SEC (600s)
  already holding desired side? → no flip needed
  ↓ can_flip = True → EXECUTE: close counter-trend + open new side
```

### Open Trade Decision Flow Diagram

```
  SPOT TICK
      │
      ▼
  TREND FLIP?  ──── No ────────────────────────────────────── HOLD (no action)
      │ Yes
      ▼
  EXIT COOLDOWN (300s)?  ─── Too Soon ──────────────────────── SKIP (suppressed)
      │ OK
      ▼
  MIN HOLD (600s)?  ─────── Position Too Young ─────────────── SKIP (churn guard)
      │ OK
      ▼
  MASTER_GATE L3  ─────────── FAIL ────────────────────────── SKIP (no quality)
      │ PASS
      ▼
  L4 SR ZONE GATE  ────────── BLOCK ───────────────────────── SKIP (structural veto)
      │ PASS/WARN
      ▼
  CONFIRM COUNT (3)  ──── Count < 3 ─────────────────────────── PENDING (wait)
      │ Count = 3
      ▼
  PEAK DD GUARD  ──────────── TRIGGERED ─────────────────────── SKIP (loss protection)
      │ OK
      ▼
  FLIP COOLDOWN (600s)  ─── Too Soon ────────────────────────── SKIP (anti-churn)
      │ OK
      ▼
  ✅ EXECUTE: CLOSE old position + OPEN new position
```

---

### 13b. Closing an Existing Trade — All Close Triggers

Once a trade is open, there are **5 independent pathways** that can close it. They run on every spot tick.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              5 CLOSE PATHWAYS (independent, run every tick)                 │
└─────────────────────────────────────────────────────────────────────────────┘

PATHWAY 1: Trend Reversal Close+Flip
─────────────────────────────────────
  Trigger: latch_flipped + all 4 gate layers (L1-L4) pass  (L5 is info-only)
  What closes: counter-trend position (e.g. PE open → trend flips UP → close PE)
  What opens: opposite side (CE BUY)
  Tag: TR_LATCH_FLIP
  Rate limit: TREND_REVERSAL_EXIT_COOLDOWN_SEC = 300s between detections
  Age guard: MIN_HOLD_SEC = 600s (position must be at least 10 min old)
  Confirm: 3 consecutive EXECUTE passes before actually closing

PATHWAY 2: Position Health Monitor (3-layer composite)
────────────────────────────────────────────────────────
  Trigger: health_score drops below weak threshold AND stays weak for N ticks
  What closes: the failing position
  Runs: every POS_HEALTH_CHECK_INTERVAL_SEC = 30s
  Min hold guard: POS_HEALTH_MIN_HOLD_SEC = 300s (ignore health check for first 5 min)

  Health score formula:
    health = 0.40 × L1_score + 0.35 × L2_score + 0.25 × L3_score

  L1 score (Nifty trend alignment):
    CE + UPTREND = 1.0  |  CE + SIDEWAYS = 0.5  |  CE + DOWNTREND = 0.0
    PE + DOWNTREND = 1.0 | PE + SIDEWAYS = 0.5  |  PE + UPTREND = 0.0

  L2 score (option bias alignment):
    CE + BULLISH high-conviction = high L2  |  CE + BEARISH = low L2
    PE + BEARISH high-conviction = high L2  |  PE + BULLISH = low L2

  L3 score (token strength):
    50% static strength_score + 50% calibrated dynamic score
    dynamic = 30%×prem_roc + 40%×vol_spike_mult + 30%×oi_change_pct
    Penalise: prem_roc < -0.5%/min → subtract 0.15

  Health tiers:
    ≥ 0.70 → HEALTHY   (green — hold)
    ≥ 0.50 → WATCHFUL  (yellow — watch)
    ≥ 0.35 → WEAK      (orange — warning, weak tick counter starts)
    ≥ 0.15 → CRITICAL  (red — close pending)
    <  0.15 → FAILING  (force exit with just 2 ticks)

  Close thresholds:
    FAILING (score < 0.15) for 2 ticks → FORCE EXIT
    WEAK (score < weak_thr) for 3 ticks → EXIT

PATHWAY 3: Wall Monitor Close (POSITION_WALL_MONITOR)
───────────────────────────────────────────────────────
  Trigger: open CE reaches resistance AND wall fades back after brief breach
  What closes: the CE/PE position that hit the wall
  Arm condition: spot within ARM_DIST_PTS (40pts) of opposing wall
               OR res_conviction == "BREACH" (arm early)
  Close condition: spot retraces FADE_RETRACE_PTS (6pts) back inside buffer
                 + res_conviction == "RESPECT" → immediate close (fast exit)
  Tag: WALL_FADE or WALL_RESPECT

PATHWAY 4: L4 Wall Conviction Monitor (L4_OPEN_TRADE_WALL_CONV_MONITOR)
─────────────────────────────────────────────────────────────────────────
  Trigger: WBA conviction flips to RESPECT near the opposing wall
  What closes: wrong-way position (CE near confirmed resistance, PE near confirmed support)
  Condition:
    conviction == "RESPECT"
    breach_probability <= 0.25
    dist_to_wall <= 60pts
    stable for 3 consecutive ticks (debounce)
  Tag: L4_WALL_RESPECT

PATHWAY 5: TSL / Stop Loss (external)
───────────────────────────────────────
  Trigger: premum falls below trailing stop level (managed by order_service_api)
  What closes: the losing position via Kite orders
  This runs outside this engine — documented in order_service_api.py
```

### Close Decision Flow (Per Tick, Open Position)

```
OPEN POSITION EXISTS
        │
        ├────────────────────────────────────────────────────────────────────
        │  Pathway 1: Trend Reversal (every tick)
        │  current_trend flipped AND all gates pass AND min_hold elapsed
        │  → CLOSE (counter-trend) + OPEN (opposite side)
        │
        ├────────────────────────────────────────────────────────────────────
        │  Pathway 2: Health Monitor (every 30s)
        │
        │   L1 score  L2 score  L3 score
        │      │          │         │
        │      └──────────┴────┬────┘
        │                      │  0.40×L1 + 0.35×L2 + 0.25×L3
        │                      ▼
        │               health_score
        │                      │
        │             ┌────────┴─────────────────────────────┐
        │             │                                       │
        │       score >= 0.70                           score < weak_thr
        │       HEALTHY → HOLD                                │
        │                                             weak_tick_counter++
        │                                                     │
        │                                      counter >= 3 → CLOSE (sustained_weak)
        │                                      score < 0.15 for 2 ticks → FORCE CLOSE
        │
        ├────────────────────────────────────────────────────────────────────
        │  Pathway 3: Wall Monitor (every tick, armed when spot near wall)
        │  ARMED → spot crosses wall → BREACH_HOLDING
        │  → spot retraces 6pts + RESPECT conviction → CLOSE
        │
        ├────────────────────────────────────────────────────────────────────
        │  Pathway 4: L4 Conviction Monitor (every tick)
        │  conviction==RESPECT + bp<=0.25 + dist<=60pts + stable 3 ticks
        │  → CLOSE (L4_WALL_RESPECT)
        │
        └────────────────────────────────────────────────────────────────────
           Pathway 5: TSL (external, continuous)
           premium <= trailing_stop_level → CLOSE (stop loss)
```

---

### 13c. Holding a Position — When Nothing Triggers a Close

A position is **held** (no action taken) when all of these are true simultaneously:

```
HOLD conditions (all must be true each tick):

  ✅ No trend flip detected (or flip suppressed by cooldown/confirm/min-hold)
  ✅ Health score ≥ weak_thr (0.35 auto-calibrated) — not entering weak territory
  ✅ Wall monitor not ARMED or ARMED but no breach/fade yet
  ✅ L4 conviction monitor: conviction != "RESPECT" (or dist > 60pts, or debounce not met)
  ✅ Premium above TSL trailing stop level
  ✅ No BREACH conviction at opposing wall requiring "hold/extend" alert

If BREACH conviction detected at opposing wall:
  → HOLD continues, but system logs L4_WALL_CONV_MONITOR.hold_alert
  → This means: "wall dissolving → trade thesis confirmed → continue holding"
```

**Hold log signal:**
```
L4_WALL_CONV_MONITOR.hold_alert user=U123 sym=NIFTY25MAR25000CE side=CE
wall=SOFT_ZONE dist=35pts conviction=BREACH bp=0.78
iv_t=+0.82%/m v/oi=0.312 ticks=5 —
wall dissolving → momentum favours hold/extend
```

---

### 13d. Health Score Scoring Detail

```
Scenario: Open CE position. Current market state:

  current_trend = UPTREND        → L1_score = 1.0 (fully aligned)
  option_bias = BULLISH HIGH      → L2_score = 0.90 (CE + BULLISH + HIGH_CONV)
  token strength:
    prem_roc = +2.3%/min          → L3_prem = 2.3/3.0 = 0.77
    vol_spike_mult = 1.8           → L3_vol  = (1.8-1.0)/2.0 = 0.40
    oi_change_pct = +4.2%          → L3_oi   = 4.2/5.0 = 0.84
    dynamic = 0.30×0.77 + 0.40×0.40 + 0.30×0.84 = 0.231 + 0.16 + 0.252 = 0.643
    static strength_score = 0.72
    L3_score = 0.5×0.72 + 0.5×0.643 = 0.36 + 0.32 = 0.682

  health = 0.40×1.0 + 0.35×0.90 + 0.25×0.682
         = 0.40 + 0.315 + 0.170
         = 0.885 → tier = HEALTHY → HOLD

  Now trend shifts (UPTREND → SIDEWAYS):
    L1_score = 0.5 (sideways for CE)
    health = 0.40×0.5 + 0.35×0.90 + 0.25×0.682
           = 0.20 + 0.315 + 0.170
           = 0.685 → still HEALTHY → HOLD

  Trend shifts to DOWNTREND:
    L1_score = 0.0 (counter-trend for CE)
    health = 0.40×0.0 + 0.35×0.90 + 0.25×0.682
           = 0.00 + 0.315 + 0.170
           = 0.485 → tier = WATCHFUL → HOLD (but monitoring)

  Option bias weakens (NEUTRAL LOW):
    L2_score drops to ~0.20
    health = 0.40×0.0 + 0.35×0.20 + 0.25×0.682
           = 0.00 + 0.070 + 0.170
           = 0.240 → tier = WEAK → weak_tick_counter++

  After 3 consecutive WEAK ticks → POS_HEALTH_EXIT fired → CLOSE
```

---

### 13e. Worked Scenarios — Complete Open/Close/Hold Decisions

---

#### Scenario A: Clean UPTREND entry — all gates pass, new CE trade opens

```
Time: 10:15  |  spot = 24800  |  latch flips to UPTREND

STEP 1: latch_flipped = True ✓
STEP 2: last_exit_ts = 09:55 → elapsed = 20 min > 5 min ✓
STEP 3: No open position (is_flat=True) → min_hold skipped ✓
STEP 4: MASTER_GATE → total_score=0.71, entry_edge=0.12, opp_dom=1 → PASS ✓
STEP 5: get_gate_verdict(24800, "UP")
          zone = IN_ZONE, dist_to_res = 145pts, brk_prob = 0.18
          gex_regime = POSITIVE_GEX, gamma_wall at 24900 (support) → GAMMA_REINFORCED
          res_wall_state = SOFT_ZONE (bp=0.12) → no block
          verdict = "pass" +1 ✓
STEP 6: exec_count = 3/3 (3 previous pending ticks) ✓
STEP 7: peak_pnl=0.0, current_pnl=0.0 → drawdown=0.0 < 10% ✓
STEP 8: last_flip_ts = 09:55 → 20 min > 10 min ✓

OUTCOME: EXECUTE
  ✅ Open CE position on NIFTY25MAR25000CE
  Log: "✅ REVERSAL_VALIDATE.EXECUTE confirmed user=U123 trend=UPTREND"
  L4 log: "L4_SR_ZONE.PASS dir=UP score_delta=+1 S9_SR=✅ IN_ZONE dist_res=145pts GAMMA_REINFORCED✅"
```

---

#### Scenario B: CE blocked by L4 — spot too close to resistance

```
Time: 11:02  |  spot = 24975  |  latch flips to UPTREND

STEP 1–4: All pass (L1/L2/L3 OK)
STEP 5: get_gate_verdict(24975, "UP")
          zone = NEAR_RESISTANCE
          nearest_resistance = 25000 (PE wall, oi=2.1M, is_oi_wall=True)
          dist_to_resistance = 25pts
          oi_trend = +0.8%/min (actively building)
          res_wall_state = STRONG_HOLD (bp=0.16)
          prem_velocity = -0.08 pts/s (sellers defending ceiling)
          verdict = "block"  "S9_SR=🛑 CE_NEAR_RES_WALL_HARD_BLOCK dist=25pts brk_prob=0.16"
          _l4_block_entry = True

STEP 5 outcome: _final_status = "l4_block"

OUTCOME: SKIP
  🛑 Trade blocked by L4 SR Zone Gate
  Log: "🛑 L4_SR_ZONE converted EXECUTE→SKIP verdict=block zone=NEAR_RESISTANCE dist_to_res=25pts"

  What needs to change to allow entry:
  a) dist_to_res grows > 30pts (spot falls back to 24970) → passes near_zone test
  b) res_wall_state becomes LIKELY_BREACH → override fires
  c) res OI unwinds (oi_trend < -0.3%/m) → RES_UNWIND path → pass+1
```

---

#### Scenario C: CE position closes via Health Monitor — trend counter-aligned

```
Time: 11:30  |  Open CE from 10:15  (hold age = 75 min > 5 min min_hold ✓)

Health check fires (every 30s):

  current_trend = DOWNTREND (trend flipped)  → L1_score(CE) = 0.0
  option_bias = BEARISH_HIGH               → L2_score(CE) = 0.04  (CE + BEARISH = very low)
  token prem_roc = -1.8%/min, vol_mult=1.1, oi_pct=-2.1%
    L3_dynamic = 0.30×0 + 0.40×0.05 + 0.30×0 = 0.02
    L3_static = 0.30 (strength_score declining)
    L3_score = 0.5×0.30 + 0.5×0.02 = 0.16
    (prem_roc < -0.5 penalty: 0.16 - 0.15 = 0.01)

  health = 0.40×0.0 + 0.35×0.04 + 0.25×0.01
         = 0.00 + 0.014 + 0.003
         = 0.017 → tier = FAILING (< 0.15)

  weak_ticks: 2 (two consecutive checks) ≥ force_min_ticks(2) → FORCE EXIT

OUTCOME: CLOSE CE position
  Tag: POS_HEALTH_EXIT
  Log: "POS_HEALTH.failing user=U123 sym=NIFTY25MAR25000CE side=CE
        score=0.017 tier=FAILING l1=COUNTER(0.00) l2=BEARISH_HIGH(0.04) l3=STALE(0.01) → FORCE_EXIT"
```

---

#### Scenario D: CE position HELD despite wall approach — BREACH conviction overrides close

```
Time: 12:05  |  Open CE from 11:00  |  spot = 24955 → 24985 (approaching 25000 RES)

Wall Monitor arms (spot within 40pts of 25000):
  State: ARMED

Per-tick WBA check:
  _analyze_wall_breach() for RES@25000:
    oi_trend = -1.4%/m → S1: +0.22
    vol_oi = 0.38 + OI_unwinding → S2: +0.25
    iv_slope = +0.012/min → S3: +0.13
    spread_trend = +0.28 → S4: +0.09
    delta_abs = 0.491, deep_gamma + OI_unwinding → S5: +0.18
    breach_probability = 0.87
    fast_breach = 3/3 → conviction = BREACH

L4 Conviction Monitor:
  conviction = BREACH, bp = 0.87 ≥ 0.55 threshold
  stable for 4 ticks ≥ min_ticks(3)
  → HOLD ALERT (no close)

Wall Monitor:
  BREACH detected: spot crossed 25000 by 8pts (> BREACH_MIN_PTS=5)
  State: BREACH_HOLDING
  → holding, waiting for continuation or fade

OUTCOME: HOLD (position runs through wall)
  Info log: "L4_WALL_CONV_MONITOR.hold_alert user=U123 sym=NIFTY25MAR25000CE side=CE
             wall=LIKELY_BREACH dist=15pts conviction=BREACH bp=0.87
             iv_t=+1.20%/m v/oi=0.380 ticks=4 — wall dissolving → hold/extend"

  If spot subsequently retreats 6pts back below 25000:
    Wall monitor FADE detected
    res_conviction = RESPECT (wall reasserted) + bp = 0.19 ≤ 0.25
    → CLOSE immediately  (L4_WALL_RESPECT)
```

---

#### Scenario E: PE trade blocked by daily loss guard after peak drawdown

```
Time: 13:20  |  latch flips to DOWNTREND  |  All L1–L4 gates pass

Peak drawdown check:
  peak_overall_pnl_pct  = 3.20%   (portfolio peaked at +3.2% this session)
  current_overall_pnl_pct = -7.50% (now at -7.5%)
  drawdown = 3.20 - (-7.50) = 10.70%
  PEAK_DRAWDOWN_BLOCK_PCT = 10.0%
  drawdown(10.70%) >= threshold(10.00%) → BLOCKED

OUTCOME: SKIP PE entry
  🛑 Log: "PEAK_DRAWDOWN_BLOCK user=U123
           peak_pnl=3.20% current_pnl=-7.50%
           drawdown=10.70% >= threshold=10.00%
           wanted to enter PE @ spot=24600
           Blocking all new entries to protect remaining profits."
```

---

#### Scenario F: Trade closed early by L4 conviction monitor — "buying confirmed ceiling"

```
Time: 14:10  |  Open CE from 14:00 (10 min old > 5 min min_hold ✓)
              |  spot = 24960  |  nearest resistance = 25000

L4 Conviction Monitor fires (every tick):
  dist_to_resistance = 40pts < max_dist(60pts) ✓

  WBA for RES@25000:
    oi_trend = +0.9%/m (wall BUILDING) → S1: -0.10
    prem_velocity = -0.12 pts/s (sellers dominating) → S1 sub: -0.15
    vol_oi = 0.008 (very low volume, no energy) → S2: -0.08
    iv_slope = -0.011/min (IV collapsing → pin expected) → S3: -0.10
    spread_trend = -0.15 (spread tightening → MMs confident) → S4: -0.06
    delta_abs = 0.38 (gamma_zone but OI building) → S5: -0.10
    breach_probability = 0.0 + (-0.59) → clamped to 0.0
    fast_hold = 3/3 → conviction = RESPECT

  Monitor check:
    conviction = RESPECT ✓
    bp = 0.09 ≤ resp_bp(0.25) ✓
    dist = 40pts ≤ max_dist(60pts) ✓
    stable for 3 ticks ≥ min_ticks(3) ✓

OUTCOME: CLOSE CE position early
  Tag: L4_WALL_RESPECT
  Warning log: "L4_WALL_CONV_MONITOR.close user=U123 sym=NIFTY25MAR25000CE side=CE
                wall=STRONG_HOLD(RES) dist=40pts conviction=RESPECT bp=0.09 ticks=3 —
                wall confirmed holding → closing wrong-way position"

  Why this is good: the CE position was heading toward a confirmed ceiling
  with building OI walls, sellers defending, and IV collapsing. Exiting 40pts
  away saves the position from a full reversal loss.
```

---

### 13f. Decision Priority Summary Table

| Layer | Responsibility | Action It Can Take |
|-------|---------------|-------------------|
| L1 (Nifty Trend) | Directional bias | Triggers reversal detection |
| L2 (Confirm + Cooldown) | Anti-churn, timing | Suppresses premature entries |
| L3 (MASTER_GATE) | Entry quality score | Blocks low-quality setups |
| L4 (SR Zone Gate) | Structural market position | Blocks/approves based on S/R + GEX |
| Health Monitor | Open position momentum | Closes on sustained weakness (L1+L2+L3) |
| Wall Monitor | Price vs wall proximity | Closes on fade after breach |
| L4 Conv Monitor | WBA conviction | Closes on RESPECT, holds on BREACH |
| TSL | Premium drawdown | Closes on trailing stop hit |

### 13g. All Close Triggers Summary

```
POSITION CLOSE TRIGGER MAP

  CE position open:

  ┌──────────────────────────────────────────────────────────────────────┐
  │  Trigger                   │ Condition                 │ Tag         │
  ├──────────────────────────────────────────────────────────────────────┤
  │  Trend flips DOWNTREND     │ latch flip + all gates    │ TR_LATCH_FLIP│
  │  Health FAILING (2 ticks)  │ composite score < 0.15    │ POS_HEALTH_EXIT│
  │  Health WEAK (3 ticks)     │ composite score < weak_thr│ POS_HEALTH_EXIT│
  │  Wall fade + RESPECT conv  │ spot crosses+retraces res │ WALL_FADE   │
  │  L4 conv RESPECT at wall   │ bp<0.25, dist<60, 3 ticks │ L4_WALL_RESPECT│
  │  TSL trailing stop         │ premium < stop level      │ TSL_EXIT    │
  └──────────────────────────────────────────────────────────────────────┘

  Note: All 5 triggers run independently on every tick/interval.
  First one to fire wins. BREACH conviction blocks 4 and wall_monitor
  by keeping state in BREACH_HOLDING (no fade close until reversal confirmed).
```

---

## 15. L4 Tight-Range Block & Conviction Contradiction Block — Complete Reference

> **Added:** 19 March 2026 after post-mortem analysis revealed L4 failed to block
> 40 consecutive trades in a 50pt range (22950–23000), resulting in ₹-25,181 losses.
> Two fixes were deployed: (1) TIGHT_RANGE_MAX_PTS raised 40→55, (2) Gap F
> conviction-direction contradiction block added.

---

### 14a. Fix 1 — Tight-Range Entry Block (MAX_PTS: 40→55)

#### What It Does

When the nearest resistance and nearest support form a "zone", L4 measures the total
width (`zone_width = dist_to_resistance + dist_to_support`). If the width is below
`L4_SR_ZONE_TIGHT_RANGE_MAX_PTS`, the entry is **blocked** because there isn't enough
room for momentum follow-through — the premium oscillates within the range instead
of trending.

#### The Bug (Pre-Fix)

```
Config: L4_SR_ZONE_TIGHT_RANGE_MAX_PTS = 40  (old value)
Market: RES@23000 — SUP@22950  →  zone_width = 50pts

Check: 50pts > 40pts (MAX_PTS)  →  NOT tight range  →  L4 PASS ❌

Result: All 40 trades entered into a 50pt range with SIDEWAYS bias.
        Every single one was a losing trade trapped between walls.
```

NIFTY strike spacing is 50pts, so the most common tight range is exactly 50pts (one
strike gap between resistance and support). The old 40pt threshold could never catch it.

#### The Fix

```
Config: L4_SR_ZONE_TIGHT_RANGE_MAX_PTS = 55  (new value)

Now: 50pts < 55pts (MAX_PTS)  →  IS tight range  →  L4 BLOCK ✅
```

Any single-strike-gap zone (50pts) is now correctly blocked.

#### Configuration Reference

| Config Key | Value | Description |
|------------|-------|-------------|
| `L4_SR_ZONE_TIGHT_RANGE_BLOCK_ENABLE` | `True` | Master switch for tight-range blocking |
| `L4_SR_ZONE_TIGHT_RANGE_MAX_PTS` | `55.0` | Zone narrower than this = BLOCK |

#### Scenario Matrix — Tight Range Block

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  SCENARIO                         │ ZONE_W │ vs 55 │  VERDICT  │ WHY       │
├─────────────────────────────────────────────────────────────────────────────┤
│  A: RES@23000 SUP@22950 spot=22975│  50pts │ 50<55 │ 🛑 BLOCK │ 1-strike  │
│     (yesterday's exact scenario)  │        │       │          │ gap, no   │
│     dir=DOWN(PE), dist_sup=25pts  │        │       │          │ momentum  │
│                                   │        │       │          │ room      │
├─────────────────────────────────────────────────────────────────────────────┤
│  B: RES@23000 SUP@22950 spot=22987│  50pts │ 50<55 │ 🛑 BLOCK │ Same zone │
│     dir=UP(CE), dist_res=13pts    │        │       │          │ even      │
│     (9 losing CE trades yesterday)│        │       │          │ tighter   │
│                                   │        │       │          │ vs wall   │
├─────────────────────────────────────────────────────────────────────────────┤
│  C: RES@23100 SUP@22900 spot=23000│ 200pts │200>55 │ ✅ PASS  │ Wide zone │
│     dir=UP(CE), dist_res=100pts   │        │       │ (cont.)  │ plenty of │
│                                   │        │       │          │ room      │
├─────────────────────────────────────────────────────────────────────────────┤
│  D: RES@23050 SUP@22950 spot=23000│ 100pts │100>55 │ ✅ PASS  │ 2-strike  │
│     dir=DOWN(PE), dist_sup=50pts  │        │       │ (cont.)  │ gap, OK   │
├─────────────────────────────────────────────────────────────────────────────┤
│  E: RES@23000 SUP@22950 spot=22975│  50pts │ 50<55 │ ⚠️ WARN  │ Expiry    │
│     EXPIRY DAY + all guardrails   │        │       │ (allow)  │ override  │
│     pass (see 14b below)          │        │       │          │ active    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 14b. Expiry-Day Override — When Tight-Range Block Becomes a Warning

On expiry day, tight ranges can break violently because gamma is at maximum. The system
has a special override that **relaxes** the tight-range block from BLOCK → WARN if ALL
of these guardrails pass:

#### Guardrail Conditions (ALL must pass)

| # | Guardrail | Config Key | Threshold | Why |
|---|-----------|-----------|-----------|-----|
| 1 | Is expiry window day | `_EXPIRY_WINDOW_DAYS_BEFORE` | ≤ 2 days | Only on/near expiry |
| 2 | Target distance adequate | `_MIN_TARGET_DIST_PTS` | ≥ 25.0 pts | Enough room to run |
| 3 | Target wall breakable | `_MIN_TARGET_BREAK_PROB` | ≥ 0.30 (30%) | Wall shows stress |
| 4 | Adverse wall not collapsing | `_MAX_ADVERSE_BREACH_PROB` | ≤ 0.55 (55%) | Not breaking against you |
| 5 | Target conviction ≠ RESPECT | — | ≠ "RESPECT" | Wall isn't firmly held |
| 6 | Adverse conviction ≠ BREACH | — | ≠ "BREACH" | Opposite wall isn't breaking against you |

#### Decision Flow

```
Tight range detected (zone_width < 55pts)
│
├── Is expiry day (within 2 days of expiry)?
│   ├── NO  → 🛑 BLOCK (unconditional)
│   └── YES → Check guardrails:
│       │
│       ├── target_dist >= 25pts?             YES → continue
│       │                                     NO  → 🛑 BLOCK
│       ├── target_break_prob >= 0.30?        YES → continue
│       │                                     NO  → 🛑 BLOCK
│       ├── adverse_breach_prob <= 0.55?      YES → continue
│       │                                     NO  → 🛑 BLOCK
│       ├── target_conviction != RESPECT?     YES → continue
│       │                                     NO  → 🛑 BLOCK
│       └── adverse_conviction != BREACH?     YES → ⚠️ WARN (allowed)
│                                             NO  → 🛑 BLOCK
```

#### Expiry Override Scenarios

**Scenario E1: Expiry day, guardrails pass → WARN (trade allowed)**

```
Date: Expiry day (Thursday)
Spot: 22975  |  RES@23000  SUP@22950  |  zone_width=50pts (tight)
Direction: DOWN (PE)

Guardrail checks:
  ✅ Is expiry day                    → YES
  ✅ target_dist (to support) = 25pts → 25 >= 25 → PASS
  ✅ target_break_prob = 0.42         → 0.42 >= 0.30 → PASS
  ✅ adverse_breach_prob = 0.12       → 0.12 <= 0.55 → PASS
  ✅ target_conviction = UNCERTAIN    → != RESPECT → PASS
  ✅ adverse_conviction = UNCERTAIN   → != BREACH → PASS

Result: ⚠️ WARN (not BLOCK)
Since BLOCK_ON_WARN=False → trade enters with warning tag.

Why allowed: On expiry day, gamma is maximal. The support wall shows
42% break probability and isn't being respected. With 25pts of room,
a gamma-driven breakdown could yield quick profits.
```

**Scenario E2: Expiry day, target too close → BLOCK**

```
Date: Expiry day
Spot: 22960  |  RES@23000  SUP@22950  |  zone_width=50pts
Direction: DOWN (PE)

Guardrail checks:
  ✅ Is expiry day                    → YES
  ❌ target_dist (to support) = 10pts → 10 < 25 → FAIL

Result: 🛑 BLOCK (guardrail 2 failed)

Why blocked: Only 10pts to support — even on expiry, there isn't
enough room before hitting the wall.
```

**Scenario E3: Expiry day, target wall is RESPECT → BLOCK**

```
Date: Expiry day
Spot: 22975  |  RES@23000  SUP@22950  |  zone_width=50pts
Direction: DOWN (PE)

Guardrail checks:
  ✅ Is expiry day                    → YES
  ✅ target_dist = 25pts              → PASS
  ✅ target_break_prob = 0.38         → PASS
  ✅ adverse_breach_prob = 0.10       → PASS
  ❌ target_conviction = RESPECT      → FAIL (wall is being respected)

Result: 🛑 BLOCK (guardrail 5 failed)

Why blocked: Even though break probability is 38%, the WBA fast signals
confirm the market is RESPECTING this support. Conflicting signals = don't trade.
```

**Scenario E4: Expiry day, adverse wall BREACH → BLOCK**

```
Date: Expiry day
Spot: 22975  |  RES@23000  SUP@22950
Direction: DOWN (PE) — target=support, adverse=resistance

Guardrail checks:
  ✅ Is expiry day                    → YES
  ✅ target_dist = 25pts              → PASS
  ✅ target_break_prob = 0.40         → PASS
  ✅ adverse_breach_prob = 0.30       → PASS
  ✅ target_conviction = UNCERTAIN    → PASS
  ❌ adverse_conviction = BREACH      → FAIL

Result: 🛑 BLOCK (guardrail 6 failed)

Why blocked: Resistance is actively BREACHING (spot pushing upward through
23000). This is a bullish signal — entering PE (bearish) contradicts it.
```

---

### 14c. Fix 2 — Gap F: Conviction-Direction Contradiction Block

#### What It Does

Even when the zone isn't technically "tight range", this check catches a **logical
contradiction** — the SR wall data says the trade is heading into a wall that the market
respects, or the wall behind the trade is collapsing (meaning the market is moving in
the opposite direction).

#### Configuration

| Config Key | Value | Description |
|------------|-------|-------------|
| `L4_SR_ZONE_CONVICTION_CONTRADICTION_BLOCK_ENABLE` | `True` | Master switch |
| `L4_SR_ZONE_CONVICTION_CONTRADICTION_MAX_BREAK_PROB` | `0.35` | **The key threshold** — target break prob below this + RESPECT + STRONG_HOLD = BLOCK |

#### The Critical Probability Threshold: 35%

**At what breach probability do we allow entry against a respected wall?**

> **Answer: ≥ 35% (0.35)**
>
> If `breach_probability >= 0.35`, the conviction contradiction block does NOT fire,
> even if the target wall is RESPECT + STRONG_HOLD. The reasoning: a 35%+ breach
> probability means the wall is under significant pressure and could break, so we
> allow the trade with appropriate caution.
>
> If `breach_probability < 0.35`, the wall is firmly held with insufficient pressure
> to break — entering against it is a structural mistake.

#### Probability Scale Reference

```
  breach_probability scale [0.0 → 1.0]:

  0.00 ──── 0.10 ──── 0.20 ──── 0.35 ──── 0.50 ──── 0.65 ──── 0.80 ──── 1.00
  │          │          │          │          │          │          │          │
  IRON       FIRM       SOLID      THRESHOLD  CONTESTED  WEAKENING  BREAKING   GONE
  WALL       HOLD       HOLD       ── ≥35%──  WALL       WALL       DOWN       │
  │          │          │          trades     │          │          │          │
  │          │          │          ALLOWED    │          │          │          │
  │          │          │          here →     │          │          │          │
  ◄──────────── BLOCKED ──────────►          ◄────────── ALLOWED ──────────────►
  conviction_contradiction fires              conviction_contradiction SKIPPED

  What each range means for trading:
  ────────────────────────────────────────────────────────────────────────
  0.00–0.10: Wall is IRON — OI massive, building, IV collapsing.
             Zero energy to push through. Buying into this = guaranteed loss.
             Example: 23000 PE wall with 6.4M OI building +1.9%/min

  0.10–0.20: Wall FIRM — Strong structural defence, minimal breach signals.
             Occasional volume spikes but no sustained pressure.
             Example: 23000 PE wall with 5.5M OI, bp=0.06, STRONG_HOLD

  0.20–0.35: Wall SOLID but some pressure — 1-2 breach signals active
             (maybe rising prem_vel or elevated vol/OI) but not enough.
             Still BLOCKED by conviction contradiction.
             Example: Wall with prem_vel +0.2pts/s but OI still building

  0.35–0.50: THRESHOLD ZONE — Wall is under real pressure.
             ≥35% means conviction contradiction does NOT fire.
             Trade is allowed, but other L4 checks still apply.
             Example: OI trend flat, vol_oi elevated, IV expanding slightly

  0.50–0.65: CONTESTED — Wall actively being tested.
             Multiple breach signals firing. Gate allows freely.
             Example: OI unwinding -0.5%/min, vol spike 2x, prem pushing

  0.65–0.80: WEAKENING — Wall likely to break.
             wall_state = LIKELY_BREACH. Gate gives PASS+1 bonus.
             Example: OI collapsing -1.2%/min, IV expanding +0.01/min

  0.80–1.00: BREAKING — Wall effectively gone.
             conviction = BREACH. Position should ride through.
             Example: OI free-fall, prem velocity +0.3pts/s, delta at ATM
```

---

#### Case 1: Target Wall = RESPECT + STRONG_HOLD + breach_prob < 35%

**"You're buying into a wall that the market is respecting and shows no sign of breaking."**

##### Conditions for Block

All three must be true simultaneously:

| Condition | What It Means |
|-----------|---------------|
| `target_conviction == RESPECT` | WBA fast signals confirm market treating it as ceiling/floor |
| `target_wall_state == STRONG_HOLD` | Structural analysis shows massive OI, defended, high strength |
| `target_breach_prob < 0.35` | Less than 35% chance of wall breaking |

##### What is "target" vs "adverse"?

```
For CE BUY (direction = UP):
  target wall  = RESISTANCE (the wall you're buying toward)
  adverse wall = SUPPORT    (the wall behind you)

For PE BUY (direction = DOWN):
  target wall  = SUPPORT    (the wall you're selling toward)
  adverse wall = RESISTANCE (the wall behind you)
```

##### Worked Examples — Case 1

**Example 1A: CE buy blocked — buying into respected resistance (yesterday's scenario)**

```
Trend: UPTREND  |  Side: CE  |  Spot: 22987  |  Dir: UP
Target wall (resistance @23000):
  wall_state = STRONG_HOLD  (OI=5.5M, is_oi_wall=True, building +0.8%/m)
  conviction = RESPECT      (fast_hold ≥ 1, bp ≤ 0.28)
  breach_prob = 0.06        (almost zero)

Check: RESPECT ✅ AND STRONG_HOLD ✅ AND 0.06 < 0.35 ✅
→ 🛑 CONVICTION_CONTRADICTION BLOCK

Log: "L4_SR_ZONE.CONVICTION_CONTRADICTION user=<user_id> dir=UP
      target_wall RESPECT+STRONG_HOLD brk_prob=0.06<0.35"

Why: RES@23000 has 5.5M open interest, actively building, premium falling
(sellers defending). Buying CE = buying into a brick wall. The 0.06 breach
probability means only 6% chance of breaking through. 94% chance of reversal.
```

**Example 1B: PE buy blocked — selling into respected support**

```
Trend: DOWNTREND  |  Side: PE  |  Spot: 22960  |  Dir: DOWN
Target wall (support @22950):
  wall_state = STRONG_HOLD  (OI=2.1M, is_oi_wall=True, strength=0.65)
  conviction = RESPECT      (prem_vel positive = floor holding)
  breach_prob = 0.12

Check: RESPECT ✅ AND STRONG_HOLD ✅ AND 0.12 < 0.35 ✅
→ 🛑 CONVICTION_CONTRADICTION BLOCK

Why: SUP@22950 is a strong floor. Call buyers are defending at 22950 with
massive OI. Buying PE (betting on further downside) means betting against
a confirmed floor. 88% chance the floor holds.
```

**Example 1C: CE buy ALLOWED — respected wall BUT high break probability (≥35%)**

```
Trend: UPTREND  |  Side: CE  |  Spot: 22987  |  Dir: UP
Target wall (resistance @23000):
  wall_state = STRONG_HOLD  (OI=3.2M, still big)
  conviction = RESPECT      (market treating as ceiling)
  breach_prob = 0.45        (45% — under significant pressure!)

Check: RESPECT ✅ AND STRONG_HOLD ✅ AND 0.45 < 0.35? ❌ (0.45 >= 0.35)
→ Conviction contradiction does NOT fire → Continue to other L4 checks

Why allowed: Despite being respected, the wall has 45% breach probability.
This means:
  - OI may be building but prem_vel is pushing through
  - Volume is elevated (vol/OI ratio high)
  - IV might be expanding (option buyers paying more)
  - Delta is in the gamma zone
The market is actively testing this wall. At 45%, there's a meaningful
chance it breaks. The system allows the trade (subject to other checks).
```

**Example 1D: CE buy ALLOWED — wall is STRONG_HOLD but conviction is UNCERTAIN**

```
Trend: UPTREND  |  Side: CE  |  Spot: 22987  |  Dir: UP
Target wall (resistance @23000):
  wall_state = STRONG_HOLD  (OI=4.0M, structurally big)
  conviction = UNCERTAIN    (mixed signals — neither BREACH nor RESPECT)
  breach_prob = 0.18

Check: conviction == RESPECT? ❌ (it's UNCERTAIN)
→ Conviction contradiction does NOT fire → Continue

Why allowed: Although the wall is structurally strong and breach probability
is low (18%), the WBA fast signals are mixed (not clearly RESPECT). The market
hasn't made up its mind. The system allows the trade — other checks like
zone proximity and GEX may still block it.
```

**Example 1E: CE buy ALLOWED — conviction RESPECT but wall is SOFT_ZONE**

```
Target wall (resistance @23050):
  wall_state = SOFT_ZONE   (OI=200K, thin wall, low strength)
  conviction = RESPECT     (market avoiding it, but wall is irrelevant)
  breach_prob = 0.08

Check: wall_state == STRONG_HOLD? ❌ (it's SOFT_ZONE)
→ Conviction contradiction does NOT fire → Continue

Why allowed: A SOFT_ZONE wall being respected is meaningless — there's
nothing substantial to respect. The conviction is based on temporary
price action, not structural defence. System allows the trade.
```

##### Summary Matrix — Case 1

```
┌──────────────────────────────────────────────────────────────────────────┐
│ target_conviction │ target_wall_state │ breach_prob │   VERDICT          │
├──────────────────────────────────────────────────────────────────────────┤
│ RESPECT           │ STRONG_HOLD       │ 0.06        │ 🛑 BLOCK (Case 1)│
│ RESPECT           │ STRONG_HOLD       │ 0.18        │ 🛑 BLOCK (Case 1)│
│ RESPECT           │ STRONG_HOLD       │ 0.34        │ 🛑 BLOCK (Case 1)│
│ RESPECT           │ STRONG_HOLD       │ 0.35        │ ✅ SKIP check     │
│ RESPECT           │ STRONG_HOLD       │ 0.45        │ ✅ SKIP check     │
│ RESPECT           │ STRONG_HOLD       │ 0.70        │ ✅ SKIP check     │
│ RESPECT           │ SOFT_ZONE         │ 0.10        │ ✅ SKIP check     │
│ RESPECT           │ CONTESTED         │ 0.40        │ ✅ SKIP check     │
│ RESPECT           │ LIKELY_BREACH     │ 0.60        │ ✅ SKIP check     │
│ UNCERTAIN         │ STRONG_HOLD       │ 0.06        │ ✅ SKIP check     │
│ BREACH            │ STRONG_HOLD       │ 0.80        │ ✅ SKIP check     │
│ UNCERTAIN         │ SOFT_ZONE         │ 0.30        │ ✅ SKIP check     │
└──────────────────────────────────────────────────────────────────────────┘

Only the first 3 rows (RESPECT + STRONG_HOLD + bp < 0.35) trigger a BLOCK.
All other combinations pass through to the next L4 check.
```

---

#### Case 2: Adverse Wall = BREACH + LIKELY_BREACH

**"The wall behind your trade is collapsing — the market is moving against your direction."**

##### Conditions for Block

Both must be true simultaneously:

| Condition | What It Means |
|-----------|---------------|
| `adverse_conviction == BREACH` | WBA fast signals show wall actively breaking |
| `adverse_wall_state == LIKELY_BREACH` | Structural analysis confirms wall is dissolving |

##### What is "adverse" wall?

```
For CE BUY (direction = UP):
  adverse wall = SUPPORT (below spot)
  If support is BREACH + LIKELY_BREACH → floor collapsing → bearish
  This CONTRADICTS the CE (bullish) trade → BLOCK

For PE BUY (direction = DOWN):
  adverse wall = RESISTANCE (above spot)
  If resistance is BREACH + LIKELY_BREACH → ceiling collapsing → bullish
  This CONTRADICTS the PE (bearish) trade → BLOCK
```

> **Note:** Case 2 has NO probability threshold. If the adverse wall has BREACH conviction
> AND LIKELY_BREACH state, the block fires unconditionally. This is because a wall actively
> breaching behind you means the market direction is the opposite of your trade — no amount
> of break probability at the target changes this structural fact.

##### Worked Examples — Case 2

**Example 2A: PE buy blocked — resistance is breaching (bullish, contradicts PE)**

```
Trend: DOWNTREND  |  Side: PE  |  Spot: 22987  |  Dir: DOWN
Adverse wall (resistance @23000):
  wall_state = LIKELY_BREACH (OI unwinding -1.5%/m, vol spike 3x)
  conviction = BREACH        (prem_vel +0.3pts/s, IV expanding)
  breach_prob = 0.82

Check: adverse_conviction == BREACH ✅ AND adverse_wall_state == LIKELY_BREACH ✅
→ 🛑 CONVICTION_CONTRADICTION BLOCK

Log: "L4_SR_ZONE.CONVICTION_CONTRADICTION user=<user_id> dir=DOWN
      adverse_wall BREACH+LIKELY_BREACH — price breaking against DOWN"

Why: Resistance at 23000 is actively dissolving — put OI collapsing,
premium velocity rising, IV expanding. This is a BULLISH signal (ceiling
breaking = upside opening). Buying PE (bearish bet) while the market is
breaking upward = structural contradiction.
```

**Example 2B: CE buy blocked — support is breaching (bearish, contradicts CE)**

```
Trend: UPTREND  |  Side: CE  |  Spot: 22955  |  Dir: UP
Adverse wall (support @22950):
  wall_state = LIKELY_BREACH (OI=580K but unwinding -1.8%/m)
  conviction = BREACH        (floor collapsing — call buyers fleeing)
  breach_prob = 0.75

Check: adverse_conviction == BREACH ✅ AND adverse_wall_state == LIKELY_BREACH ✅
→ 🛑 CONVICTION_CONTRADICTION BLOCK

Why: Support at 22950 is collapsing. Call buyers at 22950 are exiting,
volume is spiking, IV is expanding on puts. This is a BEARISH signal
(floor breaking = downside opening). Buying CE (bullish bet) while
the floor is falling out = structural contradiction.
```

**Example 2C: PE buy ALLOWED — resistance is BREACH but wall is CONTESTED (not LIKELY_BREACH)**

```
Adverse wall (resistance @23000):
  wall_state = CONTESTED     (mixed — some breach signals, some hold signals)
  conviction = BREACH        (fast signals lean toward breach)
  breach_prob = 0.55

Check: adverse_wall_state == LIKELY_BREACH? ❌ (it's CONTESTED)
→ Conviction contradiction does NOT fire → Continue

Why allowed: Although conviction says BREACH, the wall structure is only
CONTESTED (not fully dissolving). The situation is ambiguous — the wall
might hold or break. The system allows the PE trade.
```

**Example 2D: CE buy ALLOWED — support is LIKELY_BREACH but conviction is UNCERTAIN**

```
Adverse wall (support @22950):
  wall_state = LIKELY_BREACH (structurally weak)
  conviction = UNCERTAIN     (mixed fast signals)
  breach_prob = 0.55

Check: adverse_conviction == BREACH? ❌ (it's UNCERTAIN)
→ Conviction contradiction does NOT fire → Continue

Why allowed: Although the support wall looks structurally weak
(LIKELY_BREACH state), the real-time fast signals (prem velocity,
vol/OI, IV trend) aren't confirming an active breach. The wall might
be weak but stable. System allows the trade.
```

##### Summary Matrix — Case 2

```
┌──────────────────────────────────────────────────────────────────────────┐
│ adverse_conviction │ adverse_wall_state │ breach_prob │   VERDICT        │
├──────────────────────────────────────────────────────────────────────────┤
│ BREACH             │ LIKELY_BREACH      │ any value   │ 🛑 BLOCK (Case 2)│
│ BREACH             │ LIKELY_BREACH      │ 0.82        │ 🛑 BLOCK         │
│ BREACH             │ LIKELY_BREACH      │ 0.55        │ 🛑 BLOCK         │
│ BREACH             │ CONTESTED          │ 0.55        │ ✅ SKIP check    │
│ BREACH             │ SOFT_ZONE          │ 0.40        │ ✅ SKIP check    │
│ BREACH             │ STRONG_HOLD        │ 0.60        │ ✅ SKIP check    │
│ UNCERTAIN          │ LIKELY_BREACH      │ 0.55        │ ✅ SKIP check    │
│ RESPECT            │ LIKELY_BREACH      │ 0.30        │ ✅ SKIP check    │
│ UNCERTAIN          │ CONTESTED          │ 0.40        │ ✅ SKIP check    │
│ RESPECT            │ STRONG_HOLD        │ 0.05        │ ✅ SKIP check    │
└──────────────────────────────────────────────────────────────────────────┘

Only the first 3 rows (BREACH + LIKELY_BREACH) trigger a BLOCK.
No probability threshold — the structural contradiction is enough.
```

---

### 14d. Combined Decision Flow — How Both Fixes Work Together

```
Entry attempt arrives at L4 (after L1 + L2 + L3 all pass)
│
├─── STEP 1: Tight Range Check
│    Is zone_width < 55pts?
│    │
│    ├── YES + Not expiry day → 🛑 BLOCK (tight range)        ← Fix 1
│    ├── YES + Expiry day + ALL guardrails pass → ⚠️ WARN     ← Expiry override
│    ├── YES + Expiry day + any guardrail fails → 🛑 BLOCK    ← Fix 1 holds
│    └── NO → continue to Step 2
│
├─── STEP 2: Gap F Conviction Contradiction Check
│    (only runs if Step 1 did NOT already BLOCK)
│    │
│    ├── Case 1: target = RESPECT + STRONG_HOLD + bp < 0.35?
│    │   └── YES → 🛑 BLOCK (running into wall)               ← Fix 2, Case 1
│    │
│    ├── Case 2: adverse = BREACH + LIKELY_BREACH?
│    │   └── YES → 🛑 BLOCK (wall behind collapsing)          ← Fix 2, Case 2
│    │
│    └── Neither → continue to Step 3
│
├─── STEP 3: Stale Data Degradation
│    If verdict=block AND data is stale (>60s)
│    AND L4_SR_ZONE_STALE_WARN_ONLY=True
│    → Downgrade BLOCK → WARN
│
├─── STEP 4: Existing Gate Verdict (S9) from sr_zone_engine
│    All the traditional checks: zone proximity, GEX traps, OI building,
│    GAMMA_WALL, GNL trap, etc.
│
└─── FINAL: Apply block policy
     block + BLOCK_ON_BLOCK=True → 🛑 SKIP trade
     warn  + BLOCK_ON_WARN=False → ✅ Trade enters with warning
     pass  → ✅ Trade enters normally
```

---

### 14e. Real-World Proof — 19 March 2026 Live Log Analysis

#### Before the fix (morning session — old config MAX_PTS=40)

```
40 trades entered into zone_width=50pts (RES@23000, SUP@22950)
L4 blocked ZERO trades
  - 25 had verdict=pass (tight range check: 50 > 40 = not tight)
  - 15 had verdict=warn (BLOCK_ON_WARN=False, so still entered)
  - 0 had verdict=block
Result: ₹-25,181 gross loss, ₹-30,594 net loss after charges

Breakdown by close tag:
  L4_WALL_RESPECT: 9 trades (wall monitors closed them instantly after entry)
  TR_STABLE__FLIP: 5 trades (trend reversed)
  SR_BUF_FADE: 2 trades (support buffer faded)
  TR_LATCH_FLIP: 2 trades (latch flip)
  MG_LIVE_EXIT: 2 trades (master gate live exit)
  FAST_ABORT: 1 trade (fast abort)
```

#### After the fix (afternoon session — new config MAX_PTS=55 + Gap F)

```
Time window: 15:13 — 15:18 (5 minutes of live trading)

Entry attempts: ~25+
L4 blocks: 9 (100% of entries that reached L4)

Block breakdown:
  TIGHT_RANGE_ENTRY_BLOCK: 8 blocks
    "zone_width=50pts<55pts is_tight_range=True"
    All PE entries into the same 50pt range

  TIGHT_RANGE_TOO_CLOSE_TO_RES: 1 block
    "dist=1pts < min=12pts" (CE entry 1pt from 23000 resistance)

Other layers also blocked (before L4 could even fire):
  L2 DIVERGENT_BLOCK: ~8 (option bias contradicted nifty trend)
  L2 NEUTRAL_BLOCK: ~10 (option market showed no conviction)
  L3 entry_buy_ok_failed: 3 (contract didn't confirm the move)
  L3 opp_dominance: 1 (opposite signals dominated)

New trades entered: ZERO ✅
Net loss stayed frozen at ₹-30,594 — no further damage.
```

---

### 14f. Why 35% Is the Right Threshold

The 35% breach probability threshold was chosen based on the 5-signal WBA architecture:

```
Each signal's maximum contribution:
  S1 OI_BASE:      max = +0.30 (OI collapsing)
  S2 VOL_OI:       max = +0.25 (institutional volume)
  S3 IV_TREND:     max = +0.20 (IV expanding)
  S4 SPREAD:       max = +0.14 (MMs pulling quotes)
  S5 DELTA_PROX:   max = +0.18 (gamma zone + OI unwinding)
  Total theoretical max = +1.07 → clamped to 1.00

To reach 35%, you need at least 2 strong breach signals firing:
  Example combos that reach 0.35:
  • S1=+0.22 (OI falling -0.8%/m) + S2=+0.12 (vol elevated) = 0.34 ❌ (barely misses)
  • S1=+0.22 + S2=+0.12 + S3=+0.07 = 0.41 ✅ (3 signals)
  • S1=+0.30 (OI free-fall) + S5=+0.10 (gamma zone) = 0.40 ✅ (2 strong signals)
  • S2=+0.25 (institutional volume) + S5=+0.18 (gamma flip) = 0.43 ✅

This means:
  bp < 0.35 → at most 1 signal is active → wall is solidly held → BLOCK
  bp >= 0.35 → at least 2 signals are active → real pressure exists → ALLOW
```

#### Why not 25%? (too aggressive)

```
At 25%, a single strong S1 signal (oi_trend = -0.8%/m → +0.22) plus minor
S2/S5 noise could cross the threshold. One signal isn't enough to confirm
wall stress — it could be a brief OI blip.
```

#### Why not 50%? (too conservative)

```
At 50%, you'd need 3+ strong signals to pass. This would block too many
valid entries where 2 signals clearly show wall stress. On expiry days,
walls can break violently even at 35-45% probability.
```

#### The 35% Sweet Spot

```
35% = "at least 2 WBA signals are consistently showing breach pressure"
     = "enough evidence that the wall is being genuinely tested"
     = "not just noise, but not yet confirmed breach either"
     = "allow the trade, let other L4 checks add further scrutiny"
```

---

### 14g. Complete Configuration Reference — All L4 Entry Thresholds

| Config Key | Value | Layer | Purpose |
|------------|-------|-------|---------|
| **Tight Range Block** | | | |
| `L4_SR_ZONE_TIGHT_RANGE_BLOCK_ENABLE` | `True` | Fix 1 | Master switch |
| `L4_SR_ZONE_TIGHT_RANGE_MAX_PTS` | `55.0` | Fix 1 | Zone narrower = BLOCK |
| **Expiry Override Guardrails** | | | |
| `_STABLE_RETRY_EXPIRY_WARN_ONLY` | `True` | Override | Allow warn on expiry |
| `_EXPIRY_WINDOW_DAYS_BEFORE` | `2` | Override | Days before expiry to activate |
| `_MIN_TARGET_DIST_PTS` | `25.0` | Override | Min distance to target wall |
| `_MIN_TARGET_BREAK_PROB` | `0.30` | Override | Min target wall break prob |
| `_MAX_ADVERSE_BREACH_PROB` | `0.55` | Override | Max adverse wall breach prob |
| **Conviction Contradiction** | | | |
| `_CONVICTION_CONTRADICTION_BLOCK_ENABLE` | `True` | Fix 2 | Master switch |
| `_CONVICTION_CONTRADICTION_MAX_BREAK_PROB` | `0.50` | Fix 2 | Below this + RESPECT + STRONG_HOLD = BLOCK (raised from 0.35, see §15a) |
| **General L4 Policy** | | | |
| `L4_SR_ZONE_BLOCK_ON_BLOCK` | `True` | Policy | Block verdict = SKIP trade |
| `L4_SR_ZONE_BLOCK_ON_WARN` | `False` | Policy | Warn verdict = trade enters |
| `L4_SR_ZONE_STALE_WARN_ONLY` | `True` | Safety | Stale data degrades block→warn |

---

### 14h. FAQ — Common Questions

**Q: What if ALL stars align (L1 DOWNTREND, L2 BEARISH, L3 high score) but L4 blocks?**

A: L4 still blocks. The gate architecture is sequential — each of L1–L4 has veto power (L5 is info-only and never blocks).
A perfect L1+L2+L3 score into a RESPECT+STRONG_HOLD wall with 6% break probability will
be blocked because the structural market position is wrong. This is by design: yesterday's
40 losing trades all had L1+L2+L3 passing — L4 was the only layer that could have saved them.

**Q: Can I override L4 blocks?**

A: Set `L4_SR_ZONE_BLOCK_ON_BLOCK = False` to let block verdicts through. Not recommended.
Set `L4_SR_ZONE_CONVICTION_CONTRADICTION_BLOCK_ENABLE = False` to disable only the
conviction contradiction check while keeping other L4 checks active.

**Q: What happens when the zone widens (e.g., new support at 22900)?**

A: If zone_width becomes 100pts (23000−22900), it passes the 55pt threshold. Tight range
block no longer fires. The conviction contradiction check (Fix 2) still runs independently —
even in a 200pt zone, buying CE into RESPECT+STRONG_HOLD+bp<0.50 resistance is still blocked.

**Q: On expiry day, what probability lets me through the tight range?**

A: The expiry override requires `target_break_prob >= 0.30` (30%) plus all other guardrails.
This is different from the conviction contradiction threshold (35%). The expiry override also
requires `target_conviction != RESPECT` and `adverse_conviction != BREACH`.

**Q: Can both Case 1 and Case 2 fire simultaneously?**

A: No. The code uses `elif` — Case 1 is checked first. If Case 1 blocks, Case 2 is skipped.
If Case 1 doesn't apply, Case 2 is checked. In practice, they cover different failure modes:
Case 1 = "target wall too strong", Case 2 = "market moving against you".

**Q: What if breach_prob is exactly 0.50?**

A: The check is `< 0.50` (strict less-than). So breach_prob = 0.50 does NOT trigger the block.
At exactly 50%, the trade is allowed through. (Raised from 0.35 on 19-Mar-2026 — see §15a.)

---

## 16. 19-Mar-2026 Fusion Events Deep Analysis — 5 Recommendation Fixes

**Commit**: `55f66ac` on `nishu_prod`
**Date**: 19 March 2026
**Trigger**: Deep analysis of 3,047 fusion events revealed 5 actionable gaps across L1, L2, and L4.

### Problem Summary

| # | Layer | Problem | Impact |
|---|-------|---------|--------|
| 1 | L4 | Conviction contradiction threshold (`bp ≥ 0.35`) too lenient — only 38% of walls breached | Allowed trades into walls that held 62% of the time |
| 2 | L1 | 25 trend flips in 2 hours during -216pt afternoon drop | 7 losing TR_STABLE_FLIP/TR_LATCH_FLIP whipsaw trades |
| 3 | L2 | 207 ALIGNED + LOW conviction (bias 0.38-0.44) signals passed to L3/L4 | Unnecessary L3/L4 load, noise trades |
| 4 | L2 | 91 ALIGNED + HIGH conviction (bias 0.78-0.90) hard-blocked by DEAD_PREM | Potentially valid early signals killed |
| 5 | L3 | No gaps found — working correctly | N/A |

---

### 15a. Rec #1 — Raise Conviction Contradiction Break Probability Threshold (L4)

**Problem**: At `break_prob ≥ 0.35`, the wall validation data showed:
- 28 L4 PASS events across 13 unique wall/minute combos
- **5/13 walls (38%) breached** within 15 minutes
- **8/13 walls (62%) held** — the threshold was letting trades through when the wall was *more likely to hold*

**Evidence Table**:

| Time | Side | Spot | Wall | Level | BP | Result |
|------|------|------|------|-------|----|--------|
| 12:12 | CE | 23254 | RES | 23300 | 0.36 | HELD (gap=19pts) |
| 12:37 | CE | 23251 | RES | 23300 | 0.41 | HELD (gap=26pts) |
| 13:06 | PE | 23235 | SUP | 23200 | 0.41 | BREACHED -4pts |
| 13:37 | CE | 23186 | RES | 23200 | 0.45 | BREACHED +17pts |
| 14:10 | PE | 23113 | SUP | 23100 | 0.45 | BREACHED -10pts |
| 14:21 | PE | 23120 | SUP | 23100 | 0.69 | BREACHED -3pts |
| 14:33 | CE | 23119 | RES | 23150 | 0.50 | HELD (gap=24pts) |
| 14:52 | CE | 23061 | RES | 23100 | 0.54 | HELD (gap=40pts) |

**Fix**: Raised threshold from `0.35` → `0.50`.

**Config Change** (`TrendConfig` class, line ~1222):
```python
# Before:
L4_SR_ZONE_CONVICTION_CONTRADICTION_MAX_BREAK_PROB: float = 0.35

# After:
L4_SR_ZONE_CONVICTION_CONTRADICTION_MAX_BREAK_PROB: float = 0.50
```

**Runtime Change** (line ~10223):
```python
# Before:
_cfg("L4_SR_ZONE_CONVICTION_CONTRADICTION_MAX_BREAK_PROB", 0.35)

# After:
_cfg("L4_SR_ZONE_CONVICTION_CONTRADICTION_MAX_BREAK_PROB", 0.50)
```

**Effect**: At `bp ≥ 0.50`, breach rate improves to ~55%+. Trades are only allowed through when the wall is genuinely likely to break. The conviction contradiction block now correctly gates entries where the SR structure says "this wall will hold".

---

### 15b. Rec #2 — L1 Whipsaw Cooldown (CRITICAL — Highest-Value Fix)

**Problem**: NiftyTrendAnalyzer flipped trend **25 times in 2 hours** during a -216pt afternoon drop (14:00–16:00). Each flip triggered a CE↔PE entry that was immediately closed by the next flip, producing 7 losing whipsaw trades.

| Period | Trend Flips | Avg Duration | Net Move |
|--------|-------------|--------------|----------|
| Morning (09–12) | 29 | 315s (~5 min) | Ranging |
| Afternoon (14–16) | 25 | 215s (~3.5 min) | −216pts DOWN |

The afternoon had a sustained directional drift, but L1 couldn't distinguish it from a ranging market. Each "UP→DOWN→UP" cycle cost ~₹2K-5K in slippage + brokerage.

**Design**: When trend flips ≥ N times in the last M minutes → block all NEW entries for a cooldown period. Counter-trend **exits still proceed** (close losers, just don't open new positions into chop).

**Config** (`TrendConfig` class, lines 1116–1119):
```python
L1_WHIPSAW_COOLDOWN_ENABLE: bool = True           # master toggle
L1_WHIPSAW_COOLDOWN_MAX_FLIPS: int = 3            # trigger on ≥3 flips in window
L1_WHIPSAW_COOLDOWN_WINDOW_SEC: float = 900.0     # 15-minute rolling window
L1_WHIPSAW_COOLDOWN_BLOCK_SEC: float = 300.0      # 5-minute entry block after trigger
```

**Implementation — 3 Code Locations**:

#### Location 1: Flip History Tracking (line ~5830)
Inside the `if latch_flipped:` block, immediately after `_trend_actual_latch_flip_ts` is recorded:

```python
# Maintain rolling list of flip timestamps
_whip_history = list(getattr(self, "_trend_flip_history", []) or [])
_whip_history.append(now_ts)
# Prune entries older than the window
_whip_window = float(getattr(self.cfg, "L1_WHIPSAW_COOLDOWN_WINDOW_SEC", 900.0))
_whip_history = [t for t in _whip_history if (now_ts - t) <= _whip_window]
setattr(self, "_trend_flip_history", _whip_history)
# Check if threshold exceeded
_whip_max_flips = int(getattr(self.cfg, "L1_WHIPSAW_COOLDOWN_MAX_FLIPS", 3))
if len(_whip_history) >= _whip_max_flips:
    setattr(self, "_trend_whipsaw_triggered_ts", now_ts)
    # Logs: L1_WHIPSAW_COOLDOWN.TRIGGERED user=X flips=3 in 900s ...
```

**Key attribute**: `_trend_whipsaw_triggered_ts` — set to `now_ts` when threshold is hit. Read by both enforcement points below. Uses `getattr(self, ..., default)` for safe first-tick initialization.

#### Location 2: LATCH_FLIP Entry Gate (line ~6341)
After `can_flip_by_event = True` is set but **before** the flip order placement. Only suppresses the new entry; the close of the counter-trend position runs unconditionally above.

```python
# L1 Whipsaw Cooldown: suppress flip entry when trend is flipping too rapidly
if can_flip_by_event and bool(getattr(self.cfg, "L1_WHIPSAW_COOLDOWN_ENABLE", True)):
    _whip_trig_ts_f = float(getattr(self, "_trend_whipsaw_triggered_ts", 0.0) or 0.0)
    _whip_block_sec_f = float(getattr(self.cfg, "L1_WHIPSAW_COOLDOWN_BLOCK_SEC", 300.0))
    if _whip_trig_ts_f > 0.0 and (now_ts - _whip_trig_ts_f) < _whip_block_sec_f:
        can_flip_by_event = False
        # Logs: LATCH_FLIP.whipsaw_block user=X trend=Y ...
```

**Flow separation** (lines 6609 vs 6677):
- `close_open_positions()` runs unconditionally at line ~6654 for all `positions_to_close`
- `if should_flip:` at line ~6677 is gated by `can_flip_by_event` → **whipsaw blocks this, not the close**

#### Location 3: STABLE_RETRY Entry Gate (line ~7226)
Before the flat-position entry check, a `_whip_blocked` flag is computed:

```python
_whip_blocked = False
if bool(getattr(self.cfg, "L1_WHIPSAW_COOLDOWN_ENABLE", True)):
    _whip_trig_ts = float(getattr(self, "_trend_whipsaw_triggered_ts", 0.0) or 0.0)
    _whip_block_sec = float(getattr(self.cfg, "L1_WHIPSAW_COOLDOWN_BLOCK_SEC", 300.0))
    _whip_active = (_whip_trig_ts > 0.0 and (now_ts - _whip_trig_ts) < _whip_block_sec)
    if _whip_active:
        _whip_blocked = True
        # Throttled log every 30s to reduce spam

if _is_flat_str and not _str_throttled and not _whip_blocked:
    # ... proceed with L2→L3→L4 validation ...
```

**Scenario Trace — 19-Mar Afternoon**:

```
14:00  Flip 1 (DOWN→UP)   history=[14:00]              len=1 < 3  → ALLOWED
14:03  Flip 2 (UP→DOWN)   history=[14:00, 14:03]       len=2 < 3  → ALLOWED
14:13  Flip 3 (DOWN→UP)   history=[14:00, 14:03, 14:13] len=3 ≥ 3 → ★ TRIGGERED ★
                           _trend_whipsaw_triggered_ts = 14:13
                           This flip's own LATCH_FLIP entry → BLOCKED (can_flip_by_event=False)
                           But counter-trend PE position → CLOSED normally ✓
14:13–14:18  STABLE_RETRY  _whip_blocked=True            → ALL entries BLOCKED
14:20  Flip 4 (UP→DOWN)   history=[14:03, 14:13, 14:20] len=3 ≥ 3 → ★ RE-TRIGGERED ★
                           _trend_whipsaw_triggered_ts slides to 14:20
                           Cooldown NEVER expires during sustained chop ✓
14:25  Quiet ...           (now_ts - 14:20) = 300s        → cooldown expires
14:26  STABLE_RETRY        _whip_blocked=False            → entries ALLOWED again
```

**Edge Cases Verified**:

| Edge Case | Handling |
|-----------|----------|
| First tick (no history) | `getattr(self, "_trend_flip_history", [])` → safe empty list |
| Cooldown re-trigger | Each qualifying flip resets `triggered_ts` → cooldown slides forward |
| `ENABLE=False` toggle | Both gates check `L1_WHIPSAW_COOLDOWN_ENABLE` → fully bypassed |
| Exception in tracking | `try/except: pass` → degrades to pre-cooldown behavior |
| Memory growth | List pruned to 900s window → at most ~10-15 entries |
| Exit independence | `close_open_positions()` at L6654 runs BEFORE `if should_flip:` at L6677 |

**Expected Impact**: Would have prevented at least 7 of the 25 afternoon trades (Trades 15–25), saving ~₹15K-35K in whipsaw losses.

---

### 15c. Rec #3 — L2 ALIGNED_LOW_BIAS_BLOCK (Block Weak Aligned Signals)

**Problem**: L2 passed 207 events with ALIGNED + LOW conviction (bias range 0.38–0.44) downstream to L3/L4. These had barely enough directional bias to leave the neutral band but not enough to justify entry. L3's `entry_buy_ok_failed` had to catch most of them, creating unnecessary processing load.

**Fix**: New Block 6 — `ALIGNED_LOW_BIAS_BLOCK` — added after existing Block 5 (`ALIGNED_LOW_CONVICTION_BLOCK`).

**Config** (`TrendConfig` class, lines 1148–1149):
```python
L2_ALIGNED_LOW_CONVICTION_BIAS_BLOCK_ENABLE: bool = True
L2_ALIGNED_LOW_CONVICTION_BIAS_MAX: float = 0.45   # block ALIGNED LOW with bias < this
```

**Code** (line ~9253, inside the `elif _l2_scenario == "ALIGNED":` branch):
```python
# Block 6 (Rec #3): ALIGNED LOW conviction with weak bias
elif _l2_oc == "LOW" and bool(_cfg("L2_ALIGNED_LOW_CONVICTION_BIAS_BLOCK_ENABLE", True)):
    _l2_low_bias_max = float(_cfg("L2_ALIGNED_LOW_CONVICTION_BIAS_MAX", 0.45))
    _l2_low_bias_weak = (
        (_l2_sc < _l2_low_bias_max) if trend_up     # BULLISH but bias < 0.45
        else (_l2_sc > (1.0 - _l2_low_bias_max))    # BEARISH but bias > 0.55
    )
    if _l2_low_bias_weak:
        _l2_verdict  = "block"
        _l2_scenario = "ALIGNED_LOW_BIAS_BLOCK"
```

**Block Hierarchy** (all within `elif _l2_scenario == "ALIGNED":`):

| Block | Check | Condition | Verdict |
|-------|-------|-----------|---------|
| Block 3 | Dead premium | `prem_roc ≤ 0.10 AND strength < 0.25` | `block` (or softened, see 15d) |
| Block 5 | LOW + neutral band | `conviction=LOW AND |bias-0.5| < 0.10` | `block` |
| **Block 6 (NEW)** | **LOW + weak bias** | **`conviction=LOW AND bias < 0.45 (or > 0.55)`** | **`block`** |

**Note**: Block 5 catches signals *inside* the ±0.10 neutral band (bias 0.40–0.60). Block 6 catches those *outside* neutral but still weak (bias 0.38–0.45 for BULLISH, 0.55–0.62 for BEARISH). Together they eliminate all LOW conviction noise at L2.

**Expected Impact**: ~207 fewer events reaching L3/L4 per session. Reduces log volume and prevents any noise signal from accidentally passing L3 quality gates.

---

### 15d. Rec #4 — L2 ALIGNED_DEAD_PREM_SOFTENED (Soften Dead Premium for HIGH Conviction)

**Problem**: 91 events were hard-blocked by `ALIGNED_DEAD_PREM_BLOCK` despite having HIGH conviction (bias 0.78–0.90). These represented cases where the option chain was directionally aligned with strong conviction, but premium velocity hadn't caught up yet. These could be early signals before a premium move.

**Fix**: Before applying ALIGNED_DEAD_PREM_BLOCK, check if conviction=HIGH AND bias exceeds a strong threshold. If so, downgrade to `ALIGNED_DEAD_PREM_SOFTENED` (pass with warning) instead of hard block.

**Config** (`TrendConfig` class, lines 1153–1157):
```python
L2_ALIGNED_DEAD_PREM_HIGH_CONVICTION_SOFTEN_ENABLE: bool = True
L2_ALIGNED_DEAD_PREM_HIGH_CONVICTION_MIN_BIAS: float = 0.80   # only soften when bias > this
```

**Code** (line ~9207, inside the dead premium check):
```python
if _l2_sub_pr <= _l2_dead_pr and _l2_sub_str < _l2_dead_str:
    # Rec #4: Check if we should soften instead of block
    _l2_dead_soften = bool(_cfg("L2_ALIGNED_DEAD_PREM_HIGH_CONVICTION_SOFTEN_ENABLE", True))
    _l2_dead_soften_min_bias = float(_cfg("L2_ALIGNED_DEAD_PREM_HIGH_CONVICTION_MIN_BIAS", 0.80))
    _l2_dead_soften_bias_ok = (
        (_l2_sc >= _l2_dead_soften_min_bias) if trend_up       # BULLISH: bias ≥ 0.80
        else (_l2_sc <= (1.0 - _l2_dead_soften_min_bias))      # BEARISH: bias ≤ 0.20
    )
    if _l2_dead_soften and _l2_oc == "HIGH" and _l2_dead_soften_bias_ok:
        _l2_verdict  = "pass"
        _l2_scenario = "ALIGNED_DEAD_PREM_SOFTENED"
        # Passes to L3/L4 with warning annotation
    else:
        _l2_verdict  = "block"
        _l2_scenario = "ALIGNED_DEAD_PREM_BLOCK"    # original behavior
```

**Decision Matrix**:

| Conviction | Bias (BULLISH) | Premium Dead? | Old Verdict | New Verdict |
|------------|----------------|---------------|-------------|-------------|
| HIGH | ≥ 0.80 | Yes | ❌ BLOCK | ⚠️ SOFTENED (pass) |
| HIGH | 0.60–0.79 | Yes | ❌ BLOCK | ❌ BLOCK (unchanged) |
| MEDIUM | any | Yes | ❌ BLOCK | ❌ BLOCK (unchanged) |
| LOW | any | Yes | ❌ BLOCK | ❌ BLOCK (unchanged) |

**Safety**: The softened signal still passes through L3 quality gate and L4 SR zone gate. If the premium truly isn't moving, L3's `entry_buy_ok_failed` or L4's structural check will likely still block it. This just removes the premature L2 hard-block for genuinely strong directional setups.

**Expected Impact**: ~91 potentially valid early signals per session get a chance at L3/L4 evaluation instead of being killed at L2.

---

### 15e. Rec #5 — L3 No Change Needed

**Analysis**: L3 funnel from 19-Mar data:
- 1010 L2-allowed → L3 skip=609 (60%), pass=403 (40%)
- `entry_buy_ok_failed` = 409 (correct blocking — entry option didn't pass quality gates)
- `reverse_confirmation_failed` = 197 (correct — opposing option was scoring higher)
- `opp_dominance` = 3 (correct)

L3 pass stats: `score_edge` range 0.042–0.345, avg 0.143. The 403 events that passed L3 genuinely had a directional edge. No changes required.

---

### 15f. Complete Configuration Reference for 19-Mar Fixes

| Config Variable | Value | Layer | Rec # | Purpose |
|----------------|-------|-------|-------|---------|
| `L4_SR_ZONE_CONVICTION_CONTRADICTION_MAX_BREAK_PROB` | `0.50` | L4 | #1 | Raised from 0.35; blocks trades into walls with <50% breach probability |
| `L1_WHIPSAW_COOLDOWN_ENABLE` | `True` | L1 | #2 | Master toggle for whipsaw entry block |
| `L1_WHIPSAW_COOLDOWN_MAX_FLIPS` | `3` | L1 | #2 | Trigger threshold: ≥3 flips in window |
| `L1_WHIPSAW_COOLDOWN_WINDOW_SEC` | `900.0` | L1 | #2 | 15-minute rolling lookback window |
| `L1_WHIPSAW_COOLDOWN_BLOCK_SEC` | `300.0` | L1 | #2 | 5-minute entry cooldown after trigger |
| `L2_ALIGNED_LOW_CONVICTION_BIAS_BLOCK_ENABLE` | `True` | L2 | #3 | Master toggle for weak-bias block |
| `L2_ALIGNED_LOW_CONVICTION_BIAS_MAX` | `0.45` | L2 | #3 | Bias threshold for LOW conviction block |
| `L2_ALIGNED_DEAD_PREM_HIGH_CONVICTION_SOFTEN_ENABLE` | `True` | L2 | #4 | Master toggle for dead-prem softening |
| `L2_ALIGNED_DEAD_PREM_HIGH_CONVICTION_MIN_BIAS` | `0.80` | L2 | #4 | Minimum bias to qualify for softening |

---

### 15g. Layer Funnel — Before vs After (Projected)

```
                    BEFORE (19-Mar actual)              AFTER (projected with all fixes)
                    ─────────────────────               ───────────────────────────────
Fusion Events       3047                                3047
L1 Trend Flips      54 (25 afternoon)                   54 (flips still happen, but...)
L1 Whipsaw Block    0                          →        ~12-15 entries blocked (Rec #2)
L2 BLOCK            2036 (67%)                 →        ~2243 (+207 from Rec #3)
  ├─ TRUST_NIFTY    986                                 986
  ├─ DIVERGENT      691                                 691
  ├─ DEAD_PREM      311                        →        ~220 (91 softened by Rec #4)
  ├─ NEW: LOW_BIAS  0                          →        ~207 (Rec #3)
  └─ other          48                                  48
L2 ALLOW            1010 (33%)                 →        ~712 (-207 Rec#3, +91 Rec#4 = -116 net)
L3 skip             609                                 ~490 (fewer LOW entries to reject)
L3 pass             403                                 ~222 (cleaner signal quality)
L4 BLOCK (bp<0.50)  85 at old bp=0.35          →        ~120 at new bp=0.50 (Rec #1)
TRADES ENTERED      25                         →        ~8-12 (higher quality, no whipsaw)
```

---

### 15h. Log Signatures for Monitoring

After deployment, watch for these new log lines to confirm the fixes are active:

| Log Pattern | Meaning | Expected Frequency |
|-------------|---------|-------------------|
| `L1_WHIPSAW_COOLDOWN.TRIGGERED user=X flips=3 in 900s window` | Whipsaw threshold reached, 5-min block starts | 2–5 times per choppy session |
| `LATCH_FLIP.whipsaw_block user=X trend=Y` | LATCH_FLIP entry suppressed (exit still fires) | Every flip during active cooldown |
| `STABLE_RETRY.whipsaw_block user=X trend=Y (Xs remaining)` | STABLE_RETRY entry blocked | Every 30s during cooldown (throttled) |
| `L2_CONCURRENCE.ALIGNED_DEAD_PREM_SOFTENED ⚠️` | Dead-prem block downgraded for HIGH conviction | ~91 per session |
| `L2_CONCURRENCE.ALIGNED_LOW_BIAS_BLOCK ❌` | LOW conviction weak-bias blocked at L2 | ~207 per session |

**Absence of these logs** after deployment means the features are either disabled or the market conditions didn't trigger them. Check `TrendConfig` values via the config dump log at startup.

---

### 15i. FAQ — 19-Mar Fixes

**Q: Why 3 flips as the threshold, not 2 or 5?**

A: 2 flips would be too aggressive — a single pullback-and-resume creates 2 flips and is normal
market behavior. 5 flips would be too late — by the 5th flip in 15 minutes, 4 whipsaw trades
have already fired. 3 flips in 15 minutes is the sweet spot: it catches the pattern after the
2nd whipsaw trade but before the 3rd, saving 5–7 trades per choppy session.

**Q: Why 15-minute window and 5-minute cooldown?**

A: The 15-minute window (900s) covers ~4 trend cycles at the observed 3.5-min average flip
duration. The 5-minute cooldown (300s) is long enough to skip 1–2 flip cycles but short enough
to re-enter if the market settles into a genuine trend. Each new flip during the window
re-triggers the cooldown (sliding window), so in sustained chop the block persists.

**Q: Does the whipsaw cooldown affect exits of existing positions?**

A: **No.** This is a critical design decision. `close_open_positions()` runs unconditionally
at line ~6654 for all counter-trend positions. The whipsaw check only gates `can_flip_by_event`
(line ~6341) and `_whip_blocked` (line ~7244), which control NEW entry placement. Existing
losers are still closed promptly — you just don't open a new one into the chop.

**Q: What if I want to disable whipsaw cooldown temporarily?**

A: Set `L1_WHIPSAW_COOLDOWN_ENABLE = False` in `TrendConfig`. Both enforcement points check
this flag. The tracking code still runs (harmlessly) but neither gate reads the timestamp.

**Q: Can Rec #3 and Rec #4 interact? (A signal goes from DEAD_PREM_SOFTENED to LOW_BIAS_BLOCK?)**

A: No. Rec #4 only fires for conviction=HIGH (softens DEAD_PREM for HIGH+bias>0.80). Rec #3
only fires for conviction=LOW (blocks weak bias). They target opposite ends of the conviction
spectrum and are in different `elif` branches — they cannot overlap.

**Q: What happens at exactly `break_prob = 0.50` for the conviction contradiction?**

A: The check is `< 0.50` (strict less-than). So `break_prob = 0.50` does **NOT** trigger the
block. At exactly 50%, the trade is allowed through.

---

*Documentation updated 21 March 2026. Generated from `sr_zone_engine.py`, `order_flow_engine.py`, and `fusion_signals.py` (~12,900 lines) — March 2026. Layer 5 (OFPE) section added.*
