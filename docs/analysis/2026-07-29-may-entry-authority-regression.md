# May Entry Authority Structural Regression

## Scope

This report tracks the structural regression work for 2026-05-05,
2026-05-12, 2026-05-13, 2026-05-14, and 2026-05-27. The raw NIFTY legs in
`2026-06-10-nifty-raw-benchmark-2026-04-16-to-2026-06-08.md` are evidence
windows, not runtime inputs. No benchmark label, date, future leg endpoint, or
profit outcome is available to the trading engine.

Acceptance requires:

- runner participation through current market-time evidence;
- no producer-specific veto after the common final authority boundary;
- exact token and symbol identity through reservation and execution;
- no unjustified churn around the benchmark legs;
- full-day replay from 09:15 through the 15:15 entry cutoff and final close;
- focused tests plus the complete automated suite.

## 2026-05-14

### Structural defect

The same candidate episode had two incompatible owners. Stable Retry could
materialize a PE from exact-token MTEA support while DVR still proved that the
opposite CE remained actively sponsored. The producer that happened to reach
the order seam therefore determined the result. This allowed a transient PE
pass at 13:49 during the continuing bull owner.

Two rejected fixes treated selected-token VWAP sponsorship failures as generic
continuity debt. Both removed the false PE, but they also delayed or removed
the valid 11:12 CE major-runner handoff. They were rejected because a token's
own digestion is not equivalent to a live opposite owner.

### Structural correction

The final automated order seam now consumes one symmetric opposite-owner
contract for DVR, Momentum Ride, Stable Retry, and Trend Reversal. It combines:

- the canonical market-time index owner;
- fresh opposite-option sponsorship using the existing DVR semantics;
- exact current MTEA evidence;
- an exact-token confirmed immutable wall break.

A clean exact-token pass may retire stale geometric ownership only after the
opposite option stops participating. A matching confirmed wall break remains
an independent transfer fact. No entry score, price, ROC, VWAP, or momentum
threshold changed.

### Full-day acceptance

Run: `local-may14-accept-v24`

| Entry | Exit | Contract | Result |
|---|---|---|---:|
| 09:37:25 at 218.10 | 10:03:10 at 248.20 | NIFTY2651923600PE | +39,130.00 |
| 11:12:48 at 254.90 | 11:54:30 at 420.05 | NIFTY2651923350CE | +182,490.75 |
| 14:37:00 at 206.25 | 15:11:50 at 230.00 | NIFTY2651923800PE | +32,418.75 |

Realized P&L: `+254,039.50`

Coverage:

- 09:29-09:58 bear runner: captured;
- 10:59-11:52 bull major runner: captured;
- 13:12 false PE regression: no order;
- 13:49 false PE regression: no order;
- 14:33-15:09 bear runner: captured;
- post-exit churn: none.

Verification:

- focused authority/evidence tests: 441 passed plus 189 subtests;
- complete suite: 1,274 passed plus 529 subtests;
- Python compilation: passed;
- `git diff --check`: passed.

## Remaining Days

| Date | Status |
|---|---|
| 2026-05-05 | Pending fresh full-day diagnostic replay |
| 2026-05-12 | Pending fresh full-day diagnostic replay |
| 2026-05-13 | Pending fresh full-day diagnostic replay |
| 2026-05-27 | Pending fresh full-day diagnostic replay |
