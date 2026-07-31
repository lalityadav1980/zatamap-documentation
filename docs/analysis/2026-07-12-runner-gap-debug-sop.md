# Runner Gap Debug SOP

Use this workflow before changing runner logic. Dates are microscopes, not tuning targets: classify the failure class, patch that class, then prove it on the smallest replay window that recreates the required state.

## Objective

For every benchmark Runner, Major Runner, or Impulse Runner, identify whether the engine had:

- No evaluation.
- An open-position handoff block.
- Candidate reservation or materialization block.
- L4 wall/tight-range block.
- Recent-failure or cooldown block.
- Late entry.
- Early exit.
- Wrong-side overlap.

Only fix structural classes that can repeat across days. Do not add date-specific thresholds.

## Non-Negotiable Fix Invariant

- A date may appear in RCA comments, tests, and benchmark evidence, but never in a
  runtime condition. Dates identify where a structural defect was observed; they
  do not define the behavior being implemented.
- Read the surrounding guard comments before editing. Preserve the original loss
  case as a negative constraint and explain why the new proof does not reopen it.
- Do not move a numeric threshold merely to make one benchmark leg enter or exit.
  A threshold change is a separate strategy proposal and requires distributional
  evidence across protected trend and chop days, explicit review, and regression
  results before implementation.
- Prefer repairing proof ownership and gate consistency: when an earlier ceremony
  has completed, later gates must either honor that exact proof or reject it with
  a genuinely stricter, named structural requirement. Do not stack silent retries
  of the same geometry under different variable names.
- Every behavior change must include a code comment naming the failure class, the
  representative RCA evidence, why the old guard exists, and which protections
  remain active. Add positive and negative tests for the proof class.
- A target day passing is necessary but never sufficient. Re-run every protected
  date named by comments touched, then run the broader benchmark gate before the
  change can be promoted.

## Inputs

- Benchmark leg: date, start/end time, direction, points, class.
- Orders/trail data for that user/date.
- `trade.fusion_events` for the benchmark window plus the previous open-position exit window.
- App log for the same replay/paper run.
- NIFTY spot ticks and selected option premium path if the blocker is unclear.

## Precise Debug Loop

1. Anchor the benchmark leg and expected side.
2. Pull orders from 5 minutes before leg start to 5 minutes after leg end.
3. If a position was open, pull its exit reason and timestamp first; owner-transfer bugs require prior state.
4. Pull `fusion_events` for the expected side from leg start through the first missed-entry point.
5. Search the app log for the same timestamps and expected side.
6. Classify the first terminal blocker, not every downstream symptom.
7. Patch only that blocker class and preserve existing dated guard comments as regression constraints.
8. Run a targeted stateful replay that starts at the last required prior state, not the full day.
9. Validate the protected regression day named by any comment touched.
10. Run a full-day replay only after the target window proves the fix.

## Replay Efficiency Rules

- Prefer `--log-profile normal` for diagnostic replays: it keeps `fusion_events` and order lifecycle evidence without the high-volume rendered dashboards.
- Use `--log-profile debug` only for a very small window after the blocker class is known.
- For stateful windows, use `--log-profile fast` plus `--diagnostic-start-time` / `--diagnostic-end-time`. This lets the engine build state from earlier ticks while persisting `fusion_events` only during the RCA window.
- If the replay child disappears while the wrapper remains alive, stop the wrapper and treat the run as invalid infrastructure evidence, not strategy evidence.
- For stateful bugs, build or use a checkpoint at the prior required state. Example: save immediately after a `SPOT_OWNER_TRANSFER_` exit, then replay only the next 10-20 minutes for owner-transfer materialization fixes.
- Do not repeat a full-morning replay more than once for the same blocker. If one full replay is needed to create state, the next iteration must use saved state, direct fusion_events/log RCA, or a small harness.

## Useful Queries

```sql
SELECT order_timestamp, exchange_timestamp, parent_order_id, tradingsymbol,
       transaction_type, price, quantity, tag, matched_pnl
FROM trade.orders_book
WHERE zerodha_id = '<user>'
  AND exchange_timestamp BETWEEN '<start_ist>' AND '<end_ist>'
ORDER BY exchange_timestamp;
```

```sql
SELECT event_ts, side, event_type, action, symbol, left(reason, 260) AS reason
FROM trade.fusion_events
WHERE user_id = '<user>'
  AND event_ts BETWEEN '<start_ist>' AND '<end_ist>'
  AND (side = '<CE_OR_PE>' OR reason ILIKE '%<CE_OR_PE>%')
ORDER BY event_ts;
```

```bash
rg -n "12:4[1-9]|12:5[0-9]|CANDIDATE_RESERVATION|MATERIALIZATION_BLOCK|BASKET_RUNNER|SPOT_OWNER_TRANSFER" <app.log>
```

## Replay Policy

Use a stateful slice when the target bug depends on a prior position. Example: for 2026-05-13 leg 12, start before the 12:20 PE entry so the PE can exit via `SPOT_OWNER_TRANSFER_`, then inspect the 12:41-12:59 CE impulse. A full 09:15 replay is not needed for each iteration.

If the target path requires multiple prior trades before the failing leg, do not keep re-running from market open. First capture the failure once with `fusion_events`; then either seed the relevant internal state in a small harness or add checkpoint/resume support before continuing iteration.

Clear local replay data before each fresh proof run unless explicitly preserving a comparison:

```sql
DELETE FROM trade.trail_summary_day;
DELETE FROM trade.trail_data;
DELETE FROM trade.orders_book;
DELETE FROM trade.orders_tracker;
DELETE FROM trade.fusion_events;
```

Example stateful RCA replay:

```bash
python3 tools/run_timescale_multiday_backtest.py \
  --dates 2026-05-13 \
  --user-id btw_0513_leg12_rca \
  --clone-profile-from btw_0513_runners \
  --start-time 11:45:00 \
  --end-time 13:05:00 \
  --diagnostic-start-time 12:41:00 \
  --diagnostic-end-time 13:05:00 \
  --log-profile fast \
  --reset-output --reset-output-scope user --no-global-reset
```

This is not a fake memory snapshot. The replay still builds real open-position and TSL state from `--start-time`, but optional diagnostic persistence is windowed to the benchmark leg.

## 2026-05-13 Leg 12 RCA Breadcrumb

Benchmark leg: `12:41-12:59 Bull +151.5`, expected CE impulse.

Observed stateful replay:

- PE runner entered at `12:20:03`.
- PE exited profitably at `12:45:19` via `SPOT_OWNER_TRANSFER_`.
- CE was evaluated before and after the PE exit.
- Before exit, CE was correctly watched because the opposite PE was open.
- After exit, CE STABLE_RETRY was blocked by `extended_above_vwap_chase`, wall-hold revalidation, and recent-failure revalidation despite fresh CE basket expansion.

Structural class: `post_owner_transfer_impulse_materialization`. The fix must not weaken generic above-VWAP chase protection; it should only release when a profitable owner-transfer exit is followed by fresh opposite-side spot extension, multi-strike basket confirmation, and bounded selected-token participation.

## Future Structural Program: Runner Proof Contract

The broader issue is that the engine has many independent defensive gates, but no single authoritative runner proof that every downstream gate must respect. A signal can be valid at candidate reservation, then die later as a generic chase, session-extreme block, cooldown, or selected-token stall. This is why runner fixes currently feel day-by-day.

After the immediate 2026-05-13 owner-transfer fix is stable, create a first-class runner proof contract with these scenario classes:

- `owner_transfer_runner`: opposite side exited via owner-transfer, new side proves fresh spot extension plus same-side option participation.
- `extended_but_valid_runner`: selected token is above VWAP but bounded, digesting, and confirmed by basket/spot proof.
- `no_materialization_runner`: L1/L2/L3 show a trend, but no candidate becomes executable.
- `late_opening_runner`: opening runner is detected after the profitable part has already passed.
- `early_exit_runner`: entry was right, but ladder/owner-transfer exits gave back too much or exited before benchmark continuation.
- `wrong_side_transition`: system opens or holds the wrong side while benchmark/tape has flipped.

Implementation direction:

- Emit one normalized `runner_proof` payload from the first layer that proves the runner.
- Require every later gate to either honor that proof or write a named veto with its own stricter proof requirement.
- Preserve existing dated comments as regression breadcrumbs; any change touching a comment must replay that protected date.
- Build a coverage report that classifies each benchmark Runner/Major Runner/Impulse Runner as captured, late, early-exit, wrong-side, or missed.
