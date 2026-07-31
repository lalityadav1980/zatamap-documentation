# July 31 Commit Replay Comparison

Date under test: 2026-07-31  
Market session: 09:15-15:30 IST  
Source branch: `codex/runner-coverage-consolidation-20260705`

## Objective

Compare five historical commits against the same complete July 31 market feed.
The comparison focuses on causal trading quality, not only final profit:

- timely entry relative to the earliest complete current proof;
- exact token and position identity;
- runner participation;
- MFE-to-exit profit retention;
- missed opposite-side opportunity caused by stale ownership;
- churn and loss count;
- PEL and MTEA lifecycle correctness.

The first commit is a diagnostic control with full logs and persisted
`fusion_events`. The other commits run with the fast log profile and
`fusion_events` persistence disabled. Before every cleanup, the complete joined
trade ledger is exported to CSV.

## GitHub Commit Matrix

| Commit | GitHub commit date (IST) | Subject | Replay |
|---|---:|---|---|
| [`f84ab9f`](https://github.com/lalityadav1980/zatamap-trade-api/commit/f84ab9fcfe24d8d44566f1db2719773e52e1aeff) | 2026-07-31 12:49:15 | Fix live restart position lineage safety | Complete |
| [`b45c800`](https://github.com/lalityadav1980/zatamap-trade-api/commit/b45c800128c94115f7f0b06c0c5e4c5909ca4f46) | 2026-07-29 13:20:46 | Restore May 5 exact-token DVR runner authority | Fail-fast complete |
| [`6685243`](https://github.com/lalityadav1980/zatamap-trade-api/commit/66852436fc920b85c90795d0ad7f0705d19f208b) | 2026-07-30 06:12:38 | New changes by lalit 20th April | Fail-fast complete |
| [`d0a8f25`](https://github.com/lalityadav1980/zatamap-trade-api/commit/d0a8f2500c175374e0ee0314c2b4e79a8c902672) | 2026-07-24 08:36:23 | Fix immutable-arm runner participation authority | Running |
| [`9e9d386`](https://github.com/lalityadav1980/zatamap-trade-api/commit/9e9d386b0f087e2f6c57ea3707492922f8fd6bb2) | 2026-07-23 15:08:36 | Protect earned MTEA runners without masking loss exits | Pending |

Commit dates and subjects were read from the exact GitHub commit objects.

## Summary

| Commit | Round trips | Wins | Losses | Realized P&L | Entry quality | Exit quality | Status |
|---|---:|---:|---:|---:|---|---|---|
| `f84ab9f` | 2 | 1 | 1 | +30,995.25 | Both late | PEL overheld both | Failed control |
| `b45c800` | 4 | 0 | 4 | -62,799.75 | Repeated late/reversal entries | All low-MFE losses | Failed: daily stop |
| `6685243` | 4 | 0 | 4 | -62,799.75 | Repeated late/reversal entries | All low-MFE losses | Failed: daily stop |
| `d0a8f25` | TBD | TBD | TBD | TBD | TBD | TBD | Running |
| `9e9d386` | TBD | TBD | TBD | TBD | TBD | TBD | Pending |

## Commit f84ab9f

### Trades

| Side | Symbol | Entry | Exit | Held | Entry px | Exit px | MFE | MAE | Realized P&L | Exit |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---|
| PE | NIFTY2680424400PE | 09:30:35 | 09:43:18 | 12m 43s | 130.90 | 128.45 | +16.00% | -1.87% | -5,255.25 | EXIT_LADDER_LOCK |
| CE | NIFTY2680424250CE | 09:50:53 | 14:33:29 | 4h 42m 36s | 160.00 | 181.45 | +28.84% | -8.28% | +36,250.50 | EXIT_LADDER_LOCK |

### Findings

1. The PE opportunity began around 09:22, but entry occurred at 09:30:35.
   Strong option proof existed earlier; opening underlying episode authority
   remained `SIDEWAYS`/not clean and central support began only at 09:30.
2. The PE reached +16.58% in the detailed event trace, earned a 3% floor, then
   exited at -1.87%. PEL held after the floor breach and delayed authority;
   the close adapter executed correctly once authority finally arrived.
3. CE movement began around 09:38-09:39. The earliest complete current CE
   quality packets appeared around 09:40:54-09:41:05 while PE still owned the
   position. The late PE close and post-close evidence reconstruction delayed
   CE entry until 09:50:53.
4. The CE peaked at +28.84% but exited at +13.41% at 14:33:29. A stale
   same-side session-runner hold continued to override the earned floor and
   consumed the later PE opportunity.

Detailed event-level RCA is in `f84ab9f_control_findings.md`.

## Commit b45c800

The replay was stopped fail-fast after `PORTFOLIO_SL_20PCT` closed the fourth
loss at 11:15:42. No later BUY was legally possible, so processing the
remaining market ticks could not alter its trade result.

### Trades

| Side | Symbol | Entry | Exit | Held | Entry px | Exit px | MFE | MAE | Realized P&L | Exit |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---|
| CE | NIFTY2680424250CE | 09:50:53 | 09:54:00 | 3m 07s | 160.00 | 156.25 | 0.00% | -4.94% | -6,581.25 | EARLY_THESIS_INVALID |
| PE | NIFTY2680424400PE | 10:05:41 | 10:12:40 | 6m 59s | 127.70 | 119.95 | +3.37% | -6.07% | -16,623.75 | EARLY_THESIS_INVALID |
| PE | NIFTY2680424400PE | 10:33:02 | 10:37:10 | 4m 08s | 126.85 | 121.90 | +2.25% | -4.30% | -9,974.25 | EARLY_THESIS_INVALID |
| PE | NIFTY2680424400PE | 11:08:40 | 11:15:42 | 7m 02s | 123.65 | 108.95 | +3.32% | -11.89% | -29,620.50 | PORTFOLIO_SL_20PCT |

### Findings

1. All four entries failed to develop: MFE ranged from 0.00% to +3.37%.
2. Three consecutive `EARLY_THESIS_INVALID` exits were followed by another
   same-side PE entry and the daily portfolio stop.
3. Realized loss was 62,799.75, approximately 20.93% of the 300,000 starting
   capital.
4. This is a churn regression, not an exit-retention issue. The commit is
   materially worse than `f84ab9f` on July 31 despite avoiding a long overhold.

## Commit 6685243

The July 31 trade sequence is byte-for-byte identical to `b45c800` after
excluding the replay user identifier. It produced the same symbols, entry and
exit timestamps, prices, quantities, tags, MFE/MAE, exit reasons, and
62,799.75 realized loss.

The replay was stopped fail-fast after the identical `PORTFOLIO_SL_20PCT`
boundary at 11:15:42. Its normalized trade-sequence SHA-256 is:

`af07424aa67daad29a0b57420906bd62ca26bb85a6d23060cfbf57da42b020d3`

This establishes that the July 30 "20th April" commit did not change the July
31 churn behavior relative to `b45c800`.

## Remaining Commits

This section will be filled from each preserved trade CSV immediately after
the corresponding replay completes.

## Ranking

Pending completion of all five exact-commit replays.

## Evidence

- `f84ab9f_f84ab9fc_2026-07-31_trades.csv`
- `f84ab9f_f84ab9fc_2026-07-31_fusion_events.csv.gz`
- `f84ab9f_f84ab9fc_diagnostics.txt`
- `f84ab9f_control_findings.md`
- `b45c800_b45c8001_2026-07-31_trades.csv`
- `b45c800_b45c8001_metadata.txt`
- `c6685243_66852436_2026-07-31_trades.csv`
- `c6685243_66852436_metadata.txt`

Additional commit trade CSVs and metadata files are added as each replay
finishes.
