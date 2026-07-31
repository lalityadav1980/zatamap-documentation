# Coordinator V2 — Central Decision System

## Branch: `feature/coordinator-v2-central-decision`
## Base: `srini_prod` (d21530a)
## Status: OFFLINE DEVELOPMENT — test via backtesting after market hours

---

## Problem Statement

Current architecture has multiple independent strategies (STABLE_RETRY, RANGE_SCALP, MOMENTUM_RIDE, TREND_REVERSAL) each making their own entry/exit decisions with ad-hoc coordinator overrides bolted on. This creates:

1. **Cross-path conflicts** — strategies fight each other (PE entered during bull run)
2. **Scattered decision logic** — entry gates spread across 500+ lines in 4 different code sections
3. **L2 point-in-time noise** — option bias flips every 30s, blocking good entries on snapshot noise
4. **No unified risk view** — each strategy manages its own SL/exit independently
5. **Coordinator v1 is reactive** — overrides AFTER strategies decide, not BEFORE

---

## Offline Work Items

### 1. CENTRAL DECISION ENGINE (Coordinator V2)

**Current**: Each strategy independently decides → coordinator overrides after the fact
**Target**: Strategies submit REQUESTS → coordinator evaluates ALL requests → approves/rejects

```
Architecture:
  on_spot_tick()
    ├── Coordinator V2 computes market_state (already done)
    │   ├── Swing detector (HH/HL/LH/LL) ✅
    │   ├── ATR-based thresholds ✅
    │   ├── Session context (high/low) ✅
    │   ├── Short-window reversal ✅
    │   └── NEW: L2 cumulative bias
    │
    ├── Strategies evaluate and submit ENTRY REQUESTS:
    │   ├── STABLE_RETRY: "I want PE because L1=DOWNTREND, L2=BEARISH, L4=OK"
    │   ├── RANGE_SCALP: "I want CE at support, wall=STRONG_HOLD, bp=0.05"
    │   └── MOMENTUM_RIDE: "80pt move detected, sustained, want CE"
    │
    └── Coordinator V2 DECIDES:
        ├── Priority: momentum_ride > scalp_at_wall > stable_retry
        ├── Checks: market_state, session_context, existing_position, cooldowns
        ├── Checks: cumulative_L2_bias (not snapshot)
        ├── Result: APPROVE / REJECT / QUEUE
        └── Only APPROVED requests execute
```

**Files**: `fusion_signals.py` (major refactor of on_spot_tick flow)

### 2. L2 CUMULATIVE BIAS (replaces point-in-time snapshot)

**Current**: `compute_option_bias()` returns a snapshot. RANGE_SCALP checks `ok` flag.
**Target**: Rolling weighted average of L2 scores over 3-5 minutes.

```
Implementation:
  - Keep deque of last 30 L2 readings (5 min at ~10s intervals)
  - Each reading: (timestamp, score, conviction, sub_signals)
  - Weighted average: exponential decay, recent = higher weight
  - cumulative_score > 0.55 → CUMULATIVE_BULLISH
  - cumulative_score < 0.45 → CUMULATIVE_BEARISH
  - 0.45-0.55 → CUMULATIVE_NEUTRAL

  RANGE_SCALP checks cumulative bias, NOT snapshot
  STABLE_RETRY checks cumulative bias for concurrence
  Momentum_ride ignores L2 (already does)
```

**Files**: `fusion_signals.py` (compute_option_bias, RANGE_SCALP L2 gate, STABLE_RETRY L2 concurrence)

### 3. UNIFIED EXIT MANAGEMENT

**Current**: Each trade has its own TSL with different ladder/SL depending on entry tag
**Target**: Coordinator manages ALL exits through a single state machine

```
Exit hierarchy (coordinator decides):
  1. Smart exit (proportional reversal + peak retracement) — for momentum-aligned trades
  2. Risk floor (peak - 10%) — for any trade above 20% profit
  3. Coordinator force-close — for against-momentum positions
  4. Ladder (normal/expiry/scalp) — for non-momentum trades
  5. Hard SL — catastrophic protection for ALL trades

  The coordinator sets the exit MODE in order_cache:
    exit_mode = "MOMENTUM_SMART" | "LADDER_NORMAL" | "LADDER_EXPIRY" | "HARD_SL_ONLY"
  TSL reads exit_mode and applies the appropriate logic.
```

**Files**: `order_service_api.py` (TSL function), `fusion_signals.py` (coordinator exit_mode setting)

### 4. STRATEGY REQUEST/RESPONSE PROTOCOL

```python
class EntryRequest:
    strategy: str          # "STABLE_RETRY" | "RANGE_SCALP" | "MOMENTUM_RIDE"
    side: str              # "CE" | "PE"
    reason: str            # human-readable
    confidence: float      # 0.0-1.0
    data: dict             # strategy-specific data (L1/L2/L4/swing/etc)
    timestamp: float

class CoordinatorDecision:
    action: str            # "APPROVE" | "REJECT" | "QUEUE"
    reason: str            # why
    exit_mode: str         # what exit strategy to use
    priority: int          # execution priority
```

### 5. HH/LL SWING DETECTOR ENHANCEMENTS

- ATR auto-calibration for swing size (already done ✅)
- Swing quality scoring (strong HH vs weak HH based on volume)
- Multi-timeframe swings (5-min and 15-min)
- Swing-based smart exit (exit CE when Lower-High confirmed)

### 6. BACKTEST VALIDATION

Test on these dates:
- **2026-03-27** (sideways day) — should scalp well, no momentum trades
- **2026-03-30** (trending expiry) — should catch CE rally + PE selloff
- **2026-03-25** (normal day) — baseline comparison

Compare:
- Production code PnL vs Coordinator V2 PnL
- Number of trades, win rate, avg PnL per trade
- Entry timing (how early did the system detect momentum)
- False signal rate (entries that immediately hit SL)

---

## Implementation Order

1. L2 cumulative bias (smallest change, biggest impact on RANGE_SCALP)
2. Central decision engine (architecture change)
3. Unified exit management (TSL refactor)
4. Strategy request/response protocol (clean interfaces)
5. Swing detector enhancements (polish)
6. Full backtest suite

---

## Notes from Live Market Monitoring (01-Apr-2026)

### Issues Found Today:
- L2 ok/age fields were NEVER set → RANGE_SCALP always blocked (FIXED in prod)
- MOMENTUM_RIDE_ENABLE default was False → momentum never fired (FIXED in prod)
- _sc_get_oc import scope → momentum entry silently failed (FIXED in prod)
- self reference in TSL → coordinator takeover crashed (FIXED in prod)
- Auto-regime getattr default True overriding config False (FIXED in prod)
- Session block threshold 100pts too tight for normal days (FIXED: 150pts)
- DATA_READINESS 200 ticks on restart (FIXED: 50 ticks with CSV detection)
- ATR briefly dropped to 20pts after restart (self-corrected)

### Working Well:
- Swing detector: 8 points, 3 pattern transitions (LL_LH → LH_HL → HH_HL) ✅
- CSV hydration: session context restored on restart ✅
- Coordinator state transitions: RANGE ↔ MOMENTUM_DOWN correctly ✅
- SESSION_BLOCK throttle: 14 in 2 hours (was 1100+ in 35 min) ✅
- MID_SESSION_RESTART: DATA_READINESS in 25s vs 100s+ ✅
