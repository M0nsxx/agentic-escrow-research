# Evidence-gated milestone release: a testnet pilot - 2026-10-14

Status: Research reference (contributor evidence)
Scope: checking evidence before approving and releasing a Multi-Release milestone
Normative status: non-normative; does not rewrite or approve any RFC
Environment: Stellar testnet only. A self-test with the contributor's own wallets; no third-party funds.

## Purpose

The Verification and evidence envelope note asks: "Why should the money release?" This note answers it with one concrete instance. An automated approver checks anchored evidence before it approves and releases a Multi-Release milestone, and refuses when the check fails. It maps the pilot to that note's `VerificationEnvelope` sketch and records what the pilot does not establish.

## What was built

One Multi-Release escrow with three milestones of 1.0 USDC each.

- A human wallet deployed and funded the escrow, so it fixed who gets paid. The same wallet holds the platformAddress, disputeResolver and serviceProvider roles.
- An automated agent key holds the approver and releaseSigner roles. It can approve and release milestones that already exist. It cannot create milestones or change who is paid (the bounded signer pattern from the existing use case).
- Before the agent approves, a decision step runs:
  1. fetch the anchored document by CID from IPFS (if it cannot be fetched: DENY);
  2. recompute the SHA-256 of its `record` and compare it with the `content_sha256` stored in the same document;
  3. require that the record declares the same contract and milestone as the request;
  4. final rule: require an `agent` attestation, an ed25519 signature over `nirium-audit-v1:<content_sha256>`, that verifies and whose key equals the escrow's `serviceProvider`;
  5. only then approve and release. Every decision is logged, whether or not it executes.

The evidence is anchored with Nirium's Audit Trail. The decision step's code is not public, so this note describes its behavior, not its code.

The first version of the decision step (checks 1 to 3) ran on-chain. Check 4 was added afterwards.

## Evidence

| Item | Result |
| --- | --- |
| Escrow (testnet, Multi-Release) | `CADMJSOMBSHUW2LMETBUOVH2CSVY3EDISY4V7UMH7TDU2LE5XRCEULAG` |
| Funding | 3.0 USDC moved from the human wallet to the contract |
| Milestone 0 | Approved and released by hand, before the decision step existed. It does not demonstrate the gate. |
| Milestone 1 | Refused by the decision step. No approve or release was ever sent. Still unreleased. |
| Milestone 2 | Approved and released by the decision step (first version) after the check passed |
| Release effect | Receiver credited 0.997 USDC; 0.003 USDC went to Trustless Work's fee address (0.3%) |
| Contract events | Exactly six, one per transaction below; none for milestone 1 |

Transactions:

```
deploy                a0ed118a0a02ad87e09f0a743cc90d32199c931baf01b052cf49393e85719850
fund                  8769aba1b652b01ee2aafc9083fd1a775722690757e560fb43bd49f98e4856cf
milestone 0 approve   9ec5c30b63b2349975f22523db52fc81c4c572893a41526563139f3e20a09a3d
milestone 0 release   8527f36d73102be268f6cc4ef0ab74e63b1e8b156bfe3996f5be6245df1cfbba
milestone 2 approve   e86861557ad2896452a0d9fc5590de341654ddc311d4bbcd5d8763e3f6961e7b
milestone 2 release   429e550d2f05c6dd6316c8caab13605ee1f222d1ce40731d92c80f7f4bc5fc7c
```

Refusals exercised against the deployed service. None produced an on-chain transaction, and milestone 1 stayed untouched:

| Case | Anchored record (CID) | Result |
| --- | --- | --- |
| Hash declared at anchoring does not match the record (made by anchoring with an explicit hash that does not correspond to the record) | `Qmbwr5eYNex7pzvCSWVu8Q7h4bvfLzkukDu7XXtR7fnkYs` | DENY: recomputed hash does not match |
| Consistent hash, no attestation | `QmSghoBQfdXzV2i8cokxZLscNy85dAQR4pK3j8xsS75n7M` | DENY: no `agent` attestation |
| Consistent hash, valid signature from an unrelated key | `QmTDdTD79BZFmzDtqtsNixcqNrLWtMATpWFAkm64dkZkw3` | DENY: the signing key is not the escrow's serviceProvider |

The record for the milestone 2 release is `QmSpgUkiJnCfdALUtHXocKQ8GtGaG4LsxUWEoUDFDruWXN`.

The allow path under the final rule is covered by nine unit tests, including one with a real ed25519 signature by the serviceProvider. It has not yet been shown on-chain.

On-chain data and CIDs are public. The decision step's code is private. All transactions, contract events, records and the escrow state above were re-checked on 2026-10-14 against Horizon testnet, Soroban RPC and IPFS.

## Mapping to the VerificationEnvelope sketch

| Field | In the pilot | Gap |
| --- | --- | --- |
| agreementId | Escrow contract id, declared in the record (`forContract`) | None |
| milestoneId | Milestone index, declared in the record (`forMilestone`) | None |
| verifier | The decision step | Its identity is not part of any signed artifact |
| verifierType | Automated agent | None |
| criteria | Three fixed checks: hash consistency, addressed to this milestone, signed by the serviceProvider | The criteria are code, not data carried with the envelope |
| evidence[] | The anchored record, referenced by CID | The CID was passed by the caller, not read from the escrow |
| proofHash | `content_sha256` | None |
| signature | The serviceProvider's attestation over the hash | The verdict itself is not signed |
| verdict | ALLOW or DENY with a reason | Logged in the operator's database, not anchored |
| timestamp | Anchoring time in the document; decision time in the log | None |

## Observations

1. **The evidence field and who writes it.** The milestone `evidence` string was empty in all three milestones, and still is (read from the contract with `get_escrow` on 2026-10-14). In the Trustless Work OpenAPI, writing it is `POST /escrow/multi-release/change-milestone-status`, which takes `newEvidence`, `newStatus` and the `serviceProvider` address as the acting party. In this escrow the serviceProvider is a human wallet, so the approver agent could not have written it. An evidence-gated approver therefore needs the reference to arrive from the provider first, or out of band as it did here. We did not test whether another address can call that endpoint.
2. **Hash consistency is a weaker check than it sounds.** IPFS content addressing already makes a document at a CID immutable, so the same CID cannot be "altered after anchoring". What the check catches is a record whose declared hash does not match its content, and evidence addressed to a different milestone. The attestation adds who vouches for the evidence. Neither shows that the evidence is true.
3. **Role separation worked as designed.** The human created and funded the escrow, which fixed the receivers; the agent only approved and released; the dispute resolver stayed human.
4. **A shared key can collide.** The agent key is also used by an autonomous testnet loop, so an approve or release could collide on sequence number. It did not in this run. A dedicated key per role would remove the risk.

## What this does not establish

- That the evidence is true or complete. Only integrity, addressing and the provider's signature are checked.
- Anything about mainnet or third-party funds.
- That the reference came from the escrow: the CID was supplied by the caller.
- The allow path of the final rule on-chain.
- A recommended schema, API or contract change.

## Open questions

1. Should the approver read the evidence reference from the milestone's `evidence` field, and who may write it?
2. Should the verdict be a signed, anchored artifact, so a later auditor can see why a release happened?
3. Should the approver and any other automated process use separate keys?

## Supports

[RFC 0001](../rfcs/0001-agentic-escrow-flow.md), [RFC 0002](../rfcs/0002-agent-authority-levels.md), [RFC 0004](../rfcs/0004-agent-facing-tool-schema.md); the [Verification and evidence envelope](verification-and-evidence-envelope-2026-09-21.md) note; the use case [Nirium bounded milestone payouts](../use-cases/nirium-bounded-milestone-payouts.md).
