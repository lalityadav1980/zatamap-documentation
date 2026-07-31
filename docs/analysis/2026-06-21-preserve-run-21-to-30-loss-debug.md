# Preserve Run Loss Debug - 21 Apr to 30 Apr

Run root:

`/Users/<user>/Documents/Codex/2026-04-24/trade-engine-redesign/zatamap-trade-api/.trade-api-runs/debug-2026-04-21-to-2026-04-30-preserve-full-db-20260621_220737`

Scope:

- Read-only investigation.
- No `orders_book`, `orders_tracker`, `fusion_events`, or trail table cleanup was performed.
- Database rows are preserved, so this is a debug dataset, not a clean PnL comparison.

## 21-Apr Confirmed Loss Trades

| Time | Symbol | Entry | Exit | Tag | PnL | Exit % | Root shape |
| --- | --- | ---: | ---: | --- | ---: | ---: | --- |
| 11:52:39 -> 12:02:18 | `NIFTY2642124700PE` | 177.50 | 159.00 | `DVR_RECOVERY_PE` -> `EXIT_SL_LOCK` | -58,922.50 | -10.42% | below-VWAP recovery failed |
| 13:23:03 -> 13:32:36 | `NIFTY2642124700PE` | 156.75 | 140.90 | `DVR_RECOVERY_PE` -> `EXIT_SL_LOCK` | -51,512.50 | -10.11% | below-VWAP recovery failed |

### Token Sponsorship Check

`NIFTY2642124700PE` stayed below VWAP for the entire hold in both trades.

| Window | Above VWAP % | Min VWAP Gap | Max VWAP Gap | Min LTP | Max LTP |
| --- | ---: | ---: | ---: | ---: | ---: |
| 11:52 trade hold | 0.00% | -14.91% | -4.75% | 159.00 | 178.35 |
| 13:23 trade hold | 0.00% | -17.71% | -7.75% | 140.90 | 158.60 |

Spot also did not maintain clean PE continuation:

| Mark | NIFTY |
| --- | ---: |
| 11:47:39 | 24542.35 |
| 11:52:39 buy | 24521.80 |
| 12:02:18 exit | 24537.70 |
| 13:18:03 | 24547.40 |
| 13:23:03 buy | 24538.75 |
| 13:32:36 exit | 24563.15 |

## Why The Trades Were Triggered

Both trades were admitted by the discounted-VWAP recovery router.

For the first loss, the router saw a PE recovery:

- `post_opening_fast_continuation_watch`
- current VWAP gap around `-5.47%`
- repair around `+11.01%`
- premium gain around `+10.56%`
- short ROC around `+4.91`

It then graduated to:

- `DVR_ROUTER_CANDIDATE`
- `DVR_ROUTER_EXECUTE`

For the second loss, the router again graduated a below-VWAP recovery:

- current VWAP gap around `-8.87%`
- max gap had briefly reached positive earlier in the window
- token was still below VWAP at execution
- NIFTY trend context was `UPTREND` in the event context, but short-window PE spot movement still made `spot_aligned=true`

## Code-Level Gap

The current post-opening reservation confirmation in `fusion_signals.py` only requires:

- reservation age,
- premium at least +1% above reserved premium,
- side spot extension at least 5 points,
- current VWAP gap not much worse than reserved gap.

It does not require:

- actual VWAP reclaim,
- above-VWAP persistence,
- current L1 trend not being opposite,
- sustained held-token sponsorship after recovery,
- a stable post-recovery pullback survival.

This explains why below-VWAP bounces can become real orders.

There is also a maturity inconsistency:

- A 13:21 reservation logged `min_age=180s`.
- Later the same setup logged `age=65/60s`.
- That means the confirmation requirement can become easier while the reservation is already in progress.

That is a lifecycle bug. A reservation should keep its original required maturity, or restart when the required maturity changes materially.

## 21-30 Apr Loss Commonality

Current confirmed loss trades from the preserved dataset:

| Date | Symbol | Tag | PnL | Shared issue |
| --- | --- | --- | ---: | --- |
| 21-Apr | `NIFTY2642124700PE` | `DVR_RECOVERY_PE` | -58,922.50 | below-VWAP recovery never reclaimed |
| 21-Apr | `NIFTY2642124700PE` | `DVR_RECOVERY_PE` | -51,512.50 | below-VWAP recovery never reclaimed |
| 29-Apr | `NIFTY2650524350PE` | `TR_STABLE_RETRY` | -69,195.75 | selected token stayed below VWAP for 100% of hold |

The 29-Apr loss is not DVR, but it has the same selected-token sponsorship failure:

| Mark | LTP | VWAP | Gap |
| --- | ---: | ---: | ---: |
| 11:00:35 | 205.25 | 231.22 | -11.23% |
| 11:05:35 buy | 213.45 | 229.28 | -6.90% |
| 12:10:16 exit | 190.80 | 217.89 | -12.43% |

## Healthy-Trade Contrast

To avoid overfitting the fix to loss trades, the same selected-token metrics were checked on preserved profitable trades from 16/17/27/29 Apr.

| Date | Symbol | Tag | PnL | Entry VWAP Gap | Above VWAP During Hold | Max Gain | Side Spot Move |
| --- | --- | --- | ---: | ---: | ---: | ---: | ---: |
| 16-Apr | `NIFTY2642124450PE` | `TR_STABLE_RETRY` | +56,277.00 | -1.87% | 95.41% | +24.84% | +73.20 pts |
| 16-Apr | `NIFTY2642124400PE` | `TR_STABLE_RETRY` | +122,557.50 | +0.37% | 99.57% | +52.16% | +141.75 pts |
| 16-Apr | `NIFTY2642124050CE` | `DVR_RECOVERY_CE` | +24,310.00 | -6.29% | 63.81% | +21.62% | +26.25 pts |
| 17-Apr | `NIFTY2642124150CE` | `DVR_RECOVERY_CE` | +103,194.00 | +2.59% | 100.00% | +30.49% | +68.45 pts |
| 27-Apr | `NIFTY26APR24000CE` | `DVR_RECOVERY_CE` | +52,474.50 | +1.92% | 100.00% | +23.01% | +38.10 pts |
| 27-Apr | `NIFTY26APR23900CE` | `DVR_RECOVERY_CE` | +33,039.50 | -7.39% | 72.02% | +17.16% | +21.30 pts |
| 27-Apr | `NIFTY26APR24100PE` | `TR_STABLE_RETRY` | +120,510.00 | -1.43% | 93.04% | +36.61% | +40.25 pts |
| 29-Apr | `NIFTY2650524000CE` | `TR_STABLE_RETRY` | +357,054.75 | +3.85% | 99.49% | +62.23% | +172.35 pts |

This matters:

- Healthy trades can begin slightly below VWAP, so a naive "block all below-VWAP entries" would break valid opportunities.
- Healthy trades establish sponsorship after entry. Most stay above VWAP for more than 90% of the hold.
- The 16-Apr and 27-Apr DVR CE winners prove that below-VWAP DVR can be valid if it reclaims and holds.
- The 21-Apr losses and 29-Apr loss never reclaimed. Above-VWAP persistence was 0% during the hold.

Therefore the safe rule is not "must be above VWAP at entry." The safe rule is "below-VWAP materialization requires immediate reclaim/persistence or pullback-survival proof before becoming a full order."

## Generic Fix Direction

Do not solve this with a day-specific threshold.

The structural fix should be a selected-token sponsorship materialization rule shared by DVR and STABLE_RETRY:

1. If the token is still below VWAP at entry time, do not materialize from a single recovery burst.
2. Require one of:
   - VWAP reclaim plus short above-VWAP persistence, or
   - a pullback survival ceremony where premium remains firm, gap does not deteriorate, and spot continues in the trade side after the pullback.
3. Freeze reservation maturity at reservation start. If conditions change enough to reduce maturity, restart the reservation rather than graduating early.
4. For PE entries, reject if current L1 trend is clearly `UPTREND` unless the owner-transfer path has explicit opposite-side fade proof.
5. Keep DVR opening/early impulse separate. This finding is about post-opening recovery materialization.

## Current Recommendation

The RCA document fix blocked the earlier direct-continuation bypass, but this preserve run shows a second leak:

`watch/reservation -> candidate` is still too weak for below-VWAP tokens.

Next patch should target the materialization ceremony, not DVR detection itself.

## 27-Apr Missed Runner And Premature Exits

Benchmark leg 14 on 27-Apr was a real bull runner:

| Leg | Window | Direction | Points | Runner Class |
| --- | --- | --- | ---: | --- |
| 14 | 11:38-13:47 | Bull | +142.9 | Major Runner |

The order table shows no trade after the 11:36:15 PE exit, so the leg was fully missed.

The DVR audit did see the CE opportunity:

| Event | Count | First | Last |
| --- | ---: | --- | --- |
| `DVR_WATCH_COLLECT_ONLY` CE | 530 | 11:38:01 | 13:12:31 |
| `DVR_ENTRY_CANDIDATE_UNBLOCKED` CE | 384 | 11:43:06 | 13:11:36 |
| `TR_STABLE_RETRY` CE blocked | 172 | 11:41:17 | 13:19:27 |

Representative `NIFTY26APR23900CE` path:

| Time | LTP | VWAP | Gap | Spot |
| --- | ---: | ---: | ---: | ---: |
| 11:38 | 164.60 | 220.30 | -25.28% | 23985.35 |
| 11:49 | 183.60 | 216.29 | -15.11% | 24019.75 |
| 12:10 | 191.45 | 213.67 | -10.40% | 24034.55 |
| 12:35 | 224.05 | 212.89 | +5.24% | 24075.15 |
| 13:47 | 258.00 | 216.33 | +19.26% | 24125.85 |

The miss was not a lack of signal. It was a router lifecycle dead zone:

- Early: rejected because the token was still too deeply below VWAP.
- Mid: watch was discarded because the spot reset looked "spent".
- Later: rejected because the token had reclaimed VWAP and was now above the hard max gap.

Patch direction implemented after this RCA:

- Preserve a post-opening DVR deep-discount watch when premium, VWAP gap, and spot all continue improving.
- Allow that carried recovery to materialize once it reaches the near-reclaim/reclaim zone.
- Do not simply widen the global VWAP gap; require prior deep-watch context so late chases still stay blocked.

Premature exits also showed up on the same day:

| Trade | Exit | Exit PnL | Max PnL | Left On Table |
| --- | --- | ---: | ---: | ---: |
| `NIFTY26APR24000CE` | `SPOT_OWNER_TRANSFER_` | +11.42% | +22.36% | 10.94 |
| `NIFTY26APR23900CE` | `SPOT_OWNER_TRANSFER_` | +7.22% | +16.58% | 9.36 |

At those exit times, the held CE tokens were still VWAP-sponsored:

| Symbol | Exit Time | Gap | Above VWAP Last 5m |
| --- | --- | ---: | ---: |
| `NIFTY26APR24000CE` | 10:04:35 | +8.63% | 100.0% |
| `NIFTY26APR23900CE` | 11:01:15 | -0.63% | 97.3% |

Patch direction implemented after this RCA:

- Owner-transfer exits should not treat sparse/noisy reverse-option ticks as proof.
- If the held token is VWAP-sponsored, the opposite side must have a real short-window sample before it can confirm ownership transfer.

## 29-Apr Missed PE Runner And Wrong PE Loss

Benchmark leg 7 on 29-Apr was a real bearish runner:

| Leg | Window | Direction | Points | Runner Class |
| --- | --- | --- | ---: | --- |
| 7 | 13:08-14:30 | Bear | -145.3 | Major Runner |

The PE option tape confirms the opportunity was large:

| Symbol | LTP at 13:08 | Max LTP to 14:30 | Gain | Above VWAP | Min Gap | Max Gap |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| `NIFTY2650524300PE` | 156.50 | 238.10 | +52.14% | 61.47% | -17.23% | +26.61% |
| `NIFTY2650524350PE` | 177.50 | 267.50 | +50.70% | 65.20% | -13.58% | +30.07% |
| `NIFTY2650524400PE` | 201.05 | 298.95 | +48.69% | 62.12% | -16.23% | +25.45% |
| `NIFTY2650524450PE` | 226.55 | 333.00 | +46.99% | 62.90% | -15.37% | +24.60% |

The preserved 29-Apr `fusion_events` currently stop at 13:02, before this runner begins. That makes the preserved 29-Apr dataset unsuitable for proving a live router rejection after 13:08. The market tape still shows the same generic DVR shape as 27-Apr:

- PE starts deeply below VWAP around the runner start.
- PE gap repairs toward VWAP as spot starts falling.
- PE later reclaims/expands and becomes strongly sponsored.

This should be covered by the post-opening DVR carry-forward patch rather than by a day-specific PE rule.

The wrong 11:05 PE loss is the opposite shape:

| Mark | Symbol | LTP | VWAP | Gap | Spot |
| --- | --- | ---: | ---: | ---: | ---: |
| 11:05 buy | `NIFTY2650524350PE` | 213.45 | 229.28 | -6.90% | 24273.75 |
| 12:10 exit | `NIFTY2650524350PE` | 190.80 | 217.89 | -12.43% | 24289.70 |

The token stayed below VWAP and spot moved against PE. Current code logs from later 29-Apr evidence runs already show this shape being blocked as `selected_token_vwap_sponsorship_not_ready` / `selected_token_vwap_sponsorship_failed`. Regression must verify that the old 11:05 PE loss is gone while the 13:08 PE recovery can still materialize once the carry-forward lifecycle proves reclaim.
