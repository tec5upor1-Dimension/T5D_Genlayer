# Support Dispute Resolver — a GenLayer Intelligent Contract

A small, forkable template for **service-promise disputes** judged by GenLayer validators.
A user (or an agent) states what was promised, what went wrong, and links evidence. The
provider can respond. Validators, each running a different LLM, read the promise, the claim,
the response and the evidence pages, and agree on how much of the promise was kept, as one of
five payout buckets. The contract turns that bucket into a verdict and an exact escrow split.

This is the "support ticket" shape of the problems GenLayer's showcased use cases solve
(Internet Court, Collective Memory): *did the thing that was promised actually happen?*

## Flow

| Step | Method | Who |
|------|--------|-----|
| 1 | `open_case(respondent, promise, claim, evidence_url)` (payable: attached GEN is escrowed) | claimant |
| 2 | `respond(case_id, response, evidence_url)` (optional) | respondent only |
| 3 | `resolve(case_id)` | anyone |
| read | `get_case(case_id)`, `get_case_count()` | anyone |

Outcome buckets, computed **by the contract**, not by the LLM. The LLM must pick exactly one
of 0, 25, 50, 75 or 100 (its raw answer is snapped to the nearest bucket):

| Bucket (`score`) | Verdict | Respondent receives |
|------------------|---------|---------------------|
| 100 | `FULFILLED` | 100% of escrow |
| 25, 50, 75 | `PARTIAL` | 25%, 50% or 75% |
| 0 | `NOT_FULFILLED` | 0% (full refund) |

## Escrow

Attach GEN when calling `open_case` and it is held by the contract until `resolve` settles it.
The split is computed by the contract after consensus, with integer math so no wei is lost:

| Bucket | Respondent receives | Claimant refunded |
|--------|--------------------|-------------------|
| 100 | 100% | 0% |
| 75 | 75% | 25% |
| 50 | 50% | 50% |
| 25 | 25% | 75% |
| 0 | 0% | 100% |

(The respondent gets `escrow * bucket // 100`; the claimant gets the remainder, so no wei is lost.)

Accounting is stored on the case (`escrow`, `paid_to_respondent`, `refunded_to_claimant`,
`settled`), state is updated before transfers are emitted, and a resolved case can never be
settled twice. A case opened with no value attached just has nothing to settle. Transfers to
accounts go through the EVM interface and execute on finalization.

Appeals are intentionally not re-implemented here: GenLayer's Optimistic Democracy already
escalates disputed results through larger validator sets at the protocol level.

## Design choices worth copying

- **Validators bind the exact payout.** The score is a discrete payout bucket, and the
  validator function accepts the leader's result only if the bucket (and therefore the verdict
  and the settlement amount) is **identical** to the one it computed itself. There is no
  tolerance window, so every accepted result pays out exactly the same amount whichever
  validator led. A leader result that is off-bucket, or whose verdict does not match its
  bucket, is rejected. Only the free-text reasoning may differ in wording. The trade-off is
  that validators near a bucket edge can disagree, which GenLayer's appeal path resolves.
- **Contract owns the mapping.** The LLM only produces a bucket plus a short reason; the
  verdict and the payout are deterministic code, re-checked after consensus.
- **Prompt-injection hygiene.** Claim, response and fetched pages are wrapped in tags and the
  prompt states they are untrusted data. Inputs are length-capped, evidence URLs must be
  `https://`, and output is forced to strict JSON that is parsed and clamped.
- **Side effects after consensus.** All storage writes happen after `run_nondet_unsafe`
  returns, as GenVM requires.
- **Graceful evidence failure.** An unreachable URL becomes "(could not be fetched)" instead
  of aborting the case.

## Run the tests (no network needed)

```bash
pip install genlayer-test
pytest tests/direct -v
```

Tests use direct mode with mocked web and LLM calls, cover escrow payouts, and exercise the
validator path (`vm.run_validator`), including that an adjacent payout bucket, an off-bucket
score, or a verdict that contradicts its bucket are all rejected. Direct mode does not record outgoing transfers, so
payouts are asserted through the contract's own accounting; check real transfers in Studio.

## Try it in GenLayer Studio

1. Open https://studio.genlayer.com and load `contracts/support_dispute_resolver.py`.
2. Deploy (no constructor arguments).
3. Call `open_case` with a respondent address, a promise, a claim and an `https://` evidence URL.
4. Optionally call `respond` from the respondent account.
5. Call `resolve` and read the result with `get_case`.

## Ideas to extend

- Respondent-side stake (bond) that is forfeited on a `NOT_FULFILLED` verdict.
- Time limits: auto-resolve in the claimant's favor if the respondent never responds.
- Evidence categories: separate URLs for claimant and respondent with different weighting.
- Hook it to an ERC-8004 agent identity or an x402 payment receipt as evidence.

## Status

Written against the current GenLayer docs. Verified locally: `genvm-lint check` passes
(lint and validation) and all 35 direct-mode tests pass.

**Deployed to the GenLayer Rally TestNet** (2026-10-21) from the exact source in
`contracts/support_dispute_resolver.py`, then exercised end to end with a funded case.

| Item | Value |
|------|-------|
| Network | GenLayer Rally TestNet |
| Contract address | `0x2912872ad4C243023415C130E15F19797edDCfa3` |

Test case run on the deployed contract (`open_case` with 2 GEN attached, a second account as
respondent, no response and no evidence, then `resolve`):

| Field | Result |
|-------|--------|
| `status` | `RESOLVED` |
| `verdict` | `NOT_FULFILLED` |
| `score` (payout bucket) | `0` |
| `escrow` | `2000000000000000000` (2 GEN) |
| `paid_to_respondent` | `0` |
| `refunded_to_claimant` | `2000000000000000000` |
| `settled` | `true` |
