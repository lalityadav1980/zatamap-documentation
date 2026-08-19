# Reproducibility and Review Protocol

[White paper](README.md) · [DVR](DVR.md) · [Momentum Ride](MOMENTUM_RIDE.md) · [Stable Retry](TR_STABLE_RETRY.md) · [CDE](CDE.md) · [MTEA](MTEA.md) · [CTAK](CTAK.md) · [PEL](PEL.md) · [PTL](PTL.md)

**Document class:** Methods supplement<br>
**Reference baseline:** `zatamap-trade-api@11ad49ec87362aa066afa76997cb3506b28bf076`<br>
**Purpose:** Make architecture and parity claims independently auditable

## 1. Claim boundary

The white paper makes claims about software architecture, causal ordering,
identity binding, and reproducible strategy semantics. It does not claim that:

- the strategies are profitable;
- their thresholds are statistically optimal;
- paper fills reproduce broker execution;
- observed correlation establishes causation; or
- historical parity guarantees future live behavior.

Any future empirical paper must state its hypothesis, evaluation cohort,
transaction-cost model, missing-data policy, and statistical method separately.

## 2. Immutable source baseline

All implementation claims in revision 1.1 refer to the full commit:

```text
11ad49ec87362aa066afa76997cb3506b28bf076
```

Reviewers should detach or create a clean worktree at this commit. Branch names
are mutable and are not sufficient for reproduction.

## 3. Claim-to-source map

| Claim | Primary source | Verification surface |
| --- | --- | --- |
| Strategy decisions use immutable exchange-time frames | `trade_authority/decision_frame.py` | `tests/test_trade_decision_frame.py`, market-clock tests |
| DVR observes exact-token discount recovery | `discounted_recovery_detector.py` | DVR detector and CTAK DVR suites |
| Momentum first produces side-scoped evidence | `trade_authority/adapters/momentum_ride_entry_evidence.py` | Momentum entry-evidence tests |
| Stable Retry publishes current exact-token evidence | `trade_authority/adapters/stable_retry_entry_evidence.py` | Stable Retry entry-evidence tests |
| CDE explicitly owns current entry policy | `central_decision_engine.py` | CDE market-clock, direction, handoff tests |
| MTEA owns temporal continuity and reservation | `mtea_entry_authority.py`, `trade_authority/entry/` | MTEA entry-authority tests |
| CTAK binds capability to identity and lifecycle | `trade_authority/kernel.py` | trade-authority kernel tests |
| Central adapter is the sole automated entry submission seam | `entry_order_submission.py` | seam registry and static architecture tests |
| PEL separates current assessment from cross-tick continuity | `trade_authority/exit/reducer.py`, `trade_authority/exit/continuity.py` | PEL reducer and continuity tests |
| PTL owns thesis lifecycle and ordinary discretionary exit authorization | `trade_authority/exit/thesis_lifecycle.py`, `trade_authority/exit/authorization.py` | PTL lifecycle and PEL exit-authorization tests |

Test filenames can evolve after the pinned baseline. Reviewers should use
`rg --files tests | rg '<topic>'` and record the exact discovered set.

## 4. Data provenance contract

Each replay dataset should include or permit derivation of:

- exchange timestamp with timezone;
- feed monotonic sequence;
- packet index within the source WebSocket frame;
- instrument token and trading symbol;
- instrument type, strike, expiry, and lot size;
- LTP, cumulative volume, OI, and option VWAP inputs;
- NIFTY spot observations; and
- source session date.

Historical rows predating persisted sequence or packet lineage cannot support a
claim of exact tick-order parity. They can still support a weaker timestamp-
ordered study if that limitation and tie-breaking rule are explicit.

## 5. Ordering protocol

For strict replay, reconstruct the original provider frames. Order frames by
their persisted ingestion lineage and retain packet order inside each frame:

```text
provider frame / trace identity,
ingestion_sequence strictly increasing,
packet_index strictly increasing within frame
```

Do not regroup a provider frame by exchange second, promote spot packets ahead
of options, or sort by database insertion or NATS publish/receive time. A frame
may contain adjacent exchange timestamps; each packet advances replay market
time and the historical quote snapshot immediately before its strategy
processing. Reject or separately classify rows with invalid exchange timestamp,
trace, declared batch projection, sequence, or packet position. Report
duplicate/non-increasing lineage before running the experiment.

## 6. Dataset quality report

For each session, publish:

| Metric | Required |
| --- | --- |
| Session window and timezone | Yes |
| Total ticks and distinct tokens | Yes |
| NIFTY first/latest exchange timestamp | Yes |
| CE/PE first/latest exchange timestamp | Yes |
| Instrument count by type and expiry | Yes |
| Missing exchange timestamps | Yes |
| Missing sequence/packet index | Yes |
| Duplicate order keys | Yes |
| Maximum and percentile continuity gaps | Yes |
| Current-expiry CE and PE coverage | Yes |

Data-invalid sessions should remain visible in the report rather than being
silently removed after their results are known.

## 7. Isolated replay protocol

1. Create a clean code worktree at the pinned commit.
2. Use an isolated backtest user and run identifier.
3. Scope cleanup to that user/run wherever the schema permits.
4. Use a dedicated NATS subject or namespaced run ID.
5. Use an ephemeral or run-unique consumer; never share a durable with live or
   paper.
6. Configure `BACKTEST_SOURCE=nats` and the canonical tick-batch source.
7. Retain historical instrument/quote and simulated broker behavior in the
   mock provider.
8. Advance replay time from each admitted exchange batch before strategy
   processing.
9. Persist fusion events, authority receipts, order intents, fills, trail data,
   and close outcomes.
10. Stop at the declared exchange boundary and record the end-of-day policy.

The replay producer must not publish onto a live subject unless a tested
environment boundary makes cross-consumption impossible.

## 8. Parity comparison

Compare in stages so the first divergence is observable.

| Stage | Comparison key | Examples of compared fields |
| --- | --- | --- |
| Input | provider-frame lineage | trace, batch size, timestamp, sequence, packet index, token, LTP, volume, OI |
| Decision frame | frame identity | hash, spot, option snapshots, callback sequence |
| Producer | source observation | disposition, side, exact identity, reason codes |
| MTEA | episode/candidate | bucket, debt, reservation, lease, hash |
| CDE | current verdict | approved, rejection class, tag, exact identity |
| CTAK | capability | lifecycle transition, source bindings, validity |
| Intent | parent order | side, token, quantity policy, market timestamp |
| Fill | execution model | price, quantity, status, provider reason |
| PEL | owned lifecycle | assessment, continuity, state, exit authorization |

P&L is a terminal consequence and should not be used as the first parity
comparator.

## 9. Symmetry protocol

For every directional rule, construct a mirrored CE/PE test by reversing the
underlying move and option-side identity while holding absolute magnitudes and
market-time structure constant. Any divergence must be explained by explicit
market facts such as moneyness or liquidity, not by an accidental side-specific
threshold.

Required symmetry audits include:

- spot alignment;
- velocity and magnitude;
- VWAP discount/extension;
- target-wall distance;
- current-token and basket sponsorship;
- rejection recovery;
- owner transfer; and
- PEL reversal/retention policy.

## 10. Expiry protocol

Classify expiry dynamically:

```text
is_expiry_day := instrument.expiry_date == exchange_session_date
```

Report days-to-expiry distribution. Do not infer expiry from weekday. Compare
expiry and non-expiry cohorts separately because gamma, theta, spreads, and
liquidity can change the mapping from spot movement to premium path.

## 11. Statistical evaluation guidance

For empirical strategy evaluation, preregister:

- primary endpoint and unit of analysis;
- development, validation, and untouched test cohorts;
- handling of overlapping trades and multiple daily opportunities;
- commissions, taxes, spread, slippage, and partial fills;
- missing/corrupt session policy;
- confidence intervals and multiple-comparison correction;
- regime, expiry, direction, and time-of-day strata; and
- parameter sensitivity and ablation analysis.

Report entry coverage, false-start rate, maximum favorable excursion, maximum
adverse excursion, entry delay, exit delay, and opportunity capture in addition
to P&L. Avoid selecting “major runners” with future information for online
policy construction.

## 12. Documentation review checklist

- [ ] Abstract distinguishes architecture claims from return claims.
- [ ] Every acronym is defined at first use.
- [ ] Every diagram agrees with the stated authority boundary.
- [ ] Thresholds are labeled baseline configuration, not universal truth.
- [ ] Exchange time is distinguished from transport and process time.
- [ ] CE/PE symmetry is explicit.
- [ ] Expiry is contract-derived.
- [ ] Exact-token proof cannot move across strikes.
- [ ] Signals, temporal authority, policy, execution, and position authority are
      not conflated.
- [ ] Limitations and threats to validity are present.
- [ ] Source commit and verification surfaces are immutable or discoverable.
- [ ] No credential, account ID, infrastructure host, or personal path is
      published.

## 13. Change-control protocol

When implementation behavior changes:

1. update the affected monograph;
2. update the reference commit in every document;
3. identify whether the change affects a producer, MTEA, CDE, CTAK, PEL, or an
   operational adapter;
4. state whether thresholds or only architecture changed;
5. add negative and symmetry tests;
6. run a declared replay cohort and untouched holdout;
7. record the first parity divergence if one exists; and
8. publish a new document revision rather than silently rewriting the prior
   experimental claim.

## 14. Revision 1.1 validation record

On 19 August 2026, the documentation review used a clean detached worktree at
`11ad49ec87362aa066afa76997cb3506b28bf076`. The targeted verification set
covered:

- Central Trade Authority Kernel identities, receipts, and transitions;
- Discounted Volume-Weighted Average Price Recovery candidate lifecycle and
  exchange-time recovery;
- Momentum Ride and Trend-Reversal Stable Retry evidence adapters;
- Central Decision Engine cumulative-direction reconciliation;
- Market-Time Evidence Authority reservation reselection and wall receipts;
- Position Exit Evidence Ledger assessment and continuity;
- PEL Thesis Lifecycle and exit authorization; and
- NATS provider-frame replay ordering and runtime-mode handoff.

The result was 308 passing tests and 271 passing subtests with zero failures.
Two non-failing dependency deprecation warnings were observed: one for the
legacy `py_vollib` import and one for a `pandas_ta` option under pandas 3.0.
Neither warning alters the authority or replay claims in this revision, but
both should remain visible as maintenance debt.
