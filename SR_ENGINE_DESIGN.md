# Zatamap SR Zone Engine — Design Document

> **Files covered:** `sr_zone_engine.py` · `fusion_signals.py`  
> **Last updated:** 12 March 2026 (GEX block + vol veto + S5 prem ROC + S7 delta trend + S5 bug fix 284bac8)  
> **Production branch:** `nishu_prod`  
> **Latest commit:** `284bac8` — fix S5_PREM_ROC tuple error in prem_hist display

---

## Table of Contents

1. [Overview — Why This Exists](#1-overview)
2. [Architecture — How Data Flows](#2-architecture)
3. [Token Universe — What We Track](#3-token-universe)
4. [SRLevel — Per-Strike Analytics](#4-srlevel)
5. [OI Velocity — Absolute Flow Rate](#5-oi-velocity)
6. [Gamma Exposure (GEX) — Dealer Hedging Pressure](#6-gamma-exposure-gex)
7. [GEX Regime & Gamma Neutral Level (GNL)](#7-gex-regime--gnl)
8. [SRZoneResult — The Full Snapshot](#8-srzoneresult)
9. [Zone Classification](#9-zone-classification)
10. [Wall-Breach Detection](#10-wall-breach-detection)
11. [Fusion Gate (S9_SR) — How SR Votes on Trades](#11-fusion-gate-s9_sr)
12. [Gate Decision Matrix — All Paths](#12-gate-decision-matrix)
13. [REVERSAL_WATCH — Pending Watch Mechanism](#13-reversal_watch)
14. [Logging — What You See in the Logs](#14-logging)
15. [Config Knobs](#15-config-knobs)
16. [Commit History (Key Fixes)](#16-commit-history)

---

## 1. Overview

**Problem this solves:** Option strategies can go wrong if you enter a CE (call) position when a massive PE OI wall is sitting right above spot, or if you enter a PE (put) when CE support OI is 8× normal. These OI walls represent institutional positioning that will actively fight the move.

**Design philosophy:**
- Read the **option chain as institutional positioning**, not as price-prediction
- CE OI at/below spot = call buyers defending that level = **support floor**
- PE OI at/above spot = put buyers capping that level = **resistance ceiling**
- OI alone is insufficient — add **velocity**, **acceleration**, **premium flow**, and **gamma** for full institutional-grade reading

**What it does NOT do:** SR Engine does not predict trend direction. It validates whether a trend-derived entry is safe given the current OI/gamma landscape. The trend engine (`NiftyTrendAnalyzer`) decides direction; SR Engine decides whether institutional positioning allows the entry.

---

## 2. Architecture

```
Market Data Feed
     │
     ├─── option_chain_main._process_option_tick(token, strike, opt_type, ltp, oi, vol, ts)
     │         └──► SRZoneEngine.on_option_tick()
     │                   └──► _StrikeState.history (rolling 15-min deque per token)
     │
     └─── option_chain_main._on_index_tick(spot, ts)
               └──► SRZoneEngine.on_spot_tick(spot, ts)
                         └──► compute_zone(spot, ts)
                                   │
                         ┌─────────┴──────────────────────────────┐
                         │         SRZoneResult                    │
                         │  nearest_resistance / nearest_support   │
                         │  zone / tight_range / breakout_prob     │
                         │  gex / net_gex / gex_regime / gnl       │
                         └─────────┬──────────────────────────────┘
                                   │
              ┌────────────────────┼──────────────────────────┐
              │                    │                          │
  fusion_signals            REVERSAL_WATCH             get_wall_structure_str()
  _validate_trend_           5s fast-path               logged at every
  reversal() [S9 gate]       breach check               scorecard + arm event
  get_gate_verdict()
  ("pass"|"warn"|"block")
```

**Throttling:** `compute_zone()` runs at most once every 0.5 seconds per spot tick. It does not run on every tick to avoid CPU waste.

---

## 3. Token Universe

The engine tracks exactly **10 tokens** — buyer-relevant strikes only:

```
spot = 24000

CE (support floor — calls a buyer would buy at/below spot):
  24000CE  23950CE  23900CE  23850CE  23800CE   ← ATM, ATM-50, ATM-100, ATM-150, ATM-200

PE (resistance ceiling — puts a buyer would buy at/above spot):
  24000PE  24050PE  24100PE  24150PE  24200PE   ← ATM, ATM+50, ATM+100, ATM+150, ATM+200
```

**Why buyer-centric?** In the Indian NIFTY weekly options market, CE buyers at/near ATM are directional bulls defending their positions. PE buyers above spot are directional bears entering positions. Their OI levels represent real hedging commitments with financial skin in the game.

**Why 10 tokens?** Covers ±200 pts from spot, which is the institutional "magnet zone" for intraday NIFTY moves. Beyond 200 pts is far OTM and has less real-time hedging impact.

**Dynamic following:** As spot moves, the window of 10 tracked tokens shifts to always follow ATM. Old tokens outside the window are pruned from the rolling cache.

---

## 4. SRLevel

Every tracked strike produces an `SRLevel` dataclass on each `compute_zone()` call:

```python
@dataclass
class SRLevel:
    strike:             int       # e.g. 24000
    opt_type:           str       # "CE" (support) or "PE" (resistance)
    oi:                 int       # current Open Interest (contracts)
    ltp:                float     # last traded premium price
    oi_trend_pct_min:   float     # OI rate of change (%/min)
                                  # +0.003 = building (+0.3%/min threshold)
                                  # -0.003 = unwinding (-0.3%/min threshold)
    prem_trend_pct_min: float     # premium rate of change (%/min)
    vol_trend_mult:     float     # recent volume vs 15-min baseline (1.0=normal, >1.5=elevated)
    distance_pts:       float     # |spot - strike| in pts
    is_oi_wall:         bool      # True if OI ≥ 1.8× average of same-type nearby strikes
    oi_wall_mult:       float     # actual ratio (e.g. 8.0 = 8× normal = massive wall)
    strength:           float     # composite score [0..1]
    oi_acceleration:    float     # 2nd derivative of OI (%/min²); +ve = wall growing faster
    prem_velocity:      float     # premium pts/sec (last 5 ticks); stale-OI proxy
    gex:                float     # Gamma Exposure units (+ = CE/support, − = PE/resistance)
    oi_velocity_abs:    float     # |oi_trend_pct_min × oi| = absolute lots/min
```

**Strength score** is a composite of OI size, OI trend direction, wall multiplier, volume surge, proximity to spot, and OI acceleration. Range 0–1. A level with `strength=0.44` (like today's 24000 CE wall) is a strong institutional floor.

**Ranking:** Candidates are sorted by `(-strength, distance_pts)` — strongest first, then closest. This ensures the nearest *meaningful* wall is picked, not just the nearest tiny OI level.

---

## 5. OI Velocity

### The problem with % rates alone

```
Strike 24000 CE:  OI=4,500,000  oi_trend=+0.03%/min  → 1,350 lots/min added
Strike 24050 PE:  OI=1,333,280  oi_trend=+0.77%/min  → 10,267 lots/min added
```

The 24050 PE level shows a **smaller % rate** but 7.6× more absolute institutional activity. Without velocity, both levels look equally important in the heat map.

### The fix

```python
oi_velocity_abs = round(abs(oi_trend_pct_min) * oi)   # lots/min
```

**What it means:**
- `vel=1,350 lots/m` = 1,350 NIFTY contracts being added per minute = moderate institutional flow  
- `vel=40,596 lots/m` = rapid short-covering or aggressive positioning = **major institutional move**

### Displayed in logs

```
RES1(nearest) @ 24000  OI= 5,287,815  oi_t=-0.77%/m ⬇️(vel=+40596lots/m)
SUP2(mid)     @ 23950  OI=   639,600  oi_t=+5.18%/m ⬆️(vel=+33131lots/m)
```

The arrows (⬆️⬇️➡️) show direction; the `vel=` shows absolute rate. A level with both a ⬆️ arrow AND high velocity = highly active institutional accumulation.

---

## 6. Gamma Exposure (GEX)

### Why OI alone misses the picture

OI counts contracts. GEX counts **how much spot hedging those contracts force on dealers**.

Every option has a **gamma** (Γ) value — how much the delta changes per 1-point move in spot. Dealers who sold those options must continuously buy/sell spot to stay delta-neutral. This forced hedging **physically moves the market**.

### The formula

$$\text{GEX} = \text{OI} \times \text{lot\_size}(50) \times \Gamma(S, K, T, \sigma) \times S^2 \div 10^{11}$$

Where:
- **OI** = contracts at that strike  
- **lot_size=50** = NIFTY multiplier per lot  
- **Γ** = Black-Scholes gamma per strike (computed fresh each cycle)  
- **S²** = spot² (converts gamma to rupee exposure)  
- **÷1e11** = normalizer (1 GEX-unit = ₹100 Billion = ₹10,000 Crore)

### Black-Scholes Gamma

$$\Gamma = \frac{N'(d_1)}{S \cdot \sigma \cdot \sqrt{T}}$$

where $d_1 = \frac{\ln(S/K) + (r + \sigma^2/2)T}{\sigma\sqrt{T}}$

Near-ATM options have the **highest gamma** — any 1pt NIFTY move forces maximum dealer hedging. Deep OTM options have near-zero gamma — minimal hedging pressure.

### IV Estimation (Brenner-Subrahmanyam)

Instead of inverting Black-Scholes (expensive), we approximate ATM IV from the live straddle:

$$\sigma \approx \frac{\text{CE\_LTP} + \text{PE\_LTP}}{S \cdot \sqrt{2T/\pi}}$$

This uses live ATM CE + PE premiums already in the rolling cache. Falls back to `σ=0.15` if ATM ticks haven't arrived yet.

### DTE Estimation

Automatically computes calendar days to the **next Thursday 15:30 IST** (NIFTY weekly expiry):

```python
days_to_thu = (3 - ist_now.weekday()) % 7   # 3 = Thursday
```

If it's already Thursday and past 15:30, rolls to next week. This makes GEX automatically correct for expiry-week gamma explosion.

### GEX Signs

- **CE GEX = +ve** → CE support strikes → dealers are **long gamma** → they buy spot when it falls → **buy-the-dip hedging** → price magnet / mean-revert
- **PE GEX = −ve** → PE resistance strikes → dealers are **short gamma** → they sell spot when it rises → **sell-the-rally hedging** → price repulsion / momentum

### Typical Ranges

| Scenario | Typical GEX per strike |
|---|---|
| Normal mid-week (7 DTE), large OI wall | 50–300 units |
| Near expiry (1-2 DTE), same OI wall | 500–3000 units |
| Today's 24000 CE (4.5M OI, 1.1 DTE) | ~740–862 units |

Gamma explodes as expiry approaches (Γ ∝ 1/√T). This is why expiry-week moves near key strikes are so violent.

---

## 7. GEX Regime & GNL

### Three Regimes

After computing GEX for all tracked strikes:

```python
net_gex = sum(CE_GEX_all_strikes) - sum(PE_GEX_all_strikes)
```

| Regime | Condition | What it means |
|---|---|---|
| `POSITIVE_GEX 🟢` | net_gex > 10% of total | CE > PE gamma. Dealers must **buy dips** to hedge. Spot mean-reverts. Strong support floors. |
| `NEGATIVE_GEX 🔴` | net_gex < −10% of total | PE > CE gamma. Dealers must **sell rallies** to hedge. Spot trends/accelerates. Strong resistance ceilings. |
| `NEUTRAL_GEX ⚪` | within ±10% threshold | Balanced. No dominant dealer direction. Chop/consolidation mode. |

**Why 10% threshold?** Avoids noise from minor imbalances. Today's session at 12:34 had `net_gex=-92.7` on total ~1613 units = **5.7% imbalance** → correctly labelled `NEUTRAL_GEX` despite a slight PE lean.

### Gamma Neutral Level (GNL)

The GNL is the **exact spot price where dealer net gamma flips from positive to negative** — the regime boundary:

$$\text{GNL} = \text{CE\_wall\_strike} + (\text{PE\_wall\_strike} - \text{CE\_wall\_strike}) \times \frac{\text{CE\_GEX\_total}}{\text{CE\_GEX\_total} + \text{PE\_GEX\_total}}$$

**Interpretation:**
- **Spot below GNL** → POSITIVE regime → dealers buy every dip → support is real → CE entries favored
- **Spot above GNL** → NEGATIVE regime → dealers sell every rally → resistance is real → PE entries favored
- **Spot crossing GNL** → regime flip → the market's personality changes

Example from today's 12:34:
```
gnl=24024, spot=24000 → spot is 24pts BELOW GNL
→ NEUTRAL_GEX (barely in positive territory)
→ Explained the 45-minute 23988–24020 chop before the final breakdown
```

### Gamma Wall

The strike with the **highest absolute GEX** is the Gamma Wall:
```
gamma_wall = argmax(|GEX_i|) across all tracked strikes
```

The Gamma Wall acts as a **price magnet** — dealers have maximum hedging obligation there. Spot gravitates toward it when in POSITIVE regime and gets violently repelled when that wall breaks.

**Marked in logs with 🎯:**
```
SUP1(nearest) @ 24000  gex=+740.8🎯  ← this is the gamma wall
```

---

## 8. SRZoneResult

Full snapshot produced on every `compute_zone()` call (throttled to 0.5s):

```python
SRZoneResult:
  # Levels
  nearest_resistance / mid_resistance / far_resistance   (PE OI above spot)
  nearest_support    / mid_support    / far_support       (CE OI below spot)

  # Zone
  zone              = NEAR_SUPPORT | NEAR_RESISTANCE | IN_ZONE | BREAKOUT_ABOVE | BREAKDOWN_BELOW
  dist_to_resistance = pts from spot to nearest res
  dist_to_support    = pts from spot to nearest sup
  is_tight_range     = True when both walls within 2×strike_step (100pts for NIFTY)

  # Probabilities
  breakout_probability  = [0..1] chance of breaking resistance
  breakdown_probability = [0..1] chance of breaking support
  breakout_confirmed    = True = already crossed + OI unwind + prem rising
  breakdown_confirmed   = True = already crossed + OI unwind + prem falling

  # PCR
  window_pcr       = total PE OI / total CE OI across tracked window
  window_pcr_trend = change vs previous snapshot

  # GEX
  net_gex           = CE_GEX_total - PE_GEX_total (GEX-units)
  gamma_wall_strike = strike with max |GEX|
  gamma_wall_gex    = GEX at gamma wall (+ve=CE, -ve=PE)
  gex_flip_zone     = GNL in spot pts
  gex_regime        = POSITIVE_GEX | NEGATIVE_GEX | NEUTRAL_GEX
  atm_iv_est        = estimated ATM IV (Brenner-Subrahmanyam)
  dte_days          = calendar days to next Thursday expiry

  # Directional bias (for majority-vote in fusion)
  directional_bias  = UPTREND | DOWNTREND | SIDEWAYS
```

---

## 9. Zone Classification

```
                         spot
                          │
  BREAKDOWN_BELOW         │   NEAR_SUPPORT    IN_ZONE    NEAR_RESISTANCE    BREAKOUT_ABOVE
                          │◄──── 25pts ────►│           │◄───── 25pts ─────►│
  ──────────[CE OI SUP]───────────────────────────────────────────[PE OI RES]──────────
            23800                          23975    24025                    24200
                                                 24000 (spot)
```

| Zone | Meaning | Typical CE verdict | Typical PE verdict |
|---|---|---|---|
| `IN_ZONE` | Spot between walls, safe distance | pass+1 | pass+1 |
| `NEAR_SUPPORT` | Within 25pts of support floor | pass+1 (floor close) | **check OI trend** |
| `NEAR_RESISTANCE` | Within 25pts of resistance ceiling | **check OI trend** | pass+1 |
| `BREAKOUT_ABOVE` | Spot has crossed resistance | pass+1 if confirmed | block |
| `BREAKDOWN_BELOW` | Spot has crossed support | block | pass+1 if confirmed |

**Tight-range special case:** When both walls are within 100pts of each other (2×strike_step), the normal zone logic is suspended. Entries within 50pts of either wall are blocked, entries need either breakout/breakdown confirmation OR adequate distance from both walls.

---

## 10. Wall-Breach Detection

`check_wall_breach(spot, "CE" | "PE")` — called every **5 seconds** while `REVERSAL_WATCH` is armed.

This answers: "Is the OI wall that was protecting our existing position starting to collapse?"

### For CE positions (protecting support floor):

```
Signal 1: spot dropped below support strike (dist < -12.5pts) → confidence=0.95
Signal 2: support OI rapid unwind (trend < -0.6%/min = 2× threshold) → confidence ∝ rate
Signal 3: OI unwind accelerating (oi_accel < -0.001 %/m²) → +0.10 confidence
Signal 4: CE premium falling fast (prem_vel < -0.08 pts/s) → +0.15 confidence

Breach fires when: confidence >= 0.60
```

### For PE positions (protecting resistance ceiling):

Mirror logic — resistance OI unwind, spot crossing above resistance, PE premium collapsing.

### Output in logs:
```
REVERSAL_WATCH.wall_check user=<user_id> side=CE conf=0.00
  👁 WALL_WATCH CE_SUPPORT@24000 conf=0.18
  oi_trend=+0.03%/min prem_vel=+0.065pts/s spot_vs_sup=+11pts
  gex=+862.8 oi_vel=1350lots/m wall_holding
```

When `conf >= 0.60`: `🔴 WALL_BREACH` fires → wall collapse confirmed → system promotes to trade action immediately, bypassing the normal 20s recheck wait.

---

## 11. Fusion Gate (S9_SR)

`get_gate_verdict(spot, direction)` is called as **Gate S9** in the `REVERSAL GATE SCORECARD`.

It returns one of:
- `("pass", reason, +1)` → adds 1 to the scorecard score
- `("warn", reason, 0)` → neutral (no score change, no block)
- `("block", reason, 0)` → **hard veto** — can override other gate passes to force exit_only

### Gates in Fusion (K-of-4 Scoring + Hard Pre-Gates)

```
── Pre-gates (hard skip — bypass entire scorecard) ────────────────────────────
  VOL_DEAD_MARKET_VETO  vol_spike_mult < 0.70  →  SKIP  (no market conviction)
  S10_GEX_BLOCK         POSITIVE_GEX + CE entry  OR  NEGATIVE_GEX + PE entry  →  SKIP
  PEAK_DRAWDOWN_GUARD   peak_pnl drawdown >= 10%  →  BLOCK new entries

── Scoring gates (K-of-N = need 1 of 4 to execute) ───────────────────────────
  Score = S3_OI_score + S5_PREM_ROC_score + S7_GREEKS_score + S9_SR_score

Execute if: score >= 1  (of 4 active gates)
Skip if:    0 < score < 1  (ambiguous — arm REVERSAL_WATCH)
Close if:   score <= 0  (all gates failed — exit existing position)
```

**Hard VETOs bypass score** — a `block` from S9_SR or any pre-gate forces skip/exit_only regardless of scorecard.

| Gate | Status | What it checks |
|---|---|---|
| VOL_DEAD_MARKET_VETO | ✅ **ENABLED** (pre-gate, hard skip) | `vol_spike_mult < 0.70` → volume below 70% of EWMA baseline → skip |
| S3_OI | ✅ **ENABLED** (scoring) | OI flow direction: PE buildup=DOWNTREND, CE buildup=UPTREND |
| S4_VOL | ❌ DISABLED (advisory phase) | Volume confirmation — dead config, never wired |
| S5_PREM_ROC | ✅ **ENABLED** (scoring) | Premium ROC ≥ 0.02%/min confirms option momentum in entry direction |
| S6_PCR_ROC | ❌ DISABLED (advisory phase) | PCR rate of change — dead config |
| S7_GREEKS | ✅ **ENABLED** (scoring + delta-veto) | Delta sign + magnitude + IV ratio + **delta trend slope** (last N snapshots) |
| S9_SR | ✅ **ENABLED** (scoring + hard veto) | SR zone + OI wall + GEX regime |
| S10_GEX_BLOCK | ✅ **ENABLED** (symmetric hard skip) | POSITIVE_GEX blocks CE entry; NEGATIVE_GEX blocks PE entry |
| PEAK_DRAWDOWN_GUARD | ✅ **ENABLED** (portfolio guard) | Block new entries when peak PnL drawdown ≥ 10% (configurable) |

### VOL_DEAD_MARKET_VETO (pre-gate, added 84899b5)

Fires **before any scorecard** — if volume is dead, all other signals are noise.

```python
vol_spike_mult = current_tick_vol / ewma_vol_baseline
if vol_spike_mult < 0.70:   # below 70% of rolling EWMA baseline
    → SKIP entire scorecard  (log: VOL_DEAD_MARKET_VETO 🛑)
```

Log examples:
```
VOL_PRE_CHECK ✅ vol_mult=5.63 >= threshold=0.70 [NIFTY2631723700PE] — volume=HIGH (market has conviction)

VOL_DEAD_MARKET_VETO 🛑 vol_mult=0.60 < threshold=0.70 [NIFTY2631723700CE]
  — market is DEAD (no volume conviction), skipping flip.
  WHY: vol_mult < 0.70 means current tick volume is below 70% of its EWMA baseline.
```

### S5_PREM_ROC (scoring gate, added 84899b5)

Confirms the option premium is **accelerating in the entry direction** before awarding a score.

```python
prem_roc = fe.premium_roc_pct_per_min(entry_tok)   # fraction/min, e.g. 0.036 = 3.6%/min
if prem_roc >= TREND_REVERSAL_VAL_PREM_ROC_MIN (0.02):  → PASS +1  (premium building)
elif prem_roc >= 0:                                       → WARN     (positive but weak, no score)
else:                                                     → FAIL     (premium falling)
```

Log examples:
```
S5_PREM_ROC ✅ prem_roc=+0.0363%/min >= 0.0200 [NIFTY2631723700CE]
  last3_prem=[281.5, 282.3, 283.1] (↑rising) — premium ACCELERATING in entry direction (score +1)

S5_PREM_ROC ⚠️ prem_roc=+0.0105%/min (positive but < 0.0200) [NIFTY2631723700PE]
  last3_prem=[310.2, 310.4, 310.5] — weak/stalling premium, no score

S5_PREM_ROC ❌ prem_roc=-0.0041%/min < 0 [NIFTY2631723700PE]
  last3_prem=[308.2, 307.9, 307.5] (↓falling) — premium FALLING while attempting entry
  WHY: negative ROC = option value decaying, no entry momentum
```

**Bug fixed in 284bac8:** `prem_hist` is a `deque` of `(timestamp, price)` tuples. The display code originally iterated raw tuples and called `round(v, 2)` on each → `TypeError: type tuple doesn't define __round__ method`. Fix: `[round(p, 2) for _, p in prem_hist[-3:]]`.

### S7_GREEKS — Delta Trend Sub-check (added 60263c8)

In addition to delta sign/magnitude and IV checks, S7 now validates the **direction of delta change** over the last N snapshots:

```python
N = TREND_REVERSAL_VAL_GREEKS_DELTA_TREND_N   # default 4
delta_vals = [d for _, d, _, _, _ in g_hist]
delta_slope = delta_vals[-N:][-1] - delta_vals[-N:][0]   # last minus first

CE (trend_up):   need slope >= -0.005  (delta growing or flat, not shrinking)
PE (trend_dn):   need slope <= +0.005  (delta declining or flat, not rising)
```

**Why this matters:** In a POSITIVE_GEX gamma-pinned market, delta can be the correct sign (e.g. −0.46 for PE) but the slope is flat/rising because gamma pinning suppresses the move. If delta is being squeezed back toward zero while the trend engine says "DOWN", S7 now catches it and fails the gate. Gamma-pin detection via option delta slope.

Log examples:
```
S7_GREEKS ✅ delta=-0.462 (dir=OK, |delta|>=0.25) iv=0.2297 iv_mean=0.2288 (ratio=1.00>=0.9)
  vega=11.18 [NIFTY2631723700PE]
  delta_history=[-0.461, -0.463, -0.463, -0.462]
  DELTA TREND: ✅ declining(-0.001) slope=-0.001 | score +1

S7_GREEKS ❌ delta=-0.402 DELTA TREND: ❌ SHRINKING(+0.048)
  delta_history=[-0.450, -0.430, -0.420, -0.402]
  WHY: delta moving AWAY from direction (slope=+0.048 > thr=-0.005) — gamma pin suppressing move
```

### S10_GEX_BLOCK (symmetric hard skip, added 46f8def / made symmetric in 84899b5)

Blocks entries that **fight the dominant dealer gamma hedging regime**:

```
POSITIVE_GEX  (net_gex >= +100)  AND  CE entry (trend_up)   → 🛑 HARD BLOCK
  WHY: Dealers net-long gamma → forced to buy-the-dip → mean-revert toward gamma wall
       CE premium erodes as spot is pinned to the wall below

NEGATIVE_GEX  (net_gex <= -100)  AND  PE entry (trend_dn)   → 🛑 HARD BLOCK
  WHY: Dealers net-short gamma → forced to sell-the-rally → resistance ceiling
       PE premium erodes as spot is pinned to the wall above
```

Log examples:
```
S10_GEX_BLOCK 🛑 POSITIVE_GEX CE blocked: net_gex=+460.3 >= min=100
  spot_vs_wall=+46pts wall=@23700(gex=+661.0) dte=0.10d iv=1.40
  WHY: Dealers net-long gamma → buy-dip hedging → 23700 is a price MAGNET.
       CE buyer fights the exact force that will pull price BACK to wall.
       Wait for GEX flip or gamma_wall OI unwind before CE entries.

S10_POSITIVE_GEX_CE=⏭️ DISABLED (TREND_REVERSAL_VAL_POSITIVE_GEX_BLOCK_CE=False)
  [shown when flag is False — logs the fact but does not block]
```

### PEAK_DRAWDOWN_GUARD (portfolio-level guard, added 46f8def)

Protects realised gains from being eroded by a string of bad new entries:

```python
_peak_pnl  = max(portfolio_pnl seen so far today)
_drawdown  = _peak_pnl - current_portfolio_pnl
pct        = _drawdown / _peak_pnl * 100

if _peak_pnl > 0 and pct >= TREND_REVERSAL_PEAK_DRAWDOWN_BLOCK_PCT (10.0):
    should_flip = False   # 🛑 block any new entry
```

Log example:
```
PEAK_DRAWDOWN_BLOCK 🛑 peak=+₹55,588(+26%) current=+₹49,500 drawdown=₹6,088(11.0%)
  Desired side=CE blocked — resume when drawdown falls below 10.0%
  WHY: drawdown >= limit. Protect realised peak gains from further erosion.
```

---

## 12. Gate Decision Matrix

### S9_SR — Complete Decision Tree

#### TIGHT RANGE (both walls within 100pts of spot)

`min_entry_dist_pts` = 50pts (half the 100pt tight-range width)

```
CE BUY (direction=UP):                                      commit
  ① breakout_confirmed?                                      —
        → PASS+1  "TIGHT_RANGE_BREAKOUT_CONFIRMED"

  ② dist_to_res < min_entry_dist_pts (50pts)?                db93236
        brk_prob >= 0.85?  → WARN+0  "TIGHT_RANGE_TOO_CLOSE_TO_RES_BRK_HIGH"
        else               → BLOCK   "TIGHT_RANGE_TOO_CLOSE_TO_RES"

  ③ brk_prob >= 0.5?                                         —
        → WARN+0  "TIGHT_RANGE_PRE_BREAKOUT"

  ④ res_oi_build > 0.5%/min                                  2b0d386
     AND gamma_wall == res
     AND gex_regime == NEGATIVE_GEX?
        → BLOCK   "TIGHT_RANGE_GAMMA_TRAP_CE"
          (active OI accumulation at resistance ceiling
           + dealer gamma is shorting every rally)

  ⑤ gex_regime == NEGATIVE_GEX                               4bbfaef
     AND gamma_wall == res
     AND net_gex < 0
     AND (GNL - spot) >= 15pts     ← spot ≥15pts below GNL
     AND brk_prob < 0.3?
        → BLOCK   "TIGHT_RANGE_CE_BELOW_GNL"
          (spot is structurally below the gamma neutral level;
           dealer sell-hedging is a structural headwind
           even when OI build rate is low)

  ⑥ else (adequate distance, no traps)
        → PASS+1  "TIGHT_RANGE_SAFE_DIST (resistance holding)"


PE BUY (direction=DOWN):                                     commit
  ① breakdown_confirmed?                                      —
        → PASS+1  "TIGHT_RANGE_BREAKDOWN_CONFIRMED"

  ② dist_to_sup < min_entry_dist_pts (50pts)?                 db93236
        bdn_prob >= 0.85?  → WARN+0  "TIGHT_RANGE_TOO_CLOSE_TO_SUP_BDN_HIGH"
        else               → BLOCK   "TIGHT_RANGE_TOO_CLOSE_TO_SUP"

  ③ dist_to_res < min_entry_dist_pts (50pts)?                 602144e
     (spot too close to resistance CEILING while going DOWN)
        bdn_prob >= 0.85?  → WARN+0  "TIGHT_RANGE_PE_AT_RES_BDN_HIGH"
        else               → BLOCK   "TIGHT_RANGE_PE_AT_RES"
          (entering PE when spot is already near the top of
           the tight range inverts the edge — cap eats premium)

  ④ bdn_prob >= 0.5?                                          —
        → WARN+0  "TIGHT_RANGE_PRE_BREAKDOWN"

  ⑤ sup_oi_build > 0.5%/min                                   2b0d386
     AND gamma_wall == sup
     AND gex_regime == POSITIVE_GEX?
        → BLOCK   "TIGHT_RANGE_GAMMA_TRAP_PE"
          (active OI accumulation at support floor
           + dealer gamma is buying every dip)

  ⑥ else
        → PASS+1  "TIGHT_RANGE_SAFE_DIST"
```

**Why ③ (PE_AT_RES) is needed:** in a tight range, entering a PE when the spot is within 50pts of the resistance ceiling (top of the box) puts the position immediately at risk — the spot only has to drift up slightly to be sitting at the very resistance that makes PE entries dangerous. It is the symmetric risk to CE entering near support.

**Why ⑤ (CE_BELOW_GNL) differs from ④ (GAMMA_TRAP_CE):** GAMMA_TRAP requires active OI accumulation (`res_oi_build > 0.5%/min`). CE_BELOW_GNL fires even when OI is flat, because the structural sell-hedging pressure from being below GNL in NEGATIVE_GEX is independent of the current build rate.

#### NORMAL RANGE

```
CE BUY (direction=UP):
  zone=BREAKOUT_ABOVE:
    confirmed? → PASS+1  "BREAKOUT_CONFIRMED"
    else       → WARN+0  "BREAKOUT_UNCONFIRMED"

  zone=NEAR_RESISTANCE:
    oi_trend >= +0.3%/min (building):
      wall=YES OR prem_vel < -0.05 OR [GAMMA_WALL🔴 + NEGATIVE_GEX]:
                                   → BLOCK  "NEAR_RES (hard block)"
      else                         → WARN+0 "NEAR_RES (soft warn)"
    oi_trend <= -0.3%/min (unwinding):
      gamma_wall=RES AND NEGATIVE_GEX AND net_gex < 0:
                                   → WARN+0 "RES_UNWIND_BUT_GAMMA_WALL" ← new fix
      else                         → PASS+1 "RES_UNWIND"
    else (neutral OI)              → WARN+0 "NEAR_RES oi_neutral"

  zone=IN_ZONE / NEAR_SUPPORT / BREAKDOWN_BELOW:
    gamma_wall=SUP AND POSITIVE_GEX → PASS+1 "[GAMMA_REINFORCED✅]"
    else                            → PASS+1 (clean zone)

PE BUY (direction=DOWN): mirror logic — NEAR_SUPPORT instead of NEAR_RESISTANCE

  zone=NEAR_SUPPORT:
    oi_trend >= +0.3%/min:
      wall=YES OR prem_vel < -0.05 OR [GAMMA_WALL🟢 + POSITIVE_GEX + net_gex > 0]:
                                   → BLOCK  "[GAMMA_WALL🟢]"
      else                         → WARN+0
    oi_trend <= -0.3%/min:
      gamma_wall=SUP AND POSITIVE_GEX AND net_gex > 0:
                                   → WARN+0 "SUP_UNWIND_BUT_GAMMA_WALL" ← new fix
      else                         → PASS+1 "SUP_UNWIND"
```

### The Four GEX Gate Tags

| Tag | When it fires | What it means |
|---|---|---|
| `[GAMMA_WALL🟢]` | PE BUY near support, gamma wall = support, POSITIVE/NEUTRAL GEX | Hard block — dealer buy-hedging will defend this floor |
| `[GAMMA_WALL🔴]` | CE BUY near resistance, gamma wall = resistance, NEGATIVE GEX | Hard block — dealer sell-hedging will cap this ceiling |
| `[GAMMA_REINFORCED✅]` | CE BUY, gamma wall at support, POSITIVE_GEX | Confidence booster — dealer gamma reinforces the support floor |
| `[GAMMA_WALL🔴_still_active]` | CE BUY, OI unwinding at resistance, but gamma wall still there | Warning — short-covering doesn't eliminate dealer gamma pressure |

---

## 13. REVERSAL_WATCH

When scorecard returns `skip` (score > 0 but < execute threshold), the system **arms a watch** instead of doing nothing.

```
Timeline:
  12:21:52  → LATCH_FLIP detected UPTREND→DOWNTREND
  12:21:53  → Scorecard: score=1/1 → SKIP (ambiguous)
  12:21:53  → REVERSAL_WATCH.armed  timeout=180s  recheck=20s  wall_protected=True
  12:21:55  → REVERSAL_WATCH.wall_check (every 5s fast-path)
  12:22:25  → REVERSAL_WATCH.wall_check (every 5s fast-path)
  ...
  12:24:53  → REVERSAL_WATCH expired after 180s → cleared
```

**Wall-protected mode:** When `wall_protected=True` (OI wall exists near spot), the 5-second fast-path runs `check_wall_breach()` every 5 seconds. If wall breach confidence >= 0.60, the watch promotes directly to trade execution without waiting for the full 20s recheck cycle.

**Recheck scoring:** Every 20 seconds, the full 3-gate scorecard is re-run with fresh OI/GEX data. If score improves to execute threshold, the trade fires.

**Timeout:** 180 seconds. If the reversal never confirms, the watch expires and clears. No trade.

### WATCH_REVERTED Defer (commit `f906cf7`)

When a REVERSAL_WATCH is active and the latch reverts to the opposite direction (e.g. DOWNTREND watch, then latch flips back to UPTREND) with weak momentum, the system previously cleared the watch immediately — this left existing positions unprotected for 20+ minutes.

**Defer condition (all must be true):**
```
_latch_now == _pw_trend      # latch flipped back to the same direction as watch
abs(roc) < 2.0               # roc is borderline (weak conviction)
not revert_deferred already  # only defer once per watch lifecycle
```

**When deferred:**
- Watch is kept alive with `revert_deferred=True` marker
- The revert is treated as a soft hesitation, not a real regime change
- On the next recheck the full scorecard runs again with fresh data
- If roc remains weak and score stays low, the watch eventually times out normally

**When NOT deferred (clears immediately):**
- `abs(roc) >= 2.0` → strong momentum confirms the revert is real
- `revert_deferred=True` already set → second revert always clears

**Example (14:07 incident):** DOWNTREND watch active, latch flip to UPTREND with roc=+0.3. Pre-fix: CE position left open in DOWNTREND for 20 minutes. Post-fix: defer fires → watch survives → scorecard re-run 20s later with roc=−1.8 → VETO → PE entry re-attempted correctly.

---

## 14. Logging

### SR_WALL_STRUCTURE (logged at every scorecard + watch arm + 5s breach check)

```
SR_WALL_STRUCTURE  spot=24011  zone=NEAR_SUPPORT  pcr=1.93(t=+0.000)
  tight=YES  brk_prob=0.19  bdn_prob=0.00  fresh=0s  tokens=10
  net_gex=-92.7 NEUTRAL_GEX⚪  gamma_wall=@24000(gex=+740.8)
  gnl=24024  dte=1.1d  iv=0.57
  ── RESISTANCE ──
  RES1 @ 24050  OI= 1,341,600  oi_t=+0.79%/m ⬆️(vel=+10,597lots/m)  prem_vel=+0.058  x0.5 wall  gex=-202.3  dist=spot+39pts  str=0.14
  ── SUPPORT ──
  SUP1 @ 24000  OI= 4,571,190  oi_t=+0.03%/m ➡️(vel=+1,371lots/m)   prem_vel=+0.065  x8.0 🟥 OI_WALL  gex=+740.8🎯  dist=spot+11pts  str=0.47
```

**How to read it:**
- `pcr=1.93` → 93% more PE than CE OI in the tracked window → strong put-heavy bias (support)
- `tight=YES` → spot is boxed between walls < 100pts apart → choppy, confirm before entry
- `brk_prob=0.19` → only 19% chance of breaking through 24050 resistance
- `x8.0 🟥 OI_WALL` → 24000 CE OI is 8× the average → massive institutional support floor
- `gex=+740.8🎯` → gamma wall here — dealers carry maximum hedging obligation at 24000
- `vel=+1,371lots/m` → 1,371 new lots per minute being added at 24000 support

### GEX_ANALYSIS (logged at scorecard + watch arm)

```
GEX_ANALYSIS  net_gex=-92.7 NEUTRAL_GEX⚪  gamma_wall=@24000(+740.8 CE/SUPPORT)
              gnl=24024  dte=1.1d  iv_est=0.57
  CE_gex=+915.7  PE_gex=698.1  ratio=CE×1.3
  spot(24000) is 24pts BELOW GNL(24024)  [1 unit=1e11₹]
  ⚪ Balanced GEX → no dominant dealer flow → gamma-neutral near @24000
```

**How to read it:**
- `net_gex=-92.7` on `total=1613.8` → only 5.7% imbalance → NEUTRAL (below 10% threshold)
- `gnl=24024` → GNL is 24pts above current spot → barely in positive territory
- `dte=1.1d` → expiry tomorrow → gamma is at maximum, every 1pt move forces huge dealer hedging
- `iv_est=0.57` → 57% implied volatility — expiry-week panic pricing
- `[1 unit=1e11₹]` → each GEX unit = ₹10,000 crore of dealer hedging exposure

### REVERSAL GATE SCORECARD

```
📋 REVERSAL GATE SCORECARD  user=<user_id>  trend=DOWNTREND
  S3_OI ⚠️  S3_OI_FLOW=⚠️ PE_NEUTRAL pe_conv=+0.000 ...
  S7_GREEKS ✅  delta=-0.474 iv=0.2401 iv_mean=0.2406 vega=12.39
  S9_SR=✅ RES_UNWIND@24000 oi_trend=-0.77%/min ...
  [CONSENSUS] majority_bias=DOWNTREND (2/3 engines agree)
  FINAL: score=2/3 active → EXECUTE

Score  : 2/1  [██]
Thresholds: execute>=1  close<=0  skip=rest
Verdict: ✅ EXECUTE  (close + flip entry)
```

**How to read:**
- `score=2/1` → 2 gates passed, threshold is 1 → EXECUTE
- `[CONSENSUS]` → majority vote from S3+S7+S9 directional biases
- `warm=['S9']` → S9 was warming up (recently restarted) — treated more leniently

---

## 15. Config Knobs

All tunable in `fusion_signals.py` `FusionConfig` / `TrendReversalConfig`:

| Parameter | Default | Effect |
|---|---|---|
| `TREND_REVERSAL_VALIDATE_MIN_SCORE` | 1 | Min gates needed to execute (K of N) |
| `TREND_REVERSAL_VALIDATE_CLOSE_MAX_SCORE` | 0 | Score at or below = close existing position |
| `TREND_REVERSAL_VALIDATE_TOTAL_GATES` | 4 | K-of-N denominator (S3 + S5 + S7 + S9 = 4 scoring gates) |
| `TREND_REVERSAL_EXECUTE_CONFIRM_COUNT` | 1 | Consecutive execute scores before firing |
| `TREND_REVERSAL_CLOSE_CONFIRM_COUNT` | 3 | Consecutive close scores before exiting |
| `TREND_REVERSAL_MIN_HOLD_SEC` | 300 | 5 min protection after open/flip before any close |
| `TREND_REVERSAL_WATCH_TIMEOUT_SEC` | 180 | Watch expires after this many seconds |
| `TREND_REVERSAL_WATCH_RECHECK_SEC` | 20 | Watch rechecks every N seconds |
| `TREND_REVERSAL_PEAK_DRAWDOWN_BLOCK_PCT` | **10.0** ✅ ENABLED | Block new entries when drawdown from peak PnL ≥ N%. Set 0.0 to disable. |
| `TREND_REVERSAL_VAL_VOL_LOW_VETO_MAX` | **0.70** ✅ ENABLED | VOL_DEAD_MARKET veto threshold — vol_spike_mult below this = skip all gates |
| `TREND_REVERSAL_VAL_PREM_ROC_MIN` | **0.02** ✅ ENABLED | S5 gate: min premium ROC (%/min as fraction, e.g. 0.02 = 2%/min) to award score |
| `TREND_REVERSAL_VAL_POSITIVE_GEX_BLOCK_CE` | **True** ✅ ENABLED | S10: symmetric GEX block (True = POSITIVE_GEX blocks CE, NEGATIVE_GEX blocks PE) |
| `TREND_REVERSAL_VAL_POSITIVE_GEX_MIN_NET_GEX` | 100.0 | S10: minimum \|net_gex\| to trigger block (filters low-conviction GEX noise) |
| `TREND_REVERSAL_VAL_GREEKS_DELTA_TREND_N` | 4 | S7: look-back window for delta slope check (last N delta snapshots) |

All SR engine thresholds are in `sr_zone_engine.py` constants:

| Constant | Default | Effect |
|---|---|---|
| `_DEFAULT_OI_BUILD_THRESH` | 0.003 (%/min) | OI trend above this = actively building |
| `_DEFAULT_OI_UNWIND_THRESH` | -0.003 (%/min) | OI trend below this = actively unwinding |
| `_DEFAULT_NEAR_ZONE_MULT` | 0.5 | Near zone = 0.5 × strike_step = 25pts |
| `_DEFAULT_STRONG_OI_MULT` | 1.8 | Wall = strike OI ≥ 1.8× adjacent average |
| `_DEFAULT_LOT_SIZE` | 50 | NIFTY lot multiplier for GEX |
| `_DEFAULT_RISK_FREE` | 0.065 | India risk-free rate for Black-Scholes |

---

## 16. Commit History

### Foundation commits

| Commit | Change | Reason |
|---|---|---|
| `6b572ba` | S9_CONFLICT returns `skip` instead of `veto` when OI wall agrees with existing position | CE position was closed incorrectly when 24000 support was holding |
| `7c98ecb` | Added `check_wall_breach()` + REVERSAL_WATCH 5s fast-path | SR engine needed to speak immediately when wall collapses, not wait for trend reconfirmation |
| `250fea0` | Added `get_wall_structure_str()` + wall_snapshot logging at scorecard/arm/5s | Needed visibility into which walls are nearest/mid/far and how strong they are |
| `f441184` | Fixed `self.near_zone_mult` → `self.near_zone_pts` | Live AttributeError crash in `check_wall_breach()` |
| `6954473` | Added OI velocity (`oi_velocity_abs`) + full GEX implementation (Black-Scholes gamma, DTE auto-detection, ATM IV estimation, GNL, regime classification) | Institutional-grade reading: OI% alone misses absolute flow; GEX reveals dealer hedging pressure that OI doesn't capture |
| `d86cf07` | Fixed `sup` referenced before assignment in `get_gate_verdict()` | `_gamma_wall_is_sup/res` were computed before `sup/res` were assigned — caused S9_SR to error on every scorecard since `6954473` |
| `cf23b75` | `RES_UNWIND` and `SUP_UNWIND` now return `warn` (not `pass`) when gamma wall is still active | Short-covering unwinds OI but dealer gamma exposure persists — previously gave a false confidence bonus |
| `e8aaa94` | GEX-enhanced breakout/breakdown probability model | `brk_prob`/`bdn_prob` now factor in net_gex, regime, and gamma wall alignment for higher-confidence estimates |

### 11 March 2026 — live incident fixes

| Commit | Fix | Root Cause Incident | Files |
|---|---|---|---|
| `db93236` | `TIGHT_RANGE_TOO_CLOSE_TO_RES/SUP`: downgrade hard block to warn when `brk/bdn_prob ≥ 0.85` | 13:41 PE miss — `bdn_prob=0.91` hard-blocked an entry that had high breakdown confirmation. K-of-N never ran. | `sr_zone_engine.py` |
| `2b0d386` | Added `TIGHT_RANGE_GAMMA_TRAP_CE` and `TIGHT_RANGE_GAMMA_TRAP_PE` | 13:48 CE entered into a gamma trap: `res_oi=+0.84%/min`, `gamma_wall==res`, `NEGATIVE_GEX` — all three dealer-short signals present but no gate blocked entry | `sr_zone_engine.py` |
| `f906cf7` | REVERSAL_WATCH defer on `WATCH_REVERTED` when latch is hostile and `abs(roc) < 2.0` | 14:07 watch cleared on `roc=+0.3` flip, leaving a CE position open for 20+ minutes in DOWNTREND with no score protection | `fusion_signals.py` |
| `602144e` | `TIGHT_RANGE_PE_AT_RES`: block/warn PE entry when `dist_to_res < min_entry_dist_pts` | 14:27 PE entered with spot only 2pts below resistance ceiling in a tight range — symmetric risk to CE near support was unguarded | `sr_zone_engine.py` |
| `4bbfaef` | `TIGHT_RANGE_CE_BELOW_GNL`: block CE entry when `spot ≥ 15pts below GNL` in `NEGATIVE_GEX + gamma_wall==res + net_gex<0 + brk_prob<0.3` | 15:06 CE entered with spot 20pts below GNL. `GAMMA_TRAP_CE` missed it because `res_oi=0.04%/min < 0.5` threshold. Structural sell-hedge is independent of current OI build rate. | `sr_zone_engine.py` |
| `ea1c012` | Time gate: block all new TR-Latch `process_order` calls at/after 15:15 IST | New order 15 minutes before market close (15:30) carries disproportionate risk with no time to manage. Exits/TSL continue normally. | `fusion_signals.py` |

### 12 March 2026 — Gamma-pin guard (POSITIVE_GEX losses postmortem)

**Root cause of 12 March loss:** Portfolio peaked at +₹55,588 (+26%) then eroded to +₹14,826 (+7%) via 3 CE entries in a POSITIVE_GEX gamma-pinned market. `net_gex = +460 to +786`, `gamma_wall = @23700` all day. S10 was disabled and S5/vol-veto config keys existed but were not wired. Every CE entry at 23700–23750 fought the mean-reversion pin — dealers were buying every dip back to 23700.

| Commit | Change | Reason |
|---|---|---|
| `46f8def` | S10 `POSITIVE_GEX` CE-block (disabled by default) + TSL peak PnL tracking infrastructure + `TREND_REVERSAL_PEAK_DRAWDOWN_BLOCK_PCT` config key (default `0.0` = off) | First guard: stop CE entries when dealers are actively buy-hedging every dip. Peak-PnL tracker wired to track high-water mark. |
| `84899b5` | Vol dead-market veto wired as real pre-gate (`vol_spike_mult < 0.70 → skip`) + S5 `PREM_ROC` wired as real scoring gate + S10 made symmetric (NEGATIVE_GEX also blocks PE) + **enabled defaults**: `PEAK_DRAWDOWN_BLOCK_PCT=10.0`, `POSITIVE_GEX_BLOCK_CE=True` + K-of-N denominator increased 3 → 4 (S3+S5+S7+S9) | S5 and vol-veto had config keys but were never wired into the scorecard path. S10 was CE-only; made symmetric because NEGATIVE_GEX dealers sell every rally — same problem for PE entries. |
| `60263c8` | Delta trend direction sub-check in S7 (last-N delta slope: CE slope ≥ −0.005, PE slope ≤ +0.005) + rich diagnostic logging for all new gates flowing into `trade.fusion_events.gate_details` (VOL_PRE_CHECK, S5_PREM_ROC with `last3_prem`, S7 `delta_history` + slope, S10 GEX_BLOCK with full dealer-mechanics WHY, PEAK_DRAWDOWN_BLOCK) | Gamma-pinned markets: delta sign can be correct but slope is flat/reversed because gamma suppression prevents the directional move. Need slope, not just sign. Rich gate_details logs enable DB postmortem queries. |
| `284bac8` | **Bug fix** S5_PREM_ROC `TypeError: type tuple doesn't define __round__ method` — `prem_hist` stores `(timestamp, price)` tuples; `round(v, 2)` called on whole tuple instead of price component | Introduced in 60263c8 when `last3_prem=[]` display was added. Fix: `[round(p, 2) for _, p in prem_hist[-3:]]` extracts price from tuple. |

**12 March DB Postmortem Findings** (31 trend_reversal events in `trade.fusion_events`):

- `11:53:43` — last CE entry with **old code**: `S10=⏭️ DISABLED`, entered @spot=23708 into POSITIVE_GEX (+460). This was the bad entry — spot was 8pts above gamma wall @23700.
- `11:57:33` — **first new-code event**: VOL_PRE_CHECK ✅, S5_PREM_ROC ✅, S7 with delta_history. Result: SKIP (S9 wall too close).
- `12:06:27` — S5 working correctly: `prem_roc=+0.0105%/min (positive but < 0.0200)` — weak momentum, no score.
- `12:17:11` — S5 tuple bug hit: `err=type tuple doesn't define __round__ method` → fixed in 284bac8.
- `12:20:13` — VOL veto correctly fired: `vol_mult=0.60 < threshold=0.70 — market is DEAD`.
- **All post-11:57 events correctly SKIPPED** — spot trapped in 23700–23750 (50pt tight range), S9 blocking because OI wall @23700 (OI=3.7M, gex=+660🎯) was only 24–31pts below spot. System behavior was **correct**, not weird.

---

## Summary: The Design Philosophy

```
Trend Engine    → "NIFTY is trending DOWN"              (NiftyTrendAnalyzer)
         │
         ▼
VOL_DEAD_MARKET_VETO → vol_spike_mult < 0.70? → SKIP   ✅ ENABLED (added 84899b5)
  (below 70% of EWMA volume baseline → no conviction → all signals are noise)
         │
         ▼
S10_GEX_BLOCK   → POSITIVE_GEX + CE?  OR  NEGATIVE_GEX + PE?  → SKIP  ✅ ENABLED (added 84899b5)
  (dealer gamma is actively fighting the entry direction; mean-reversion pin)
         │
         ▼
PEAK_DRAWDOWN_GUARD → peak drawdown ≥ 10%? → BLOCK     ✅ ENABLED (added 46f8def)
  (protect realised peak gains from being eroded by a bad new entry)
         │
         ▼
OI Flow Gate (S3) → "PE OI is building"                ✅ ENABLED (scoring)
         │
         ▼
Prem ROC Gate (S5) → premium ROC ≥ 0.02%/min?         ✅ ENABLED (scoring, added 84899b5)
  (option premium accelerating in entry direction)
         │
         ▼
Greeks Gate (S7)  → "Delta=-0.47, IV healthy,          ✅ ENABLED (scoring + delta-veto)
         │           delta TREND declining (slope ≤ +0.005)"  (delta slope added 60263c8)
         ▼
SR Gate (S9)      → "23700 CE wall ×2.0, POSITIVE_GEX"  ✅ ENABLED (scoring + hard veto)
         │              Are we too close to a wall?
         │              Is the wall growing or shrinking?
         │              Are dealers hedging FOR or AGAINST us?
         ▼
Score K-of-4      → Execute / Skip / Close               (need ≥1 of 4 scoring gates)
         │
         ▼
REVERSAL_WATCH    → Re-evaluate every 20s if ambiguous   (anti-ping-pong)
         │         Wall-breach fast-path every 5s
         ▼
Time Gate         → Block new entries at/after 15:15 IST (market closes 15:30)
         │         Exits and TSL are never blocked
         ▼
Trade Execution   → Close counter-trend + Open new-side  (PROCESS_ORDER)
```

**What flows into DB (`trade.fusion_events.gate_details`):**

Every gate log line is captured in `_val_details: list[str]` and persisted as a JSONB array column. All new gate details — VOL_PRE_CHECK, S5_PREM_ROC with `last3_prem`, S7_GREEKS with `delta_history` and slope, S10_GEX_BLOCK with dealer-mechanics WHY, PEAK_DRAWDOWN_BLOCK — are queryable for postmortem analysis:

```sql
SELECT event_ts AT TIME ZONE 'Asia/Kolkata', tag, action, score, spot, gate_details
FROM trade.fusion_events
WHERE user_id = '<user_id>' AND event_type = 'trend_reversal'
ORDER BY event_ts;
```

The SR engine does not add noise — it adds **structural context**. A trend reversal in a vacuum could be a false signal. A trend reversal where the OI wall that was defending the old direction is now **unwinding at 40,000 lots/min** with **dealer gamma turning negative** is a high-confidence institutional confirmation.
