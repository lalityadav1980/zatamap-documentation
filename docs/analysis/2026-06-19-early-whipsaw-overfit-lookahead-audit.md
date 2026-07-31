# Early/Whipsaw Gate Overfit And Lookahead Audit

Date: 2026-06-19

Scope: verify whether the separate opening-impulse, whipsaw-participation, and discounted-VWAP recovery audit work is day-specific, overfit, or using future data.

## Current Evidence

Active validation run:

```text
tmux: full-dvr-audit-r4
run root: .trade-api-runs/full-2026-04-16-to-2026-06-18-dvr-audit-r4
mode: DISCOUNTED_RECOVERY_DETECTOR_MODE=audit
speed: 0
```

20-Apr current observed orders while replay was in progress:

```text
09:19:57 BUY  NIFTY2642124200CE @ 209.35 TR_STABLE_RETRY
09:26:23 SELL NIFTY2642124200CE @ 247.45 EXIT_TARGET_VWAP, pnl +49,530
09:30:53 BUY  NIFTY2642124450PE @ 184.90 TR_STABLE_RETRY
```

This means the opening impulse was not missed by the early detector in the clean no-CSV path. The first CE entered inside the 09:16-09:25 benchmark impulse and exited profitably.

## No Lookahead Found In Current Gate Path

The following checks were performed:

```text
rg for hard-coded 2026-04-20, target symbols, benchmark/future fields in:
opening_impulse_participation_gate.py
whipsaw_participation_gate.py
discounted_recovery_detector.py
fusion_signals.py
order_service_api.py
db_backtest_runner.py
option_chain_main.py
```

Findings:

- `opening_impulse_participation_gate.py` uses only runtime features: selected side, selected token path, VWAP state, L2/L3/L4 evidence, trend, and session minutes.
- `whipsaw_participation_gate.py` uses only runtime features: selected token premium path, VWAP sponsorship, L2/L3/snapshot/wall evidence, and open-position context.
- `discounted_recovery_detector.py` is audit-only and observes incoming option ticks. It does not place orders.
- `benchmark_leg_annotator.py` is used only by DVR audit payloads in `fusion_signals.py`; no order path imports it or uses benchmark fields for entries/exits.
- CSV restore paths in `option_chain_main.py` are hard-disabled in `BACKTEST_MODE`.
- `db_backtest_runner.py::_day_ohlc` is bounded to replay `time_patcher.now()`, not end-of-day.

Conclusion: current early/whipsaw entry gates do not appear to use benchmark labels, future outcome, or full-day CSV hydration for trading decisions.

## Remaining Overfit Risks

These are not proof of overfit, but they are risks to manage before promotion:

- Thresholds are still hand-authored and have environment override hooks. Defaults live in code, but run manifests should log effective modes/thresholds.
- A profitable 20-Apr result alone is not sufficient. The same behavior must be checked on 16-Apr, 20-Apr, 23-Apr, 24-Apr, 27-Apr, 28-Apr, 29-Apr, 30-Apr, and 18-Jun.
- DVR benchmark annotations are valuable for analysis but must remain audit-only. They should never become order-routing inputs.

## Structural Interpretation

20-Apr currently shows that early detection can identify the opening impulse without future data. The more important remaining question is not "can we enter?" but "are exits and handovers class-aware?"

Opening impulse trades should not be managed exactly like full session runners. A short, high-velocity opening impulse can peak quickly and reverse into the next benchmark leg. If the trade has reached meaningful profit and opposite-side transfer evidence starts building, a recorded ladder/profit floor should be allowed to materialize instead of being suppressed only because the held token is still above VWAP.

This is a structural rule class, not a 20-Apr-specific patch:

```text
opening impulse entry
-> meaningful peak achieved
-> giveback starts
-> opposite side / spot transfer begins
-> enforce profit-floor or handover audit
```

The next safe step is audit-only first:

```text
opening_impulse_profit_floor_candidate
-> transfer evidence
-> floor_should_materialize
-> compare against actual exit
```

Only after this fires on valid impulse reversals and stays quiet on true session runners should it become an exit rule.

## Promotion Rule

Do not promote any new rule only because it improves one day. Promotion requires:

```text
1. No benchmark/future data in the decision path.
2. Effective thresholds logged in the run output.
3. Isolated replay on target day.
4. Cross-day impact replay on known winners and known loss days.
5. No regression on 16-Apr, 20-Apr, 23-Apr, 24-Apr, 27-Apr, 28-Apr, 29-Apr, 30-Apr, and 18-Jun.
```
