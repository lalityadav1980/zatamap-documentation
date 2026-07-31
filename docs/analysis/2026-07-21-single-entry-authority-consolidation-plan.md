# Single Entry Authority Consolidation Plan

Date: 2026-07-21
Branch: `codex/runner-coverage-consolidation-20260705`
Inventory baseline: `7c0a91d93f305b82a7b010a907f6a80a31c3dc12`

## Decision

Stop accepting changes because one historical day becomes profitable. Historical
dates are RCA evidence and regression fixtures, not production conditions and
not order targets. Replace the overlapping entry paths with one causal authority
contract before resuming outcome-driven runner work.

This plan does not promise to enter every hindsight benchmark leg or make every
session profitable. It establishes deterministic, causal decision ownership and
then measures whether the strategy has out-of-sample edge.

## Scope And Effort

The entry source inventory is expected to take 4-8 focused engineering hours.
A semantic audit of entry plus every exit authority is closer to 1-2 focused
engineering days. The first pass must cover every order call, alias, market
veto, cooldown, reservation mutation, cross-side owner, and contract bridge.
The current central modules are large enough that a text search alone is not
sufficient:

| Module | Approximate lines | Current role |
| --- | ---: | --- |
| `fusion_signals.py` | 84,634 | Feature calculation, all entry producers, MTEA adapters, legacy gates, exit monitors |
| `order_service_api.py` | 28,543 | Entry revalidation, position transfer, order execution, TSL and PEL exits |
| `stable_retry_evidence_cache.py` | 4,195 | Cross-tick evidence, reservations and shared side arbitration |
| `central_decision_engine.py` | 3,257 | Separate entry/exit verdicts, cumulative L2 and strategy cooldowns |
| `index_runner_state.py` | 3,406 | Past-only runner and owner calculations |

Expected implementation effort after the inventory:

| Milestone | Engineering estimate | Exit condition |
| --- | --- | --- |
| Entry authority inventory and classification | 4-8 hours | Every entry execution seam and market re-decision has an owner and disposition |
| Exit authority inventory and classification | 1 additional focused day | Every close seam is classified as emergency, risk, market-driven or execution lifecycle |
| Pure contract and invariant tests | 1-2 focused days | Contract can be evaluated without DB, cache or broker side effects |
| Producer adapters and atomic cutover | 2-4 focused days | All automated entry paths require the same contract |
| Legacy deletion and exit separation | 1-3 focused days | No post-contract market decision remains; PEL is independently owned |
| Deterministic regression | 1-3 days of machine time | Repeated local/remote parity and locked acceptance matrix pass |
| Paper soak | At least 5-10 sessions | No authority mismatch, lifecycle loss, or unexplained order divergence |

The complete replacement is therefore not a safe one-day patch. The inventory
and pure contract can be completed first without changing trading behavior.

## Current Status

| Work item | Status | Evidence |
| --- | --- | --- |
| Enumerate direct order seams | Complete for the default authority modules | Seven seams: six automated and one manual API |
| Enumerate contract/evidence/CDE/L4 calls | Complete first AST pass | Reproducible with `tools/audit_entry_authority_paths.py` |
| Trace route-local entry chains | Complete first pass | DVR, Momentum Ride, Stable Retry, trend flip/flat and Range Scalp mapped below |
| Classify all decision-time clock reads | In progress | 115 candidates; broker polling and audit throttles still need to be separated from decisions |
| Trace all exit authorities | Pending | 24 direct close seams found; exit consolidation follows entry cutover |
| Implement pure V1 contract | Complete | `single_entry_authority.py`; no I/O, cache, broker or wall-clock dependency |
| Shadow producer integration | Complete at final verdicts and the shared reducer | Five final-seam ignored-return adapters plus one diagnostic-only MTEA reducer adapter cover DVR, Momentum Ride, Stable Retry/TR Stable Retry and Trend Reversal |
| Shadow comparison reporting | Complete | `tools/report_entry_authority_shadow.py` separates reducer readiness from actionable final-order authority and deduplicates by market bucket/snapshot |
| Sequential acceptance replay | In progress | April 20 source-frozen baseline preserved; reducer-stage April 20 and May 11/12 runs follow |

## V1 Core Implemented

`single_entry_authority.py` now provides a side-effect-free
`EntryAuthorityContractV1` and evaluator. This is not yet connected to order
execution. Its current responsibilities are deliberately limited to authority:

1. Require explicit positive market timestamps; no wall-clock fallback exists.
2. Require exact side, token, symbol, expiry, trace, episode and session identity.
3. Admit only DVR Recovery, Momentum Ride, Stable Retry and Trend Reversal as
   producer intents. Range Scalp is disabled and rejected by V1.
4. Deduplicate SUPPORT, WAIT, HARD_REJECT and corroborated-opposite evidence in
   fixed market-time buckets.
5. Keep WAIT neutral, derive categorical rejection identity from structured
   reason codes, and exclude audit text from decisions and hashes.
6. Require current exact-token SUPPORT and valid current causal checks.
7. Keep reservation wall and token identity immutable.
8. Issue a deterministic decision hash and tamper-evident contract hash.
9. Verify policy version, exact execution identity and market-time expiry without
   recomputing trend, L2/L3/L4, VWAP, wall or momentum logic.

Focused V1 tests cover CE/PE symmetry, packet-cadence parity, rejection recovery,
opposite cancellation, sibling-token cancellation isolation, stale/future facts,
reservation identity, contract expiry, policy mismatch and the Range Scalp kill
switch. The complete repository suite passed 959 tests after this milestone.

## Shadow Adapters Implemented

`entry_authority_shadow.py` translates facts already computed by a legacy route
into V1 without reading or mutating the evidence cache. It has no order, close,
database, Redis, broker, logging or wall-clock owner. The five final-order calls
are standalone expression statements. A sixth call at the common MTEA reducer
returns an audit mapping used only by bounded diagnostics. Static tests prohibit
both forms from influencing any entry predicate during the shadow phase.

| Producer | Shadow evidence source | Current limitation |
| --- | --- | --- |
| DVR Recovery | Existing exact-token DVR MTEA episode and immutable reservation | Observed only after current DVR preflight/execution binding |
| Momentum Ride | Existing exact-token Momentum Ride MTEA episode | Observed only after route guard and CDE |
| Stable Retry / TR Stable Retry | Existing Stable Retry MTEA episode and reservation | Observed at the final central evidence verdict |
| Trend Reversal close-and-flip | Labeled final exact-token route fact | Has not yet published cross-time observations into MTEA |
| Trend Reversal flat entry | Labeled final exact-token route fact | Has not yet published cross-time observations into MTEA |
| Shared MTEA reducer | Typed current observation plus the exact candidate's central episode | Measures hypothetical evidence readiness; explicitly not an actionable order intent |

The adapter resolves expiry only from the exact token in the in-memory
`market.instruments` universe. It never parses expiry from a symbol or assumes a
weekday. It also exposed and fixed two pure authority defects before wiring:
side-scoped support can no longer authorize an exact token, and sparse opposite
buckets can no longer masquerade as a sustained cancellation streak.

Each deduplicated log starts with `ENTRY_AUTHORITY_V1.SHADOW` and records the
legacy verdict, V1 status, comparison, exact token/expiry, market timestamp,
support requirement, hard debt, waits, opposite streak, reason codes and
contract id. Optional rows use
`trade.fusion_events.event_type = 'entry_authority_v1_shadow'`. Configuration:

```text
ENTRY_AUTHORITY_V1_SHADOW_ENABLE=true
ENTRY_AUTHORITY_V1_SHADOW_FUSION_EVENT_ENABLE=true
```

`MATCH_EXECUTE` and `MATCH_BLOCK` are agreement. `V1_WOULD_BLOCK` and
`V1_WOULD_EXECUTE` are migration evidence, not trading instructions. Missing
instrument expiry or malformed evidence fails the shadow result closed while
the legacy route continues unchanged. The complete repository suite passed
975 tests after shadow wiring.

Reducer-stage checks preserve categorical semantics. An incomplete current
observation is a neutral `WAIT`, not fabricated rejection debt. Rejected token
identity or a malformed accepted observation is `HARD_REJECT`; episode
cancellation is explicitly side scoped. The reducer payload is embedded in the
existing bounded `entry_evidence_episode` event, so it does not add another
high-volume event stream.

## First Source-Frozen Replay Baseline

The pre-reducer-shadow April 20 replay is preserved under user
`bt_20260420_eac_shadow_v1` and run id
`local-eac-shadow-apr20-v1-20260721`. It completed successfully with five
round trips and realized P&L of **+₹114,013.25**:

| Entry | Exit | Contract | Entry route | Exit route | P&L |
| --- | --- | --- | --- | --- | ---: |
| 09:40:51 | 09:55:00 | `NIFTY2642124400PE` | `TR_STABLE_RETRY` | `EXIT_LADDER_LOCK` | +₹11,375.00 |
| 09:55:02 | 10:52:01 | `NIFTY2642124200CE` | `TR_STABLE_RETRY` | `EXIT_LADDER_LOCK` | +₹82,095.00 |
| 13:01:54 | 13:31:04 | `NIFTY2642124500PE` | `TR_STABLE_RETRY` | `EXIT_LADDER_LOCK` | +₹50,293.75 |
| 13:43:52 | 13:51:50 | `NIFTY2642124300CE` | `MOMENTUM_RIDE_CE` | `EARLY_THESIS_INVALID` | -₹24,518.00 |
| 14:32:32 | 15:02:20 | `NIFTY2642124500PE` | `TR_STABLE_RETRY` | `EXIT_LADDER_LOCK` | -₹5,232.50 |

This is a causal baseline, not an acceptance result. It establishes four
separate findings:

1. The 09:16 opening CE impulse was still missed and the first PE was late.
2. The 09:48-10:40 CE major runner and 12:52-13:27 PE runner were captured.
3. The 13:43 Momentum Ride CE loss was already authorized by both legacy and V1;
   it is a producer/current-fact problem, not evidence-ledger suppression.
4. The final PE reached about +18.45% before the Position Exit Evidence Ledger
   authorized an exit at a loss just as the final PE runner resumed. That is a
   separate Phase 5 exit-classification defect and must not be repaired inside
   entry authority.

The first shadow version observed only candidates that reached a final route
seam, so it could not measure how much earlier the reducer itself became ready.
The reducer-stage adapter was added specifically to close that observability
gap without changing legacy execution.

Use the reporter against a replay user as follows:

```bash
python tools/report_entry_authority_shadow.py \
  --dsn postgresql://<user>:<password>@127.0.0.1:5432/zatamap_market \
  --user-id bt_20260420_eac_shadow_v2 \
  --date 2026-04-20
```

It reports `MTEA_EVIDENCE_REDUCER` and `FINAL_ORDER_SEAM` independently, with
the first legacy-ready and V1-ready market timestamps. Reducer readiness is
hypothetical evidence authority; only a final-seam contract can represent an
actionable order intent.

## April 20 Entry And Exit Acceptance

The six-round-trip control immediately before the entry-history and PEL
corrections was preserved under user `bt_eac_pel_postfix_apr20_20260721`. It
realized **+₹134,985.50**. Two weaknesses were causal defects rather than
day-profit targets:

1. MTEA had approved the exact first PE at 09:31, but position lifecycle
   construction reconstructed one older opposite component and delayed entry
   until 09:40.
2. The final PE had earned a +18.45% peak and was recovering on its exact token
   at 15:02, but PEL retained only the slower ROC. A later packet in the same
   fixed bucket then overwrote recovery and helped manufacture an exit at a
   loss.

The generic corrections are deliberately separate:

- `EntryHistoryResolutionV1` is an immutable receipt for a centrally approved
  exact side, token, symbol and market tick. It resolves only historical
  anti-chase debt. Current gates, wall geometry, cancellation and exact-token
  participation remain mandatory.
- Exact held-token recovery uses tick market time and is monotonic inside one
  fixed PEL bucket. Packet ordering cannot turn a recovery bucket into an
  independent discretionary exit vote. Hard loss, authenticated opposite L3 or
  MTEA transfer, and proved immutable-wall reclaim remain categorical.
- Recovery authority is limited to a position that already owns the existing
  mature-runner lifecycle. A low-MFE underwater trade records the uptick for
  audit but cannot use it to erase loss-invalidation debt. This uses existing
  lifecycle state and adds no price, score, ROC, P&L or duration threshold.

The final source-frozen verifier used user
`bt_eac_mtea_pel_apr20_full_v7_20260722` and run id
`local-eac-mtea-pel-apr20-full-v7-20260722`. It completed with six round trips
and **+₹191,256.00**:

| Entry | Exit | Contract | Entry route | Exit route | P&L |
| --- | --- | --- | --- | --- | ---: |
| 09:22:07 | 09:30:38 | `NIFTY2642124250CE` | `TR_STABLE_RETRY` | `EXIT_LADDER_LOCK` | +₹13,172.25 |
| 09:31:18 | 09:54:27 | `NIFTY2642124450PE` | `TR_STABLE_RETRY` | `EXIT_LADDER_LOCK` | +₹36,851.75 |
| 09:54:58 | 10:52:01 | `NIFTY2642124200CE` | `TR_STABLE_RETRY` | `EXIT_LADDER_LOCK` | +₹81,445.00 |
| 13:01:54 | 13:31:04 | `NIFTY2642124500PE` | `TR_STABLE_RETRY` | `EXIT_LADDER_LOCK` | +₹50,293.75 |
| 13:43:52 | 13:51:50 | `NIFTY2642124300CE` | `MOMENTUM_RIDE_CE` | `EARLY_THESIS_INVALID` | -₹24,518.00 |
| 14:32:32 | 15:14:59 | `NIFTY2642124500PE` | `TR_STABLE_RETRY` | `REPLAY_EOD_CLOSE` | +₹34,011.25 |

The trace proves the lifecycle distinction rather than only the improved total.
At 13:51:46 the losing CE had a raw exact-token uptick, but no mature peak, so
its loss streak remained at two and reached three at 13:51:50. At 15:02:28 the
mature final PE's earlier recovery owned that fixed bucket; the next bucket
provided only one profit-lock vote and renewed recovery reset it to zero. No
15:02 sell occurred.

The acceptance snapshot passed **1,010 tests**, `py_compile` and
`git diff --check`. Relevant SHA-256 hashes:

```text
single_entry_authority.py          006ae0e670bda46e3acfba82abe9936ac425b19fe6db4b54ec682eae8517a1c6
entry_authority_shadow.py          ceec0dad93397688b2662efa4ae048a42559e01b14606fd3d232fd9111568e88
fusion_signals.py                  adc68aa9b373087bdebb0dc800e3e6eef4bb8fdfc5e23f545787e45702d51baf
mtea_position_lifecycle.py         33582e76da8ad98680c9fe3e8bfe25cc4644cadf49296aa7d786d67c724f8ca4
position_exit_evidence_cache.py    d6e29ff696da288c3737862a86af01d4d74c03b50d43ef294f62bb4bf269685b
order_service_api.py               b989bb5da5077914eb959c4edcb3642d6e660702c7f7309df17dc469d3843838
test_position_exit_evidence_cache  8ffeba1ac60e85ae3769f16e89f6978b64b05e43bc23660c268bc3493599f2f4
```

This accepts the two April 20 defects under the frozen tree. It does not claim
that April 20 profit is an optimization target or that the broader April, May
and July touched-date regression is complete.

## Target Ownership

```text
market ticks
  -> causal feature facts (L1/L2/L3/L4 and exact-token metrics)
  -> scenario intent producers
  -> MTEA evidence reducer
  -> SingleEntryAuthority.evaluate()
  -> immutable EntryAuthorityContract
  -> order service identity/risk verification
  -> broker or DB executor
```

Scenario producers remain useful. `DVR_RECOVERY`, `MOMENTUM_RIDE`,
`STABLE_RETRY`, `TR_STABLE_RETRY`, opening impulse, owner transfer and wall
break describe why a candidate exists. They must not own independent order
placement, cancellation, cooldown, or cross-side state.

## Evidence Vocabulary

Every producer emits one typed observation:

| Observation | Meaning | Cross-tick effect |
| --- | --- | --- |
| `SUPPORT` | Current exact-token and same-side evidence is qualified | Adds one deduplicated market-time bucket |
| `WAIT` | Evidence is incomplete, warming, wall-held or neutral | No failure debt and no timed blindness |
| `HARD_REJECT` | Current candidate has a categorical defect | Adds one structured and scoped defect identity |
| `CANCEL` | Exact token/side lifecycle is no longer valid | Terminates only the matching authority scope |

Human-readable reasons are audit text only. A structured reason code, scope,
side, token and market-time bucket own deduplication. CE and PE histories are
independent. Packet frequency cannot manufacture support or rejection debt.

## Contract V1

`EntryAuthorityContractV1` will contain:

```text
version and policy_version
trace_id and episode_id
user_id and session_date
side, exact token, symbol and expiry
scenario intents and producer set
decision_market_ts and expires_market_ts
original wall and reservation identity
support/wait/rejection bucket summary
current categorical safety checks
position-transfer intent, when applicable
decision snapshot hash
```

The contract is immutable. The order layer can verify identity, freshness,
position/risk limits, quantity, margin and broker response. It cannot recalculate
trend, momentum, L2/L3/L4, VWAP sponsorship, chase state, wall state, or runner
ownership after authorization.

## Gate Classification

All existing checks will receive one disposition:

| Class | Examples | Destination |
| --- | --- | --- |
| Operational hard safety | Invalid token, stale/missing market data, entry cutoff, position conflict, portfolio loss lock, margin | Contract safety section or order verifier |
| Current causal market fact | L1 direction, L2 bias, exact-token participation, L4 wall state, VWAP geometry | Producer fact; evaluated once by central authority |
| Cross-time evidence | Repeated support, hard-reject recovery, reservations, opposite corroboration | MTEA reducer only |
| Execution integrity | Contract identity, pending duplicate, broker acknowledgement, matching persisted position | Order service only |
| Audit-only legacy | Previous cooldown counters, duplicated owner reconstruction, text-derived fingerprints | Logged during shadow phase, then deleted |

## Preliminary Execution Inventory

This table is the first source-backed pass. Line numbers describe baseline
`7c0a91d` and will move as the migration proceeds.

| Entry path | Execution seam | MTEA today | Preliminary disposition |
| --- | --- | --- | --- |
| DVR recovery | `fusion_signals.py:33833` | Yes | Convert to intent adapter; keep no direct order call |
| Momentum Ride | `fusion_signals.py:47955` via `_sc_process` | Yes | Convert to intent adapter; remove route-local final authority |
| Stable Retry / TR Stable Retry | `fusion_signals.py:74234` | Yes | Convert to intent adapter; retire the surrounding duplicated preflight chain |
| Trend reversal close-and-flip | `fusion_signals.py:49384` | No | Must enter central evidence/contract path before cutover |
| Trend entry while flat | `fusion_signals.py:49615` | No | Must enter central evidence/contract path before cutover |
| Range Scalp | `fusion_signals.py:76468` via `_rs_process` | No | Disabled and excluded from V1; retire the dormant direct-order route during cutover |
| HTTP/manual order endpoint | `order_service_api.py:593` | No | Explicitly classify as manual/operator or require supplied contract |

## Current Route Chains

The route name is an intent source, not an authority boundary. This table records
the actual current path and the migration disposition. A check marked `move`
remains part of the policy but is evaluated once inside the new authority. A
check marked `delete` is duplicated or incompatible authority. A check marked
`verify` belongs after authorization because it concerns execution integrity.

| Route | Current causal chain | Active overlapping state | Disposition |
| --- | --- | --- | --- |
| DVR recovery | detector and quality ceremony -> 180-second route debounce -> active-position/handoff checks -> fusion preflight -> CDE -> MTEA -> order-service contract reconstruction -> execution | Per-side `_dvr_router_last_*_ts`, DVR binding cache, stable reservation and central evidence all influence one entry | Move detector facts and handoff intent to one observation; delete route debounce as market authority; pass the resulting contract explicitly; verify only exact execution identity and position state |
| Momentum Ride | rolling coordinator -> warmup/session-age watch -> L1 direction -> shared fusion preflight -> route guard -> CDE -> MTEA -> order service | Coordinator state, L1 deferral, fusion score and CDE coordinator/structure re-evaluate related facts; old post-exit cooldown is already audit-only | Preserve warmup as data-readiness fact; move current market facts into one evaluation; keep cooldown audit-only; delete repeated direction decisions after contract creation |
| Stable Retry / TR Stable Retry | selected-token candidate -> wall/reservation lifecycle -> materialization revalidation -> 30-second order debounce -> candidate reservation enforcement -> multiple pre-order vetoes -> CDE -> MTEA -> order service | Route candidate store, wall reservation, recent failure state, order debounce, MTEA reservation and order-cache projection can each block or mutate the candidate | MTEA becomes the only cross-time store; move current token/wall facts into typed observations; reduce debounce to duplicate-request integrity keyed by contract id; delete text/counter authority |
| Trend close-and-flip | reversal/master-gate decision -> route cooldown/whipsaw checks -> token resolution -> wall-clock cutoff -> direct order -> legacy order ownership guard | No MTEA contract; mutable order cache acts as approval receipt; order service recomputes index ownership | Adapt reversal into intent and current facts; use market-time contract cutoff; require central contract before close-and-flip; make transfer atomic |
| Trend entry while flat | master-gate decision -> CT-close cooldown -> MG-exit cooldown -> token resolution -> wall-clock cutoff -> direct order -> legacy order ownership guard | Process-time `mg_live_exit_wall_ts`, route cooldowns and cached trend approval independently suppress or permit entry | Represent prior exit as market-time evidence; move re-entry policy to central authority; delete wall-clock and legacy owner decisions |
| Range Scalp | range geometry -> session-age veto -> loss/exit/SL cooldowns -> L1/L2/L4 proposal -> token reversal guard -> CDE -> direct order -> legacy order ownership guard | Dormant route had four route-local locks plus CDE and the legacy guard; `live_trade=True` is hardcoded | Keep disabled, reject it as a V1 producer, and delete the dormant direct-order route during cutover |
| Manual endpoint | caller parameters/default token -> process order -> legacy order ownership guard | No automated evidence contract and a concrete default instrument | Keep visibly operator-only with a separate signed manual authorization; never classify it as an automated strategy contract |

### Active Lock Disposition

| Existing lock | Current effect | Required treatment |
| --- | --- | --- |
| DVR 180-second router debounce | Returns before later same-side evidence reaches MTEA | Replace with idempotent bucket deduplication; never suspend observation |
| Momentum Ride post-exit cooldown | Logged through `_entry_evidence_timed_lock_audit`; does not currently veto | Retain during shadow comparison, then delete |
| Stable Retry 30-second order debounce | Blocks before contract creation based on route-local last-order time | Replace with contract-id pending/executed dedupe in the order layer |
| Trend CT/MG post-exit cooldowns | Block a new candidate before shared evidence is evaluated | Convert the underlying exit and side ownership facts into MTEA observations; no process-time fallback |
| Range loss, post-exit and post-SL cooldowns | Absolute predicates in the route's entry condition | Express genuine position/risk safety centrally; classify ordinary recovery as WAIT and continue validating |
| CDE strategy cooldown | Checked on every CDE entry but has no production setter call | Remove as latent authority after shadow proof; do not revive it |

### Boundary Rule

The future order boundary is intentionally narrow:

```text
producer facts + MTEA episode
  -> SingleEntryAuthority.evaluate(..., market_ts)
  -> EntryAuthorityContractV1
  -> verify contract identity/freshness + portfolio/position/margin safety
  -> execute exactly the authorized token
```

There must be no coordinator, trend, L2/L3/L4, VWAP, wall, chase, session-age,
runner-owner or route-cooldown calculation after `EntryAuthorityContractV1` is
issued. A changed market packet creates a new observation and contract decision;
the order layer cannot silently reinterpret the old one.

### Confirmed Overlaps

1. `central_decision_engine.py` already calls itself the central decision engine,
   but MTEA later performs another cross-time decision and `order_service_api.py`
   can perform another owner decision. There are at least three authorities.
2. CDE owns strategy cooldown state independently of MTEA. `set_cooldown()` uses
   process time while `evaluate_entry()` uses request market time.
3. The central order contract covers only DVR, Momentum Ride and Stable Retry in
   `order_service_api.py:17140-17148`. Range Scalp, trend flip, WTP and unknown
   tags fall through to the legacy runner ownership reconstruction.
4. Trend flip and flat-entry use `datetime.now()` for their 15:15 cutoff at
   `fusion_signals.py:49373` and `fusion_signals.py:49604`. Replay decisions must
   use the supplied market timestamp.
5. Flat-entry also reads `mg_live_exit_wall_ts` for a market decision. A process
   timestamp cannot be a causal replay lock.
6. Range Scalp passes `live_trade=True` at `fusion_signals.py:76474`, bypassing
   the manager's paper/backtest/live mode.
7. `process_order()` invokes `_index_runner_entry_ownership_guard()` before
   execution. Framework tags may return early after a valid central contract;
   every tag outside that taxonomy receives a large second market decision with
   its own runner thresholds and cache proofs.
8. The entry contract is currently reconstructed from shared cache at the order
   boundary. Contract V1 should be passed explicitly by the caller and verified,
   not rediscovered by tag and mutable state.
9. CDE repeats producer-owned market decisions: coordinator direction, session
   extremes, cumulative L2, structural trend and route-specific exceptions are
   evaluated after each producer has already evaluated L1-L4. Some exceptions
   fail open, so CDE is neither a pure safety verifier nor the final authority.
10. CDE declares per-strategy cooldown state and checks it during every entry.
    No production caller currently invokes `set_cooldown()`, making this latent
    authority rather than a reliable lifecycle. Its setter also uses wall time.
11. The manual HTTP endpoint defaults to a concrete NIFTY token and calls
    `process_order()` without a contract. It must remain explicitly operator-only
    or require a signed manual authorization; it cannot masquerade as automation.

### First Automated Count

The initial AST pass over the default authority modules found:

| Seam type | Count |
| --- | ---: |
| Order execution | 7: six automated seams plus one manual API seam |
| Entry evidence observations | 33 |
| CDE entry decisions | 5, including CDE's own evaluator call |
| Entry contract verification calls | 3 |
| Evidence authority reads | 7 |
| L4 entry authority reads | 3 |
| Direct position-close calls | 24 across producer and order-service lifecycles |
| Direct process-clock reads | 115 |

The 115 clock reads are not all defects. Broker polling and log throttling may
use monotonic/process time. Any read affecting market decisions, cooldowns,
evidence freshness, entry cutoffs, exits or replay state must use market time.

## Inventory Method

Use `tools/audit_entry_authority_paths.py` to enumerate imported order aliases and
authority calls without importing the application:

```bash
python tools/audit_entry_authority_paths.py
python tools/audit_entry_authority_paths.py --include-clocks
python tools/audit_entry_authority_paths.py --json > outputs/entry-authority-seams.json
```

The generated list is only a starting point. Each seam must be traced backward
to all mutable state and forward to order/exit lifecycle effects.

## Migration Sequence

### Phase 0: Freeze And Baseline

1. Record source, config, market-data and profile hashes.
2. Preserve accepted and degraded replay ledgers separately.
3. Do not accept a change based on one day's P&L.
4. Do not mix feed/restart work with decision-policy work in one commit.

### Phase 1: Complete Authority Map

1. Enumerate every order and close call, including local import aliases.
2. Enumerate CDE calls, MTEA observations, reservation mutations and cooldowns.
3. Enumerate process-clock reads that affect decisions.
4. Classify every check as safety, current fact, cross-time evidence, execution
   integrity, or audit-only legacy.
5. Produce a deletion/migration table before changing behavior.

### Phase 2: Pure Authority Core

1. Add a side-effect-free `single_entry_authority.py` module.
2. Define typed facts, observations, lifecycle transitions and contract schema.
3. Make market time a required argument with no wall-clock fallback.
4. Make categorical cancellation scope explicit: TOKEN, SIDE, POSITION or SESSION.
5. Add deterministic snapshot hashing and policy versioning.

### Phase 3: Shadow Integration

1. Adapt every producer to emit facts and observations only.
2. Run old and new decisions together; only old code may trade in shadow mode.
3. Publish one bounded comparison event per candidate bucket.
4. Explain every divergence by rule identity, never by daily profit.

### Phase 4: Atomic Cutover

1. Switch all automated strategies together to the new contract.
2. Require an explicit valid contract at `process_order()`.
3. Keep operational risk, duplicate-order and broker checks in order service.
4. Remove post-contract market re-decisions and route-local cooldowns.
5. Reject unknown automated entry tags instead of silently using legacy logic.

### Phase 5: Exit Separation

1. Keep hard loss, EOD and broker emergency exits independent.
2. Convert all market-driven exits into PEL observations.
3. Issue one immutable `ExitAuthorityContract` before close execution.
4. Prevent entry evidence from directly mutating position exit state.

## Required Tests

### Pure Invariants

1. One opposite L2 observation cannot cancel same-side authority.
2. Three independent corroborated opposite buckets can cancel it.
3. Repeated packets in one bucket count once.
4. WAIT never creates failure debt.
5. A free-text reason change cannot create a new hard rejection.
6. Exact-token and symbol identity are mandatory and symmetric for CE/PE.
7. An immutable reservation cannot move its original wall.
8. No future timestamp can enter a decision snapshot.
9. One-second and five-second process cadence produce the same contracts.
10. No automated order can execute without one active contract.

### Integration Invariants

1. Every order has exactly one contract trace and policy version.
2. No market gate runs after the contract boundary.
3. Local and remote produce identical orders for the same source/config/data.
4. Paper, replay and live differ only at the execution adapter.
5. An accepted broker response without a matching position remains ambiguous.
6. Position transfer consumes the old position and installs the new contract
   atomically.

## Regression Policy

Known April, May and July sessions form the touched regression set. They can
detect reintroduced defects but cannot prove generalization. A missed hindsight
runner or a red day is not by itself a software bug.

Acceptance metrics:

1. Deterministic order parity on repeated runs.
2. Zero future-data or process-clock decisions.
3. Zero unexplained post-contract blocks.
4. Aggregate expectancy and drawdown after realistic charges.
5. Churn and stop-out rate.
6. Entry delay measured from the first causal actionable contract, not the
   benchmark turning point.
7. Premium capture measured from the actually selectable token.
8. One locked, previously unseen evaluation period. Inspecting and tuning that
   period consumes it; another untouched period is then required.

## Change Discipline

Every behavior change must include:

1. The invariant being corrected.
2. Historical dates only as evidence reproducing the invariant.
3. A scenario-neutral unit test for both CE and PE where applicable.
4. The expected impact on all producers and order/exit boundaries.
5. Focused tests, full tests, deterministic regression and an explicit rollback.

Threshold changes are a separate strategy-research activity. They cannot be
mixed with authority consolidation or justified by making one date profitable.

## Completion Definition

The consolidation is complete only when:

1. All automated entry paths emit intent into one authority.
2. `process_order()` requires and verifies the immutable contract.
3. No strategy-specific order call, blind cooldown or owner reconstruction can
   bypass that contract.
4. All market decisions use exchange/tick time.
5. Exit authority is separate and singular.
6. Touched regression, locked validation and paper soak pass without tuning to
   individual daily outcomes.

## July 21 EOD Lifecycle Identity Acceptance

### Incident

While a CE position remained open, a rejected PE replacement reached the central
entry boundary and wrote its candidate trace into the shared top-level order
cache. The cache then contained the held CE symbol/token/quantity and the
unfilled PE parent. At 15:15, the parentless `Market_Close` sweep used that
hybrid identity and inserted the CE SELL under the PE parent. The symbol became
flat, but the original CE tracker and trail summary remained open; the later
`EOD_EXIT` correctly named the CE parent but could no longer sell zero inventory.

### Structural Correction

1. Pre-fill authority is stored only as `pending_entry_candidate`.
2. DVR, Stable Retry and both Trend Reversal entry paths no longer write
   candidate trace, parent or entry tag into active position fields.
3. An exact confirmed BUY fill promotes the pending trace/token/symbol contract
   into the active MTEA/PEL lifecycle.
4. Paper/replay close identity comes only from persisted `orders_tracker`, with
   `orders_book` fallback. Mutable cache may enrich diagnostics but cannot choose
   symbol, token, quantity or parent.
5. A non-EOD caller with a different parent fails closed. Categorical EOD close
   ignores the stale requested parent, records the mismatch, and closes the
   canonical persisted parent.
6. Per-user DB close serialization makes repeated 15:15 sweeps idempotent.
7. DailyCloser uses only `EOD_EXIT`; exactly 15:15 is outside the signal window.
   The shared order backstop uses the same exclusive 15:15 cutoff, so fusion,
   paper/live execution and replay cannot disagree during the final 15 minutes.

### Observability

Every DB close records `position_identity_source`,
`canonical_parent_order_id`, `requested_parent_order_id`,
`parent_identity_canonicalized`, and `pending_entry_candidate_trace` in
`close_dbg`. Logs and the optional lifecycle fusion event also include tracker
deletion and trail-summary reconciliation results.

### Verification

- Incident-shaped hybrid CE/PE cache closes under the persisted CE parent.
- A wrong-parent ordinary flip is blocked with
  `close_parent_identity_mismatch`.
- A second EOD sweep returns `no_open_position` and creates no second SELL.
- Candidate authority preserves the held lifecycle until exact fill promotion.
- DailyCloser emits `EOD_EXIT`; 15:14:59 is tradable and 15:15:00 is not.
- The order layer accepts 15:14:59 and rejects 15:15:00 for every BUY route.
- Focused authority, DVR, MTEA and PEL suites pass.
- Full suite: 1,020 tests pass.
- `py_compile` and `git diff --check` pass.

No price, score, VWAP, momentum, or market-evidence threshold changed.

## July 24 Side-Debt And Exit-Wall Authority Acceptance

### Incidents

The remote `0002358` regression exposed two shared authority defects.

1. On 23 and 27 April, `same_side_spot_chase_exhausted` was computed from
   underlying spot extension but stored as token-local debt. A selector roll
   therefore discarded the rejection and admitted a sibling strike one second
   later. The defect was symmetric: a side-level market condition was being
   attached to an option-token identity.
2. On 27 April, PEL treated every lifecycle whose role said
   `STRUCTURAL_CONTINUATION_WALL` as an authenticated L4 option wall. The
   position actually carried a causal spot-probe wall and an empty immutable
   wall identity. L4 reclaim could never authenticate it, so the position gave
   back a `+15.9%` peak and waited into the hard-loss region.

The dates reproduce the invariants; no date is read by production code.

### Structural Correction

1. The central MTEA observation builder normalizes
   `same_side_spot_chase_exhausted` to SIDE scope for every producer. Exact
   token-quality defects remain TOKEN scoped, and CE/PE ledgers remain
   independent.
2. A sibling strike cannot erase side-owned anti-chase history. Recovery still
   requires independent fixed market-time support buckets; callback frequency
   cannot manufacture evidence.
3. MTEA lifecycle identity, a structural-role label, and a canonical immutable
   option-wall identity are now separate facts.
4. Only a positively validated immutable wall identity receives L4 structural
   hold/reclaim authority. Causal spot/probe provenance remains visible in the
   lifecycle context but cannot suppress ordinary PEL forever while waiting for
   an impossible L4 proof.
5. The authenticated replacement-position handshake remains strict even for a
   non-wall lifecycle. This preserves an already-earned profit-lock episode and
   prevents an incoming candidate from resetting it into a fresh reversal
   ceremony.
6. No price, score, VWAP, momentum, duration, or support threshold changed.

### Verification

- Symmetric CE/PE unit proof: side-owned spot-chase debt survives a sibling
  strike roll and recovers only after three independent market-time buckets.
- Symmetric CE/PE PEL proof: a non-wall MTEA position can use ordinary PEL
  reversal/profit-lock evidence; a canonical wall still requires authenticated
  release.
- Canonical-wall proof: a structural role without immutable identity cannot
  claim L4 exit veto authority; a validated option-wall identity can.
- Focused remote suites: 227 tests pass.
- Full remote suite: 1,045 of 1,046 tests pass. The sole failure is unchanged on
  the base checkout: `test_cde_market_clock` patches global `time.time()` and
  Python logging itself consults that patched function while creating a
  `LogRecord`.
- `py_compile` and `git diff --check` pass.
- 20 April A/B control: patched and base both buy
  `NIFTY2642124250CE` at `09:22:07` for `INR 199.90` and sell at
  `09:30:38` for `INR 209.55`, realizing `+INR 13,172.25`.
- 23 April: the old `09:31:20` sibling-strike CE loss is absent. The subsequent
  CE setup arms, but its live multi-strike basket-participation proof remains
  incomplete; this is recorded as a separate runner-coverage gap, not hidden as
  acceptance of the later runner.
- 27 April: the old opening CE loss is absent. The valid PE enters at
  `09:33:00` for `INR 141.90`, reaches a `+15.9%` peak, and exits through two
  independent PEL profit-lock buckets at `09:43:40` for `INR 142.65`,
  realizing `+INR 1,462.50`. No impossible L4 reclaim wait occurs.

The full-day 20 April no-event cross-run continues as a broader regression; its
ledger is additive evidence and is not used to redefine these invariants.

## July 24 Immutable-Arm Participation Authority

### Incident

After the side-debt correction removed the invalid 23 April `09:31:20` CE
loss, the valid `09:36-10:05` CE runner still remained blocked. The exact
`24100 CE` reservation armed at `09:39:08` behind the canonical `24,200`
continuation wall and the index subsequently held the break. At materialization,
however, the ordinary 900-second basket recomputed `strike_move` from an older
window that included the preceding opposite leg. It therefore recognized only
a sibling strike even though every nearby CE had advanced from the immutable
reservation snapshot.

Raw premium evidence from the arm to the held break showed approximately
`+4.7%` to `+5.6%` across the nearby CE basket. Across the complete benchmark
runner, those same contracts gained roughly `+25%` to `+30%`. The failure was
therefore a proof-horizon mismatch, not weak breadth and not an entry threshold
problem.

### Structural Correction

1. A regular exact-session wall reservation now owns a causal arm snapshot for
   each token and symbol.
2. Current path ROC, giveback, strength, price floor and VWAP envelope checks
   remain current-tick requirements.
3. Only a failed `strike_move` check may be satisfied by advancement from the
   immutable arm quote. The required move remains the existing configured
   `STABLE_RETRY_WALL_RESERVATION_MIN_CANDIDATE_MOVE_PCT`; its value is
   unchanged.
4. Token and symbol must both match. A sibling can prove breadth but cannot
   donate identity, price, or execution authority to the reserved token.
5. The ordinary post-validation path and the neutral-L2 reservation consumer
   now call the same participation resolver. Callback order can no longer
   change the proof horizon.
6. Opening-specific proof profiles retain their existing route-owned horizons.
7. Compact arm-relative checks are persisted with reservation events and
   included in confirmation logs. Full candidate rows are not duplicated in
   the event payload.

No price, score, VWAP, momentum, duration, support-count, or move threshold was
changed.

### Focused Proof

- Final source-frozen 23 April replay user:
  `bt_apr23_armrel_v4_35e2494_20260423`.
- The old `09:31:20` CE loss remained absent.
- Exact reserved token `NIFTY26APR24100CE` entered at `09:42:25` for
  `INR 268.60`, quantity `1,040`, via `TR_STABLE_RETRY`.
- The token reached `INR 319.75` at `10:05:25`, approximately `+19.0%` from
  entry.
- PEL ignored transient contrary snapshots while the position contract
  remained supported. Three independent reversal buckets then produced
  `MTEA_FINAL_CLOSE_AUTHORITY.PASS`.
- The exact position exited naturally at `10:10:30` for `INR 294.35`,
  realizing `+INR 26,780` and retaining approximately `+9.6%`.
- No invalid churn followed. A `24300 PE` entry at `10:33:30` belonged to the
  separate benchmark `10:05-11:44` bear major runner, but remained late. Its
  `10:35` close was the verifier boundary and is not natural-exit proof.
- 27 April cross-regression avoided the old opening CE loss and completed the
  valid `09:33:00-09:43:40` PE lifecycle for `+INR 1,462.50`.
- 4 May completed the `09:15-09:40` old-loss negative-control window with zero
  orders.
- Focused fusion suite: 268 tests pass.
- Full repository suite: 1,048 of 1,049 tests pass. The sole failure is the
  unchanged `test_cde_market_clock` logging/mock interaction already present on
  the base checkout.
- `py_compile` and `git diff --check` pass.

### Replay Isolation Finding

The first transient acceptance processes were started as root with
`HOME=/home/administrator`. Their auxiliary handlers created root-owned
`gamma_signals.log` and `fusion_signals.log`, and paper subsequently failed
during logger import. Those processes were stopped, ownership was repaired,
and paper returned to a stable websocket-connected state.

Final acceptance ran as `administrator` with an isolated HOME, a fresh `bt_*`
user, and user-scoped cleanup. Transient replay launchers must retain these
properties; a diagnostic process may never share writable runtime files with
paper or live.

## July 23 Stable Retry Final-Reducer Acceptance

### Incident

On 2026-05-04, Stable Retry accumulated five source-lane support buckets for a
09:28 CE candidate. That lane was intended to be evidence-only, but its shadow
diagnostic looked executable and the older flow could reach an order before the
source-neutral MTEA result owned the final decision. A second ordering defect was
visible at 10:44: a central wall-handoff WAIT overwrote
`selected_token_live_premium_stalled`, so the current categorical rejection was
never recorded. The next transient pass then appeared cleaner than the evidence
actually was.

The date is reproduction evidence only. The defect applies symmetrically to CE
and PE whenever a producer-local wall handoff, a central WAIT, and a current
token rejection occur on the same callback.

### Structural Correction

1. Producer lane readiness remains non-executable:
   `entry_authority=false`, owner `MTEA_CENTRAL_CONTRACT`.
2. Reducer shadow telemetry now reports `EVIDENCE_READY_ONLY` or
   `EVIDENCE_WAIT_ONLY`; it suppresses the contract id because no order
   capability exists at that stage.
3. A central wall-handoff WAIT no longer replaces the producer's current token
   verdict. The current verdict is persisted first as typed WAIT or HARD_REJECT.
4. Central wall proof remains input evidence only. Its WAIT is deferred to the
   final reducer instead of short-circuiting evidence collection.
5. Stable Retry rebuilds the exact MTEA contract after its current token,
   snapshot, whipsaw and CDE checks. Only that final contract can feed
   `EntryAuthorityContractV1` and execution.
6. Existing exact-token, VWAP, ROC, wall, CDE, position and execution checks are
   unchanged. No threshold or date branch was added.

### Verification

- Symmetric CE/PE unit proof: a central WAIT cannot hide a current token
  rejection; a clean current pass still reaches the final reducer.
- Fusion suite: 265 tests pass.
- MTEA cache suite: 82 tests pass.
- Order contract suite: 37 tests pass.
- Entry execution suite: 8 tests pass.
- Entry shadow suite: 21 tests pass.
- DVR exact-execution/remap suite: 14 tests pass, including proof that the
  retired source-local `entry_authority=true` flag cannot bind an execution
  token without explicit producer evidence and the central exact-token
  preflight.
- Full repository suite: 1,042 tests pass.
- `py_compile` and `git diff --check` pass.
- Clean replay user: `bt_may04_mtea_v23_20260504`.
- Replay window: 2026-05-04 09:15-10:52, full state and diagnostic events.
- 09:28 source lane: `5/1`, but `EVIDENCE_READY_ONLY`,
  `entry_authority=false`, `contract=None`; no order.
- Valid runner preserved:
  `09:53:38 NIFTY2650524300PE BUY 125.75` to
  `10:29:30 SELL 164.45`, quantity 2,210, realized `+INR 85,527`.
- No 10:31 CE tail order.
- At 10:44 the exact-token stall is persisted as one categorical debt,
  required recovery becomes three independent buckets, and no delayed order
  appears through 10:52.
- Final replay ledger: one round trip, two order rows, realized
  `+INR 85,527`.
