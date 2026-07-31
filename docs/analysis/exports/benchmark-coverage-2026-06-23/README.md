# Benchmark Coverage Export - 2026-06-23

This folder packages the raw NIFTY benchmark and the current replay coverage
analysis in one shareable location.

## Files

- `2026-06-10-nifty-raw-benchmark-2026-04-16-to-2026-06-08.md`  
  Source benchmark document copied from the `zatamap` repo.
- `benchmark_100pt_leg_replay_coverage_current.md`  
  Replay coverage for benchmark legs where `abs(NIFTY points) > 100`.
- `benchmark_100pt_leg_replay_coverage_current.csv`  
  Machine-readable version of the same coverage report.

## Recreate

Run from the `zatamap-trade-api` repo:

```bash
.venv/bin/python tools/generate_benchmark_100pt_coverage.py \
  --benchmark-md /Users/<user>/Documents/Codex/2026-04-24/trade-engine-redesign/zatamap/docs/analysis/2026-06-10-nifty-raw-benchmark-2026-04-16-to-2026-06-08.md \
  --out-dir docs/analysis/exports/benchmark-coverage-2026-06-23 \
  --min-abs-points 100 \
  --start-date 2026-04-16 \
  --end-date 2026-06-22
```
