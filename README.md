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
| 3b | `report_evidence_failure(case_id)` (re-checks the evidence; counts confirmed failures) | claimant or respondent |
| 3c | `recover(case_id)` (after 3 consecutive confirmed failures: full escrow back to the claimant) | claimant or respondent |
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

## Recovery: broken evidence cannot lock escrow

Because `resolve` fails closed, a respondent could submit an `https://` URL that always returns
404 and block every settlement. The evidence URL cannot be changed, so the contract has a
bounded exit that does **not** depend on the blocking party:

1. Either party calls `report_evidence_failure(case_id)`. Validators re-run the evidence
   check and must agree on the **exact** result: `OK`, `EVIDENCE_UNAVAILABLE` or
   `EVIDENCE_INSUFFICIENT`. A confirmed failure increments `failed_checks`; if the evidence
   works again the counter resets to 0, so only *consecutive* failures count. This call never
   moves escrow.
2. After `RECOVERY_THRESHOLD` (3) consecutive confirmed failures, either party calls
   `recover(case_id)`. The **entire** escrow is returned to the claimant (the depositor), the
   case becomes `RECOVERED` and `settled`, and nothing is paid to the respondent.

Guarantees: only the claimant or respondent can report or recover; recovery needs consensus-
confirmed failures, never one party's word; a payout to the respondent still requires
evidence (`resolve`); and a case that is `RESOLVED` or `RECOVERED` is closed for good, so
`resolve`, `recover` and `report_evidence_failure` all revert afterwards (no double release).

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
`settled`, `failed_checks`, `last_failure`), state is updated before transfers are emitted, and a resolved case can never be
settled or recovered twice. A case opened with no value attached just has nothing to settle. Transfers to
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
empty, too-short or insufficient evidence leaves the escrow untouched. The recovery tests cover an unavailable respondent URL,
permanently insufficient evidence, recovery of the full escrow balance (including odd amounts),
authorization, the consecutive-failure counter, validator agreement on the failure reason, and
repeat settlement or recovery being rejected. Direct mode does not record outgoing transfers, so
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
| Contract | `0x4Fa714bA78151e21C93Ea1C4D53e5188FB3C3D1E` ([explorer](https://explorer-rally-testnet.genlayer.com/address/0x4Fa714bA78151e21C93Ea1C4D53e5188FB3C3D1E)) |
| Source | `contracts/support_dispute_resolver.py` (the committed file, 466 lines) |
| Deploy tx | [`0x885ca978…685dbf8e`](https://explorer-rally-testnet.genlayer.com/tx/0x885ca9785b9f8525a0d6f0912bedec5016a224f04c3bf34ba7631bf0685dbf8e) |

Live recovery run on this deployment (1 GEN escrow, evidence URL that always returns 404):

1. `open_case` with 1 GEN ([tx](https://explorer-rally-testnet.genlayer.com/tx/0xa9c6cd52de3e75a60e4a333eb67d70f6cb6a72fe5dfc3013bbbf77dc225aad08)).
2. `report_evidence_failure` three times, each confirmed `EVIDENCE_UNAVAILABLE` by consensus
   ([1](https://explorer-rally-testnet.genlayer.com/tx/0x3f9f1964557b0b1e3bb43ae3891711ff719a23c84152ee855771a9596e0a5469),
   [2](https://explorer-rally-testnet.genlayer.com/tx/0x31a002eec141cbfa5cf957517c94f25368533c147cebf676c4c7e3224ac46a95),
   [3](https://explorer-rally-testnet.genlayer.com/tx/0x3e7e28187a187f4b3b009b22966c70e27b0c15d0e02087de4744ccf8af3960db)).
   `get_case` then showed `failed_checks: 3`, escrow untouched, case still `OPEN`.
3. `recover` ([tx](https://explorer-rally-testnet.genlayer.com/tx/0x10acb37ac98bdc36b919b4051676b7ebb9169f067b824dee9a01835f769ef755))
   returned the full 1 GEN to the claimant
   ([transfer](https://explorer-rally-testnet.genlayer.com/tx/0xa4bbae2585473e43b1746ada91be21f66b47db36cac82974b016ad09aa7e2920));
   the contract balance is 0.
4. A second `recover` was rejected with `case already recovered`
   ([tx](https://explorer-rally-testnet.genlayer.com/tx/0x8e3faaca15765dd8dcbc50c21dece02e36d3e7698d65746bf35850409b407aae)).

An earlier deployment of the evidence-based settlement path (without recovery) also settled a case
from real evidence (`FULFILLED`, full escrow to the respondent); the `resolve` logic is the same
(now sharing a `judge` helper) and is covered by the direct-mode tests.

## Status

Written against the current GenLayer docs. Verified locally: `genvm-lint check` passes
(lint and validation), all 66 direct-mode tests pass, and the contract is deployed and exercised on Rally TestNet (see above).
