# Entry Gate Consolidation Workshop - 2026-07-05

This note explains why the performance dashboard shows many "strategies" even though the engine really has a smaller number of entry ideas. The raw `entry_tag` currently mixes four dimensions in one string:

| Dimension | Examples | Why it matters |
|---|---|---|
| Entry family | `DVR_RECOVERY`, `TR_STABLE_RETRY`, `MOMENTUM_RIDE` | Which detector decided the trade was valid. |
| Side | `CE`, `PE` | Bullish or bearish option side. |
| Runner proof | `RUNNER`, `STABLE_RETRY_R`, `BASKET`, `NEAR_RECLAIM_RUNNER` | Whether the trade is meant to be a scalp/recovery or a trend runner. |
| Exit policy | normal ladder, stable retry ladder, runner ladder, basket ladder, momentum ladder | How much room the open trade gets before profit locks exit. |

Because these dimensions are encoded into one tag, the dashboard splits a few families into many apparent strategies. That makes the system harder to audit and makes day-by-day fixes easy to lose.

## Current Tags From The Leaderboard

| Raw tag | Canonical family | Lifecycle | Intended purpose | Current exit behavior |
|---|---|---|---|---|
| `MOMENTUM_RIDE_PE` | `MOMENTUM_RIDE` | `momentum_runner` | Join strong spot momentum after the momentum/time gates pass. | `LADDER_MOMENTUM`, tight locks from +3%. |
| `TR_STABLE_RETRY` | `STABLE_RETRY` | `recovery` | Retry a trend/option-led candidate after the normal validation path has enough quality. | `LADDER_STABLE_RETRY`. |
| `TR_STABLE_RETRY_R` | `STABLE_RETRY` | `selected_token_runner` | Same stable retry route, but selected token itself proves VWAP/path runner sponsorship. | `LADDER_STABLE_RETRY_RUNNER`. |
| `TR_STABLE_RETRY_BASKET` | `STABLE_RETRY` | `basket_runner` | Basket confirms same-side runner even when selected token alone is imperfect or late. | `LADDER_STABLE_RETRY_BASKET`. |
| `DVR_RECOVERY_CE` / `DVR_RECOVERY_PE` | `DVR_RECOVERY` | `recovery` | Discounted/VWAP recovery candidate. | Plain DVR falls through to normal/expiry ladder unless another suffix is present. |
| `DVR_RECOVERY_CE_RUNNER` / `DVR_RECOVERY_PE_RUNNER` | `DVR_RECOVERY` | `runner_confirmed` | DVR candidate with strict runner proof; preserves DVR ownership but uses runner breathing room. | `LADDER_STABLE_RETRY_RUNNER`. |
| `DVR_RECOVERY_CE_NEAR_RECLAIM_RUNNER` | `DVR_RECOVERY` | `runner_confirmed` | Near-reclaim no-owner DVR materialization that has runner proof. | `LADDER_STABLE_RETRY_RUNNER`. |
| `DVR_RECOVERY_PE_STABLE_RETRY_R` / `DVR_RECOVERY_CE_STABLE_RETRY_R` | `DVR_RECOVERY` | `selected_token_runner` | DVR handoff promoted through stable retry runner proof. | `LADDER_STABLE_RETRY_RUNNER`. |

## What Is Different Between The Gates

### DVR Recovery

Purpose: enter a discounted option recovery before the market has a clean conventional trend entry.

Strength: catches early option recoveries that STABLE_RETRY may reject as incomplete.

Risk: plain `DVR_RECOVERY_*` can represent very different trades: a small VWAP recovery, a real runner starting, or a late exhausted recovery. The dashboard shows this clearly: plain `DVR_RECOVERY_PE` traded more often and has weak expectancy, while `DVR_RECOVERY_*_RUNNER` is much stronger.

Structural issue: runner proof is currently encoded as tag suffixes. Plain DVR is not clearly separated from runner DVR in reporting or policy.

### Stable Retry

Purpose: re-attempt a trend/option-led trade after the engine has enough quality evidence and the selected token can pass validation.

Strength: safer than raw momentum because it has validation layers and debounce.

Risk: it can be too late for documented NIFTY runners because it waits for recovery/retry proof after the move has already matured.

Structural issue: `TR_STABLE_RETRY`, `TR_STABLE_RETRY_R`, and `TR_STABLE_RETRY_BASKET` are treated as separate strategies, but they are one family with different runner lifecycles.

### Momentum Ride

Purpose: join a strong spot momentum move.

Strength: simple and high conviction when the clock/gates align.

Risk: opening time gates and cooldown can make it evaluate too late, which was one cause of live/backtest drift on 29-Jun.

Structural issue: it is better separated than DVR/STABLE_RETRY, but it still needs the same canonical lifecycle reporting so the dashboard can compare it against other runner routes.

## The Real Maintainability Problem

The engine currently lets raw tags decide behavior by substring:

| Code decision | Current mechanism | Problem |
|---|---|---|
| Which ladder to use | `if "STABLE_RETRY_R" in entry_tag` / `if "RUNNER" in entry_tag` | A naming change can change exit behavior. |
| Dashboard strategy grouping | Raw `entry_tag` | Same family splits into many strategies. |
| RCA comparison | Manual tag interpretation | Hard to know if a failure is family-level or lifecycle-level. |
| Future fixes | Add another suffix | Each fix risks creating another strategy variant. |

This is why the gate count keeps growing. We are using tags as both labels and control flow.

## Consolidation Plan

### Phase 1: Normalize Tags Without Changing Trading Behavior

Add a side-effect-free taxonomy parser:

| Canonical field | Values |
|---|---|
| `family` | `DVR_RECOVERY`, `STABLE_RETRY`, `MOMENTUM_RIDE`, `RANGE_SCALP`, `UNKNOWN` |
| `side` | `CE`, `PE`, or null |
| `variant` | `plain_recovery`, `direct_runner`, `near_reclaim_runner`, `stable_retry_handoff`, `selected_token_runner`, `basket_runner`, `spot_momentum` |
| `lifecycle` | `recovery`, `runner_confirmed`, `selected_token_runner`, `basket_runner`, `momentum_runner`, `scalp` |
| `exit_policy` | `recovery`, `stable_retry`, `runner`, `basket_runner`, `momentum`, `range_scalp`, `normal` |
| `source_gate` | Human-readable gate owner for RCA. |

This is now implemented in `entry_tag_taxonomy.py` with tests for every leaderboard tag.

### Phase 2: Report By Canonical Family And Lifecycle

The performance page should show two levels:

| Level | Example |
|---|---|
| Family leaderboard | `DVR_RECOVERY`, `STABLE_RETRY`, `MOMENTUM_RIDE` |
| Lifecycle drilldown | `DVR_RECOVERY:recovery`, `DVR_RECOVERY:runner_confirmed`, `STABLE_RETRY:basket_runner` |

This will immediately answer whether the weak performance is a family problem or only a plain-recovery lifecycle problem.

### Phase 3: Replace Substring Exit Routing With Taxonomy Routing

After Phase 1 is verified, `order_service_api.py` should use `classify_entry_tag(entry_tag)` for ladder selection. This keeps behavior equivalent at first, but removes the fragility where a tag suffix accidentally changes the exit policy.

### Phase 4: Reduce New Tag Variants

Future entries should store:

| Storage | Meaning |
|---|---|
| `entry_tag` | Short backward-compatible family tag, for example `DVR_RECOVERY_PE`. |
| `signal_reason.entry_taxonomy` | Canonical metadata: variant, lifecycle, exit policy, source gate. |
| `fusion_events.ctx.entry_taxonomy` | Same metadata for RCA. |

That prevents new variants like `DVR_RECOVERY_PE_STABLE_RETRY_R` from becoming a new dashboard strategy.

## Review Of The Leaderboard

| Observed group | Interpretation |
|---|---|
| `DVR_RECOVERY_*_RUNNER` positive | DVR is useful when runner proof exists. Do not remove DVR; separate its runner lifecycle. |
| Plain `DVR_RECOVERY_PE` weak | Plain PE recovery is overbroad and should be reviewed separately from DVR runner trades. |
| `TR_STABLE_RETRY_BASKET` positive but risky | Basket proof can help runner capture, but must not override selected-token exhaustion without digestion/reclaim proof. |
| `TR_STABLE_RETRY_R` small positive | Selected-token runner proof works, but entries can be late because the token proof arrives after index runner has matured. |
| `DVR_RECOVERY_*_STABLE_RETRY_R` weak | Hybrid tags are a smell. This should be reported as DVR family with stable-retry-handoff lifecycle, not a separate strategy. |

## Proposed Decision Rules Going Forward

1. Do not create a new raw tag for every RCA fix.
2. Add or adjust canonical metadata instead: family, variant, lifecycle, exit policy, source gate.
3. Any behavior change must state which lifecycle it affects, for example `DVR_RECOVERY:recovery` or `STABLE_RETRY:basket_runner`.
4. Every fix must be measured against the runner coverage gate and protected regression days.
5. Plain DVR and plain STABLE_RETRY should not silently inherit runner behavior; runner behavior requires explicit lifecycle proof.

## Immediate Next Step

Use the taxonomy parser to regenerate the strategy leaderboard grouped by canonical family and lifecycle. Then review weak groups one by one:

1. `DVR_RECOVERY:recovery`
2. `DVR_RECOVERY:selected_token_runner`
3. `STABLE_RETRY:basket_runner`
4. `STABLE_RETRY:selected_token_runner`

That gives us a maintainable workshop path: fix a lifecycle class, not a day and not a tag suffix.
