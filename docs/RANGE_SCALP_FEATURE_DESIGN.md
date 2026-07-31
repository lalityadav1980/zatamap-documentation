# Range Scalp Mode — Complete Feature Design Document

**Version:** 2.0
**Date:** 2026-03-27
**Fixes:** W14, W14b, W14c-v2, W14d, W14e
**Branch:** nishu_prod
**Kill Switch:** `RANGE_SCALP_ENABLE = False` or `RANGE_SCALP_AUTO_REGIME = False`

---

## 1. Problem Statement

The existing system is designed for **trend-following** trades — it detects NIFTY momentum (UPTREND/DOWNTREND) and buys CE/PE accordingly. This works well in trending markets but **fails in range-bound/sideways markets** because:

- L3 (entry quality) blocks entries due to insufficient edge (±0.07 vs 0.08 threshold)
- L4 (SR zone) blocks entries because spot is in a "tight range" (50pts between S/R)
- Result: System sits idle while option premiums oscillate 4-8% within the range

**Today's example (27-Mar-2026):**
- NIFTY gap-down from 23306 to 23186 (-0.52%)
- After the initial drop, NIFTY ranged between ~22850-22950 for hours
- System detected DOWNTREND + BEARISH L2 = wanted PE
- But L4 blocked: "tight range 50pts" and "PE only 23pts above support"
- PE premiums moved 10-15% within the range — all missed

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    MARKET REGIME DETECTOR                     │
│              (Fix-W14b — runs every 30 seconds)              │
│                                                              │
│  Inputs: ATR, |slope|, flip_count, zone_width, price_range   │
│                                                              │
│  Output: TRENDING | RANGE_BOUND | SLOW_DRIFT | CHOPPY        │
│                                                              │
│  TRENDING    → Range Scalp OFF, normal trend-following ON     │
│  RANGE_BOUND → Range Scalp ON                                │
│  SLOW_DRIFT  → BOTH modes ON (trend + scalp coexist)         │
│  CHOPPY      → Reduce all activity                           │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    RANGE SCALP ENGINE                         │
│              (Fix-W14 — runs every tick cycle)                │
│                                                              │
│  Step 1: Is spot at an EDGE? (within 20pts of S/R wall)      │
│  Step 2: Check breach probability of the wall                │
│  Step 3: Determine side (PE/CE) based on L1 + wall position  │
│  Step 4: Execute via process_order() with RANGE_SCALP tag     │
│  Step 5: Full position size (process_order calculates qty)    │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              SCALP TSL LADDER (Fix-W14c-v2)                  │
│        (in manage_trailing_stop_loss — data-driven)          │
│                                                              │
│  Detects RANGE_SCALP tag → switches to scalp ladder           │
│  SL: 7% (data: median entry drawdown 2-5%, P75=7%)          │
│  Target: 15% (let runners run)                               │
│  Ladder starts at 4% profit (2% is just noise)               │
│  Room: 3.5% (covers median 5-6% retrace from PE/CE data)    │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Market Regime Detector (Fix-W14b)

### 3.1 Classification Logic

Every 30 seconds, the detector reads 5 signals and classifies the market:

| Signal | Source | How it's used |
|--------|--------|---------------|
| ATR (pts) | `spot_cache["nifty_atr"]["primary_atr_points"]` | Low ATR = range, High ATR = trending |
| \|slope\| (bps/min) | `spot_cache["nifty_trend"]["slope_bps_per_min"]` | Flat slope = sideways |
| Flip count (30min) | `_trend_flip_history` list | Many flips = choppy |
| Zone width (pts) | `sr_zone_engine.last_result.zone_width_pts` | Established range detection |
| Price range (pts) | `session_high - session_low` | Overall day range |

**Classification Priority (highest first):**

```
if flips >= 5 in 30min:
    regime = CHOPPY          # Too noisy for any strategy

elif ATR >= 50pts AND |slope| >= 3 bps/min AND flips <= 2:
    regime = TRENDING        # Clear directional move

elif ATR <= 35pts AND |slope| <= 2 bps/min AND zone <= 120pts:
    regime = RANGE_BOUND     # Spot oscillating between walls

else:
    regime = SLOW_DRIFT      # Gradual move — both strategies work
```

### 3.2 Regime Impact on Trading

| Regime | Range Scalp | Trend Following | Position Size |
|--------|-------------|-----------------|---------------|
| TRENDING | OFF | Normal | Full |
| RANGE_BOUND | ON | Still active but L3/L4 often blocks | Full |
| SLOW_DRIFT | ON | ON | Full |
| CHOPPY | OFF | Reduced (whipsaw cooldown) | Reduced |

### 3.3 Restart Recovery (Fix-W14d)

```
Restart
  │
  ├─► Tier 1: JSON state file (~/.zatamap_regime_state.json)
  │   Written every 30s with: regime, scalp_active, range_start_ts, trade_count
  │   If file age < 10 min → INSTANT restore (0 delay)
  │
  ├─► Tier 2: CSV bootstrap (NIFTY50_YYYY-MM-DD.csv)
  │   Read last 30 min of today's tick data
  │   Compute price_range + slope → classify regime
  │   Picks LARGEST file (avoids corrupt/duplicate small files)
  │   Delay: ~1-2 seconds (CSV read)
  │
  └─► Tier 3: Live detection
      Wait for first 30s evaluation cycle
      Delay: 30 seconds
```

---

## 4. Range Scalp Entry Logic (Fix-W14)

### 4.1 Entry Decision Flow

```
For every tick cycle:
  │
  ├─ Is RANGE_SCALP enabled? (auto-regime or manual)
  │   └─ No → skip
  │
  ├─ Is there an open position?
  │   └─ Yes → skip (only 1 position at a time)
  │
  ├─ Read SR Zone Data from sr_zone_engine.last_result:
  │   dist_to_resistance, dist_to_support, zone_width,
  │   res_breach_prob, sup_breach_prob, wall_state, conviction
  │
  ├─ QUALIFY: Is spot in a valid range?
  │   Range width: 40-120pts (too narrow = noise, too wide = trending)
  │   Range held: >= 5 minutes (established, not transient)
  │   Data fresh: SR zone age <= 30 seconds
  │   Trade limit: < 3 scalps in this range session
  │   Not in loss cooldown: >= 5 min since last scalp loss
  │
  ├─ EDGE CHECK: Is spot near a wall?
  │   Within 20pts of resistance → AT_RESISTANCE
  │   Within 20pts of support    → AT_SUPPORT
  │   More than 20pts from both  → MIDDLE ZONE → SKIP ❌
  │
  ├─ BREACH PROBABILITY CHECK:
  │   At resistance:
  │     bp <= 30% → wall holds → BUY PE (reversal)
  │     bp >= 50% + L1 UP → wall breaks → BUY CE (breakout)
  │   At support:
  │     bp <= 30% → wall holds → BUY CE (reversal)
  │     bp >= 50% + L1 DOWN → wall breaks → BUY PE (breakdown)
  │
  ├─ MACRO BIAS ADJUSTMENT:
  │   When L1 trend aligns with scalp direction:
  │     Breach threshold relaxed by +10% (30% → 40%)
  │
  └─ EXECUTE: process_order() with full investment_percentage
      (process_order calculates quantity from available funds)
```

### 4.2 The 6 Use Cases

#### Use Case 1: Reversal at Resistance (Wall Holds → Buy PE)

```
NIFTY range: 22850-22900

  22900 ═══ RESISTANCE ═══  breach_prob = 0.15 (LOW)
    ▲                        conviction = RESPECT
    │  Spot arrives at 22892 (8pts from resistance)
    │
    │  L4 says: breach_prob 0.15 ≤ 0.30 → wall holds
    │  Action: BUY PE
    │  Tag: RANGE_SCALP_PE_AT_RES
    │  Size: Full (process_order calculates from available funds)
    │
    │  PE premium bought at ₹255
    │  Spot reverses to 22860
    │  PE premium rises to ₹270 (+5.9%)
    │  Scalp ladder: peak 5.9% → lock at +2.5% (3.5% room)
    │  Retrace to ₹266 → still above lock
    │  Continues to ₹275 → peak 7.8% → lock at +4.5%
    │  Final exit at ₹266.48 (+4.5%)
    │
    ▼  Profit: +4.5%
  22850 ═══ SUPPORT ═══════
```

#### Use Case 2: Reversal at Support (Wall Holds → Buy CE)

```
NIFTY range: 22850-22900

  22900 ═══ RESISTANCE ═══
    │
    │  Spot drops to 22858 (8pts from support)
    │
    │  L4 says: breach_prob 0.20 ≤ 0.30 → support holds
    │  Action: BUY CE
    │  Tag: RANGE_SCALP_CE_AT_SUP
    │  Size: Full
    │
    │  CE premium bought at ₹310
    │  Spot bounces to 22890
    │  CE premium rises to ₹330 (+6.5%)
    │  Scalp ladder: peak 6.5% → lock at +2.5% (3.5% room)
    │  Retrace: CE drops to ₹322 → still above lock ₹317.75
    │  Exits when retrace hits lock → +2.5%
    │
    ▼  Profit: +2.5%
  22850 ═══ SUPPORT ═══════  breach_prob = 0.20 (LOW)
```

#### Use Case 3: Breakdown Through Support (Wall Breaks → Buy PE)

```
NIFTY range: 22850-22900, L1 = DOWNTREND

  22900 ═══ RESISTANCE ═══
    │
    │  Spot at 22855 (5pts from support)
    │
    │  L4 says: breach_prob 0.62 ≥ 0.50 → support breaking!
    │  L1 says: DOWNTREND → confirms breakdown direction
    │  Action: BUY PE (ride the breakdown)
    │  Tag: RANGE_SCALP_PE_BREAKDOWN
    │
    │  PE premium bought at ₹260
    ▼  Spot breaks 22850, drops to 22790
  22850 ═══ SUPPORT ═══════  breach_prob = 0.62 (HIGH)
    │  PE premium surges to ₹295 (+13.5%)
    │  Scalp ladder: peak 13.5% → lock at +9.0% (3.0% room)
    ▼  Exits at lock → +9.0%

  Profit: +9.0% (breakdown runner)
```

#### Use Case 4: Breakout Through Resistance (Wall Breaks → Buy CE)

```
NIFTY range: 22850-22900, L1 = UPTREND

  22900 ═══ RESISTANCE ═══  breach_prob = 0.55 (HIGH)
    ▲  Spot at 22895, resistance weakening
    │
    │  L4 says: breach_prob 0.55 ≥ 0.50 → resistance breaking!
    │  L1 says: UPTREND → confirms breakout direction
    │  Action: BUY CE (ride the breakout)
    │  Tag: RANGE_SCALP_CE_BREAKOUT
    │
    │  CE premium bought at ₹315
    │  Spot breaks 22900, rises to 22960
    │  CE premium surges to ₹355 (+12.7%)
    │  Scalp ladder: peak 12.7% → lock at +9.0%
    │
  22850 ═══ SUPPORT ═══════

  Profit: +9.0% (breakout runner)
```

#### Use Case 5: Macro-Biased Reversal (L1 Aligns → Relaxed Threshold)

```
NIFTY range: 22850-22900, L1 = DOWNTREND (macro bearish)

  22900 ═══ RESISTANCE ═══  breach_prob = 0.35
    ▲
    │  Spot at 22888 (12pts from resistance)
    │
    │  Normal: bp 0.35 > 0.30 threshold → SKIP
    │  BUT: L1 = DOWNTREND, wanting PE at resistance = MACRO ALIGNED
    │  Threshold relaxed: 0.30 + 0.10 = 0.40
    │  bp 0.35 ≤ 0.40 → PASS with relaxed threshold
    │
    │  Action: BUY PE (macro-supported reversal)
    │  Tag: RANGE_SCALP_PE_AT_RES_MACRO
    │  Size: Full
    │
    │  PE premium bought at ₹258
    │  Spot reverses to 22860
    │  PE premium rises to ₹276 (+7.0%)
    │  Scalp ladder: peak 7.0% → lock at +4.5% (3.5% room from 8% rung)
    │
    ▼  Profit: +4.5%
  22850 ═══ SUPPORT ═══════
```

#### Use Case 6: Slow Drift Continuation (New Ranges Form)

```
NIFTY slowly drifting down: macro = DOWNTREND

  Time 10:00  Range: 22950-23000
    │  At 23000 (resistance) → Buy PE → exits at +4.5%
    │  Spot drifts to 22940, breaks 22950 support
    ▼
  Time 11:00  Range: 22880-22940 (NEW range formed)
    │  At 22940 (new resistance) → Buy PE → exits at +4.5%
    │  Spot drifts to 22870, breaks 22880 support
    ▼
  Time 12:00  Range: 22830-22870 (NEWER range formed)
    │  At 22870 (new resistance) → Buy PE → exits at +6.5%
    ▼

  3 scalp trades riding the drift: +4.5% + 4.5% + 6.5% = +15.5% total
```

---

## 5. Scalp TSL Ladder (Fix-W14c-v2 — Data-Driven)

### 5.1 Data Analysis (27-Mar-2026)

Analyzed 22900PE (5,630 ticks) and 22800CE (28,338 ticks):

| Metric | 22900PE | 22800CE |
|--------|---------|---------|
| Median peak-to-trough retrace | **5.0%** | **6.7%** |
| P75 retrace | 7.8% | 8.3% |
| Median drawdown from entry | 2.1% | 4.7% |
| P90 drawdown from entry | 5.4% | 8.4% |
| Median max profit (10min) | 4.3% | 2.2% |

**Key insight:** Options retrace 5-7% routinely after reaching peak profit.
A ladder with 1.5% room would exit on normal noise. Need 3.5% room minimum.

### 5.2 How the Scalp Ladder is Detected

When `manage_trailing_stop_loss_15pct_only_with_pretty_log()` starts:

```python
entry_tag = get_order_cache(userid).get("entry_tag")  # e.g. "RANGE_SCALP_PE_AT_RES"

_is_range_scalp = entry_tag.startswith("RANGE_SCALP")
_is_scalp_from_cache = order_cache.get("range_scalp_mode", False)
_is_scalp = _is_range_scalp or _is_scalp_from_cache

if _is_scalp:
    _active_ladder = LADDER_SCALP        # data-driven scalp ladder
    LOSS_PCT_DEFAULT = 0.07              # 7% SL
    PROFIT_PCT_DEFAULT = 0.15            # 15% target
elif entry_is_expiry_day:
    _active_ladder = LADDER_EXPIRY       # 3% room
else:
    _active_ladder = LADDER_NORMAL       # 4% room
```

### 5.3 Ladder Comparison

**Normal Trend Ladder (4% room):**
```
Peak >=  8% → lock  4%  (4% room) ← First rung
Peak >= 10% → lock  6%  (4% room)
Peak >= 12% → lock  8%  (4% room)
Peak >= 15% → lock 11%  (4% room)
Peak >= 20% → lock 16%  (4% room)
Peak >= 30% → lock 26%  (4% room)

SL: -10% | Target: +30% | First rung: +8%
```

**Scalp Ladder (3.5% room — data-driven from PE/CE tick analysis):**
```
Peak >=  4% → lock  0.5%  (3.5% room) ← First rung at 4% (not 2%)
Peak >=  6% → lock  2.5%  (3.5% room)
Peak >=  8% → lock  4.5%  (3.5% room)
Peak >= 10% → lock  6.5%  (3.5% room)
Peak >= 12% → lock  9.0%  (3.0% room, tighter for runners)
Peak >= 15% → lock 12.0%  (3.0% room, protect the runner)

SL: -7% | Target: +15% | First rung: +4%
```

### 5.4 Why These Values

| Aspect | Normal (Trend) | Scalp (Range) | Why |
|--------|---------------|---------------|-----|
| First rung | +8% | +4% | 2% is noise (happens 97x in 1-min windows). 4% is a real directional move |
| Room | 4% | 3.5% | Covers median 5-6% retrace without getting stopped out of winners |
| Hard SL | -10% | -7% | 5% SL was inside normal entry drawdown (median 2-5%). 7% gives breathing room |
| Target | +30% | +15% | 10% was cutting winners short. 15% lets breakdown/breakout runners profit |
| Rungs | 10 | 6 | Focused on the 4-15% profit zone where scalps live |

### 5.5 Scalp Ladder Walk-Through Example

**PE bought at ₹255, scalp ladder active:**

```
Time   LTP     PnL%   Ladder State                    Action
─────  ──────  ─────  ──────────────────────────────  ──────────
09:30  ₹255    0.0%   Below all rungs                 Hold (SL at ₹237.15 = -7%)
09:31  ₹258    +1.2%  Below rung 1 (need +4%)         Hold — this is just noise
09:32  ₹253    -0.8%  Drawdown                        Hold — normal entry drawdown
09:33  ₹260    +2.0%  Still below rung 1              Hold — 2% is still noise
09:34  ₹265    +3.9%  Almost at rung 1                Hold — almost there
09:35  ₹266    +4.3%  ★ Hit rung 1! Peak=4.3%         LOCK at +0.5% (₹256.28)
09:36  ₹270    +5.9%  ★ Hit rung 2! Peak=5.9%         LOCK at +2.5% (₹261.38)
09:37  ₹264    +3.5%  Pullback (still above lock)     Hold — lock protects at ₹261.38
09:38  ₹262    +2.7%  Deeper pullback                 Hold — still above ₹261.38
09:39  ₹271    +6.3%  ★ Hit rung 2 again, peak=6.3%   LOCK stays at +2.5%
09:40  ₹276    +8.2%  ★ Hit rung 3! Peak=8.2%         LOCK at +4.5% (₹266.48)
09:41  ₹270    +5.9%  Retrace                         Hold — lock at ₹266.48
09:42  ₹267    +4.7%  Approaching lock                Hold
09:43  ₹266    +4.3%  Touches lock!                   ★ EXIT at ₹266.48 (+4.5%)

                       Net profit: +4.5% on full position
```

**Failed scalp example (direct SL hit):**

```
Time   LTP     PnL%   Ladder State                    Action
─────  ──────  ─────  ──────────────────────────────  ──────────
09:30  ₹255    0.0%   Below all rungs                 Hold (SL at ₹237.15 = -7%)
09:31  ₹253    -0.8%  In loss                         Hold — normal entry drawdown
09:32  ₹250    -2.0%  SL trigger zone approaching      SL trigger arm at -2% (dynamic)
09:33  ₹248    -2.7%  In trigger zone                  Waiting for recovery or SL
09:34  ₹252    -1.2%  Brief recovery — disarms trigger  SL trigger disarm
09:35  ₹246    -3.5%  Drops again                      SL trigger re-arms
09:36  ₹240    -5.9%  Deep loss                        Still within 7% SL
09:37  ₹237    -7.1%  ★ Hard SL hit!                   EXIT at ₹237 (-7.1%)

                       Net loss: -7.1% on full position
                       vs -10% on normal trade = saved 3%
```

---

## 6. Macro Trend Integration

### 6.1 How Macro Bias Affects Scalp Decisions

The L1 (NIFTY trend) provides a macro directional bias. The scalp engine uses this in one way:

**Breach Threshold Relaxation (+10%)**

When the macro trend AGREES with the scalp direction, the breach probability threshold is relaxed:

```
Normal:  breach_prob must be ≤ 0.30 for reversal trade
Aligned: breach_prob can be ≤ 0.40 for reversal trade (relaxed by 0.10)
```

| Macro | Scalp Direction | Aligned? | Threshold |
|-------|----------------|----------|-----------|
| DOWNTREND | PE at resistance | YES | 0.40 (relaxed) |
| DOWNTREND | CE at support | NO | 0.30 (strict) |
| UPTREND | CE at support | YES | 0.40 (relaxed) |
| UPTREND | PE at resistance | NO | 0.30 (strict) |
| SIDEWAYS | Any | NO | 0.30 (strict) |

**Position Size:** Always FULL — process_order calculates quantity from available funds.
No artificial size reduction. Risk is managed by the 7% SL and scalp ladder.

### 6.2 Macro + Range Interaction Matrix

```
┌──────────────┬──────────────────────────────────────────────┐
│              │             NIFTY POSITION                     │
│  MACRO TREND │  At Resistance       │  At Support            │
├──────────────┼──────────────────────┼────────────────────────┤
│              │  PE (reversal)       │  CE (reversal)         │
│  DOWNTREND   │  ★ BEST TRADE        │  Counter-macro         │
│              │  bp threshold: 0.40  │  bp threshold: 0.30    │
│              │  Tag: _MACRO         │  Tag: _AT_SUP          │
│              │  Size: FULL          │  Size: FULL            │
├──────────────┼──────────────────────┼────────────────────────┤
│              │  CE (reversal)       │  PE (reversal)         │
│  UPTREND     │  Counter-macro       │  ★ BEST TRADE          │
│              │  bp threshold: 0.30  │  bp threshold: 0.40    │
│              │  Tag: _AT_RES        │  Tag: _MACRO           │
│              │  Size: FULL          │  Size: FULL            │
├──────────────┼──────────────────────┼────────────────────────┤
│              │  PE (reversal)       │  CE (reversal)         │
│  SIDEWAYS    │  bp threshold: 0.30  │  bp threshold: 0.30    │
│              │  Tag: _AT_RES        │  Tag: _AT_SUP          │
│              │  Size: FULL          │  Size: FULL            │
└──────────────┴──────────────────────┴────────────────────────┘
```

---

## 7. Risk Management

### 7.1 Position-Level Safeguards

| Safeguard | Value | Purpose |
|-----------|-------|---------|
| Hard SL | -7% | Cut failed scalps (covers P75 entry drawdown) |
| Ladder start | +4% | Lock profit on real moves (ignores 2% noise) |
| Ladder room | 3.5% | Survive median 5-6% retrace without stopping out |
| Hard target | +15% | Let breakdown/breakout runners profit fully |
| L2 adverse exit | Active | If L2 reverses with HIGH conviction → cut early |

### 7.2 Session-Level Safeguards

| Safeguard | Value | Purpose |
|-----------|-------|---------|
| Max trades per range | 3 | Don't overtrade a single range |
| Loss cooldown | 5 min | Pause after a failed scalp |
| Range maturity | 5 min | Don't scalp a range that just formed |
| Middle zone block | >20pts from wall | NEVER enter in the middle |
| Range width limits | 40-120pts | Too narrow = noise, too wide = trending |
| Data freshness | SR age <= 30s | Don't act on stale zone data |
| Choppy regime block | >=5 flips/30min | System goes defensive in chaos |

### 7.3 Worst-Case Scenario

```
Max loss per scalp trade:   -7% on full position
Max scalps per range:       3 trades
Max loss per range session: 3 × (-7%) = -21% at risk

But: 5-min cooldown after each loss means max 3 losses = 15+ min
     In practice: range breaks after 1-2 losses → regime shifts → scalp OFF
     Also: L2 adverse exit may cut earlier than -7% SL
```

---

## 8. Entry Tag Reference

| Tag | Meaning | Macro | Direction |
|-----|---------|-------|-----------|
| `RANGE_SCALP_PE_AT_RES` | PE reversal at resistance | Not aligned | Reversal |
| `RANGE_SCALP_PE_AT_RES_MACRO` | PE reversal at resistance | DOWNTREND aligned | Reversal |
| `RANGE_SCALP_CE_AT_SUP` | CE reversal at support | Not aligned | Reversal |
| `RANGE_SCALP_CE_AT_SUP_MACRO` | CE reversal at support | UPTREND aligned | Reversal |
| `RANGE_SCALP_PE_BREAKDOWN` | PE on support break | DOWNTREND required | Breakout |
| `RANGE_SCALP_CE_BREAKOUT` | CE on resistance break | UPTREND required | Breakout |

---

## 9. Configuration Reference

### 9.1 Range Scalp Entry (fusion_signals.py)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `RANGE_SCALP_ENABLE` | True | Master switch |
| `RANGE_SCALP_AUTO_REGIME` | True | Auto-detect regime (overrides ENABLE) |
| `RANGE_SCALP_NEAR_WALL_DIST_PTS` | 20.0 | Max distance from wall to enter |
| `RANGE_SCALP_MIN_RANGE_WIDTH_PTS` | 40.0 | Minimum range width |
| `RANGE_SCALP_MAX_RANGE_WIDTH_PTS` | 120.0 | Maximum range width |
| `RANGE_SCALP_MIN_RANGE_DURATION_SEC` | 300.0 | Range must hold 5+ min |
| `RANGE_SCALP_REVERSAL_MAX_BREACH_PROB` | 0.30 | Max breach prob for reversal |
| `RANGE_SCALP_BREAKOUT_MIN_BREACH_PROB` | 0.50 | Min breach prob for breakout |
| `RANGE_SCALP_BREAKOUT_REQUIRE_L1_ALIGN` | True | L1 must agree for breakout |
| `RANGE_SCALP_MACRO_ALIGNED_BREACH_RELAX` | 0.10 | Relax threshold when aligned |
| `RANGE_SCALP_MAX_TRADES_PER_RANGE` | 3 | Max scalps per range |
| `RANGE_SCALP_COOLDOWN_AFTER_LOSS_SEC` | 300.0 | 5-min pause after loss |
| `RANGE_SCALP_LOSS_EXIT_PCT` | 0.07 | 7% hard SL |
| `RANGE_SCALP_PROFIT_TARGET_PCT` | 0.15 | 15% hard target |

### 9.2 Regime Detector (fusion_signals.py)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `REGIME_EVAL_INTERVAL_SEC` | 30.0 | Re-evaluate every 30s |
| `REGIME_TRENDING_MIN_ATR_PTS` | 50.0 | ATR above this = trending |
| `REGIME_TRENDING_MIN_SLOPE_BPS` | 3.0 | Slope above this = trending |
| `REGIME_TRENDING_MAX_FLIPS_30MIN` | 2 | Max flips for trending |
| `REGIME_RANGE_MAX_ATR_PTS` | 35.0 | ATR below this = range |
| `REGIME_RANGE_MAX_SLOPE_BPS` | 2.0 | Slope below this = range |
| `REGIME_CHOPPY_MIN_FLIPS_30MIN` | 5 | Flips above this = choppy |

### 9.3 Scalp Ladder (order_service_api.py)

| Rung | Peak Profit | Lock Profit | Room |
|------|-------------|-------------|------|
| 1 | 4% | 0.5% | 3.5% |
| 2 | 6% | 2.5% | 3.5% |
| 3 | 8% | 4.5% | 3.5% |
| 4 | 10% | 6.5% | 3.5% |
| 5 | 12% | 9.0% | 3.0% |
| 6 | 15% | 12.0% | 3.0% |

Hard SL: -7% | Hard Target: +15%

---

## 10. Feature Flags — Quick Reference

```python
# ── Range Scalp ──
RANGE_SCALP_ENABLE = True/False          # Master kill switch
RANGE_SCALP_AUTO_REGIME = True/False     # Auto-detect regime (overrides ENABLE)

# ── Macro/Gap Features (independent of Range Scalp) ──
GAP_BIAS_GUARD_ENABLE = True/False       # W12k: Block counter-gap entries
GAP_TREND_ALIGNMENT_ENABLE = True/False  # W13: Relax L3/L4 for with-gap entries

# Disabling macro features does NOT affect Range Scalp or normal trend-following.
# Each feature is independently flag-gated.
```

---

## 11. How to Disable

**Kill entire Range Scalp feature:**
```python
RANGE_SCALP_ENABLE = False      # in fusion_signals.py FusionConfig
```

**Keep scalp but disable auto-regime:**
```python
RANGE_SCALP_AUTO_REGIME = False  # manual control only
RANGE_SCALP_ENABLE = True        # or False to disable
```

**Disable only macro/gap features (scalp + normal continue):**
```python
GAP_BIAS_GUARD_ENABLE = False           # W12k off
GAP_TREND_ALIGNMENT_ENABLE = False      # W13 off
```

**Rollback to stable branch (pre-W12k, only W12i+W12j):**
```bash
# On server:
cd /home/ubuntu/zatamap-trade-api
git checkout nishu_prod_stable
sudo systemctl restart zatamap.service
```
