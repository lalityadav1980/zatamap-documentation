# Whipsaw Participation Gate Toggle

This gate is a reversible experiment for `TR_STABLE_RETRY` entry materialization.
It checks whether the selected option token is actually participating in the
current whipsaw leg before `process_order` is called.

## Modes

Set `WHIPSAW_PARTICIPATION_GATE_MODE` before starting app/backtest:

- `off`: no audit and no behavior change.
- `audit`: write `fusion_events` evidence only, no trade behavior change.
- `enforce`: block entries classified as late chase, fading token path, weak VWAP
  sponsorship, low token quality, path-not-ready, or side-confirmation-pending.

Example:

```bash
export WHIPSAW_PARTICIPATION_GATE_MODE=enforce
```

Emergency rollback:

```bash
export WHIPSAW_PARTICIPATION_GATE_MODE=off
```

## What It Audits

The audit payload is stored under `payload.extra.whipsaw_participation_gate` in
`trade.fusion_events`.

It records:

- selected side, symbol, token
- selected-token premium path: move, ROC, giveback from peak
- option VWAP sponsorship: gap, gap change, ticks above VWAP, sponsorship reason
- snapshot quality: score, tier, premium ROC
- side evidence: side-adjusted L2 score, L3 total/edge, opposite-side status
- wall pressure toward the target side
- verdict: `participate`, `watch_path_not_ready`,
  `block_late_chase_exhausted`, etc.

## Wall-Hold Promotion Safety

The gate can promote a candidate through older materialization blocks only when
the promotion evidence matches the block type. A `wall_holding_revalidation` or
`no_breakout_confirmation` block is stricter than a plain recent failure:
strong VWAP alone is not enough. The selected token must also show strong
snapshot quality, premium expansion, wall/breakout pressure, and token movement
before the gate can clear a wall-holding block.

This protects late CE/PE candidates near a respected wall where the option is
above VWAP but the structure still says "no breakout."

## Delete Path

If the experiment fails, remove:

- `feature_toggles.py`
- `whipsaw_participation_gate.py`
- the `WHIPSAW_PARTICIPATION_GATE` block in `fusion_signals.py`

The feature is intentionally isolated so removal does not require redesigning
the rest of the signal engine.
