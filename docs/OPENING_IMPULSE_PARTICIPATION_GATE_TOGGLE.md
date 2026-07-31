# Opening Impulse Participation Gate Toggle

This is a reversible audit-first path for the first 15 minutes after market
open. It does not remove the session-age gate. It observes candidates that are
currently blocked by `SESSION_AGE_GATE` and can later bypass that block only
when option-token sponsorship, VWAP/reclaim, premium expansion, L2/L3 side
alignment, and underlying impulse are all clean.

## Modes

Set this before starting app/backtest:

```bash
export OPENING_IMPULSE_PARTICIPATION_GATE_MODE=off
export OPENING_IMPULSE_PARTICIPATION_GATE_MODE=audit
export OPENING_IMPULSE_PARTICIPATION_GATE_MODE=enforce
```

- `off`: no audit and no behavior change.
- `audit`: write `fusion_events` evidence only.
- `enforce`: only verdict `opening_impulse_participate` may bypass the
  15-minute session-age gate.

Emergency rollback:

```bash
export OPENING_IMPULSE_PARTICIPATION_GATE_MODE=off
```

## What It Audits

The payload is stored under
`trade.fusion_events.extra.opening_impulse_participation_gate`.

It records:

- selected side, symbol, token
- session age versus configured session-age wait
- selected-token premium path: move, ROC, giveback
- option VWAP sponsorship: gap, gap change, ticks above VWAP, sponsorship reason
- L2 side-adjusted score and conviction
- L3 total, edge, buy_ok, and fail reasons
- L4 verdict/block state
- NIFTY ROC/slope impulse
- verdict such as `opening_impulse_participate`,
  `watch_data_warmup`, `watch_opening_impulse_late`,
  `watch_vwap_not_clean`, or `watch_premium_path_not_clean`

## Default Guardrails

The detector is intentionally stricter than the normal whipsaw gate:

- session window: 5 to 15 minutes after open by default
- first five minutes stay watch-only so the engine has real option/VWAP data
- L2 side score must be high conviction and at least `0.92`
- L3 total must be at least `0.74`
- selected token must be VWAP sponsored or reclaiming VWAP cleanly
- premium path must be expanding with low giveback
- underlying impulse must be present
- if the last-30s NIFTY ROC has paused, aligned trend can still pass only when
  selected-token move, path ROC, VWAP repair, and slope remain strong

These are environment-overridable with `OPENING_IMPULSE_*` variables, but the
first validation pass should run in `audit` before any enforcement.

## Delete Path

If the experiment fails, remove:

- `opening_impulse_participation_gate.py`
- this document
- the `OPENING_IMPULSE_PARTICIPATION_GATE` block in `fusion_signals.py`

The path is isolated so it can be deleted without changing the rest of the
engine.
