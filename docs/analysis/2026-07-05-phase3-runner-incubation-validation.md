# 2026-07-05 Phase 3 Runner Incubation Validation

Branch: `codex/runner-coverage-consolidation-20260705`

Scope: local-only validation for the runner coverage consolidation work. This
phase targets premature ladder exits during structurally valid NIFTY runner
formation. It does not tune for a date label; it uses live/replay index ticks,
option VWAP sponsorship, and side-symmetric CE/PE rules.

## Structural Problem

The engine already had an active index-runner hold that could defer
`EXIT_LADDER_LOCK`, but it only activated after the NIFTY move had fully matured.
On 2026-05-13, the CE leg was already VWAP-sponsored and the index was building a
same-side runner, yet the strict runner threshold was not mature at the first
pullback. The old path released the ladder floor at 09:56 and exited the CE
before the runner continued.

The full-day replay shows this is broader than a single ladder-exit bug. The
09:40 CE leg is both late-entry and early-exit, and the rest of the day exposes
the same missing lifecycle concept in different forms: missed runner conversion,
late stale entries, and wrong-side overlap into the next benchmark leg.

## Code Changes Under Validation

- Backtest TSL hook now starts replay TSL from the actual `process_order` success
  payload shape. Earlier local replay could under-test the production TSL path.
- A runner-incubation hold was added before the strict active-runner hold:
  VWAP-sponsored entries can defer low-rung ladder-floor release while the NIFTY
  same-side runner is forming.
- The incubation hold has a bounded grace window, side-symmetric CE/PE logic, and
  does not overrule breakeven, hard stop, or confirmed transfer exits.
- Unit coverage was added for relaxed pre-active runner detection and digestion
  tolerance.

## 2026-05-13 Leg 4 Validation

Benchmark target leg:

| Leg | Time | Direction | Points | Class |
| --- | --- | --- | ---: | --- |
| 4 | 09:40-10:34 | Bull | +207.0 | Major Runner |

Baseline behavior from the workshop run:

| Trade | Entry | Old Exit | Old PnL | Failure Class |
| --- | --- | --- | ---: | --- |
| `NIFTY2651923150CE` | 09:52:36 | 09:56:02 | ~Rs 9.4k | EARLY_EXIT |

Final local replay:

Run id:
`phase3-0513-v17-final-candidate-incubation-20260705`

User id:
`bt_phase3_0513_20260705`

| Trade | Entry | Exit | Matched PnL | Result |
| --- | --- | --- | ---: | --- |
| `NIFTY2651923450PE` | 09:30:16 | 09:42:30 | Rs 51,935.00 | unchanged from accepted baseline |
| `NIFTY2651923150CE` | 09:52:36 | 10:10:42 | Rs 21,914.75 | improved CE runner hold |

The CE trade held roughly 14 minutes longer than the old 09:56 exit and improved
matched PnL by about Rs 12.5k. This confirms the incubation guard is addressing
the intended failure mode without changing the earlier PE trade on this day.

However, this is only a partial improvement, not a complete runner capture. The
benchmark gate still classifies leg 4 as `EARLY_EXIT` because the trade exited
at 10:10:42, about 23 minutes before the benchmark leg ended at 10:34.

Premium/VWAP check for the selected CE confirms the option data was valid and
the runner had more premium available:

| Timestamp | NIFTY | `NIFTY2651923150CE` LTP | CE VWAP | Note |
| --- | ---: | ---: | ---: | --- |
| 09:40:00 | 23275.50 | 340.25 | 393.68 | benchmark leg start |
| 09:52:36 | 23341.75 | 385.90 | 377.19 | actual entry, already +13.42% from leg start |
| 09:56:02 | 23367.10 | 399.10 | 379.45 | old early-exit point |
| 10:10:42 | 23397.50 | 416.55 | 387.56 | new exit |
| 10:28:02 | - | 473.60 | - | selected CE max during leg |
| 10:34:00 | 23456.00 | 460.75 | 397.28 | benchmark leg end |

Leg 4 diagnosis after premium check:

- `LATE_ENTRY`: the engine entered 12.6 minutes after the benchmark leg start,
  after the selected CE had already moved +13.42%.
- `EARLY_EXIT`: the new hold still exited 23.3 minutes before leg end; the
  selected CE later reached +22.73% from entry and was still +19.39% from entry
  at leg end.
- `DATA_OK`: this is not a fake premium or bad tick issue. The CE traded above
  VWAP after entry and had meaningful continuation.

Important diagnostic near the final CE exit:

- `INDEX_RUNNER_LADDER_HOLD` deferred exits while strict same-side CE runner was
  active.
- The final exit occurred when strict runner state dropped to `candidate` because
  recent adverse moved just beyond the active cap.
- A looser recent-adverse experiment was tested and rejected because it worsened
  the earlier PE behavior on the same day.

## Rejected Experiment

An experiment scaled the active-runner recent-adverse cap with extension. It was
reverted because it made 2026-05-13 worse overall: the earlier PE was held too
long and the CE did not improve enough to justify the extra risk. This branch
therefore keeps the bounded incubation hold, not a broad mature-runner cap
relaxation.

## Remaining Work

This phase improves the 2026-05-13 early-exit class, but it does not claim the
runner lifecycle is complete. The next protected regression set should verify
that the bounded incubation rule does not break chop/whipsaw protection:

- 2026-05-04: whipsaw/high opportunity, protects against short-lived CE/PE flips.
- 2026-05-08: mixed-bias runner with chop.
- 2026-05-14: benchmark-skip/data-quality day with major runners.
- 2026-04-21: same-side profit re-entry guard protection day.
- 2026-04-29: major-runner coverage day.
- 2026-06-23, 2026-06-24, 2026-06-29: June runner coverage follow-up set.

## 2026-05-13 Full-Day Final Gate

The final replay completed through 15:30 IST with 6 entries, 6 exits, and
`Rs 19,841.25` matched PnL. That looks profitable in isolation, but the runner
coverage gate shows the system under-covered the documented opportunity:

| Leg | Class | Side | Benchmark Window | Status | Current Diagnosis |
| ---: | --- | --- | --- | --- | --- |
| 3 | Impulse Runner | PE | 09:24-09:40 | `CAPTURED` | Opening PE was covered by the 09:30-09:42 PE trade. |
| 4 | Major Runner | CE | 09:40-10:34 | `EARLY_EXIT` | Improved vs baseline, but still exits too early and misses remaining CE premium. |
| 7 | Impulse Runner | PE | 10:45-10:59 | `MISSED` | PE impulse was not converted into an order. |
| 8 | Major Runner | CE | 10:59-11:45 | `LATE_ENTRY` | CE entered at 11:42, after most of the leg had already played out, and later stopped out. |
| 11 | Runner | PE | 12:14-12:41 | `WRONG_SIDE_OVERLAP` | Late CE from leg 8 remained open into the PE runner window until 12:36. |
| 12 | Impulse Runner | CE | 12:41-12:59 | `LATE_ENTRY` | CE entered at 12:56, near the end of the impulse, then stopped during the following PE runner. |
| 13 | Runner | PE | 12:59-13:28 | `WRONG_SIDE_OVERLAP` | The late CE from leg 12 remained open into this PE runner until 13:17. |
| 21 | Runner | PE | 14:50-15:22 | `EARLY_EXIT` | A PE was open before this runner, but EOD close at 14:59 captured only 31% of the benchmark window. |

Final coverage summary:

| Status | Count |
| --- | ---: |
| `CAPTURED` | 1 |
| `EARLY_EXIT` | 2 |
| `LATE_ENTRY` | 2 |
| `MISSED` | 1 |
| `WRONG_SIDE_OVERLAP` | 2 |

Final trade tape:

| Trade | Entry | Exit | Matched PnL | Diagnosis |
| --- | --- | --- | ---: | --- |
| `NIFTY2651923450PE` | 09:30:16 | 09:42:30 | Rs 51,935.00 | Captured opening PE impulse. |
| `NIFTY2651923150CE` | 09:52:36 | 10:10:42 | Rs 21,914.75 | Improved but still early exit on leg 4. |
| `NIFTY2651923350CE` | 11:42:22 | 12:36:46 | Rs -26,715.00 | Late leg-8 chase that overlapped wrong-side PE leg 11 and stopped out. |
| `NIFTY2651923450CE` | 12:56:53 | 13:17:38 | Rs -28,301.00 | Late leg-12 chase that overlapped wrong-side PE leg 13 and stopped out. |
| `NIFTY2651923550PE` | 13:46:17 | 14:15:01 | Rs -864.50 | PE attempt during post-runner chop; gave back from +12.72% max to flat/negative. |
| `NIFTY2651923600PE` | 14:19:06 | 14:59:59 | Rs 1,872.00 | Early relative to leg 21, but EOD close ended runner coverage after only 31% of the benchmark window. |

This means Phase 3 should continue as a full-day lifecycle fix, not another
threshold-only adjustment. The next structural target is runner ownership:
detect forming runners early enough to enter, avoid stale entries after the
runner is already mature, hold while same-side runner ownership persists, and
release or reverse cleanly when the benchmark-side runner changes.

Artifacts:

- Coverage report: `outputs/phase3-0513/runner_coverage_final.md`
- Trade tape: `outputs/phase3-0513/trade_tape.csv`

## 2026-05-13 Phase 5 Update: DVR Runner Handoff

Commit:
`c6dc823 Capture 13-May DVR runner handoff`

The Phase 3 incubation work fixed only part of 2026-05-13. The unacceptable
remaining structural miss was leg 11:

| Leg | Time | Direction | Points | Class | Previous Failure |
| ---: | --- | --- | ---: | --- | --- |
| 11 | 12:14-12:41 | Bear | -86.0 | Runner | PE runner blocked by stale opposite-side ownership / handoff proof gap |

The fix is intentionally not date-specific. It adds a trace-scoped DVR handoff
proof that must pass through the signal layer, CDE, and final order ownership
guard. The order guard only accepts the handoff when all of these are true:

- The proof is tied to the exact trace id, token, side, and runner tag.
- DVR has fresh same-side recovery metrics, not just an old premium bounce.
- The requested side has fresh NIFTY follow-through.
- The previous owner is weakening in a runner-compatible way.
- The same rule path is side-symmetric for CE and PE.

Final local replay:

| Field | Value |
| --- | --- |
| Run id | `phase5-0513-final2-20260705` |
| User id | `bt_phase5_0513_final_20260705` |
| Orders | 14 rows, 7 entries, 7 exits |
| Matched PnL | Rs 153,468.25 |

Final 2026-05-13 trade tape:

| Entry | Exit | Symbol | Entry Tag | Exit Reason | Matched PnL |
| --- | --- | --- | --- | --- | ---: |
| 09:30:16 | 09:42:06 | `NIFTY2651923450PE` | `MOMENTUM_RIDE_PE` | `EXIT_LADDER_LOCK` | Rs 53,868.75 |
| 09:52:36 | 10:39:42 | `NIFTY2651923150CE` | `DVR_RECOVERY_CE` | `EXIT_LADDER_LOCK` | Rs 94,480.75 |
| 10:49:45 | 11:03:16 | `NIFTY2651923600PE` | `DVR_RECOVERY_PE_RUNNER` | `SPOT_OWNER_TRANSFER_` | Rs 114,712.00 |
| 11:38:17 | 11:50:06 | `NIFTY2651923350CE` | `TR_STABLE_RETRY` | `SPOT_OWNER_TRANSFER_` | Rs 124,640.75 |
| 12:20:02 | 12:45:19 | `NIFTY2651923600PE` | `DVR_RECOVERY_PE_RUNNER` | `SPOT_OWNER_TRANSFER_` | Rs 152,460.75 |
| 13:46:17 | 14:15:01 | `NIFTY2651923550PE` | `DVR_RECOVERY_PE_STABLE_RETRY_R` | `EXIT_LADDER_LOCK` | Rs 151,596.25 |
| 14:19:06 | 14:59:59 | `NIFTY2651923600PE` | `TR_STABLE_RETRY_R` | `REPLAY_EOD_CLOSE` | Rs 153,468.25 |

Runner coverage after the Phase 5 handoff fix:

| Status | Count |
| --- | ---: |
| `CAPTURED` | 5 |
| `LATE_ENTRY` | 1 |
| `MISSED` | 1 |
| `WRONG_SIDE_OVERLAP` | 1 |

Captured legs:

| Leg | Benchmark Window | Side | Class | Actual Coverage |
| ---: | --- | --- | --- | --- |
| 3 | 09:24-09:40 | PE | Impulse Runner | 09:30-09:42 PE |
| 4 | 09:40-10:34 | CE | Major Runner | 09:52-10:39 CE |
| 7 | 10:45-10:59 | PE | Impulse Runner | 10:49-11:03 PE |
| 11 | 12:14-12:41 | PE | Runner | 12:20-12:45 PE |
| 21 | 14:50-15:22 | PE | Runner | 14:19-14:59 PE, clipped by EOD |

Remaining 13-May gaps were deliberately classified rather than loosened blindly:

- Leg 8 CE remains `LATE_ENTRY`; this is a broader stale-entry/post-profit-roll
  class and needs cross-day regression before changing.
- Leg 12 CE remains `WRONG_SIDE_OVERLAP`; the PE runner was correctly held until
  12:45 and the later CE proof looked extended/chased.
- Leg 13 PE remains `MISSED`; inspected events did not show enough same-side
  proof, premium alignment, and opposite-side fading to justify another PE rule.

This closes the structural 13-May DVR handoff miss without overfitting the
remaining whipsaw legs. Those remaining classes should stay on the regression
gap list until they repeat across multiple protected days.

Artifacts:

- Coverage report: `outputs/phase5-0513-final2/runner_coverage.md`
- Trade tape: `outputs/phase5-0513-final2/trade_tape.csv`
