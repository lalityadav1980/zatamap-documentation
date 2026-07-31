# Branching Strategy

This repository uses a simple promotion model so live trading code has one
source of truth and scattered fix branches do not become deployment candidates.

## Branch Roles

- `main`: production mirror only. Fast-forward or merge into this branch only
  after a `release/*` branch has been replay-tested, approved, deployed, and
  tagged.
- `develop`: integration source of truth. All completed feature, parity, safety,
  and regression fixes are merged here first.
- `feature/*` or `codex/*`: one isolated concern per branch. These branches are
  not deployable by themselves.
- `release/*`: cut from `develop` for a specific live deployment candidate.
  Only stabilization fixes may be added after the cut.
- `hotfix/*`: urgent production fix. Merge the hotfix into both `main` and
  `develop`, then cut or update a `release/*` candidate.

## Promotion Rules

1. Merge feature branches into `develop` only after compile/import/smoke checks.
2. Cut `release/<name>` from `develop`.
3. Replay-test the release branch with recorded market ticks before production.
4. For paper/live safety, `ZATAMAP_TRADE_MODE=live` must be explicit before
   broker orders can route to Zerodha. Missing mode fails closed to paper DB
   execution.
5. Deploy only from a tested `release/*` branch, never directly from a feature
   branch.
6. After deployment is accepted, tag the release and update `main`.

## Current Canonical Branches

- Integration: `develop`
- Current release candidate: `release/live-2026-06-30-integrated`
- Both currently point to the same consolidated fix line.
