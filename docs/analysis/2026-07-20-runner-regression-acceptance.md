# Runner Regression Acceptance: April-May-July 2026

## Purpose

This ledger records replay acceptance against the raw market opportunities in
`2026-06-10-nifty-raw-benchmark-2026-04-16-to-2026-06-08.md`. The benchmark is
an observation source, never an input to live, paper, or replay decisions.

Acceptance rules:

- Evaluate runner, major-runner, and impulse-runner coverage without future-leg
  information.
- Reject date checks, benchmark lookahead, and date-specific threshold changes.
- Describe every correction as a symmetric market scenario and code invariant.
- Preserve hard risk stops and current-tick categorical safety vetoes.
- Re-run previously passing dates after any shared entry or exit authority change.
- Record exact entries, exits, tags, and realized P&L from a clean replay.

## Scope And Status

| Date | Benchmark focus | Status | Clean replay P&L | Notes |
|---|---|---|---:|---|
| 2026-04-16 | Bear runner, bear major runner, bull runner | PASS | +INR 226,307.25 | All three target legs covered; one valid late PE swing also captured |
| 2026-04-20 | Opening CE, morning PE/CE, afternoon PE runner | ENTRY/EXIT PASS TO 11:30 | +INR 129,275.25 focused | Three winning round trips; exact-token CE lease and natural PEL exit proved; afternoon/full-day coverage pending |
| 2026-04-23 | Opening CE impulse/runner, PE major runner, later runners | ENTRY PASS / EXIT PENDING | +INR 34,944 focused | Exact 24100 CE enters at 09:42; slice close is artificial, so natural exit remains under test |
| 2026-04-27 | Opening whipsaw, 09:28 PE swing, 09:39 CE impulse | PARTIAL PASS | +INR 1,462.50 focused | Bad CE blocked; PE profit-lock exits before hard-loss region |
| 2026-04-29 | Bull major runner, PE major runner | RCA | n/a | Current fast lane exposed premature CE episode and failed recovery |
| 2026-04-30 | PE runner, CE major runner | PENDING | n/a | Must verify CE retention and prevent post-exit churn |
| 2026-05-04 | Three PE runners | PENDING | n/a | Must separate recoverable opening pullback from stale retry authority |
| 2026-05-06 | PE runner, CE major runner | PASS | +INR 294,794.50 | Two winning round trips; PE retained through recoverable digestion, then transferred to CE as option participation reversed |

## 2026-04-16

Benchmark targets:

| Leg | Window | Direction | Points | Class |
|---:|---|---|---:|---|
| 2 | 09:25-09:59 | Bear | -124.3 | Runner |
| 4 | 11:08-12:07 | Bear | -173.7 | Major Runner |
| 13 | 14:03-14:51 | Bull | +102.6 | Runner |

Pre-fix regression observed on remote commit `ceb1617`:

| Entry | Exit | Contract | Result |
|---|---|---|---:|
| 09:32:30 | 09:51:01 | 24450 PE | +INR 30,712.50 |
| 12:02:30 | 13:51:04 | 24250 PE | -INR 25,691.25 |
| 14:11:43 | 14:55:31 | 24050 CE | +INR 48,675.25 |

The second PE had shifted from the working 11:10 exact-token entry to 12:02,
after the major runner was mature, and then held through the reversal into a
loss.

Clean acceptance on commit `b7ba3c0`:

| Entry | Exit | Contract | Entry tag | Exit tag | Realized P&L |
|---|---|---|---|---|---:|
| 09:32:30 | 09:51:01 | 24450 PE | TR_STABLE_RETRY | EXIT_LADDER_LOCK | +INR 30,712.50 |
| 11:10:54 | 12:11:01 | 24400 PE | TR_STABLE_RETRY | EXIT_LADDER_LOCK | +INR 112,320.00 |
| 14:11:34 | 14:55:31 | 24050 CE | TR_STABLE_RETRY | EXIT_LADDER_LOCK | +INR 48,509.50 |
| 14:55:56 | 15:14:59 | 24300 PE | TR_STABLE_RETRY | REPLAY_EOD_CLOSE | +INR 34,765.25 |

Outcome: all required runner windows were covered without a late entry or an
early structural exit. The 14:55 PE was also directionally valid for the
14:51-15:08 bear swing and proves that the 15:15 entry/close boundary is active.

Generic invariants exercised:

- Instrument expiry and lot size come from `market.instruments`; L4 does not
  hardcode Thursday expiry or a 50-unit lot.
- Market-Time Evidence Authority carries exact-token and immutable-wall proof
  across DVR, Momentum Ride, STABLE_RETRY, and TR_STABLE_RETRY arbitration.
- Position Exit Evidence holds an authenticated continuation while its original
  wall remains valid; a single L1/L2 or token-VWAP fluctuation cannot close it.
- Confirmed wall reclaim, corroborated opposite structure, earned profit locks,
  the hard loss stop, and EOD remain authoritative.

Evidence:

- Replay user: `btstructm1_20260416`
- Run: `local-structural-matrix-v1-20260720-2026-04-16`
- Full-day process result: `rc=0`
- Fusion events and debug logs were enabled.

## Cross-Regression Queue

Any production change found in the remaining dates must preserve the 16 April
exact-token entries, continuation holds, reversal exits, and final 15:15 close.
The next diagnostic order is 29 April, 30 April, 4 May, 6 May, then a clean
full-day 23 April recheck.

## 2026-04-20, 2026-04-23 And 2026-04-27 Authority Cross-Check

Base commit: `0002358d7c1a67527443964d4d2872a0cbb6e76d`.

### Entry Debt

Both 23 and 27 April produced the same causal defect. The anti-chase rejection
described underlying side geometry, but the token-scoped ledger forgot it when
the selector rolled strikes. The corrected central builder stores that category
as SIDE-owned evidence for both CE and PE. This is not a blanket side veto:
independent later market-time support can recover the episode.

Observed outcomes:

- 23 April: no order at the old `09:31:20` losing entry. The later CE
  reservation is not blocked by stale side debt; it is waiting on current
  multi-strike participation. Runner coverage remains open.
- 27 April: no order at the old `09:21:03` losing CE entry. The valid bear swing
  is admitted as `NIFTY26APR24100PE` at `09:33:00`.

### Exit Wall Identity

The 27 April PE lifecycle named a structural wall but had no immutable option
wall identity. Such a contract cannot call L4 reclaim authority and therefore
cannot own an indefinite L4 exit veto. PEL now retains the causal spot-probe
provenance while using ordinary fixed-bucket profit/reversal evidence.

Focused ledger:

| Entry | Exit | Contract | Entry tag | Exit tag | Realized P&L |
|---|---|---|---|---|---:|
| 09:33:00 | 09:43:40 | 24100 PE | TR_STABLE_RETRY | EXIT_LADDER_LOCK | +INR 1,462.50 |

The token peaked `+15.9%`; PEL stayed in HOLD while its live token structure was
supported, then recorded profit-lock evidence at `09:43:34` and `09:43:40`.
The second independent bucket authorized the close. This replaces the old
impossible-wall wait that carried the same position toward `-10%`.

### April 20 Control

The first version of the exit correction weakened the separate MTEA
replacement-position handshake and changed a profitable exit into a loss. A
paired source-frozen A/B run identified that interaction, and the handshake was
restored without restoring impossible wall authority.

Final paired result:

| Tree | Entry | Exit | Contract | Realized P&L |
|---|---|---|---|---:|
| Base `0002358` | 09:22:07 @ 199.90 | 09:30:38 @ 209.55 | 24250 CE | +INR 13,172.25 |
| Corrected | 09:22:07 @ 199.90 | 09:30:38 @ 209.55 | 24250 CE | +INR 13,172.25 |

This A/B failure-and-repair is retained deliberately: it proves that cross-day
acceptance, rather than making one date profitable, determined the final rule.

## 2026-04-23 Immutable-Arm Participation Check

The remaining CE miss was caused by measuring current strike participation from
a mature rolling window instead of the immutable reservation snapshot. That
window still included the preceding opposite leg and erased a genuine
continuation from its `strike_move` check.

The shared resolver now permits only that single check to use arm-relative
movement. Every current-tick path, giveback, strength, price-floor and VWAP
constraint remains authoritative. The exact token and symbol must match, and
the existing configured move requirement is unchanged.

Final isolated ledger:

| Entry | Natural exit | Contract | Entry tag | Exit tag | Realized P&L |
|---|---|---|---|---|---:|
| 09:42:25 @ 268.60 | 10:10:30 @ 294.35 | 24100 CE | TR_STABLE_RETRY | SPOT_OWNER_TRANSFER_ | +INR 26,780 |

The premium peaked at `319.75` at `10:05:25`, approximately `+19.0%` from
entry. The exit retained approximately `+9.6%` after the benchmark runner had
ended. PEL first kept the position through transient contrary snapshots, then
recorded corroborated reversal evidence in three independent market-time
buckets. `MTEA_FINAL_CLOSE_AUTHORITY.PASS` authorized the exact parent, token,
and symbol close at `10:10:30`; no source-local strategy forced the exit.

At `10:33:30`, the verifier entered `24300 PE` during the separate benchmark
`10:05-11:44` bear major runner. This was not post-exit churn, but it remained a
late entry. The configured verifier ended at `10:35`, so its `+INR 2,691`
`REPLAY_EOD_CLOSE` is artificial and is not natural-exit acceptance for that
second runner.

Generic invariants exercised:

- Current behavior is measured relative to the event that created the
  reservation, not an unrelated rolling-window origin.
- Sibling breadth cannot replace the exact reserved execution identity.
- Both reservation consumers use the same resolver.
- The event log records the arm price, current relative move, required move,
  native move verdict, relative move verdict, and token/symbol identity result.
- No date or benchmark data is available to production code.

Cross-regression:

- 27 April avoided the old opening CE loss, entered the valid `24100 PE` at
  `09:33:00`, and exited naturally at `09:43:40` for `+INR 1,462.50`.
- 4 May stayed flat through the old `09:28` CE-loss window and completed the
  `09:15-09:40` negative-control slice with zero orders.

The first transient acceptance launch ran as root with
`HOME=/home/administrator`, which created root-owned auxiliary logs and caused
the paper service to fail during logger import. That lane was stopped. Final
acceptance ran as `administrator` with an isolated HOME and a fresh `bt_*`
user. Paper log ownership was repaired and the service returned to a stable
websocket-connected state. Future transient replays must preserve that runtime
isolation invariant.

## 2026-07-25 Cross-Day Loss RCA

The remote diagnostic lanes on commit `add3484` were compared with the raw
benchmark only after their orders and evidence streams were preserved. The
benchmark supplied the opportunity windows below; it was not available to the
decision engine.

| Date | Preserved observation | Benchmark context | Shared causal finding |
|---|---|---|---|
| 2026-04-27 | `24100 PE` 09:33:40-09:45:41, -INR 46,182.50 | Entry occurred late inside the 09:28-09:39 bear fast swing; the day's major runner is CE 11:38-13:47 | A newer anti-chase rejection was erased by support accumulated before that rejection |
| 2026-06-08 | Two early PE losses before 09:53 | Opening CE impulse 09:15-09:34; benchmark runners begin at 12:51 | Same anti-chase/old-support interaction remains a cross-check target |
| 2026-06-17 | `24100 PE` 10:14:45-10:25:41, -INR 28,242.50 | CE runner 09:19-11:32 | A complete `selected_token_vwap_sponsorship_not_ready` verdict was stored as neutral history; one later packet was treated as a clean candidate |
| 2026-07-03 | `24400 PE` 10:16:21-10:49:03, -INR 28,080 | Bull slow leg 10:04-12:13; PE runner begins 12:13 | A complete `selected_token_sustained_move_missing` verdict did not create continuity recovery |
| 2026-07-09 | `23950 CE` 10:25:43-10:36:24, -INR 27,777.75 | Bear slow leg 10:01-10:37 | Repeated `extended_above_vwap_chase` packets remained non-authoritative until old support released the CE |
| 2026-06-02 | Diagnostic only | Benchmark is `DATA_GAP_SKIP / DATA_QUALITY_REVIEW` | The date cannot certify a strategy change until feed coverage is reconciled |

These are not six date-specific entry patterns. They reduce to three shared
invariants:

1. `selected_token_sustained_move_missing` and
   `selected_token_vwap_sponsorship_not_ready` are current-tick WAIT vetoes,
   but they start central continuity recovery. The next callback cannot call
   itself a clean candidate; three independent later market-time buckets must
   reproduce the same side and exact token.
2. `same_side_spot_chase_exhausted` and `extended_above_vwap_chase` are one
   typed anti-chase lifecycle family. Evidence collected before the latest
   rejection can prequalify the episode, but cannot recover it. Recovery
   requires later SIDE, exact TOKEN, and exact continuation-probe buckets.
3. A completed execution lease is exact geometry, not permission to skip the
   central ledger. It must consume categorical, continuity, anti-chase, and
   reservation-expiry recovery checks from the same source-neutral authority.

### 2026-07-27 June 8 Focused Acceptance

Run `local-jun08-mtea-loss-window-v2-20260727` replayed `09:15-10:05`
from `market.ticks` with full fusion evidence under user
`btj8mtea2_20260608`. This was a focused loss-window acceptance, not a
full-day runner certification.

- The former `09:36:51` PE loss did not execute. WPG's completed
  `block_opening_soft_repair_unconfirmed` verdict is now one lane-local
  `HARD_REJECT` identity for either CE or PE. Repeated packets dedupe, and a
  later candidate must earn three independent post-rejection market-time
  recovery buckets.
- There was no second early PE loss before `09:53`.
- One CE entered at `09:53:26` during benchmark leg 5
  (`09:50-09:59` Bull, `+70.9` points) and exited at `10:04:57` via
  `EXIT_LADDER_LOCK` for `+INR 273`.
- WPG calculations, scores, and numeric thresholds were unchanged. The fix is
  evidence taxonomy and authority ordering only.
- The order cutoff now consumes the explicit signal/replay timestamp before
  any operational clock fallback. A test proves a `15:14:59` replay tick
  remains entry-eligible even when the process clock is `17:00`.

### 2026-07-27 June 17 Focused Acceptance

Run `local-jun17-mtea-loss-window-v1-20260727` replayed `09:15-10:35`
with full fusion evidence under user `btj17mtea_20260617`.

- The correct `NIFTY2662323900CE` entered at `09:31:26`, twelve minutes into
  the benchmark's `09:19-11:32` CE runner.
- The former `10:14:45` wrong-side PE loss did not execute. The CE remained
  the authenticated owner through the entire old loss window.
- The diagnostic slice marked `+INR 48,041.50` at `10:35`. Its
  `REPLAY_EOD_CLOSE` is the configured slice boundary, not a strategy exit;
  full-day PEL retention remains to be certified separately.

The 27 April exit also exposed an independent capital-protection defect. The
code comment said the hard loss floor was unconditional, while the executable
condition disabled it whenever profit-lock state was active. Profit ladders may
tighten a stop; they cannot suppress the categorical floor.

The candidate implementation is symmetric for CE and PE and shared by DVR,
Momentum Ride, Stable Retry and TR Stable Retry. It changes no price, score,
VWAP, ROC, duration, support-count, or date threshold. Its isolated snapshot
passes 1,079 repository tests, including exact-token identity, three-bucket
market-time recovery, reset from the latest repeated anti-chase rejection,
execution-lease fail-closed behavior, and unconditional hard-floor tests.
Replay acceptance remains pending; no result in this section is labeled fixed
until clean April 16, April 20, and remote loss-day reruns pass.

## 2026-07-26 April 20 Final-Wall Authority Ordering

The source-frozen April 20 replay exposed a same-tick split decision rather than
a weak signal. At `09:54:58`, the central MTEA exact-token execution lease and
the candidate-reservation audit both authorized `NIFTY2642124200CE`. The final
Stable Retry wall helper then ignored that resolved candidate authority,
re-read the producer's older `ARMED_BEHIND_WALL` projection, and emitted
`wall_reservation_authoritative_wait`.

The shared final-wall helper now consumes the frozen authoritative candidate
resolution when one exists. It falls back to the immutable producer wall only
when the central resolver has not decided. Therefore:

- an authenticated exact-token execution lease can supersede only its own stale
  producer wall projection;
- an ordinary armed wall still blocks;
- a sibling token still fails identity;
- opening-wall, current premium, current L4, and other categorical vetoes remain
  authoritative;
- CE and PE use the same resolver and tests;
- no score, price, VWAP, ROC, duration, or support threshold changed.

Clean focused replay:

| Entry | Exit | Contract | Entry tag | Exit tag | Realized P&L |
|---|---|---|---|---|---:|
| 09:22:07 @ 199.90 | 09:30:38 @ 209.55 | 24250 CE | TR_STABLE_RETRY | EXIT_LADDER_LOCK | +INR 13,172.25 |
| 09:37:08 @ 195.85 | 09:53:50 @ 220.45 | 24450 PE | DVR_RECOVERY_PE_RUNNER | SPOT_OWNER_TRANSFER_ | +INR 35,178.00 |
| 09:54:58 @ 210.40 | 11:09:10 @ 272.65 | 24200 CE | TR_STABLE_RETRY | SPOT_OWNER_TRANSFER_ | +INR 80,925.00 |

The final CE peaked at `294.95` (`+40.2%` from entry). PEL held while the exact
token remained above an improving VWAP and the original wall held, then
authorized the natural close after three independent reversal buckets. The
candidate PE immediately after the close remained blocked because its premium
was still about 13% below token VWAP; no flip churn occurred through 11:30.

Evidence:

- Replay user: `bt_mtea_pel_natural_exit_20260420_v30`
- Run: `local-mtea-pel-natural-exit-20260420-v30`
- Process result: `rc=0`
- Orders: 6 rows, 3 buys, 3 sells
- Realized P&L: `+INR 129,275.25`
- Repository verification: 1,109 tests passed

This focused run certifies the morning authority and exit lifecycle only. The
afternoon PE runner and final full-day P&L remain part of the remote clean
cross-regression; they are not inferred from the benchmark.

## 2026-07-26 April 16 Entry-Liveness Control

Commit `2863dc7` was replayed from `09:15` through `12:30` after the required
six-table cleanup. This control targeted the shared liveness regression where
the second PE had moved from its historical `11:10` entry to `11:58-12:02` and
subsequently lost money.

| Entry | Exit | Contract | Entry tag | Exit tag | Realized P&L |
|---|---|---|---|---|---:|
| 09:32:15 @ 233.55 | 10:05:00 @ 278.45 | 24450 PE | TR_STABLE_RETRY | SPOT_OWNER_TRANSFER_ | +INR 52,533.00 |
| 11:10:34 @ 231.50 | 12:11:50 @ 326.90 | 24400 PE | TR_STABLE_RETRY | SPOT_OWNER_TRANSFER_ | +INR 111,618.00 |

The second PE reproduced the expected entry to within three seconds, reached a
`+52.5%` premium peak, and exited naturally with `+41.4%` premium capture.
This verifies that current exact-token SUPPORT is observed before the shared
authority contract is resolved and that the same contract remains available to
the position lifecycle. It does not rely on a date branch or a changed numeric
threshold.

Evidence:

- Replay user: `bt_mtea_v21_apr16_2863dc7`
- Run: `local-mtea-v21-apr16-2863dc7`
- Process result: `rc=0`
- Orders: 4 rows, 2 buys, 2 sells
- Realized P&L through 12:30: `+INR 164,151.00`
- Historical first-two-trade comparison: `+INR 178,834.50`
- Difference: `-INR 14,683.50`

This is an entry-liveness and continuation-ownership pass, not full-day
acceptance. The afternoon CE and full-day result remain part of the independent
remote replay.

## 2026-07-26 May 6 Underwater Exit Continuity

The clean opening replay distinguished a valid real-time entry from a position
exit defect. At `09:21:55`, all available opening-side and exact-token evidence
supported the `24250 PE`; the option continued from `207.95` to `213.30` while
NIFTY continued lower before reversing. The loss cannot honestly be removed by
using later benchmark knowledge or by tightening an entry threshold.

PEL did expose a generic continuity error after the reversal:

- opposite-owner reversal evidence reached two independent 10-second buckets;
- at `09:23:39`, opposite L2 and the adverse exact-token premium channels
  remained actionable, but L1 ownership was absent for one packet;
- the untyped WAIT erased the two-bucket reversal episode;
- L1 returned one second later, but the hard loss floor arrived before three
  buckets could be accumulated again.

The shared CE/PE classifier now emits
`underwater_opposite_owner_confirmation_wait` for that exact neutral state.
The reducer preserves an existing reversal vote through it without creating a
new vote. Same-side ownership, VWAP support, exact-token recovery, improving
sponsorship, or loss of opposite L2 still produces a real HOLD/reset. No price,
P&L, VWAP, ROC, duration, or bucket threshold changed.

Focused replay result:

| Entry | Exit | Contract | Entry tag | Exit tag | Realized P&L |
|---|---|---|---|---|---:|
| 09:21:55 @ 207.95 | 09:23:40 @ 187.60 | 24250 PE | TR_STABLE_RETRY | SPOT_OWNER_TRANSFER_ | -INR 27,777.75 |

The pre-fix run exited at `09:23:44` for `-INR 29,415.75`. The corrected event
stream is `reversal 1/3 -> reversal 2/3 -> neutral WAIT preserving 2/3 ->
reversal 3/3`, after which final close authority passed. This is a focused PEL
proof, not May 6 full-day acceptance; the benchmark PE and CE runners remain in
the independent full-day regression matrix.

Evidence:

- Replay user: `bt_mtea_v22_may06_pelwait`
- Run: `local-mtea-v22-may06-pelwait`
- Process result: `rc=0`
- Orders: 2 rows, 1 buy, 1 sell
- Focused realized P&L: `-INR 27,777.75`

## 2026-07-26 April 16 Full-Day Temporal Join Acceptance

The earlier April 16 control proved the second PE liveness correction but left
the afternoon CE uncertified: one replay did not enter until `14:31:41`, about
21 minutes after the preserved `14:10:31` reference.

The diagnostic trace exposed duplicate authority rather than a weak market
signal. Central MTEA had already accumulated independent SIDE and exact-token
support for the `24050 CE`, and the current Stable Retry route had passed its
live L1-L4, selected-token, whipsaw and CDE checks. The final discounted reducer
then discarded that temporal proof and started a second three-bucket ceremony
from zero. The duplicate ceremony, not a price threshold, caused the delay.

The shared final reducer now joins an existing central temporal quorum to one
complete current route. It fails closed unless all of the following remain
true:

- exact CE/PE token and symbol identity;
- current same-tick TOKEN SUPPORT from DVR, Momentum Ride, or Stable Retry;
- complete current route and exact source identity;
- fresh source-neutral MTEA history with all lifecycle checks passed;
- independent SIDE and exact-token quorum under the existing central bucket
  policy;
- no opposite reservation or unrecovered categorical debt;
- a prior exact-token probe and a non-deteriorating current quote.

No score, price, VWAP, ROC, duration, support-count, or date threshold changed.
The rule is symmetric for CE and PE and shared by all three automated entry
producers. Bounded log and `fusion_events` proof now records the temporal
checks, failed checks, bucket counts, exact identity, first qualified quote,
and current quote without copying the full nested cache snapshot.

Clean full-day result:

| Entry | Exit | Contract | Entry tag | Exit tag | Realized P&L |
|---|---|---|---|---|---:|
| 09:32:02 @ 234.90 | 10:05:00 @ 278.45 | 24450 PE | DVR_RECOVERY_PE_STABLE_RETRY_R | SPOT_OWNER_TRANSFER_ | +INR 50,953.50 |
| 11:10:34 @ 231.50 | 12:11:50 @ 326.90 | 24400 PE | TR_STABLE_RETRY | SPOT_OWNER_TRANSFER_ | +INR 111,618.00 |
| 14:11:29 @ 244.10 | 14:55:31 @ 289.00 | 24050 CE | TR_STABLE_RETRY | EXIT_LADDER_LOCK | +INR 49,614.50 |
| 14:55:56 @ 223.65 | 15:14:59 @ 251.80 | 24300 PE | TR_STABLE_RETRY | REPLAY_EOD_CLOSE | +INR 34,765.25 |

The corrected CE entry is 58 seconds after the preserved reference, instead of
21 minutes late. It covers the benchmark `14:03-14:51` bull runner, survives
the normal pullback, and exits after the option's `297.90` peak. The final PE
is aligned with the benchmark `14:51-15:08` bearish fast swing and is flattened
at the configured `15:15` cutoff.

Evidence:

- Replay user: `bt_mtea_v41_apr16_full`
- Run: `local-mtea-v41-apr16-full`
- Process result: `rc=0`
- Orders: 8 rows, 4 buys, 4 sells
- Realized P&L: `+INR 246,951.25`
- Focused pre-acceptance replay: `local-mtea-v40-apr16-ce`
- Symmetric route/counterexample suite: 303 tests passed
- Full repository suite: 1,122 tests passed
- `py_compile` and `git diff --check`: passed

Final combined-tree replay `bt_mtea_v44_apr16_final` reproduced all eight order
rows above exactly after the PEL correction was applied. The result remained
`4W / 0L`, with `+INR 246,951.25` realized. This closes the risk that the
April 20 exit correction could move the recovered April 16 entries or exits.

## 2026-07-26 April 20 Directional-Transfer Authentication

The `13:01:54` 24500 PE entered during the benchmark `12:52-13:27` bear
runner. Before this correction, three raw L1/L2 owner snapshots closed it at
`13:03:30` even though:

- opposite L3 was not actionable;
- opposite MTEA had no replacement-entry authority;
- the exact PE token was recovering; and
- no authenticated original-wall reclaim was available.

The legacy owner reconstruction is now audit-only while an exact MTEA position
lifecycle is active. Current PEL profit lock, loss invalidation, hard risk,
actionable opposite L3, and mature exact-token premium reversal remain
authoritative. No price or evidence-count threshold changed.

The corrected PE closed at `13:30:10` for `+INR 49,887.50`, rather than the
false `13:03:30` loss. The full replay completed with six round trips and
`+INR 116,038.00`. Two later CE losses remain separate entry-quality findings;
they were not hidden by changing exit behavior.

Evidence:

- Replay user: `bt_mtea_v43_apr20_pel`
- Run: `local-mtea-v43-apr20-pel`
- Process result: `rc=0`
- Orders: 12 rows, 6 buys, 6 sells
- Realized P&L: `+INR 116,038.00`

## 2026-07-26 April 27 PEL Cross-Regression

April 27 distinguishes a lucky legacy exit from a causal exit. The control
closed the 24100 PE at `11:41:10` after three L1/L2 owner votes. At that time
the option remained about 28% above its own VWAP, option ROC and VWAP ROC were
positive, and the premium subsequently recovered from about 157 to 170.

With directional transfer authenticated centrally, those packets remain WAIT.
The exact-token lifecycle later emitted three independent
`mtea_mature_peak_premium_reversal` buckets from `11:48:08` through
`11:48:20`; PEL then closed at 152.50 for `+INR 76,752.00`. This proves that
the correction does not trap the position: legacy direction is demoted, while
the existing exact-premium reversal channel remains live.

The following CE did not enter by the focused `12:05` cutoff. Its exact token
was still roughly 14% below VWAP and had negative live ROC when the execution
lease was released, so the current final route correctly refused to convert
historical support into an order. This remains an entry-quality/missed-runner
case for the loss-date matrix; it is not repaired by restoring the legacy
owner vote or weakening a threshold.

Evidence:

- Replay user: `bt_mtea_v46_apr27_pel_rca`
- Run: `local-mtea-v46-apr27-pel-rca`
- Process result: `rc=0`
- Orders through 12:05: 8 rows, 4 buys, 4 sells
- Realized P&L through 12:05: `+INR 165,148.75`

## 2026-07-27 May 6 PE-to-CE Handoff Acceptance

The prior full replay held the `24200 PE` until `13:17:08`. It donated a
`+50.63%` option peak down to `+24.45%` and kept ownership during the following
CE major runner. The same position also experienced a superficially similar
pullback at `11:29`, but that earlier pullback recovered and the PE later made
its session peak. A safe correction therefore could not simply tighten the
profit ladder.

The PEL release rule now distinguishes canonical structural walls from
geometric provenance walls. A canonical immutable wall remains authoritative.
For a noncanonical lifecycle, exact-token VWAP sponsorship can be released only
when the earned floor is breached, the live same-side index runner has ended,
premium ROC and VWAP gap are deteriorating, and existing opposite-owner/L2
evidence reaches three independent market-time buckets. The rule is symmetric
for CE and PE and changes no score, price, VWAP, ROC, duration, or bucket
threshold.

The clean trace proved both sides of that invariant:

- At `11:29:08`, one reversal bucket appeared. Renewed same-side runner and
  exact-token sponsorship restored HOLD, so no false SELL occurred.
- The PE premium peaked at `13:04:17` at `309.55`, after the index benchmark
  reversal. Nearby CE premiums were still falling through that timestamp.
- From `13:08:07`, independent mature-premium reversal buckets accumulated.
  PEL authorized exit at `13:08:40`, and the CE entered 24 seconds later.
- The CE remained held through the benchmark `12:53-14:28` major runner and
  exited during the subsequent bearish pullback. No later churn appeared.

Clean full-day result:

| Entry | Exit | Contract | Entry tag | Exit tag | Realized P&L |
|---|---|---|---|---|---:|
| 10:29:32 @ 205.50 | 13:08:40 @ 279.70 | 24200 PE | TR_STABLE_RETRY | SPOT_OWNER_TRANSFER_ | +INR 101,283.00 |
| 13:09:04 @ 305.65 | 14:33:44 @ 518.30 | 23900 CE | TR_STABLE_RETRY | EXIT_LADDER_LOCK | +INR 193,511.50 |

The CE was worth `518.30` at exit, after reaching `578.70`; at the benchmark
14:28 endpoint it remained above `+80%` from entry. The complete day contained
exactly two round trips and realized `+INR 294,794.50`, compared with the
broken `-INR 18,379` regression.

Evidence:

- Replay user: `bt_mtea_v60_may06_pel_handoff`
- Run: `local-mtea-v60-may06-pel-handoff`
- Process result: `rc=0`
- Orders: 4 rows, 2 buys, 2 sells
- Realized P&L: `+INR 294,794.50`
- Focused PEL suite: 90 tests passed
- Evidence-cache suites: 200 tests passed
- Fusion-route suites: 304 tests passed
- Full repository suite: 1,133 tests passed
- `py_compile` and `git diff --check`: passed
