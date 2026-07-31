# 2026-04-23 RCA: Whipsaw Disaster

Run: `.trade-api-runs/full-2026-04-17-to-2026-06-08-cont-r1`

Status: multiday replay was stopped after 2026-04-23 showed a large realized loss.

## Trade-Level Damage

| Trace | Entry | Exit | Symbol | Tag | Exit Tag | Qty | PnL | PnL % |
| --- | --- | --- | --- | --- | --- | ---: | ---: | ---: |
| TK260423-093100-961 | 09:31:00 | 09:33:11 | NIFTY26APR24100CE | MOMENTUM_RIDE_CE | EXIT_LADDER_LOCK | 3900 | -25,935.00 | -2.49% |
| TK260423-093702-1323 | 09:37:02 | 09:42:24 | NIFTY26APR24250PE | TR_STABLE_RETRY | EXIT_SL_LOCK | 4095 | -105,036.75 | -10.28% |
| TK260423-095137-2198 | 09:51:37 | 10:32:04 | NIFTY26APR24150CE | TR_STABLE_RETRY | EXIT_SL_LOCK | 3445 | -93,187.25 | -10.10% |

Total 2026-04-23 realized PnL: `-224,159.00`.

## Benchmark Alignment

2026-04-23 was tradable, not a bad-data day. The issue was timing and ownership handover:

- 09:31 CE fired after the opening bull impulse had matured and while L4/GEX said resistance was reinforced.
- 09:37 PE fired just after the bearish scalp ended, while spot had already started the next bullish runner.
- 09:51 CE was directionally valid inside the 09:36-10:05 bull runner, but it was held through the 10:05 bearish owner transfer until a -10% stop.

## Evidence

Token path evidence:

| Trace | Entry | Prior 5m Max vs Entry | Max During Hold | Min During Hold | VWAP Read |
| --- | ---: | ---: | ---: | ---: | --- |
| 09:31 CE | 266.65 | +0.94% | +4.78% | -2.49% | Entry was late; wall/GEX blocked but was downgraded to warn |
| 09:37 PE | 249.45 | +0.82% | -0.10% | -10.28% | Never got real premium follow-through |
| 09:51 CE | 267.70 | +0.67% | +7.56% | -10.10% | Sponsored, but no structural exit on owner transfer |

Spot context:

- 09:34-09:43 spot moved `24205.40 -> 24235.10`; PE entry at 09:37 was against immediate tape.
- 09:48-10:05 spot moved into the bull leg peak; CE entry was acceptable, but by 10:06-10:32 spot rolled down into the bearish leg and CE should have exited structurally.

Fusion evidence:

- 09:37 PE was promoted by `WHIPSAW_PARTICIPATION_GATE_PROMOTE` even though the old blocker was `same_side_spot_chase_exhausted` and wall pressure was only `0.318`.
- 09:51 CE was promoted even though the old blocker was `selected_token_sustained_move_missing`, the selected token was already `+18.41%` above VWAP, and target-wall pressure was `0.0`.
- 09:31 CE had `L4_SR_ZONE.GEX_DIRECTION_CONTRADICTION` with `break_prob=0.00`, but `OPTION_SPONSORED_OVERRIDE` downgraded the hard block to warning.

## Patch

The patch is intentionally narrow:

- `whipsaw_participation_gate.py` now blocks recent spot-chase failures unless target-wall participation is strong enough.
- `whipsaw_participation_gate.py` now blocks recent sustained-move failures when the token is already extended far above VWAP and target-wall pressure is weak.
- `fusion_signals.py` no longer allows option sponsorship to downgrade a hard GEX-direction contradiction when target breach probability is below the sponsored override threshold.

Direct gate checks after patch:

| Case | Result |
| --- | --- |
| 20-Apr good CE `TK260420-095312-2293` | still `participate` |
| 23-Apr PE `TK260423-093702-1323` | now `block_recent_spot_chase_not_revalidated` |
| 23-Apr CE `TK260423-095137-2198` | now `block_recent_extended_no_wall_participation` |

## Remaining Gap

This patch prevents two bad 23-Apr promotions and the hard-GEX warning downgrade. It does not yet solve the bigger exit lifecycle gap: a valid CE inside a bull leg must exit on structural owner transfer around 10:05-10:10 instead of waiting for a -10% stop.
