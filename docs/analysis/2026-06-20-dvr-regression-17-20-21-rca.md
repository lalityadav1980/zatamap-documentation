# DVR Regression RCA - 17/20/21 Apr Replay

Run under review:

- Branch: `backtest-box-69`
- Replay session: `verify-17to21-logging-r1`
- Run root: `.trade-api-runs/verify-2026-04-17-to-2026-04-21-logging-r1-20260620_174620`
- Scope: `2026-04-17` to `2026-04-21`
- Purpose: document production/local replay regressions before making another code change.

This document is intentionally RCA-first. The current failure mode is not one bad number. We are seeing DVR entry materialization and exit management interact in a way that fixes one day while degrading another.

## Confirmed 17-Apr Evidence

### Trade 1 - CE winner under-captured

Order lifecycle:

| Field | Value |
| --- | --- |
| Symbol | `NIFTY2642124150CE` |
| Parent | `TK260417-094804-1985` |
| Entry | `2026-04-17 09:48:04` @ `199.90` |
| Exit | `2026-04-17 10:34:35` @ `235.30` |
| Tag | `DVR_RECOVERY_CE` -> `EXIT_TARGET_VWAP` |
| Realized | `+48,321.00` |
| Exit PnL pct | `+17.71%` |
| Peak PnL pct | `+30.39%` |
| Left on table | `12.68%` |
| Above VWAP during hold | `100.0%` |
| Entry VWAP | `194.85`; entry was `+2.59%` above VWAP |
| Exit VWAP | `207.70`; exit was still `+13.29%` above VWAP |

Important trail sequence:

| Time | Price | PnL pct |
| --- | ---: | ---: |
| 10:25:05 | 250.00 | 25.06 |
| 10:26:20 | 256.65 | 28.39 |
| 10:27:50 | 260.65 | 30.39 |
| 10:31:21 | 248.50 | 24.31 |
| 10:34:21 | 239.60 | 19.86 |
| 10:34:35 | 235.30 | 17.71 |

RCA:

- Entry was structurally acceptable: `DVR_ROUTER_EXECUTE` fired after `opening_post_auction_direct_continuation_confirmed`.
- The exit was too loose after a 30% peak. The held token remained strongly above VWAP the entire hold.
- This is not a signal-entry issue. It is a post-target runner-release issue: once peak is near 30%, VWAP defer should not allow a giveback to 17% unless the token remains in a very strong same-side continuation state.

Generic fix direction:

- Add a post-target profit-protection floor for DVR/session-runner trades.
- Do not disable VWAP defer. Instead, require both:
  - held token still strongly VWAP-sponsored and VWAP improving, and
  - spot/side continuation still aligned.
- If those are not both true after a 25-30% peak, release/exit earlier.

### Trade 2 - PE medium winner round-tripped to loss

Order lifecycle:

| Field | Value |
| --- | --- |
| Symbol | `NIFTY2642124450PE` |
| Parent | `TK260417-115921-9862` |
| Entry | `2026-04-17 11:59:21` @ `249.65` |
| Exit | `2026-04-17 13:41:41` @ `231.00` |
| Tag | `DVR_RECOVERY_PE` -> `EXIT_SL_LOCK` |
| Realized | `-20,608.25` |
| Exit PnL pct | `-7.47%` |
| Peak PnL pct | `+8.35%` |
| Left on table | `15.82%` |
| Above VWAP during hold | `0.0%` |
| Entry VWAP | `275.41`; entry was `-9.35%` below VWAP |
| Exit VWAP | `269.03`; exit was still `-14.13%` below VWAP |

Important trail sequence:

| Time | Price | PnL pct |
| --- | ---: | ---: |
| 12:05:39 | 269.80 | 8.07 |
| 12:16:00 | 248.35 | -0.52 |
| 12:53:24 | 270.50 | 8.35 |
| 13:00:57 | 262.05 | 4.97 |
| 13:09:15 | 247.50 | -0.86 |
| 13:40:00 | 239.50 | -4.07 from entry by tick, trail later closed -7.47 |
| 13:41:41 | 231.00 | -7.47 |

Critical router evidence:

At `11:59:09`, the router correctly skipped similar PE candidates:

```text
DVR_ROUTER_SKIP post_opening_spot_reset_already_spent:
pullback=43.7 recovery=39.5 ratio=0.91 gap=-9.39% premium_gain=+4.04%
```

At `11:59:21`, the same broad setup was allowed by a fast-continuation bypass:

```text
DVR_ROUTER_WATCH post_opening_fast_continuation_direct_confirmed:
gap=-9.35% repair=+7.28% premium_gain=+4.43% roc=+0.89
short_roc=+3.42 accel=+2.53 spot_move=-31.15 giveback=-0.34%
DVR_ROUTER_EXECUTE order_placed
```

RCA:

- The router had already identified that the post-opening reset was mostly spent.
- The later `post_opening_fast_continuation_direct_confirmed` path overrode that risk and allowed entry while the selected token never reclaimed VWAP.
- After entry, the trade reached a medium peak around +8%, but there was no DVR-specific medium-profit lock. It round-tripped to loss.

Generic fix direction:

- `post_opening_fast_continuation_direct_confirmed` must not bypass spent-reset protection.
- If a post-opening DVR token is still below VWAP, it should need either:
  - unspent reset/recovery runway, or
  - a fresh pullback-survival/reclaim ceremony after the spent reset.
- Add a DVR medium-peak protection rule, but not the earlier broad owner-transfer exit. The safer rule should require deterioration in held-token evidence, such as VWAP gap failing to improve, premium fading, and spot no longer continuing with the held side.

## Confirmed 20-Apr Evidence

Replay reached 20-Apr and reproduced both healthy captures and the reported bad PE.

### Healthy early sequence

| Parent | Symbol | Entry | Exit | Tag | Exit PnL | Peak | Left on table |
| --- | --- | --- | --- | --- | ---: | ---: | ---: |
| `TK260420-092002-303` | `NIFTY2642124200CE` | 09:20:02 @ 217.65 | 09:30:49 @ 235.95 | `TR_STABLE_RETRY` -> `SPOT_OWNER_TRANSFER_` | +8.41% | +21.87% | 13.46% |
| `TK260420-093619-1280` | `NIFTY2642124400PE` | 09:36:19 @ 167.50 | 09:49:27 @ 201.90 | `DVR_RECOVERY_PE` -> `EXIT_TARGET_VWAP` | +20.54% | +25.31% | 4.77% |
| `TK260420-095314-2295` | `NIFTY2642124200CE` | 09:53:14 @ 192.30 | 10:38:42 @ 281.20 | `TR_STABLE_RETRY` -> `EXIT_TARGET_VWAP` | +46.23% | +52.11% | 5.88% |

Notes:

- The 09:36 PE used the safer ceremony: `recent_opposite_pullback_survival_confirmed`.
- The 09:53 CE shows the STABLE_RETRY runner exit path working well: +46.23% realized after a +52.11% peak.
- The 09:20 CE still has exit under-capture: peak +21.87%, realized +8.41%. That supports a generic profit-protection issue, but it is less severe than the 17-Apr CE because it exited on an owner-transfer handover.

### Bad 12:13 PE

Order lifecycle:

| Field | Value |
| --- | --- |
| Symbol | `NIFTY2642124550PE` |
| Parent | `TK260420-121329-10710` |
| Entry | `2026-04-20 12:13:29` @ `197.65` |
| Exit | `2026-04-20 12:26:35` @ `184.00` |
| Tag | `DVR_RECOVERY_PE` -> `EXIT_SL_LOCK` |
| Realized | `-20,406.75` |
| Exit PnL pct | `-6.91%` |
| Peak PnL pct | `+4.22%` |
| Left on table | `11.13%` |
| Above VWAP during hold | `0.0%` |
| Entry VWAP | `219.40`; entry was `-9.91%` below VWAP |

Critical router evidence:

Before the buy, the router repeatedly identified missing reset/runway:

```text
DVR_ROUTER_SKIP post_opening_deep_discount_spot_reset_missing:
pullback=9.8 recovery=22.0 reason=spot_reset_missing

DVR_ROUTER_SKIP post_opening_spot_reset_missing:
pullback=8.6 recovery=31.05 low_age=831.0 high_age=732.0
```

Then the fast-continuation bypass overrode that and bought:

```text
DVR_ROUTER_WATCH post_opening_fast_continuation_direct_confirmed:
gap=-9.91% repair=+10.15% premium_gain=+4.60% roc=+0.92
short_roc=+3.51 accel=+2.59 spot_move=-22.95 giveback=+0.00%
DVR_ROUTER_EXECUTE order_placed
```

RCA:

- This is the same failure class as the 17-Apr PE.
- The router knew the post-opening reset was missing, then allowed a direct-continuation shortcut.
- The selected token stayed below VWAP for the whole hold and never developed durable sponsorship.

Comparison against the old expected 13:01 PE:

| Candidate | Entry | Entry VWAP | Max in window | Min in window | Above VWAP pct |
| --- | ---: | ---: | ---: | ---: | ---: |
| Bad `4550PE` 12:13 | 197.65 | 219.40 | 206.90 | 184.00 | 0.0% |
| Expected `4600PE` 13:01 | 230.80 | 245.05 | 287.90 | 224.75 | 66.3% |

The 13:01 candidate had materially better sponsorship after the reset matured. The fix should therefore preserve the watch/reserve state instead of letting the first short-ROC burst spend the trade slot.

### Missed 13:01 PE

In the current replay, the old expected `NIFTY2642124600PE` was not entered.

At `13:01:24`, it was skipped:

```text
DVR_ROUTER_SKIP post_opening_spot_reset_missing:
pullback=16.5 recovery=42.9 low_age=899.0 high_age=516.0
```

At `13:02:24`, it was still skipped, but for a different reason:

```text
DVR_ROUTER_SKIP router_quality_failed:
phase=post_opening score=8.0/8.0 gap=-5.23% repair=+15.27%
premium_gain=+13.08% roc=+2.62 short_roc=+0.56 accel=-2.05
spot_move=-35.50 giveback=-6.39% medium=True deep=True
```

RCA:

- The router is too loose at `12:13` because `post_opening_fast_continuation_direct_confirmed` can bypass reset-missing.
- The same router is too rigid at `13:01` because it treats reset-missing as a hard skip, then later requires short-window acceleration after the better candidate has already cooled.
- This proves the next fix cannot simply be "require reset" or "relax reset". The correct fix is a stateful reservation lifecycle: continue watching a candidate that has good long-window repair, and only materialize after fresh pullback survival or durable sponsorship appears.

## Confirmed 21-Apr Evidence

Replay is still in progress, but the first reported loss is already confirmed.

### Trade 1 - PE direct-continuation loss

Order lifecycle:

| Field | Value |
| --- | --- |
| Symbol | `NIFTY2642124650PE` |
| Parent | `TK260421-102257-4078` |
| Entry | `2026-04-21 10:22:57` @ `164.85` |
| Exit | `2026-04-21 10:31:39` @ `148.25` |
| Tag | `DVR_RECOVERY_PE` -> `EXIT_SL_LOCK` |
| Realized | `-49,634.00` |
| Exit PnL pct | `-10.07%` |
| Peak PnL pct | `+4.28%` |
| Left on table | `14.35%` |
| Above VWAP during hold | `1.93%` |
| VWAP gap during hold | min `-13.13%`, max `+0.87%` |

Important tick sequence:

| Time | LTP | VWAP | VWAP gap |
| --- | ---: | ---: | ---: |
| 10:20:00 | 157.30 | 172.97 | -9.06% |
| 10:22:00 | 157.90 | 172.26 | -8.34% |
| 10:23:00 | 163.45 | 172.09 | -5.02% |
| 10:28:00 | 171.35 | 171.46 | -0.06% |
| 10:31:00 | 158.05 | 171.15 | -7.65% |
| 10:35:00 | 130.10 | 168.78 | -22.92% |

Critical router evidence:

Before the buy, the router skipped the same family of PE candidates:

```text
DVR_ROUTER_SKIP post_opening_spot_reset_missing:
pullback=9.8 recovery=26.35 low_age=891.0 high_age=806.0
```

At the buy, the direct-continuation shortcut overrode that caution:

```text
DVR_ROUTER_WATCH post_opening_fast_continuation_direct_confirmed:
gap=-4.21% repair=+19.53% premium_gain=+11.88% roc=+2.38
short_roc=+1.57 accel=-0.80 spot_move=-18.75 giveback=+0.00%
DVR_ROUTER_EXECUTE order_placed
```

RCA:

- This is the same failure class as the 17-Apr PE and 20-Apr 12:13 PE.
- The selected token briefly approached VWAP but did not establish durable sponsorship.
- `post_opening_fast_continuation_direct_confirmed` is currently allowed to materialize a trade even after the router has just seen reset/runway problems.
- The short-window premium burst was not enough; the trade needed a fresh pullback-survival/reclaim ceremony before execution.

### Trade 2 - PE direct-continuation loss

Order lifecycle:

| Field | Value |
| --- | --- |
| Symbol | `NIFTY2642124600PE` |
| Parent | `TK260421-113447-8388` |
| Entry | `2026-04-21 11:34:47` @ `95.60` |
| Exit | `2026-04-21 11:38:32` @ `85.60` |
| Tag | `DVR_RECOVERY_PE` -> `EXIT_SL_LOCK` |
| Realized | `-47,450.00` |
| Exit PnL pct | `-10.46%` |
| Peak PnL pct | `+7.53%` |
| Left on table | `17.99%` |
| Above VWAP during hold | `0.91%` |
| VWAP gap during hold | min `-16.72%`, max `+1.45%` |

Important tick sequence:

| Time | LTP | VWAP | VWAP gap |
| --- | ---: | ---: | ---: |
| 11:30:00 | 85.15 | 104.00 | -18.13% |
| 11:32:00 | 90.35 | 103.67 | -12.85% |
| 11:34:00 | 94.70 | 103.30 | -8.33% |
| 11:36:00 | 102.45 | 103.14 | -0.67% |
| 11:37:00 | 92.25 | 102.98 | -10.42% |
| 11:38:00 | 91.70 | 102.87 | -10.86% |
| 11:39:00 | 86.45 | 102.71 | -15.83% |

Critical router evidence:

Before the buy, the router repeatedly identified missing reset/runway:

```text
DVR_ROUTER_SKIP post_opening_spot_reset_missing:
pullback=5.95 recovery=35.0 low_age=900.0 high_age=873.0

DVR_ROUTER_SKIP post_opening_spot_reset_missing:
pullback=1.25 recovery=34.25 low_age=900.0 high_age=896.0
```

At the buy, the same direct-continuation shortcut materialized:

```text
DVR_ROUTER_WATCH post_opening_fast_continuation_direct_confirmed:
gap=-7.39% repair=+22.41% premium_gain=+12.87% roc=+2.57
short_roc=+4.03 accel=+1.45 spot_move=-19.00 giveback=-2.15%
DVR_ROUTER_EXECUTE order_placed
```

RCA:

- This confirms the pattern from 17-Apr PE, 20-Apr 12:13 PE, and 21-Apr first PE.
- The PE had a sharp recovery burst, but it did not have durable VWAP sponsorship.
- The token briefly approached VWAP, then immediately failed back below it.
- The router must not convert this shape into an immediate order. It should remain a watch/reserve until either the token survives a pullback or reclaims VWAP with continuation.

## Code Pinpoint

The repeated loss is rooted in the current post-opening DVR materialization flow in `fusion_signals.py`.

Relevant behavior:

- The branch computes `post_opening_fast_continuation_confirmed`.
- That branch can suppress `recovery_quality_failed`.
- The later spot-reset/runway validation only executes when `not post_opening_fast_continuation_confirmed`.
- Therefore a fast short-window recovery burst can skip the reservation/reset lifecycle and go straight to `DVR_ROUTER_CANDIDATE` / order placement.

That matches all confirmed bad PE entries:

| Date | Symbol | Before buy | Buy reason | Outcome |
| --- | --- | --- | --- | --- |
| 17-Apr | `NIFTY2642124450PE` | `post_opening_spot_reset_already_spent` | `post_opening_fast_continuation_direct_confirmed` | medium peak to loss |
| 20-Apr | `NIFTY2642124550PE` | `post_opening_spot_reset_missing` | `post_opening_fast_continuation_direct_confirmed` | small peak to loss |
| 21-Apr | `NIFTY2642124650PE` | `post_opening_spot_reset_missing` | `post_opening_fast_continuation_direct_confirmed` | small peak to loss |
| 21-Apr | `NIFTY2642124600PE` | `post_opening_spot_reset_missing` | `post_opening_fast_continuation_direct_confirmed` | medium peak to loss |

This is why changing one threshold keeps regressing other days. The issue is lifecycle ordering, not a date-specific parameter.

## Cross-Day Generic Fix Plan

The safe fix should split DVR into independent lifecycle stages:

1. `watch`: discounted token recovery exists, but no trade permission yet.
2. `reserve`: same side is improving and opposite side is fading, but reset/runway is not spent.
3. `materialize`: token quality, spot/side continuation, and unspent runway pass together.
4. `hold`: if the trade becomes a runner, use VWAP-aware hold logic.
5. `profit_protect`: protect either large peak or medium peak based on held-token evidence, not generic owner-transfer alone.

Entry-side rules:

- Keep opening impulse separate from post-opening DVR.
- Post-opening direct continuation must not bypass `spot_reset_already_spent`.
- If the selected token is below VWAP, require stricter proof:
  - VWAP gap improving,
  - premium rising,
  - same-side spot continuation,
  - reset not already spent, or fresh pullback-survival/reclaim after reset.
- Apply symmetrically to CE and PE.

Exit-side rules:

- Large peak case: after 25-30% peak, do not allow >7-8% giveback unless token remains strongly VWAP-sponsored and same-side spot continuation persists.
- Medium peak case: after 7-10% peak on DVR trades, do not allow a round trip to SL if held-token sponsorship deteriorates and spot continuation fails.
- Keep the old broad `DVR_MEDIUM_PEAK_HANDOVER_EXIT` disabled until the stricter evidence-based version is validated.

Regression plan before any promotion:

1. Validate focused replay on `2026-04-16`, `2026-04-17`, `2026-04-20`, `2026-04-21`.
2. Validate known regression days: `2026-04-23`, `2026-04-24`, `2026-04-27`, `2026-04-28`, `2026-04-29`, `2026-04-30`.
3. Only then run a full clean cumulative replay from `2026-04-16` to `2026-06-19`.

## Current Conclusion

The repeated regression is real: DVR is not yet a complete institutional router. The detector is useful, but enforcement currently lets a fast-continuation shortcut override spent-runway warnings, and exits do not distinguish large-runner protection from medium-peak protection.

The next code change should therefore be small but structural:

- tighten post-opening DVR materialization around spent reset / fresh reclaim,
- add evidence-based profit protection for DVR trades,
- keep both changes trace-logged so we can prove their impact across days.

## Implemented Fix - 2026-06-20

Code changes made after this RCA:

1. `fusion_signals.py`: `post_opening_fast_continuation_confirmed` is no longer allowed to suppress router quality failure or bypass spot-reset/reservation validation.
   - It is now logged as `post_opening_fast_continuation_watch`.
   - Execution still requires the normal post-opening spot-reset / reservation / fresh-reclaim lifecycle.
   - This directly targets the repeated 17/20/21 Apr PE losses where a fast below-VWAP burst became an order after `spot_reset_missing` or `spot_reset_already_spent` had already been logged.

2. `order_service_api.py`: DVR hard-target deferral now uses a tighter code-default giveback cap once a DVR trade reaches the hard-target zone.
   - New default: `hard_target_vwap_defer_dvr_target_max_giveback_pct = 8%`.
   - This applies only to DVR recovery trades that are already in target-defer/session-runner mode and have not matured into the larger peak bucket.
   - It does not alter normal `STABLE_RETRY` runner target behavior.

What was intentionally not fixed yet:

- The old broad `DVR_MEDIUM_PEAK_HANDOVER_EXIT` remains disabled. Re-enabling it would be overfitting because it previously cut a valid 16-Apr DVR CE runner too early.
- Medium-peak exits should only be revisited after the entry lifecycle fix is validated, because the confirmed medium-peak losses should now be blocked before entry.

Focused regression in progress:

- `2026-04-21` replay to noon, run id `verify-2026-04-21-dvr-lifecycle-fix-r1-20260620_204246`.

## Focused Regression Results

### 21-Apr focused replay

Run id: `verify-2026-04-21-dvr-lifecycle-fix-r1-20260620_204246`

Window: `09:15` to `12:05`

Result:

- Old bad `10:22:57 NIFTY2642124650PE` buy did not occur.
- Old bad `11:34:47 NIFTY2642124600PE` buy did not occur.
- The router now emits `post_opening_fast_continuation_watch`, then blocks on `router_quality_failed` or `post_opening_spot_reset_missing`.
- This confirms the direct-continuation bypass was the leak.

### 17-Apr focused replay

Run id: `verify-2026-04-17-dvr-lifecycle-fix-r1-20260620_205424`

Window: `09:15` to `14:00`

Result:

| Trade | Before fix | After fix |
| --- | ---: | ---: |
| `NIFTY2642124150CE` exit PnL | `+17.71%` | `+22.06%` |
| `NIFTY2642124150CE` peak | `+30.39%` | `+30.39%` |
| `NIFTY2642124150CE` left on table | `12.68%` | `8.33%` |
| `NIFTY2642124450PE` loss | `-7.47%` | blocked |

The 11:59 PE now logs `post_opening_fast_continuation_watch`, then blocks on `post_opening_spot_reset_already_spent`. That is the intended behavior.

### 20-Apr focused replay

Run id: `verify-2026-04-20-dvr-lifecycle-fix-r1-20260620_210553`

Window: `09:15` to `13:35`

Result:

| Trade | Status |
| --- | --- |
| 09:20 CE | Preserved: `+8.41%` |
| 09:36 DVR PE | Preserved: `+20.54%`, peak `+25.31%` |
| 09:53 CE | Preserved: `+46.23%`, peak `+52.11%` |
| 12:13 bad PE | Blocked |
| 13:01 expected PE | Still missed |

The 12:13 bad PE now logs `post_opening_fast_continuation_watch`, then blocks on `post_opening_spot_reset_missing`.

The 13:01 PE is still missed because the router sees `post_opening_spot_reset_missing` at `13:01:24`, then by `13:02:24` short-window acceleration has cooled and it is rejected by `router_quality_failed`. This is not safe to force with a threshold tweak; it needs a future stateful carry-forward reservation that can keep watching long-window repair through a reset-missing phase and materialize only after fresh pullback survival.

### Current classification

Fixed safely:

- Bad post-opening below-VWAP DVR PE materialization caused by `post_opening_fast_continuation_direct_confirmed`.
- DVR target-defer over-giveback after a 25-30% peak, partially improved without changing non-DVR runner exits.

Not fixed yet:

- 20-Apr 13:01 PE missed opportunity. This is an enhancement, not a safety fix. Forcing it now would likely re-open the 17/20/21 loss class.

Intentionally not changed:

- `DVR_MEDIUM_PEAK_HANDOVER_EXIT` remains disabled. Re-enabling it now would be overfitting because it previously broke valid DVR runners.

### 16-Apr guardrail replay

Run id: `verify-2026-04-16-dvr-lifecycle-fix-r1-20260620_212340`

Window: `09:15` to `15:05`

Result:

| Trade | Status |
| --- | --- |
| 09:31 PE | Preserved: `+20.83%`, peak `+23.30%` |
| 11:10 PE | Preserved: `+45.15%`, peak `+51.49%` |
| 14:11 DVR CE | Preserved but under-captured: `+8.98%`, peak `+21.56%`, exit `SPOT_OWNER_TRANSFER_` |

The lifecycle safety fix did not break the two key PE captures on 16-Apr, and the afternoon DVR CE still triggers. However, the CE still exits too early via `SPOT_OWNER_TRANSFER_` before target. This should not be fixed in the same patch because it is a different exit class: pre-target sponsored-runner hold versus owner-transfer exit. A broad owner-transfer veto here could overfit 16-Apr and break real reversal protection elsewhere.

## 23/24-Apr Regression Findings

These two dates exposed a second failure class. The first 17/20/21 fix blocked the obvious post-opening DVR fast-continuation leak, but 23/24-Apr showed that old range-scalp reversal tags and DVR owner-transfer validation could still admit wrong-side or premature recovery trades.

### 23-Apr focused replay

Known bad shape:

- The day was a gap-down / mixed-trend day with multiple runner legs, but several historical loss trades came from late or wrong-side reversal behavior.
- The bad `RANGE_SCALP_PE_AT_RES_MACRO` style entry around the late morning was not a true PE breakdown; it was a reversal-at-wall scalp on a day where the selected token did not have enough durable sponsorship.

Focused replay result after the range-scalp guard:

- Replay reached `2026-04-23 13:03:57`.
- Orders: `0`.
- The old bad `11:13 RANGE_SCALP_PE_AT_RES_MACRO NIFTY26APR24250PE` did not fire.

RCA:

- This supports keeping `RANGE_SCALP_CE_BREAKOUT` / `RANGE_SCALP_PE_BREAKDOWN` available under strict guard, but not allowing `*_AT_SUP*` / `*_AT_RES*` reversal tags to place orders by default.
- The safe design is audit-first for reversal-at-wall range scalps. A real breakout/breakdown is a different signal class from a support/resistance bounce.

Implemented behavior:

- `RANGE_SCALP_REVERSAL_MODE = "audit"` by default.
- Tags containing `AT_SUP` or `AT_RES`, excluding `BREAKOUT` / `BREAKDOWN`, are persisted as `range_scalp_reversal_audit` events with tag `RANGE_SCALP_REVERSAL_AUDIT`.
- Breakout/breakdown tags still continue through the strict selected-token guard.

Impact:

- This is a safety fix, not a profit enhancement.
- It can create no-trade outcomes on weak reversal setups, which is acceptable until the audit proves a positive expectancy subset.

### 24-Apr focused replay

Original loss pattern:

| Time | Symbol | Tag | Outcome |
| --- | --- | --- | ---: |
| 11:06:57 | `NIFTY26APR23900CE` | `DVR_RECOVERY_CE` | loss |
| 11:21:23 | `NIFTY26APR23900CE` | `DVR_RECOVERY_CE` | loss |
| 12:34:42 | `NIFTY26APR23700CE` | `DVR_RECOVERY_CE` | loss |

The important clue is that all three losses were CE, while PE had remained structurally strong after the morning PE winner.

Observed owner evidence:

| Window | Target CE state | Prior PE state |
| --- | --- | --- |
| 11:06 | CE recovering but still below VWAP | `NIFTY26APR24100PE` still strongly above VWAP and up after exit |
| 11:21 | CE recovering but still below VWAP | same PE still above VWAP and still expanded after exit |
| 12:34 | CE recovery attempt | PE sponsorship still active enough to reject clean owner transfer |

The key distinction versus valid 16-Apr CE:

- 24-Apr PE was still gaining materially after the PE exit.
- 16-Apr prior PE had stopped expanding materially by the valid CE recovery window.

RCA:

- DVR was treating target-side recovery as sufficient for entry.
- That is not enough. A discounted CE recovery is only actionable if PE ownership has actually transferred or faded.
- Without owner-transfer validation, DVR can buy a recovering CE against a tape still owned by PE.

Implemented behavior:

- Added a DVR owner-still-active veto:
  - target side is still materially below VWAP,
  - a profitable opposite-side exit exists during the same session,
  - the previous opposite token remains strongly above VWAP,
  - the previous opposite token has gained materially since its exit.
- The veto is symmetric for CE and PE.
- Long session memory is used only for the hard old-owner veto.
- The normal recent-opposite transfer ceremony keeps its shorter lookup so old wins do not delay valid later handovers.

Focused validation:

| Date | Result |
| --- | --- |
| 24-Apr r3 | Passed: only morning PE winner remained; CE loss trades were blocked |
| 16-Apr guard | Passed for PE winners; 14:10 CE miss remains a separate router materialization issue |
| 20-Apr guard | Early CE/PE/CE sequence preserved; 13:01 PE still missed for separate `spot_reset_missing` / `router_quality_failed` reasons |

24-Apr final focused evidence:

- Run reached `2026-04-24 13:15:05`.
- Orders:
  - `09:30:04 NIFTY26APR24100PE BUY 194.65`
  - `09:48:39 NIFTY26APR24100PE SELL 233.85`
  - Realized: `+56,056.00`
- No `DVR_RECOVERY_CE` orders appeared.
- `recent_opposite_owner_still_active` skips: `86`, from `10:53:07` to `12:56:40`.

Important non-fix:

- This does not solve the 20-Apr 13:01 PE missed opportunity.
- After splitting long owner memory from the normal ceremony, that PE is no longer blocked by stale CE ownership, but it still fails on `post_opening_spot_reset_missing` / `router_quality_failed`.
- That is a future DVR reservation-materialization problem, not part of the 24-Apr safety fix.

## Implemented Fix - 2026-06-21

Code status:

- Branch: `backtest-box-69`
- Commit: `e17f6e050b85bcabda6472c57bfc686cfdb79c22`
- Commit message: `Refine DVR recovery routing logic (follow-up to 17/20/21-Apr fix)`

Functional changes:

1. Range-scalp reversal audit mode:
   - `RANGE_SCALP_REVERSAL_MODE = "audit"` by default.
   - `RANGE_SCALP_CE_BREAKOUT` and `RANGE_SCALP_PE_BREAKDOWN` remain enforceable under strict guard.
   - `*_AT_SUP*` / `*_AT_RES*` reversal tags are logged but do not place orders unless explicitly promoted later.

2. DVR owner-transfer validation:
   - Added session-scoped owner-still-active detection.
   - A target-side DVR recovery cannot execute while the previously profitable opposite side is still VWAP-sponsored and expanding after exit.
   - The long memory is intentionally isolated from the normal owner-transfer ceremony to avoid delaying valid later handovers.

3. Replay/live parity:
   - The range-scalp selected-token guard now uses the replay tick timestamp rather than wall-clock time.
   - This avoids live/backtest divergence in guard freshness checks.

Current safe conclusion:

- 24-Apr CE losses are now correctly classified as false owner-transfer / old-PE-still-active.
- 23-Apr range-scalp reversal losses are now blocked/audit-only by design.
- 16-Apr and 20-Apr still expose opportunity-capture gaps, but those are separate from this safety patch:
  - 16-Apr afternoon CE: DVR materialization/quality gate too slow or too strict.
  - 20-Apr 13:01 PE: DVR reservation should carry long-window repair through reset-missing and materialize only after fresh proof.

Do not fold those enhancement fixes into the same patch; they need their own focused regression set.
