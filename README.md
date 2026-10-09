# Support Dispute Resolver — a GenLayer Intelligent Contract

A small, forkable template for **service-promise disputes** judged by GenLayer validators.
A user (or an agent) states what was promised, what went wrong, and links evidence. The
provider can respond. Validators, each running a different LLM, read the promise, the claim,
the response and the evidence pages, and agree on a score from 0 to 100. The contract turns
that score into a verdict.

This is the "support ticket" shape of the problems GenLayer's showcased use cases solve
(Internet Court, Collective Memory): *did the thing that was promised actually happen?*

## Flow

| Step | Method | Who |
|------|--------|-----|
| 1 | `open_case(respondent, promise, claim, evidence_url)` (payable: attached GEN is escrowed) | claimant |
| 2 | `respond(case_id, response, evidence_url)` (optional) | respondent only |
| 3 | `resolve(case_id)` | anyone |
| read | `get_case(case_id)`, `get_case_count()` | anyone |

Verdict bands, computed **by the contract**, not by the LLM:

| Score | Verdict |
|-------|---------|
| 70–100 | `FULFILLED` |
| 31–69 | `PARTIAL` |
| 0–30 | `NOT_FULFILLED` |

## Escrow

Attach GEN when calling `open_case` and it is held by the contract until `resolve` settles it.
The split is computed by the contract after consensus, with integer math so no wei is lost:

| Verdict | Respondent receives | Claimant refunded |
|---------|--------------------|-------------------|
| `FULFILLED` | 100% | 0% |
| `PARTIAL` | score% (e.g. 40 of 100 pays 40%) | the remainder |
| `NOT_FULFILLED` | 0% | 100% |

Accounting is stored on the case (`escrow`, `paid_to_respondent`, `refunded_to_claimant`,
`settled`), state is updated before transfers are emitted, and a resolved case can never be
settled twice. A case opened with no value attached just has nothing to settle. Transfers to
accounts go through the EVM interface and execute on finalization.

Appeals are intentionally not re-implemented here: GenLayer's Optimistic Democracy already
escalates disputed results through larger validator sets at the protocol level.

## Design choices worth copying

- **Equivalence on structure, not prose.** The validator function accepts the leader's result
  only if the verdict band matches *and* the score is within ±20. The free-text reasoning is
  allowed to differ in wording, so honest validators converge without needing identical text.
- **Contract owns the thresholds.** The LLM only produces a number plus a short reason; the
  verdict mapping is deterministic code.
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

Tests use direct mode with mocked web and LLM calls, cover escrow payouts, and also exercise
the validator path (`vm.run_validator`). Direct mode does not record outgoing transfers, so
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
(lint and validation) and all 27 direct-mode tests pass. **Not yet deployed** to Studio or
testnet, so run it there once before submitting. Confirm the `Depends` hash on the first
line matches the current Studio template.
