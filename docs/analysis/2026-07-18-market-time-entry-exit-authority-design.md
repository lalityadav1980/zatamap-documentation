# Market-Time Entry And Exit Authority Design

## Status

This document defines the structural correction being validated after the April,
May, and July runner regressions. Dates are evidence and acceptance cases only.
No rule in this design may inspect a trading date, expected profit, benchmark
leg, or replay-only future information.

The implementation has three cooperating components:

- Market-Time Evidence Authority (MTEA) owns cross-tick entry evidence and
  recovery.
- L4 owns current structural support/resistance evidence for an option buyer.
- Position Exit Evidence Ledger (PEL) owns cross-tick exit confirmation.

None of these components may independently create an order. A BUY requires one
authenticated final entry contract. A SELL requires either a categorical safety
exit or one authenticated final exit contract.

Implementation is intentionally staged. The central contracts, gate facade,
order-boundary verification, lifecycle persistence, and PEL ownership are now
integrated on `codex/runner-coverage-consolidation-20260705`. Cross-day replay
remains the acceptance authority; passing unit tests is necessary but is not
sufficient to call the migration complete.

## Why The Regression Happened

The system has been deterministic, but it has not had one decision owner. A
candidate can pass one route and then be re-decided by a legacy guard using a
different snapshot. Conversely, historical side evidence can survive long
enough to release an entry after the current underlying has changed direction.
Exit code can classify the same position by its tag rather than by the market
evidence that admitted it.

The April evidence exposes both failure classes:

- April 17: a CE entered at the tail of the opening impulse; the long midday CE
  runner was missed while stale PE ownership produced losses.
- April 20: the current tree fragmented the morning CE runner, entered a PE in
  the 12:17-12:52 bull leg, and entered the real 12:52 PE runner late. The old
  profitable behavior partly depended on accidental L4 blocks and inconsistent
  exit ownership, so restoring the old output is not a valid fix.
- April 23/24: late tail entries and missed sustained runners show that the same
  ownership and final-authority conflict is not isolated to April 20.
- May 11/12: a valid DVR/MTEA decision was rejected by a second order-layer
  proof, while free-text hard-reject identities manufactured excessive debt.
- July 8/9/17: blind cooldowns and one opposite snapshot delayed or cancelled
  otherwise valid same-side momentum.

L4 data-integrity fixes remain mandatory: contract expiry and lot size come from
instrument metadata, timestamps use their actual unit, option and spot samples
are synchronized, and CE/PE flow is interpreted from the option buyer's side.
Those fixes removed accidental blocks; they did not cause the underlying
split-authority defect.

## April RCA Ledger

This ledger records evidence, not desired P&L. A row is complete only after a
clean full-day replay and cross-day verification on one frozen source snapshot.

| Evidence day | Observed use case | Structural gap | Invariant / status |
| --- | --- | --- | --- |
| 2026-04-16 | Opening and midday PE runners plus afternoon CE handoff | L4 derived IV/GEX from a hard-coded Thursday expiry, fixed lot size, mixed timestamp units, and unsynchronized spot; ordinary exit proposals could also bypass one position owner | Exact instrument metadata and causal spot pairing are implemented. Entry/PEL full-day acceptance remains pending. |
| 2026-04-17 | Early impulse followed by a long CE continuation | Stale opposite ownership and duplicate final authority can admit a tail trade yet suppress the later runner | One final entry receipt is implemented; full-day replay remains pending. |
| 2026-04-20 | Alternating morning runners and the 12:52 PE runner | Runtime benchmark leakage, duplicate order preflight, cross-side reclaim bypass, a circular policy-remap authority check, tag-derived profit protection, legacy early exit, and finally an incoming CE transfer that ignored the healthy held PE lifecycle | Future-data firewall, two-phase exact-token remap validation, one preflight, source-neutral mature-peak handoff, and held-lifecycle reconciliation are implemented. The 2026-07-23 full-state replay through 14:05 preserved the accepted morning and 13:01 PE ledgers, then proved that an unearned 13:43 CE accumulates three ordinary loss-invalidation buckets rather than receiving runner probe protection. Full-day acceptance remains pending. |
| 2026-04-21 | Corrupt/incomplete option chain | Replay previously fabricated option metadata for unmatched ticks | Exact-token instrument validation now fails closed; this date remains a data-quality SKIP. |
| 2026-04-22 | Missing regular-session opening feed | A cold partial day cannot certify opening ownership | Benchmark marks the date SKIP; no trading-rule change is allowed to compensate. |
| 2026-04-23 | 09:36 CE runner followed by the 10:05 PE major runner | Canonical L4 exposed an audit-only legacy policy wait as live MTEA quality debt; after the corrected profitable CE exit, a completed exact-token position handoff was re-decided by ordinary VWAP and a shorter 60-second premium window | Legacy-only L4 policy is audit-only only when canonical market data and identity are complete. Exact position-handoff sponsorship is source-neutral and may replace only stale VWAP/short temporal ceremony; v8 clean acceptance is in progress. |
| 2026-04-24 | Opening and midday PE runners followed by the 14:06 CE runner | Reported late entries and missing continuations require classification against the unified authority snapshot | Pending clean regression after April 23 closes. |
| 2026-04-27 / 28 | Multi-leg handoffs, including the 10:58 April 28 PE | Concurrent source changes invalidated earlier replay certification | Existing work is preserved; both days require a clean frozen-snapshot rerun. |
| 2026-04-29 / 30 | Long morning/afternoon runners | Partial coverage and loss entries implicate the same ownership, selected-token, and exit lifecycle boundaries | Pending clean regression; no day-specific threshold work is accepted. |
| 2026-05-04 | Three separate PE runners at 09:41, 11:48, and 13:12 inside a whipsaw day | An elapsed post-stop veto could suppress a genuinely new exact-token episode, while owner-transfer and target/VWAP tags could bypass PEL. The first PEL correction then protected every unconfirmed wall probe, including unearned losing positions on April 20. | A post-stop retry can release only for fresh, post-exit exact-token MTEA proof in independent market-time buckets. Strategic exit tags now require PEL. Wall-probe protection requires an already-armed profit lifecycle: the 09:53 PE survived its -4.81% pullback after a +12.01% peak and exited at 10:29 for +₹77,980.50. Later May 4 runners remain pending. |

### 2026-07-23 PEL Cross-Regression

The May 4 correction was accepted only after April 20 exposed and closed its
cross-day boundary:

1. May 4's 09:53 PE had already reached a +12.01% peak, armed profit
   protection, and accumulated 11 HOLD buckets. Its 10:00 spot-wall crossing
   was an unconfirmed reclaim probe, so PEL kept it in WAIT. The exact token
   recovered, reached +50.7% by 10:23, and exited at 10:29:30 for
   +₹77,980.50.
2. April 20's 13:43 CE had peaked only +0.25%, had no profit-protection
   lifecycle, and lost both token VWAP and its terminal owner. Treating its
   wall probe like the May 4 runner delayed the valid exit to the hard stop.
3. Probe protection therefore requires the existing
   `profit_protection_armed` lifecycle. It does not introduce a price, score,
   time, or day threshold. An unearned underwater position remains eligible
   for the ordinary symmetric loss contract.
4. In the corrected April 20 replay, terminal opposite ownership, actionable
   opposite L2, lost token VWAP, negative VWAP ROC, and an underwater position
   produced independent loss buckets at 13:51:35, 13:51:40, and 13:51:50. The
   third bucket authorized `loss_thesis_invalidation_confirmed` at -8.94%,
   before the categorical hard stop.

The frozen acceptance snapshot passed 1,027 unit tests. The April 20 replay
through 14:05 and the May 4 replay through 10:35 both completed successfully
from empty trade tables with full fusion evidence.

## L4 Correctness Preconditions

MTEA is intentionally unable to override an L4 structural failure. L4 must
therefore fail closed when its inputs are unavailable and must not manufacture
structure from defaults:

- Expiry and lot size are resolved from the exact `market.instruments` token.
  There is no weekday-based expiry or fixed-lot fallback in a decision.
- Elapsed time is computed from timestamp values with their real pandas unit.
  `datetime64[us]` is not converted as if it were nanoseconds.
- Each option observation is paired with a causal, bounded-age spot observation.
  Historical option prices are never repriced against the latest spot.
- OI rates retain one unit contract: fractional change per minute. A value of
  `0.005` means 0.5% per minute; consumers do not compare it with a percentage
  integer such as `0.5`.
- Premium and OI polarity is normalized from the option buyer's perspective.
  A positive PE buyer-flow signal supports PE; it is not interpreted using a CE
  sign convention.
- Missing expiry, lot size, synchronized spot, or market timestamp makes the
  derived IV/GEX feature unavailable. It cannot silently become a PASS, RESPECT,
  or BREAK vote.

These are data-contract corrections, not threshold changes. Their acceptance
tests use both CE and PE tokens, different expiries and lot sizes, pandas time
units, stale/unsynchronized packets, and OI rates around the intended unit.

## Entry Invariants

### Evidence is not execution

MTEA stores independent CE and PE episodes using strategy market time. It
deduplicates observations into fixed market-time buckets and classifies them as
SUPPORT, WAIT, HARD_REJECT, or CORROBORATED_OPPOSITE. WAIT is neutral. Free-text
diagnostics never define debt identity.

Historical SIDE evidence may recover a rejected episode, but it is never
executable. A final contract must authenticate all of the following on the
current market tick:

1. Requested side, exact token, exact symbol, source, trace, and tag.
2. Current exact-token premium participation and freshness.
3. Current L2/L3 option-buyer support and current L4 authority.
4. Current underlying ownership: the immutable wall remains valid and no
   categorical opposite transfer or confirmed wall reclaim is active.
5. Central episode readiness with no active opposite reservation.
6. The order has not been remapped to an unproved sibling token.

MTEA may release only temporal policy such as session age, repeated ceremony,
or anti-chase classification. It may never release a current-token failure,
underlying contradiction, wall reclaim, stale data, L4 structural block, or
position conflict.

An incomplete MTEA recovery episode does not become an extra entry gate: a
clean candidate with no categorical debt may execute through its existing
current route. The current-underlying sub-contract is always authoritative,
however, so confirmed opposite ownership, stale market evidence, wall reclaim,
or an active opposite reservation fails closed even while the outer recovery
contract is still inactive. A caller may consult the legacy continuation record
only when no structured MTEA contract was produced at all; it may not replace a
current-tick rejection while building the resulting position lifecycle.

The first independent exact-token probe is also the causal continuation wall.
A later admission is valid only while spot remains on the same directional side
of that probe: CE at or above it, PE at or below it. The probe timestamp must be
strictly earlier than admission market time. This creates no points, score, or
percentage threshold and prevents entry with a lifecycle wall already reclaimed.

### Current underlying authority

The missing entry channel is a structured, same-tick underlying contract. It is
not a new momentum threshold. It records:

- market timestamp and observed spot;
- expected option side;
- immutable original wall when a reservation exists;
- whether that wall is still broken;
- whether a confirmed opposite owner or held contradiction exists;
- the current raw/latched trend fields as audit data.

The contract fails closed when market time or current spot is missing. Raw or
latched trend disagreement alone is not necessarily a veto because those
classifiers can lag an early runner. A confirmed wall reclaim, active opposite
owner, or route-owned categorical contradiction is a veto. This preserves early
momentum without letting stale PE history authorize a BUY during a current bull
reversal.

### One final boundary

DVR, MOMENTUM_RIDE, STABLE_RETRY, and TR_STABLE_RETRY publish through the same
MTEA facade. The final order service verifies the immutable contract; it does
not run another momentum model. Route-local legacy cooldowns, skip counters, and
ownership proofs become bounded audit inputs or categorical current-tick vetoes.
They cannot independently reset, release, or delay an accepted central episode.

### Cross-side discounted reclaim

A profitable exit on one side may seed a useful opposite-side reclaim proposal,
but it cannot sponsor the new option token. This distinction is mandatory
because the old post-runner rule combined a recent profitable exit, a DVR
candidate, and improving premium geometry, then directly bypassed the current
token's VWAP and live-ROC waits.

The April 20 acceptance replay exposed the resulting split decision: at
11:10:21 MOMENTUM_RIDE classified the exact PE token as unsponsored while
STABLE_RETRY admitted the same token through a reclaim rule originally added
for an April 16 handoff. The PE was bought one second before the spot low and
subsequently lost more than 20% of premium value.

The corrected authority is scenario-based and symmetric:

1. The legacy reclaim detector may publish a bounded proposal with its current
   DVR, premium, and previous-position evidence.
2. An ordinary exact-token continuation contract cannot promote that proposal.
3. Only a current, identity-bound `MTEA_EXACT_EXECUTION_LEASE` or
   `MTEA_WALL_BREAK_HANDOFF` can release the below-VWAP reclaim.
4. Without that central wall contract, current VWAP, micro-ROC, acceleration,
   giveback, freshness, L2/L3/L4, and underlying checks remain authoritative.
5. A failed proposal records WAIT and remains eligible for later recovery; it
   does not create a blind cooldown or categorical debt.

No historical threshold changed. The change removes a second decision owner
and routes cross-side discounted entry through the same immutable wall lifecycle
used by every MTEA source.

### Position ownership is not replacement-entry suitability

The April 20 morning replay exposed a circular authority defect after the entry
and exit ledgers were first unified. A PE position had correctly retained its
immutable wall while the next CE episode accumulated sustained side evidence.
The proposed CE was still subject to exact-token and anti-chase entry checks,
but those checks also prevented an executable opposite BUY from existing. PEL
then waited for that executable BUY before releasing the PE. The held position
therefore prevented the replacement contract that was required to end it.

The structural correction separates the two questions:

1. MTEA may publish a non-executable, side-only
   `MTEA_OPPOSITE_POSITION_TRANSFER` contract after the normal three independent
   10-second SIDE support buckets.
2. The current support bucket must contain structured L2/L3/L4 and same-tick
   underlying proof. Packet count, raw L2, stale proof, and TOKEN-only support
   cannot create the contract.
3. Side-scoped categorical rejection debt must recover. Anti-chase debt remains
   attached to replacement-entry suitability and cannot hide loss of ownership
   by the held side.
4. PEL rebinds the contract to the exact held side and current strategy tick,
   then requires its own three independent reversal buckets before SELL.
5. The replacement BUY remains subject to its ordinary exact-token MTEA and
   current gate contract. Position transfer neither chooses nor sponsors it.

This distinction is symmetric for CE and PE and introduces no price, score,
VWAP, or momentum threshold. The focused April 20 acceptance replay provided
the first runtime proof: CE support reached 3/3 at 09:52:44, PEL reached 3/3 at
09:53:04 and closed the PE for +17.52%, the replacement CE entered through
TR_STABLE_RETRY at 09:53:39, survived the old 10:12 false-exit window, and
closed on corroborated reversal at 10:39:30 for +42.70%. The three morning
positions realized INR 194,935 with no loss or churn. This replay is evidence
for the invariant, not a date condition in the implementation.

The next full replay exposed the missing half of that contract. At 13:14 an
incoming CE episode was valid in isolation, but the held 24450/24500 PE still
owned its immutable 24,433.4 continuation wall and traded above an improving
token VWAP. Treating incoming-side authority as sufficient closed the PE at
+8.61% after a +18.37% peak even though its causal position evidence had not
failed. The generic handoff invariant is therefore two-sided:

1. The incoming side must own an authenticated current MTEA transfer contract.
2. The held MTEA lifecycle must independently release through confirmed
   original-wall reclaim, categorical exact-token sponsorship failure (token
   below VWAP, negative VWAP ROC, and deteriorating VWAP gap together), or an
   already-earned floor combined with negative premium ROC and deteriorating
   VWAP gap. The last path catches a real premium reversal before lagging VWAP
   crosses price and reuses existing floor/ROC facts rather than a new limit.
3. While the held wall and exact token remain sponsored, the incoming contract
   is recorded as HOLD/audit evidence and cannot build PEL reversal buckets.
4. Hard risk, earned profit-lock, and loss-thesis invalidation remain independent
   and are evaluated before transfer arbitration.
5. The rule is identical for held CE and held PE positions and adds no threshold.

The same audit found an environment-parity control-flow bug in the broker/live
flip path. DB/paper mode restored the authenticated replacement contract after
closing the old side, but the equivalent live block was indented below an
unconditional failure return and could never run after a successful close. A
live flip could therefore place the correct replacement option with no immutable
wall or PEL provenance. Cleanup plus restoration now live in one fail-closed
helper used by both paths. An unconfirmed close mutates nothing, cache-cleanup
failure blocks the replacement BUY, and only a fully authenticated contract is
restored. This is an execution-lifecycle correction; it changes no market gate.

## Exit Invariants

The position lifecycle must be derived from authenticated entry evidence, not
from an entry-tag suffix. An MTEA-admitted runner carries exact side/token/symbol,
entry market time, original wall, entry premium, and admission source into PEL.

PEL evaluates every current market tick. While the exact position remains
structurally sponsored:

- a one-snapshot premium pullback is WAIT, not an exit;
- a synthetic mature-profit floor is audit-only;
- a HOLD observation resets any pending ordinary profit-lock streak;
- token VWAP, VWAP rate of change, LTP-to-VWAP geometry, and peak giveback are
  recorded together rather than allowing one volatile LTP print to decide.

Categorical exits remain authoritative: hard loss limit, invalid position
identity, or an independently proven token breakdown. Immutable-wall reclaim
releases structural HOLD but does not manufacture an opposite direction by
itself; an authenticated transfer or earned profit/loss channel must still own
the exit. Earned profit locks may exit only after independent market-time
buckets confirm the same defect.

The explicit configured profit target and EOD are policy exits and remain
independent of PEL. All ordinary route-local proposals, including ladder locks,
L2 exits, Momentum Ride risk floors, DVR handovers, and CDE exit guidance, are
advisory for an MTEA-authenticated position until current PEL authority agrees.
The hard loss limit is never deferred.

The final SELL API enforces the same contract. This is necessary because older
fusion monitors for wall fade, L4 wall respect, fast abort, held-token health,
and composite position health call the close API directly instead of passing
through the trailing-stop proposal helper. For an MTEA-owned position those
calls remain observable proposals, but cannot execute without an exact current
PEL contract. The only independent releases are categorical hard-risk/EOD/target
policy. An authenticated opposite-entry contract can begin transfer only after
the held lifecycle independently releases; it cannot evict a healthy held wall
by itself. A declared but token-mismatched MTEA lifecycle fails closed rather
than silently falling back to legacy exit behavior.

A refused close is not a close lifecycle event. Callers must preserve the open
position cache, PEL state, monitor evidence, and retry eligibility; replay must
not notify its TSL clock that the position closed. This prevents a 409 refusal
from deleting MTEA ownership and making the next legacy proposal authoritative.

Entry and exit evidence use separate ledgers. Entry evidence cannot keep a
broken position alive, and exit evidence cannot retroactively justify an entry.

## Implementation Map

- `mtea_entry_authority.py` builds the symmetric same-tick underlying contract.
- `stable_retry_evidence_cache.py` owns source-neutral market-time episodes,
  reservations, exact-token probes, and fixed-bucket recovery.
- `fusion_signals.py` is the common MTEA producer for DVR, MOMENTUM_RIDE,
  STABLE_RETRY, and TR_STABLE_RETRY. It carries one full contract through the
  final candidate and order context.
- `mtea_position_lifecycle.py` freezes exact entry identity, first-probe wall,
  and the PEL boundary policy for the resulting position.
- `order_service_api.py` verifies the final BUY contract, persists the lifecycle
  in the order cache, selects source-neutral runner risk behavior, and requires
  PEL authority again at the final SELL boundary for every ordinary close path.
- `position_exit_evidence_cache.py` remains the sole cross-tick owner of ordinary
  exit confirmation.
- `db_backtest_runner.py` requires exact replay-day `market.instruments`
  metadata for every consumed NIFTY option token. It cannot derive expiry, lot
  size, strike, or tick size from a symbol fallback.

Runtime source contains no benchmark leg annotator or benchmark-derived field.
The benchmark document is an offline acceptance oracle only. A static test
guards that firewall. Replay also detects option ticks that do not join exact
instrument metadata and fails the day before decision code runs; April 21 is the
current positive proof of this data-quality behavior.

The order boundary performs one preflight and produces one immutable receipt
bound to user, side, token, symbol, tag, trace, and market timestamp. The DB or
broker executor verifies that receipt and cannot rerun ownership/momentum logic
against a later snapshot. This closes the former gap where a candidate passed
MTEA and was then re-decided inside `process_order` or its executor.

Every contract and ledger timestamp is derived from the strategy market tick.
Missing tick time fails closed; process wall time is never evidence. Evidence
clock availability is independent of log level, diagnostic windows, and fusion
event publication. In DB replay, BUY/SELL fills and PEL read the same canonical
replay-wide option-chain tick buffer. The legacy DB-price helper's wall-clock
compatibility fallback is explicitly rejected by MTEA/PEL consumers.

### Regular-session ownership

The shared market-data admission boundary accepts decision evidence only from
09:15:00 through 15:30:00 IST, based on the packet's exchange timestamp. A
09:00 auction packet that arrives late is still pre-open data and is rejected
before any L1-L5, MTEA, reservation, or PEL cache mutates. The decision must not
use callback arrival time to reclassify that packet as regular-session data.

Pre-open data may be stored by a separate diagnostic or market-data path, but
it cannot:

- seed a CE or PE evidence bucket;
- define the first continuation probe or immutable wall;
- create or cancel a reservation;
- vote in current L3/L4 ownership; or
- affect a position exit observation.

The process cache is cleared at session rollover and position/episode terminal
events. It is the one source read by every entry route; route-local copies are
audit projections and may never resurrect an expired central reservation.

Timestamp validity is enforced before feature mutation. An option packet with
no parseable exchange timestamp is logged once per token and dropped before it
can update Flow, premium ROC, L2/L3, L4, or MTEA. Cross-strike recalculation,
selected-token freshness, data-readiness stabilization, coordinator state,
regime evaluation, and Range-Scalp evaluation all use the same packet market
clock. They do not fall back to `time.time()`. Wall time remains permitted only
for operational concerns such as log throttling, network timeout, and broker
request latency; it is never a decision or evidence input.

## Legacy Authority Migration

The migration is deliberately behavioral rather than date-driven. For each
entry and exit path, every legacy check is classified into one of three roles:

1. **Categorical current-tick veto**: stale or missing market data, exact-token
   invalidity, position conflict, immutable-wall reclaim, confirmed opposite
   ownership, or hard risk policy. These remain authoritative and are included
   in the central contract.
2. **Evidence observation**: same-side support, neutral/warming state,
   deduplicated categorical rejection, or corroborated opposite structure.
   These update MTEA/PEL but cannot order by themselves.
3. **Audit-only policy**: repeated ceremony, packet-count skip counters, blind
   cooldowns, free-text reasons, tag-derived lifecycle guesses, and duplicate
   order-layer momentum proofs. These cannot change the central verdict.

No legacy caller may run a second market model after the final contract. The
order boundary verifies identity, freshness, market timestamp, and the signed
authority result. This is the generic repair for the observed late-entry and
missed-runner class: remove conflicting decision ownership rather than relax a
threshold until one historical day becomes profitable.

### Shared entry-guard authority matrix

The common selected-token guard applies the following fixed classification to
all four entry families. This matrix is behavioral; no row contains a date or a
route-specific price threshold.

| Legacy condition | With no current exact MTEA contract | With current exact MTEA contract |
| --- | --- | --- |
| Post-flip elapsed wait | Authoritative | Audit-only; continue through current gates |
| Post-loss same-side elapsed wait | Authoritative | Audit-only; continue through current gates |
| Post-loss opposite elapsed wait | Authoritative | Audit-only; continue through current gates |
| Post-profit opposite elapsed wait | Authoritative | Audit-only; continue through current gates |
| Post-profit epoch/re-entry elapsed lock | Authoritative | Audit-only; continue through current gates |
| Post-Master-Gate-exit elapsed wait | Authoritative | Audit-only; continue through current gates |
| Session-age ceremony | Authoritative fallback | Audit-only; continue through current gates |
| Same-side spot anti-chase extension | Authoritative | Classified as continuation; current gates still run |
| Current exact-token ROC/deceleration/giveback | Authoritative | Authoritative |
| Current exact-token freshness/identity | Authoritative | Authoritative |
| Current L2/L3/L4 or underlying contradiction | Authoritative | Authoritative |
| Immutable-wall reclaim/opposite reservation | Authoritative | Authoritative |

Only the named history-lock rows are accepted by the closed
`_mtea_legacy_history_guard_resolution` whitelist. Unknown reasons fail closed,
so a current premium or structural defect cannot be mislabeled as historical
ceremony by a new caller. Every demotion records strategy market time, authority
class, side, token, symbol, trace, and the original legacy detail in the entry
guard audit; the same payload is available to the normal bounded fusion-event
publication path.

The previous wall-handoff implementation incorrectly treated a fresh negative
selected-token ROC as temporal digestion merely because MTEA was active. That
made historical support stronger than the current traded token. The corrected
boundary preserves the current deceleration/stall veto and leaves the MTEA
episode alive as WAIT so a later independent support bucket can recover without
a blind cooldown.

The elapsed locks are evaluated after exact-token resolution. STABLE_RETRY no
longer stops validation before it can produce a SUPPORT, WAIT, or HARD_REJECT
observation. An ordinary candidate is still blocked by an active history lock;
only a same-tick, exact-side, exact-token MTEA contract can demote it. Tight-range
trade limits, stale-latch contradiction, position ownership, and current market
quality remain authoritative because they are current risk, not elapsed policy.

Post-flip close time is stamped after successful cache cleanup from the
exchange fill timestamp. The old implementation used `time.time()` and wrote
the value before clearing the cache, so replay/live ages could diverge and the
stamp could disappear immediately. Missing exchange time now produces an
`INVALID_MARKET_TIMESTAMP` diagnostic and no history mutation; there is no wall
clock fallback.

## Logging And Fusion Events

Every state transition logs and, when enabled, publishes a bounded fusion event
with:

- strategy market timestamp and 10-second bucket;
- source, side, token, symbol, trace, and episode id;
- observation class, categorical reason code, and rejection scope;
- support, rejection, and opposite-corroboration counts;
- immutable wall and current underlying authority checks;
- current token/L2/L3/L4 checks;
- final authority result and failed check names;
- entry lifecycle carried to PEL and the exit evidence decision.

Packet frequency must not change counts. Replay at one-second and five-second
process cadence must produce the same evidence transitions and decisions.

## Acceptance Matrix

Unit and integration proof must include:

- repeated identical rejects do not manufacture debt;
- three independent supports recover an episode after categorical debt;
- one opposite L2 snapshot cannot cancel a same-side reservation;
- sustained corroborated opposite evidence cancels symmetrically for CE/PE;
- stale SIDE history plus a current underlying contradiction cannot execute;
- a current wall-held exact-token continuation can execute for every entry gate;
- token remapping fails closed;
- current-tick categorical vetoes remain authoritative;
- a central order contract cannot be re-decided by an order-layer momentum gate;
- repeated packets in one bucket and TOKEN-only support cannot manufacture a
  position-transfer contract;
- sustained side-only ownership can release a held opposite position through
  PEL only after the held wall or exact-token sponsorship independently releases,
  without granting or selecting the replacement BUY;
- an authenticated incoming transfer remains HOLD while a symmetric CE/PE held
  lifecycle has both wall and token sponsorship;
- categorical held-token sponsorship failure can release transfer even before
  wall reclaim, while hard risk remains immediate;
- runner lifecycle is persisted for every MTEA source, not only DVR tags;
- PEL holds recoverable pullbacks but exits confirmed reversal and hard loss;
- direct legacy fusion close calls cannot bypass PEL or falsely report a close;
- a categorical hard-risk close and an authenticated opposite-entry transfer
  still reach the final SELL boundary;
- decisions are identical across replay process cadence.
- elapsed entry locks cannot stop evidence collection before exact-token
  resolution, but still block an ordinary unproved re-entry;
- a flip close persists its exchange market timestamp after cache cleanup and
  never falls back to process time.
- a cross-side discounted reclaim remains WAIT under ordinary exact-token MTEA
  and becomes executable only after central exact-wall materialization;
- disabling logs or fusion-event writes does not remove the MTEA/PEL market
  clock or change entry/exit decisions.

Replay acceptance covers the known scenarios without encoding their dates:

- April 16, 17, 20, 23, 24, 27, 28, 29, and 30;
- May 4, 6, 11, 12, 14, 15, 18, 19, and 20;
- July 8, 9, 13, 16, and 17.

Primary acceptance is structural: expected runner side, timely participation,
no opposite-side tail entry, no unjustified early exit, and no new churn.
Profit is a secondary diagnostic because a profitable output can still be
caused by future leakage or an invalid gate.

## Policy-Selected Token Binding RCA

The April 20 morning replay exposed a circular execution-identity dependency.
DVR had sustained PE side evidence for `NIFTY2642124550PE`, while runtime
contract policy selected `NIFTY2642124400PE` as the execution-safe strike. The
old sequence immediately required the selected 24400 token to own an existing
router or STABLE_RETRY reservation. When none existed, it returned
`remapped_execution_token_unproven` at 09:38:31. The shared fusion preflight
that evaluates the exact selected token was downstream of that return, so the
token was rejected before it could ever earn the proof being requested.

Forcing the source strike is not a valid correction. The policy resolver may be
protecting expiry, liquidity, or contract geometry, and May 19 demonstrated a
case where the detector/watch token was not the execution-safe contract.
Transferring the 24550 proof to 24400 is equally invalid because option-token
VWAP, freshness, IV/OI, and participation are identity-specific.

The generic correction is a two-phase execution contract:

1. An otherwise valid `remapped_execution_token_unproven` result may become
   `DVR_EXECUTION_TOKEN_VALIDATION_PENDING`. This state preserves source and
   execution identities but has `entry_authority=false`.
2. The policy-selected token proceeds through the existing shared L2/L3/L4,
   selected-token, wall, ownership, freshness, and anti-chase preflight.
   This is mandatory for a pending remap even when the ordinary opening DVR
   route would otherwise be exempt from the post-opening shared preflight.
3. CDE must approve the same current market tick.
4. MTEA must publish final TOKEN SUPPORT for the exact execution token and
   symbol with matching tag, trace, and market timestamp. SIDE support or a
   sibling token cannot satisfy this step.
5. Only then is `DVR_EXACT_EXECUTION_PREFLIGHT_AUTHORITY` created and written as
   an immutable order-cache receipt. The final order layer verifies the receipt
   and does not rerun momentum.
6. A failed preflight, stale/mismatched token, active opposite reservation,
   incomplete CDE result, or unavailable cache receipt fails closed.

This changes no score, premium, VWAP, ROC, wall, or session threshold. It is
CE/PE symmetric and applies to plain DVR, DVR runner, near-reclaim, and DVR
STABLE_RETRY handoff taxonomies. Unit acceptance proves pending is
non-executable, exact current proof can bind both sides, and stale/sibling/
opposite evidence remains blocked. Replay acceptance must still prove that the
result improves timely participation without restoring a loss-tail entry.

## April 20 Exit-Monitor Control-Flow RCA

The first held-position reconciliation was intentionally conservative: an
opposite entry episode could not evict a sponsored MTEA position until the
immutable wall reclaimed, exact-token sponsorship failed categorically, or an
earned profit floor failed alongside negative exact-token LTP ROC and a
deteriorating VWAP gap. This removed the false 13:14 PE transfer without making
the replacement BUY easier.

The next replay exposed a separate production lifecycle defect. At 09:32:20,
PEL correctly recorded the first independent `PROFIT_LOCK` bucket for the held
CE and refused the legacy `EXIT_LADDER_LOCK` proposal while waiting for its
second bucket. `_do_close` returned false as designed, but the caller interpreted
that as "monitor finished" and returned from the entire trailing-stop loop.
The position remained open while receiving no further exit evidence. The same
failure was possible after a broker ambiguity, mutex conflict, hard-stop close
failure, deferred-target proposal, or ordinary target proposal in paper/live.

The structural invariant is now explicit: a close proposal may terminate the
position monitor only after the shared SELL boundary confirms persisted closed
quantity. A refused or unconfirmed close logs
`TSL_15ONLY.CLOSE_NOT_CONFIRMED_CONTINUE` and returns to the next market-time
observation. Deferred-target state is consumed only after confirmed close. No
entry/exit threshold, date, side, or benchmark outcome participates in this
rule.

The v18 replay exposed a separate exit-classification defect after the repaired
CE entry. At 11:09 the position still had its immutable wall, traded 21.4% above
its exact-token VWAP, had positive token-VWAP ROC, and had no actionable
opposite L3 or MTEA transfer. A short negative LTP ROC compressed the LTP/VWAP
gap, and the old classifier called that compression "VWAP sponsorship fade."
It exited at 273.70; the exact token subsequently reached 314.00 and remained
near 310 through the next bullish leg. Gap compression measures LTP volatility;
it does not prove that VWAP sponsorship itself is falling.

PEL now distinguishes those channels without adding a threshold. For an
immutable MTEA position, an earned-floor breach becomes profit-lock evidence
only when negative LTP ROC and a deteriorating LTP/VWAP gap are accompanied by
either categorical loss of exact-token VWAP sponsorship or authenticated
stronger opposite structure (opposite MTEA transfer or actionable opposite L3).
Raw L1/L2 cannot supply the second channel. A negative VWAP slope while LTP
remains sponsored is HOLD; synchronized categorical LTP/VWAP deterioration or
a proved handoff retains the existing two-bucket profit-lock ceremony.
Unreserved positions retain their established ladder policy, and hard exit,
categorical token loss, and wall reclaim remain authoritative. The cache
revalidates the same contract independently so a caller cannot bypass it by
submitting `PROFIT_LOCK` directly.

The v19 replay exposed the temporal half of the same exit contract. The held PE
crossed its already-earned floor at 09:54:57 while an exact CE transfer was
authenticated. A small bounce above the floor then changed the caller's
per-tick boolean to false, erasing the first evidence bucket. By the time the PE
wall reclaimed and PEL rebuilt two consecutive buckets, the exact CE entry
window had expired. A trailing-floor crossing is historical position evidence,
not a property that becomes untrue on the next quote.

PEL therefore latches a floor breach by exact position identity and market
timestamp. The latch clears only when that option establishes a genuinely new
profit peak; it neither creates an exit nor relaxes a gate. A later bucket must
still reproduce negative LTP ROC and deteriorating LTP/VWAP geometry together
with categorical token/VWAP sponsorship loss or authenticated opposite
MTEA/actionable L3. This lets two independent handoff buckets survive a harmless
bounce while the 11:09 CE case remains HOLD: its token remained sponsored and
it had no stronger opposite structure. Audit and fusion payloads expose
current-floor state, latched state, breach market time, and the peak at breach.

The v21 replay exposed a separate route-taxonomy leak before the afternoon
acceptance point. The 09:40:51 PE had already reached the existing shared
mature-runner peak at +23.80%. By 09:52:44, MTEA had authenticated the opposite
CE with three independent SIDE buckets, while the exact PE had negative LTP ROC
and a deteriorating LTP/VWAP gap. Its token VWAP still rose because VWAP is a
lagging sponsorship anchor. PEL correctly rejected raw L1/L2, but the caller
withheld profit protection from STABLE/TR_STABLE positions until their older
route-specific ladder floor was crossed. The first profit-lock bucket therefore
arrived only at +4.15%, and the confirmed SELL at 09:55:04 retained just +5.46%.

The generic correction does not move that ladder threshold. An MTEA position
whose peak already satisfies the existing mature-runner definition may begin a
two-bucket PEL profit-lock ceremony when all current facts agree: authenticated
opposite MTEA transfer, negative exact-token LTP ROC, and a deteriorating
LTP/VWAP gap. Entry tag, source, CE/PE side, date, and desired return do not
participate. Raw L1/L2 cannot release the position, and authenticated opposite
structure with positive held-premium ROC remains HOLD; that negative proof
preserves the 13:14 recoverable PE pullback, whose LTP/VWAP geometry was still
improving. This rule applies equally to DVR, MOMENTUM_RIDE, STABLE_RETRY, and
TR_STABLE_RETRY positions and is independently revalidated by PEL rather than
trusted from the trailing-stop caller.

The v22 counterexample prevented that correction from being accepted in
isolation. The earlier PE handoff improved as intended: DVR entered at 09:36:36,
the option peaked +25.64%, PEL closed at 09:52:54 with +16.96%, and the next CE
entered at 09:53:39. But that CE then peaked only +9.86% and was closed at
10:00:50, forty minutes before the benchmark major runner ended. No opposite
MTEA transfer existed. The old floor latch promoted two buckets because exact
token VWAP ROC changed from approximately flat to `-0.01` and then `-0.06`;
LTP still traded above VWAP, the immutable 24,304.5 wall held, and the move
subsequently resumed. Treating a floating-point sign as categorical
sponsorship failure made a stable anchor more volatile than the LTP it was
meant to stabilize.

The corrected semantic contract introduces no slope threshold. Without
authenticated opposite MTEA/L3, a latched floor plus negative LTP ROC and gap
deterioration may release a mature MTEA winner only after the exact token
actually loses VWAP sponsorship and token VWAP ROC is negative. Authenticated
opposite structure may still release before the lagging VWAP cross, which
preserves the timely 09:52 handoff. The immutable wall therefore outranks
numerical VWAP noise, while categorical token failure on a mature winner,
proved transfer, hard loss, target, and EOD remain authoritative.

The v23 replay proved why maturity is part of that contract. It correctly
rejected the `-0.01` slope exit at 10:00:50, but one second across the next
market-time bucket the CE traded ₹0.33 below VWAP and the old floor latch closed
at 10:01:10. Peak profit had been only +9.86%, no opposite MTEA/L3 existed,
NIFTY remained 27.7 points beyond the immutable continuation wall, and the
option recovered above VWAP within the next twenty seconds before continuing
the major runner. A brief exact-token cross is therefore not allowed to promote
a low-MFE route floor over causal underlying ownership. Without authenticated
opposite structure, token/VWAP failure protects a position through the shared
existing mature-runner state; below that state, wall reclaim or categorical
hard/loss policy must end the lifecycle. This is symmetric and reuses the
existing maturity definition rather than adding a new number.

The v25 full-day replay proved a different boundary in the afternoon. The PE
runner entered on time at 13:02:09, survived the former 13:14 false exit, and
still had an intact 24,433.4 continuation wall and rising token VWAP at 13:18.
During a short underlying rebound, one CE episode accumulated recovered support
from STABLE_RETRY and became valid MTEA opposite-position evidence after three
categorical SIDE contradictions. The mature-peak shortcut treated that
debt-recovered episode as a clean proactive handoff and sold the PE at 183.30
(+7.41%). The same PE then rose to 226.45 before the 13:27 benchmark reversal.
This was not a missing bucket or threshold problem: the flat-state entry
contract and the incumbent-position eviction contract had been made equivalent.

They are now intentionally asymmetric. Recovered categorical debt may still
make an opposite candidate eligible for a new BUY after the existing MTEA
support ceremony; it does not disappear and its thresholds are unchanged. But
while the held position retains both its immutable wall and token sponsorship,
the proactive mature-peak shortcut requires an explicitly debt-free current
opposite episode. A debt-recovered episode must wait for the incumbent lifecycle
to release through wall reclaim, categorical token/VWAP sponsorship failure,
earned-floor failure, actionable direct opposite L3, or hard risk policy. This
preserves the clean 09:52 PE-to-CE transfer, whose opposite episode had zero SIDE
categorical debt, while rejecting the noisy 13:17 countertrend handoff. Missing
or malformed debt metadata fails closed at the position boundary. Fusion and
PEL audit payloads now expose both authenticated transfer authority and the
separate `opposite_mtea_transfer_clean_episode` decision.

## Canonical L4 WAIT Must Not Become MTEA Support

The April 20 v26 replay retained the 13:02 PE runner correctly, but later bought
`NIFTY2642124300CE` at 13:43:52 during a local premium spike and stopped out at
13:51:50. The current L4 buyer-structure contract had classified that tick as
`BUYER_STRUCTURE_QUALITY_WAIT`: the relevant wall/quality state was not ready,
but it was not a categorical defect and therefore correctly did not create
recovery debt. Its legacy compatibility surface represented the same result as
`WARN` with `blocked_entry=false`. MTEA consumed only those wrapper fields and
mistook the neutral wait for positive support.

The corrected boundary consumes L4's structured buyer authority and categorical
reason codes. `BUYER_STRUCTURE_QUALITY_WAIT` and an unavailable canonical
contract remain current-tick WAIT, never support and never hard debt. A plain
canonical pass remains support. A separately authenticated same-tick,
same-side, exact-token L4 route may still release the wait; verbose reason text
and a generic WARN cannot. The rule is symmetric for CE/PE and shared by DVR,
Momentum Ride, STABLE_RETRY, and TR_STABLE_RETRY. No wall, score, price, or
momentum threshold changed.

The v27 acceptance replay exposed one producer-side variant of the same defect.
At 13:43:52 spot was only 0.4 points beyond the target wall and the canonical
zone still had `breakout_confirmed=false`, but the legacy gate represented
`BREAKOUT_UNCONFIRMED` as a generic WARN before the authority contract was
built. The producer now publishes a structured
`target_break_confirmation_wait` for both `BREAKOUT_ABOVE`/CE and
`BREAKDOWN_BELOW`/PE until the existing confirmation flag becomes true. This
uses zone state and booleans rather than parsing the reason sentence; confirmed
breaks still become PASS immediately under the original confirmation rules.

The v28 fail-fast replay proved that this field must not be folded into L4's
global `quality_wait`. Doing so changed position ownership and closed the valid
09:20 CE at 09:32 even though its token remained above VWAP. The canonical
contract therefore publishes `target_break_confirmation_wait` as an independent
market fact. MTEA combines it into an entry-only `entry_support_wait`; position
ownership and PEL continue to consume the unchanged global quality contract.
This separation preserves the profitable CE hold while preventing the 13:43
unconfirmed-wall packet from manufacturing new-entry support.

The v29 fail-fast replay exposed the remaining consumer distinction. Suppressing
the unconfirmed-break packet entirely also removed the legitimate PE ownership
transfer that had closed the opening CE at 09:30 with profit. The CE then held
until a later wall reclaim and exited at 09:32 after giving the impulse back.
MTEA now records this case under a dedicated `POSITION_TRANSFER` support scope.
PEL may aggregate that symmetric CE/PE scope for risk reduction, but continuation
history, exact-token recovery, reservation materialization, and every entry gate
ignore it. Global data-quality WAIT remains invalid for both decisions. This
models the actual authority difference: an unconfirmed opposite wall may justify
reducing an already-earned position, but cannot justify increasing risk with a
new BUY.

The v30 fail-fast replay then exposed the required bridge between those two
decisions. The valid opposite transfer closed the PE at 09:52:54, and the exact
CE passed confirmed L2/L3/L4 at 09:53:39. Because the earlier packets were now
correctly stored only as non-executable `POSITION_TRANSFER`, continuation
history had one current SIDE bucket instead of three entry buckets and rejected
the handoff. Promoting the transfer buckets back to entry support would restore
the 13:43 unconfirmed-wall loss, so that is not a valid correction.

The bridge is a confirmed-close capability, not evidence relabeling:

1. PEL may close a position from `POSITION_TRANSFER` only through its existing
   independent market-time ceremony.
2. Only after the shared SELL boundary confirms persisted closed quantity does
   the central cache record a bounded `MTEA_POSITION_TRANSFER_HANDOFF` receipt.
3. The receipt binds the closed position, proposed side, exact token, symbol,
   transfer proof, premium, spot, and market-time bucket. It expires within the
   existing MTEA window and cannot remap to a sibling strike.
4. A later 10-second bucket must independently reproduce current SIDE and exact
   TOKEN support through the normal L2/L3/L4 and underlying contract. The
   receipt alone is never executable, and one same-bucket callback cannot
   consume it.
5. Ordinary categorical hard rejection still blocks the handoff. Anti-chase is
   retained in audit but no longer reclassifies a just-confirmed ownership
   transfer as a newly discovered chase.
6. Confirmed consumption clears the receipt with the shared side state. Failed
   or ambiguous SELL responses create no receipt.

This contract is symmetric for CE and PE and is consumed from the same central
resolver by DVR, Momentum Ride, STABLE_RETRY, and TR_STABLE_RETRY. It changes no
price, score, ROC, VWAP, wall, or momentum threshold. Logs and `fusion_events`
publish receipt creation/rejection, exact identity, market time, expiry, current
confirmation, and selected revalidation policy; diagnostic publication is not
part of the decision path.

## Shared EOD Market-Time Policy

Replay previously forced positions flat at 15:00 while the live DailyCloser and
several signal paths used 15:15. That split made late-session regression depend
on the execution environment. The default strategy EOD is now one shared
15:15 IST policy, with replay inheriting the live/paper value unless an explicit
backtest override is supplied. The same timestamp is the exclusive BUY cutoff:
15:14:59 remains signal time, while 15:15:00 belongs only to `EOD_EXIT`. An
explicit earlier operational BUY cutoff remains supported, but there is no
hidden order-layer runway contradicting the signal engine. Both boundaries use
strategy market time in replay; neither uses process wall time.

## Post-Transfer Anti-Chase Causal-Order RCA

The April 20 replay then exposed a causal-order defect after the first PE-to-CE
ownership transfer. Exact `NIFTY2642124200CE` support became source-neutral and
ready at 09:52:44, with ten completed side and exact-token buckets by 09:54:14.
The held PE prevented a replacement BUY, which was correct. A
`same_side_spot_chase_exhausted` observation first appeared later at 09:54:19.
The cache nevertheless selected `ANTI_CHASE_DISCOUNT_CONFIRMATION` solely
because an anti-chase row existed anywhere in the rolling window. It therefore
reclassified a previously observed continuation as a newly discovered chase
and required the advancing CE premium to become cheaper than its old proof.
After the PE closed, current CE L2/L3/L4 and exact-token proof passed, but this
retroactive policy kept the major runner blocked.

The generic invariant is chronological:

- anti-chase remains audit evidence and is never erased;
- a continuation is prequalified only when the normal quorum of independent
  SIDE support, exact-token support, and exact-token probe buckets all completed
  before the first anti-chase market-time bucket;
- support in the same bucket as the rejection cannot prequalify the episode;
- a prequalified continuation uses
  `PREQUALIFIED_CONTINUATION_CONFIRMATION` and does not demand a cheaper quote;
- if anti-chase precedes qualification, or only side-level evidence existed,
  the existing `ANTI_CHASE_DISCOUNT_CONFIRMATION` remains authoritative;
- current exact-token layers, immutable first-probe wall, categorical debt,
  opposite reservation, and final token identity remain mandatory in both
  paths.

This changes no price, ROC, VWAP, momentum, wall, or score threshold. It fixes
the temporal meaning of existing evidence symmetrically for CE and PE and for
every MTEA consumer. Logs and fusion events include the selected policy,
earliest anti-chase market time, prequalification result, and pre-anti-chase
side/token/probe bucket counts.

## Final Entry Evidence Must Not Re-Decide Central History

The April 20 v31 replay proved one final split-authority defect after the
confirmed PE-to-CE handoff. At 09:53:39, exact `NIFTY2642124200CE` L2/L3/L4,
underlying continuation, CDE, position-handoff identity, and MTEA temporal
authority all passed on the same market tick. The route then wrote its final
TOKEN SUPPORT for lifecycle/audit, but immediately treated that route-local
projection as a second historical decision. Its older raw lane still contained
one anti-chase rejection, so `effective_entry_ready=false` overruled the
already-authenticated central contract and suppressed the order.

The final resolver now has one narrow responsibility boundary:

- without an authenticated MTEA contract, the existing route-local
  `entry_authority && effective_entry_ready` rule is unchanged;
- with MTEA, only an active exact-token continuation/execution contract for the
  same CE/PE side, token, symbol, and market tick can own historical recovery;
- the route's final SUPPORT observation must itself be accepted, current, and
  marked as a complete contract from the exact DVR, Momentum Ride, or
  STABLE/TR_STABLE source;
- cancellation, reservation identity, and opposite central reservation remain
  independent fail-closed checks;
- route preflight, current selected-token freshness/ROC/acceleration/giveback,
  L2/L3/L4, CDE, and order-service binding still execute before this boundary;
- the final SUPPORT write remains in the cache and diagnostics, but cannot
  perform a second recovery calculation over history that MTEA just resolved.

This is symmetric for CE and PE and shared by DVR, Momentum Ride,
STABLE_RETRY, and TR_STABLE_RETRY. It changes no market threshold and grants no
authority to a stale, sibling-token, wrong-source, cancelled, or
opposite-reservation-blocked contract. The `MTEA.FINAL_EVIDENCE_AUTHORITY` log
identifies every order that used the central path so replay, paper, and live can
prove the integration rather than infer it from an eventual trade.

The v32 fail-fast replay proved that the same invariant must cross the final
BUY boundary. Fusion released the exact CE at 09:53:39 and emitted
`MTEA.FINAL_EVIDENCE_AUTHORITY`, but `order_service_api` returned 403 because it
again required the raw lane's `entry_authority` and `effective_entry_ready`.
Every identity and current-safety check in that verifier had passed, including
the MTEA token, symbol, market time, underlying contract, and position
lifecycle. The order layer now accepts the unchanged legacy-ready path or a
recognized exact MTEA capability as the history authority. The MTEA capability
must come from `fusion_current_layers` or central wall materialization; all
side/token/symbol/tag/trace/stage, current SUPPORT, market-time, reservation,
opposite-owner, underlying, and lifecycle checks remain mandatory. The order
layer therefore verifies the final capability without running another recovery
model.

## Confirmed Wall Continuation Must Survive Statistical Flicker

The April 20 v33 replay proved a different failure after the morning authority
chain was repaired. At 13:01:54, exact `NIFTY2642124500PE` passed current
underlying, L2, L3, and canonical L4 with `BREAKDOWN_CONFIRMED@24450`. MTEA
correctly stored the first exact-token quote and waited for an independent later
market-time bucket. At 13:02:04 the same token was stronger, L2/L3 and the
underlying remained aligned, and spot still held below 24,450, but L4's
instantaneous statistical confirmation changed to `BREAKDOWN_UNCONFIRMED`.
That one pulse converted the callback to entry-only `POSITION_TRANSFER`, so the
new central framework could not consume its own earlier confirmed proof.

Two structural defects caused the miss:

- the canonical buyer-structure contract did not publish the target wall's
  strike/token/symbol identity, forcing consumers to infer the wall from prose;
- continuation history used the first post-break spot as a wall proxy, so a
  harmless pullback could look like a reclaim even while the real wall remained
  broken.

The corrected contract changes no threshold:

1. Canonical L4 publishes structured target-wall identity and the existing
   `target_break_confirmed` boolean for both CE and PE.
2. A continuation probe may attach that wall only when the current exact-token
   side proof is fully active and the canonical break is genuinely confirmed.
3. The central cache preserves the first confirmed wall immutably in its fixed
   market-time bucket. Packet callbacks cannot replace it with a later wall.
4. A later bucket may revalidate an L4 target-break WAIT only for the same exact
   selected token, the same canonical wall strike/token/symbol, current valid
   L2/L3/underlying proof, and a wall that remains directionally broken.
5. Quality WAIT, hard veto, categorical debt, opposite reservation, token
   remap, changed wall, same-bucket callback, and wall reclaim remain fail-closed.
6. A first-time unconfirmed cross has no prior confirmed-wall capability and is
   still blocked. This preserves the v27 13:43 CE protection.

The resolver is part of the shared MTEA contract, so DVR, Momentum Ride,
STABLE_RETRY, and TR_STABLE_RETRY consume the same CE/PE-symmetric authority.
Logs and `fusion_events` include the immutable wall, current wall, reason,
market bucket, every check, and failed checks. Human-readable L4 verdict text is
audit-only and never reconstructed into authority.

## CDE Categorical Vetoes Must Create Recovery Debt

The April 20 v34 full-day verifier fixed the `13:02:04` PE entry and its
position lifecycle, then exposed a separate late-session authority gap. At
`14:28:42`, CDE rejected a CE at the session high with
`SESSION_BLOCK:side=CE_at_extreme` while coordinator state was `RANGE` and the
structural trend was `CONFLICTED`. Fusion classified that categorical veto as
an uncategorized WAIT. At `14:28:47`, inside the same 10-second market bucket
and at the same spot, a transient session-bypass pass reused support accumulated
before the veto. The CE entered exactly as the market reversed and stopped for
a loss.

The correction does not alter the session-extreme threshold or any entry
score. CDE now publishes `CDE_ENTRY_REJECTION_EVIDENCE_V1` with a stable reason
code, token scope, and recovery policy:

- session-extreme, coordinator, and structural conflicts are lane-local
  `HARD_REJECT` observations requiring independent later support buckets;
- cumulative L2 disagreement, cooldown, warming, and incomplete current data
  remain `WAIT`, so one opposite snapshot cannot reset a valid side episode;
- the human-readable rejection string remains diagnostic only;
- DVR, Momentum Ride, STABLE_RETRY, and TR_STABLE_RETRY consume the same
  structured contract;
- a pass later in the rejection's own market-time bucket cannot repay it.

This preserves MTEA's recovery ability while preventing historical support
from outranking a newer categorical safety decision.

## Exact Position Handoff Must Own Its Temporal Entry History

The April 23 v6 replay entered the 09:36 CE runner at `09:41:48` and, after the
canonical L4 legacy-policy correction, exited at `10:10:51` for a protected
profit as independent PE evidence took ownership. PEL then recorded an exact
`NIFTY26APR24350PE` handoff receipt at premium `251.70`. At `10:11:40`, a later
market-time bucket reproduced the same token and symbol with premium `252.80`;
L2, L3, canonical L4, underlying continuation, MTEA history, and every receipt
identity check passed.

Two older entry checks nevertheless re-decided that completed history:

- ordinary selected-token VWAP rejected the discounted option at roughly 11%
  below VWAP, even though discounted participation is the purpose of the
  authenticated handoff;
- after VWAP was made capability-aware, the local 60-second premium window
  started after the earlier PE rise and reported `move=-0.02%`, while a
  one-tick ROC reported `-8.31%/min`. The central ledger still held seven
  independent exact-token buckets and a later quote above the immutable
  handoff quote.

The correction is capability-based, not a wider threshold:

1. A successful PEL close creates a bounded exact side/token/symbol receipt in
   strategy market time.
2. A later fixed bucket must reproduce current exact-token L3, current side
   layers, current underlying continuation, and a quote no worse than the
   receipt quote.
3. That capability is source-neutral for DVR, Momentum Ride, STABLE_RETRY, and
   TR_STABLE_RETRY. A route may verify it but cannot rebuild different history.
4. The capability may substitute stale VWAP geometry and the duplicate short
   rolling move/one-tick ROC ceremony. It cannot release token freshness,
   identity, categorical debt, wall reclaim, opposite ownership, CDE, final
   order proof, or genuine multi-horizon premium deceleration.
5. A worse quote, same market bucket, expired receipt, sibling token, unknown
   source, or incomplete current layer fails closed.

The focused regression also exposed a dormant implementation error in the
multi-horizon deceleration guard: it read an undefined `_path` variable and
swallowed the exception. It now reads the actual `path` snapshot. Symmetric
tests prove the specialized handoff still blocks when both the rolling path and
live premium are deteriorating.

## Exact MTEA Authority Retires Route-Local History Re-Decisions

The April 23 v8 fail-fast replay proved the handoff contract itself was healthy.
At `10:11:40`, exact `NIFTY26APR24350PE` had a confirmed position-transfer
receipt, seven independent exact-token support buckets, current L2/L3/L4 and
underlying proof, and an accepted DVR/CDE decision. Two legacy consumers still
overruled that same completed history after the final MTEA boundary:

- STABLE_RETRY's current selected-token guard passed, but an older aggregate
  snapshot-strength floor independently blocked the order;
- DVR's final MTEA evidence was ready, but its execution flag was rebuilt from
  a detector-specific allow-list that could not represent source-neutral exact
  MTEA continuation.

Those consumers no longer own historical entry authority:

1. Current categorical token, market-data, L2/L3/L4, underlying, CDE, wall,
   ownership, and order-state vetoes remain fail-closed.
2. Once those current checks pass and the exact position-handoff capability is
   active for the same market tick, the old aggregate snapshot floor is audit
   data. It cannot re-score the history already authenticated by MTEA.
3. DVR execution authority accepts either its unchanged detector proof or an
   authenticated source-neutral MTEA exact-token continuation for the same
   side, token, symbol, trace, and market tick.
4. The order boundary recognizes that MTEA source only for an unremapped exact
   token. A sibling strike or changed identity remains blocked and must build
   its own evidence.

This changes no score, VWAP, ROC, acceleration, wall, or momentum threshold.
It removes duplicate decision ownership while retaining legacy values only in
logs until the cross-date suite proves they can be deleted safely.

## Exact Continuation Owns a Restarted Short Move Ceremony

The clean April 29 replay proved the morning `09:23-10:28` CE major runner,
then exposed a separate shared entry-authority defect during the `13:08-14:30`
PE major runner. At `13:17:28`, current L2/L3/L4 and underlying direction
passed, the canonical opposite CE owner had released, and MTEA held independent
exact `24400 PE` support buckets. The current premium tick was strongly positive
and the rolling premium path was still rising. The legacy 60-second move window
had restarted after the first recovery and measured only `+0.66%`, so its
ordinary `+1.00%` ceremony overruled the completed exact-token history.

The correction does not lower or remove that ordinary anti-chase gate:

1. An ordinary candidate still needs the existing 60-second premium move.
2. Only an authenticated same-tick `MTEA_EXACT_TOKEN_CONTINUATION`, with the
   exact side/token/symbol, released opposite owner, current VWAP-geometry
   capability and a fresh or anti-chase revalidation policy, may classify the
   restarted move window as audit-only.
3. The selected token must still have current live premium participation, a
   non-deteriorating rolling path, controlled giveback and current snapshot
   strength. A positive micro print against a declining path remains blocked.
4. Multi-horizon deceleration, current identity/freshness, categorical debt,
   wall/opposite evidence, CDE, final evidence and order-risk gates remain
   authoritative.
5. DVR, Momentum Ride, STABLE_RETRY and TR_STABLE_RETRY consume the same
   source-neutral capability for CE and PE.

This changes no price, score, VWAP, ROC, wall, momentum or session threshold.
It removes only a duplicate temporal history decision after MTEA has already
proved the exact continuation in independent strategy-market-time buckets.

## Position Replacement Requires Final Entry Preflight

The post-entry April 29 acceptance trace found a different authority split. The
`24400 PE` entered at `13:17:38`, survived its initial pullback and reached an
`18.18%` peak while its immutable wall and exact-token VWAP sponsorship still
held. At `13:56:22`, a provisional CE episode completed directional transfer
evidence, so PEL closed the healthy PE. The same CE then failed its final
selected-token move gate on that market tick. Partial opposite ownership had
therefore evicted a position that no executable replacement could take over.

The source-neutral handoff contract now has two explicit capabilities:

1. `MTEA_OPPOSITE_POSITION_TRANSFER` remains directional risk evidence. It may
   participate after wall reclaim, exact-token sponsorship invalidation, an
   earned-floor release, or another independent held-lifecycle failure.
2. `MTEA_REPLACEMENT_ENTRY_PREFLIGHT` proves that one exact opposite token has
   completed every current entry gate in the current fixed market-time bucket.
   DVR, Momentum Ride, STABLE_RETRY and TR_STABLE_RETRY publish this through the
   same final TOKEN SUPPORT boundary they already use before `process_order`.
3. The proactive mature-peak shortcut requires both capabilities, a clean
   opposite episode, held-token LTP/gap deterioration, and PEL's independent
   profit-lock buckets. Directional L1-L4 evidence alone cannot close a healthy
   wall-held winner.
4. The final preflight does not execute or select a replacement order. After
   the SELL is persisted, the existing exact-token handoff receipt still binds
   the later BUY and all current order/risk checks remain authoritative.

This adds no market threshold. It makes the entry-to-exit handoff atomic at the
decision-contract level: a healthy winner cannot be proactively replaced by a
candidate that has not passed its own final gate.

### April 29 acceptance replay

The clean full-day replay `local-apr29-replacement-preflight-v3-20260720`
validated both sides of the contract with full logs and persisted fusion
events:

- `23950 CE` entered at `09:27:59` for `290.40`, survived the old `10:31`
  provisional PE transfer, reached a `68.63%` peak, and exited at `13:36:15`
  for `445.75` (`+Rs 151,466.25`) only after the held lifecycle weakened.
- `24350 PE` entered at `13:41:03` for `210.85`. Its post-entry low was
  `206.90` (`1.87%` MAE), it reached a `30.47%` peak, survived the old
  `13:56` false CE transfer point, and closed on the last tradable quote at
  `15:14:59` for `272.70` (`+Rs 80,405.00`).
- The two-trade day realized `+Rs 231,871.25` with no loss trade or churn.

The clock entry is later than the hindsight `13:08` swing boundary, but it is
not a late-premium chase: the selected PE was still below its VWAP through the
initial chop, entered with only `1.87%` subsequent drawdown, and captured the
substantive premium expansion. This is the required acceptance shape: current
tradable-token evidence, not a future benchmark boundary, owns timing.

## Legacy Authority Retirement Inventory

Legacy code is not permitted to remain as a hidden second decision after a
central MTEA/PEL capability. The retirement rule is explicit:

- packet-count cooldown and repeated free-text failure debt are replaced by
  fixed market-time evidence buckets and structured categorical identities;
- route-local historical recovery after final MTEA authority is audit-only;
- order service verifies immutable identity/current-risk fields on the issued
  capability and does not run a second momentum model;
- detector-specific execution allow-lists may not reject a valid source-neutral
  exact-token MTEA contract;
- aggregate snapshot/history scores may remain diagnostic, but cannot veto an
  exact handoff after their corresponding current guard passed;
- token remap, stale/malformed market data, position conflict, hard stop,
  categorical current-tick veto, wall reclaim, and confirmed opposite ownership
  remain independent authority and are not classified as legacy.

Every remaining `False`/`403` path after `MTEA.FINAL_EVIDENCE_AUTHORITY` is part
of the static and replay inventory. It must be classified as either an immutable
identity/current-risk verifier or removed from decision authority. A date-level
profit improvement is not acceptance proof.

## Rollout

1. Keep the current multi-day replay on one committed source snapshot.
2. Implement pure contract helpers and symmetric tests in an isolated worktree.
3. Run focused tests, then the complete test suite.
4. Replay the smallest causal windows with full logs and fusion events.
5. Run the complete cross-date matrix with preserved per-user ledgers.
6. Integrate, commit, and push only after the structural checks pass.
7. Deploy paper first and inspect MTEA/PEL transition logs before live rollout.
