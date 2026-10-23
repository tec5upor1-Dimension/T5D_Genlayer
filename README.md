# Support Dispute Resolver — a GenLayer Intelligent Contract

A small, forkable template for **service-promise disputes** judged by GenLayer validators.
A user (or an agent) states what was promised, what went wrong, and **links public evidence**.
The provider can respond. Validators, each running a different LLM, fetch and read the evidence
pages and agree on how much of the promise was kept, as one of five payout buckets. The contract
turns that bucket into a verdict and an exact escrow split. **If the evidence cannot be
fetched, or is not enough to decide, nothing is settled** (see "Fail closed" below).

This is the "support ticket" shape of the problems GenLayer's showcased use cases solve
(Internet Court, Collective Memory): *did the thing that was promised actually happen?*

## Flow

| Step | Method | Who |
|------|--------|-----|
| 1 | `open_case(respondent, promise, claim, evidence_url)` (payable: attached GEN is escrowed; `evidence_url` is **required**, `https://`) | claimant |
| 2 | `respond(case_id, response, evidence_url)` (optional; any evidence URL given must load too) | respondent only |
| 3 | `resolve(case_id)` | anyone |
| read | `get_case(case_id)`, `get_case_count()` | anyone |

Outcome buckets, computed **by the contract**, not by the LLM. The LLM must pick exactly one
of 0, 25, 50, 75 or 100 (its raw answer is snapped to the nearest bucket):

| Bucket (`score`) | Verdict | Respondent receives |
|------------------|---------|---------------------|
| 100 | `FULFILLED` | 100% of escrow |
| 25, 50, 75 | `PARTIAL` | 25%, 50% or 75% |
| 0 | `NOT_FULFILLED` | 0% (full refund) |

## Fail closed: evidence or no settlement

Claims and responses are written by interested parties, so they are treated as unverified
allegations. The outcome rests only on the evidence pages, and `resolve` **reverts** (no state
change, no payout, escrow stays in the case) unless all of these hold:

- the claimant gave an `https://` evidence URL when opening the case (it is required);
- every evidence page (claimant's and, if given, respondent's) was fetched with HTTP 200 and
  contains at least 40 characters of readable text, otherwise `EVIDENCE_UNAVAILABLE`;
- the adjudicator explicitly reports `"evidence_sufficient": true` (a strict boolean),
  otherwise `EVIDENCE_INSUFFICIENT`.

Validators fail closed together. If the leader fails, a validator agrees only if it fails for
the **same reason** on its own fetch; a validator that can fetch and judge the evidence rejects
the leader's failure, so a leader cannot falsely block a resolvable case. A leader result is
likewise rejected if the validator cannot fetch the evidence or finds it insufficient. A case
that failed closed can be resolved later, once the evidence is reachable.

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
- **Evidence is mandatory.** There is no fallback that lets the model decide without pages it
  actually read; missing or unreadable evidence aborts the resolution.

## Run the tests (no network needed)

```bash
pip install genlayer-test
pytest tests/direct -v
```

Tests use direct mode with mocked web and LLM calls, cover escrow payouts, and exercise the
validator path (`vm.run_validator`), including that an adjacent payout bucket, an off-bucket
score, or a verdict that contradicts its bucket are all rejected, and that missing, unreachable,
empty, too-short or insufficient evidence leaves the escrow untouched. Direct mode does not record outgoing transfers, so
payouts are asserted through the contract's own accounting; check real transfers in Studio.

## Try it in GenLayer Studio

1. Open https://studio.genlayer.com and load `contracts/support_dispute_resolver.py`.
2. Deploy (no constructor arguments).
3. Call `open_case` with a respondent address, a promise, a claim and a **public `https://`
   evidence URL that returns real text** (the evidence is mandatory).
4. Optionally call `respond` from the respondent account.
5. Call `resolve` and read the result with `get_case`.

## Ideas to extend

- Respondent-side stake (bond) that is forfeited on a `NOT_FULFILLED` verdict.
- Time limits: auto-resolve in the claimant's favor if the respondent never responds.
- Evidence categories: separate URLs for claimant and respondent with different weighting.
- Hook it to an ERC-8004 agent identity or an x402 payment receipt as evidence.

## Deployment (GenLayer Rally TestNet)

| | |
|---|---|
| Contract | `0x6928ae0A7f85416F9839472b9f780713b6cd44eF` ([explorer](https://explorer-rally-testnet.genlayer.com/address/0x6928ae0A7f85416F9839472b9f780713b6cd44eF)) |
| Source | `contracts/support_dispute_resolver.py` at commit `db1f0446a09e9486dc6011b971066bb94c910a6f` |
| Deploy tx | [`0xe0a85003…44cefe6e`](https://explorer-rally-testnet.genlayer.com/tx/0xe0a850038aa369797dd609f0d8178ef4ed5652039d7fd71fdd5e2de544cefe6e) |

Live runs on this deployment (escrow of 2 GEN each):

1. **Fail closed.** Case 1 pointed at a GitHub HTML page whose text does not document the promise.
   `resolve` ([tx](https://explorer-rally-testnet.genlayer.com/tx/0x48f383987246b1347dc0f9989619d61bce7b1d5e4435f1e4aea52cf56ced0007))
   rolled back with `EVIDENCE_INSUFFICIENT` and consensus accepted the rollback. The case stayed
   `OPEN`, `settled: false`, and the 2 GEN escrow was untouched.
2. **Settled from evidence.** Case 2 pointed at the raw README text.
   `resolve` ([tx](https://explorer-rally-testnet.genlayer.com/tx/0x7d707cdf4463b971d7b69db7857d42802729a7f8b7ea4101d1c9575dad6f285f))
   returned `FULFILLED`, score 100, and `get_case` shows `paid_to_respondent` = 2 GEN,
   `refunded_to_claimant` = 0, `settled: true`.

## Status

Written against the current GenLayer docs. Verified locally: `genvm-lint check` passes
(lint and validation) all 53 direct-mode tests pass, and the contract is deployed and exercised on Rally TestNet (see above).
