# Replay Coverage For Benchmark Legs > 100 NIFTY Points

Current replay max event: `2026-05-07 11:39:55 IST`

Only benchmark legs with `abs(points) > 100` are included. Data-gap/no-data days are excluded from the summary but retained in the CSV.

## Summary

| Date | 100pt Legs | Captured | Partial/Late | Wrong Side | Audit No Order | Not Triggered | Covered/Partial % |
|---|---:|---:|---:|---:|---:|---:|---:|
| 2026-04-16 | 3 | 2 | 1 | 0 | 0 | 0 | 100.0% |
| 2026-04-17 | 2 | 2 | 0 | 0 | 0 | 0 | 100.0% |
| 2026-04-20 | 5 | 1 | 2 | 2 | 2 | 0 | 60.0% |
| 2026-04-23 | 3 | 0 | 0 | 0 | 3 | 0 | 0.0% |
| 2026-04-24 | 2 | 0 | 1 | 0 | 1 | 0 | 50.0% |
| 2026-04-27 | 1 | 0 | 1 | 1 | 0 | 0 | 100.0% |
| 2026-04-28 | 1 | 0 | 1 | 0 | 0 | 0 | 100.0% |
| 2026-04-29 | 2 | 1 | 0 | 1 | 0 | 0 | 50.0% |
| 2026-04-30 | 3 | 2 | 0 | 0 | 1 | 0 | 66.7% |
| 2026-05-04 | 3 | 0 | 1 | 0 | 2 | 0 | 33.3% |
| 2026-05-05 | 2 | 0 | 0 | 0 | 0 | 2 | 0.0% |

## Leg Details

| Date | Leg | Time | Side | Points | Class | Status | Entry | Exit | Symbol | PnL | Coverage | Events/Reason |
|---|---:|---|---|---:|---|---|---|---|---|---:|---:|---|
| 2026-04-16 | 2 | 09:25-09:59 | PE | -124.3 | Runner | partial_or_late_capture | 09:31:50 | 09:46:01 | NIFTY2642124450PE | 56277.0 | 0.42 | 482 events; rejected_control (131); range_scalp_reversal_audit (105); watch_collect_only (66) |
| 2026-04-16 | 4 | 11:08-12:07 | PE | -173.7 | Major Runner | captured_major_leg | 11:10:37 | 13:45:21 | NIFTY2642124400PE | 134550.0 | 0.96 | 267 events; rejected_control (193); ignore_collect_only (60); LATCH_FLIP: UPTREND → DOWNTREND (4) |
| 2026-04-16 | 13 | 14:03-14:51 | CE | +102.6 | Runner | captured_major_leg | 14:13:00 | 14:57:49 | NIFTY2642124150CE | 33891.0 | 0.79 | 673 events; watch_collect_only (255); range_scalp_reversal_audit (142); same_side_position_already_open (98) |
| 2026-04-17 | 7 | 09:43-10:27 | CE | +114.4 | Runner | captured_major_leg | 09:48:04 | 15:15:00 | NIFTY2642124150CE | 211399.5 | 0.89 | 324 events; rejected_control (156); range_scalp_reversal_audit (67); ignore_collect_only (47) |
| 2026-04-17 | 11 | 12:00-14:46 | CE | +100.7 | Runner | captured_major_leg | 09:48:04 | 15:15:00 | NIFTY2642124150CE | 211399.5 | 1.00 | 1148 events; rejected_control (848); ignore_collect_only (295); LATCH_FLIP: SIDEWAYS → UPTREND (3) |
| 2026-04-20 | 1 | 09:16-09:25 | CE | +144.2 | Impulse Runner | partial_or_late_capture | 09:20:02 | 09:32:56 | NIFTY2642124200CE | -2028.0 | 0.55 | 21 events; STABLE_RETRY_BLOCKED: UPTREND (4); CANDIDATE_RESERVATION: rejected reason=extended_above_vwap_chase (3); WHIPSAW_PARTICIPATION_GATE: block_e |
| 2026-04-20 | 2 | 09:25-09:48 | PE | -131.5 | Runner | partial_or_late_capture_with_opposite_overlap | 09:40:49 | 09:52:13 | NIFTY2642124400PE | 103684.75 | 0.31 | 469 events; range_scalp_reversal_audit (121); watch_collect_only (118); entry_candidate_unblocked (53) |
| 2026-04-20 | 3 | 09:48-10:40 | CE | +164.8 | Major Runner | captured_major_leg_with_opposite_overlap | 09:53:14 | 11:10:18 | NIFTY2642124200CE | 246853.75 | 0.90 | 567 events; rejected_control (190); watch_collect_only (135); same_side_position_already_open (76) |
| 2026-04-20 | 8 | 12:52-13:27 | PE | -116.3 | Runner | triggered_or_audited_no_order |  |  |  |  | 0.00 | 775 events; watch_collect_only (211); entry_candidate_unblocked (158); rejected_control (73) |
| 2026-04-20 | 12 | 15:01-15:24 | PE | -100.9 | Runner | triggered_or_audited_no_order |  |  |  |  | 0.00 | 182 events; rejected_control (74); ignore_collect_only (48); STABLE_RETRY_BLOCKED: DOWNTREND (24) |
| 2026-04-23 | 3 | 09:36-10:05 | CE | +108.7 | Runner | triggered_or_audited_no_order |  |  |  |  | 0.00 | 449 events; range_scalp_reversal_audit (151); STABLE_RETRY_BLOCKED: UPTREND (134); rejected_control (120) |
| 2026-04-23 | 4 | 10:05-11:44 | PE | -128.0 | Major Runner | triggered_or_audited_no_order |  |  |  |  | 0.00 | 1750 events; range_scalp_reversal_audit (649); STABLE_RETRY_BLOCKED: DOWNTREND (249); watch_collect_only (215) |
| 2026-04-23 | 7 | 12:31-13:24 | CE | +105.0 | Runner | triggered_or_audited_no_order |  |  |  |  | 0.00 | 1384 events; range_scalp_reversal_audit (705); watch_collect_only (311); STABLE_RETRY_BLOCKED: UPTREND (123) |
| 2026-04-24 | 1 | 09:15-09:57 | PE | -193.8 | Runner | partial_or_late_capture | 09:30:04 | 09:48:39 | NIFTY26APR24100PE | 219128.0 | 0.44 | 268 events; rejected_control (127); range_scalp_reversal_audit (34); STABLE_RETRY_BLOCKED: DOWNTREND (32) |
| 2026-04-24 | 10 | 14:06-14:51 | CE | +111.5 | Runner | triggered_or_audited_no_order |  |  |  |  | 0.00 | 1195 events; watch_collect_only (303); entry_candidate_unblocked (212); range_scalp_reversal_audit (143) |
| 2026-04-27 | 14 | 11:38-13:47 | CE | +142.9 | Major Runner | partial_or_late_capture_with_opposite_overlap | 12:07:40 | 14:31:45 | NIFTY26APR23950CE | 190190.0 | 0.77 | 1611 events; watch_collect_only (530); rejected_control (276); same_side_position_already_open (226) |
| 2026-04-28 | 8 | 10:58-11:33 | PE | -122.8 | Runner | touched_only | 11:06:42 | 11:13:46 | NIFTY26APR24250PE | 79491.75 | 0.20 | 912 events; watch_collect_only (202); entry_candidate_unblocked (124); ignore_collect_only (107) |
| 2026-04-29 | 4 | 09:23-10:28 | CE | +231.5 | Major Runner | captured_major_leg | 09:27:38 | 13:36:10 | NIFTY2650524000CE | 1173012.75 | 0.93 | 380 events; rejected_control (254); watch_collect_only (66); same_side_position_already_open (27) |
| 2026-04-29 | 7 | 13:08-14:30 | PE | -145.3 | Major Runner | wrong_side_overlap |  |  |  |  | 0.00 | 1089 events; watch_collect_only (248); range_scalp_reversal_audit (211); rejected_control (184) |
| 2026-04-30 | 1 | 09:15-09:30 | PE | -108.3 | Impulse Runner | triggered_or_audited_no_order |  |  |  |  | 0.00 | 46 events; STABLE_RETRY_BLOCKED: DOWNTREND (30); OPENING_IMPULSE_PARTICIPATION_GATE: watch_opening_impulse_late reason=opening_impulse_window_expired ( |
| 2026-04-30 | 3 | 09:46-10:17 | PE | -140.9 | Runner | captured_major_leg | 09:50:38 | 10:15:58 | NIFTY2650524000PE | 636402.0 | 0.82 | 202 events; rejected_control (141); ignore_collect_only (43); STABLE_RETRY_BLOCKED: DOWNTREND (8) |
| 2026-04-30 | 6 | 11:14-13:58 | CE | +259.3 | Major Runner | captured_major_leg | 11:02:20 | 14:17:39 | NIFTY2650523700CE | 1562151.5 | 1.00 | 934 events; rejected_control (509); watch_collect_only (242); same_side_position_already_open (116) |
| 2026-05-04 | 5 | 09:41-10:23 | PE | -137.0 | Runner | touched_only | 10:13:09 | 10:23:18 | NIFTY2650524250PE | 1185258.75 | 0.23 | 829 events; watch_collect_only (233); entry_candidate_unblocked (145); rejected_control (115) |
| 2026-05-04 | 11 | 11:48-12:17 | PE | -136.3 | Runner | triggered_or_audited_no_order |  |  |  |  | 0.00 | 683 events; watch_collect_only (171); entry_candidate_unblocked (123); STABLE_RETRY_BLOCKED: DOWNTREND (100) |
| 2026-05-04 | 15 | 13:12-13:35 | PE | -110.4 | Runner | triggered_or_audited_no_order |  |  |  |  | 0.00 | 267 events; rejected_control (103); range_scalp_reversal_audit (87); STABLE_RETRY_BLOCKED: DOWNTREND (61) |
| 2026-05-05 | 6 | 09:55-10:57 | PE | -131.5 | Major Runner | not_triggered |  |  |  |  | 0.00 | 0 events;  |
| 2026-05-05 | 13 | 12:32-13:11 | CE | +128.6 | Runner | not_triggered |  |  |  |  | 0.00 | 0 events;  |
| 2026-05-06 | 11 | 11:55-12:53 | PE | -110.8 | Runner | wrong_side_overlap |  |  |  |  | 0.00 | 483 events; rejected_control (277); STABLE_RETRY_BLOCKED: DOWNTREND (89); range_scalp_reversal_audit (67) |
| 2026-05-06 | 12 | 12:53-14:28 | CE | +322.6 | Major Runner | triggered_or_audited_no_order |  |  |  |  | 0.00 | 1576 events; STABLE_RETRY_BLOCKED: UPTREND (392); range_scalp_reversal_audit (335); rejected_control (228) |

CSV: `docs/analysis/exports/benchmark-coverage-2026-06-23/benchmark_100pt_leg_replay_coverage_current.csv`