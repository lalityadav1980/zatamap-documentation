# RCA: 2026-05-04 First Trade Loss - TR_STABLE_RETRY CE

## Scope

This is a read-only RCA for the first loss in the current May cumulative replay. No code changes were made.

Run root:

`/Users/<user>/Documents/Codex/2026-04-24/trade-engine-redesign/zatamap-trade-api/.trade-api-runs/full-2026-05-04-to-2026-05-31-cumulative-r1-20260623_184035`

Trade:

| Field | Value |
|---|---|
| Date | 2026-05-04 |
| Parent order | `TK260504-093040-941` |
| Symbol | `NIFTY2650524100CE` |
| Entry | 09:30:40 @ 227.60 |
| Exit | 09:32:57 @ 204.50 |
| Entry tag | `TR_STABLE_RETRY` |
| Exit tag | `EXIT_SL_LOCK` |
| Quantity | 22,685 |
| Realized PnL | -524,023.50 |
| Max pct | +0.40% |
| Min pct | -10.15% |

## Benchmark Context

Benchmark day label:

`ROUND_TRIP_MIXED_BIAS / WHIPSAW_HIGH_OPPORTUNITY`

Relevant benchmark legs:

| Leg | Time | Direction | Points | Class |
|---:|---|---|---:|---|
| 2 | 09:17-09:30 | Bull | +77.8 | Fast Swing |
| 3 | 09:30-09:32 | Bear | -37.0 | Scalp / Noise Leg |
| 4 | 09:32-09:41 | Bull | +88.3 | Impulse Runner |
| 5 | 09:41-10:23 | Bear | -137.0 | Runner |

The CE was entered exactly at the transition from leg 2 to leg 3, after the opening bull fast swing had already matured and just as the two-minute bear scalp started.

## Market And Token Evidence

Spot:

| Window | Open | High | Low | Close | Move |
|---|---:|---:|---:|---:|---:|
| 09:15-09:30:40 | 24096.20 | 24241.35 | 24096.20 | 24225.65 | +129.45 |
| 09:30:40-09:32:57 | 24225.65 | 24235.00 | 24196.30 | 24196.30 | -29.35 |
| 09:30:40-09:45 | 24225.65 | 24288.00 | 24181.05 | 24279.25 | +53.60 |

Selected CE minute path:

| Minute | Open | High | Low | Close | VWAP gap open | VWAP gap close |
|---|---:|---:|---:|---:|---:|---:|
| 09:30 | 224.40 | 231.00 | 222.20 | 231.00 | +1.99% | +4.80% |
| 09:31 | 230.35 | 230.35 | 215.65 | 218.70 | +4.49% | -0.83% |
| 09:32 | 219.85 | 221.50 | 203.45 | 203.45 | -0.31% | -7.56% |
| 09:33 | 200.45 | 209.75 | 193.85 | 208.40 | -8.89% | -4.76% |
| 09:34 | 209.80 | 228.00 | 209.80 | 227.85 | -4.12% | +4.08% |

The selected CE did not prove follow-through after entry. It went from +3.31% above VWAP at entry to -8.89% below VWAP by 09:33.

Opposite PE evidence during the same hold:

| Symbol | Entry LTP | Entry VWAP gap | Max during CE hold | Max pct during CE hold | Min pct during CE hold |
|---|---:|---:|---:|---:|---:|
| `NIFTY2650524250PE` | 116.10 | -11.99% | 138.85 | +19.60% | -2.45% |
| `NIFTY2650524300PE` | 141.00 | -10.36% | 166.15 | +17.84% | -2.66% |
| `NIFTY2650524350PE` | 169.00 | -9.45% | 196.60 | +16.33% | -2.28% |

The correct short-term owner transfer appeared on PE while the CE position was open.

## Engine Evidence At Entry

At 09:30:40, `fusion_events` show:

- `L1`: UPTREND
- `L2`: ALLOW, BULLISH, HIGH conviction, side score 0.913
- `L3`: pass, entry total score 0.7789, edge 0.1096
- `L4`: warning, not block
- `WHIPSAW_PARTICIPATION_GATE`: participate, reason `token_side_and_path_participating`
- `CANDIDATE_RESERVATION`: `entry_not_enforced`, reason `opening_window_exempt`

Important warning fields:

| Field | Value |
|---|---|
| L4 condition | `TIGHT_RANGE_CE_WALL_HOLDING` |
| Zone | `NEAR_RESISTANCE` |
| Resistance conviction | `RESPECT` |
| Resistance breach probability | 0.00 |
| Breakout confirmed | false |
| Breakout confidence | 0.00 |
| Tight-range override | `STRONG_CONVICTION_BYPASS` |
| Wall holding against side | true |
| Target wall confirmed | false |
| Candidate reservation reason | `opening_window_exempt` |

This means the engine had enough evidence to know this was a wall-hold / no-breakout setup, but opening-window exemption and strong L2/L3 allowed the entry anyway.

## Why The Recent Fix Did Not Cover It

The recent fix was for DVR reservation materialization and VWAP-aware target exit protection. This trade was not a DVR entry and did not fail because target/VWAP exit closed early.

This trade used:

`TR_STABLE_RETRY -> EXIT_SL_LOCK`

So it bypassed the new DVR-specific reservation fix by design.

## Root Cause

Primary RCA:

Opening-window STABLE_RETRY allowed a CE entry after an already-mature opening bull swing, even though L4 said the trade was near resistance, the resistance wall was holding, breakout was not confirmed, and the candidate reservation guard was not enforced because of `opening_window_exempt`.

Secondary RCA:

The bad CE position blocked the emerging PE owner-transfer evidence. PE DVR/router events started showing recovery/watch and later ceremony strength, but were skipped because broker already had the CE position open.

Failure shape:

`mature opening CE push -> resistance wall holding -> opening exemption bypass -> CE buy -> immediate PE owner transfer -> CE stop loss`

## Live-Safe Fix Direction

Do not solve this by weakening stop loss or tuning the CE threshold. The stop behaved correctly after a bad entry.

Candidate structural guard for later implementation:

- For opening-window `TR_STABLE_RETRY`, do not allow `opening_window_exempt` to bypass candidate reservation when all are true:
  - L4 says wall is holding against the side.
  - Target wall is not confirmed breached.
  - Breakout/breakdown confirmation is false or near zero.
  - Selected token is merely above VWAP, not deeply reclaiming from discount.
  - Opposite-side discounted recovery begins improving while selected token VWAP gap deteriorates.

This is not a day-specific rule. It targets a general failed shape: opening continuation chase into an unbroken wall.

## Regression Impact To Check Before Any Fix

Before promoting any code change, test this guard against:

- 2026-04-16 09:31 PE winner, to ensure valid opening participation is not blocked.
- 2026-04-20 opening/early whipsaw sequence, to ensure early CE/PE materialization still works.
- 2026-04-29 morning major bull runner, to ensure true breakout continuation is still allowed.
- 2026-05-04 later PE runner at 10:13, to ensure the good PE trade remains.

