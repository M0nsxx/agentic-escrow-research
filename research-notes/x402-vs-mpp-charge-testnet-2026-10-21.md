# x402 and MPP Charge on One Endpoint: Testnet Settlements Compared - 2026-10-21

Status: Research note  
Scope: two direct-payment rails served by the same agent endpoint on Stellar testnet  
Normative status: non-normative; does not rewrite or approve any RFC

## Purpose

The payment-rail note asks when a direct payment, an MPP session or an auth-capture flow should bypass escrow, and which recovery semantics from payment rails should feed RFC 0004. This note adds settled testnet evidence for two rails on the same paid endpoint: x402 (`exact`, through a facilitator) and MPP Charge (per request, in both pull and push mode, without a facilitator).

The comparison that matters for escrow research is not speed or price. It is **who submits the transaction, and what the client holds if delivery fails after the money moved.**

## Evidence Classification

| Kind | What | Status |
| --- | --- | --- |
| Testnet, on-chain | 5 MPP Charge settlements (2026-10-15) and 2 x402 settlements (2026-10-21) to the same receiving account | Public, re-checked on Horizon testnet 2026-10-21 |
| Testnet, historical | One push-mode payment that settled with no delivery (2026-09-29) | Observed; only the transaction hash prefix was kept |
| Mainnet | x402 settles on mainnet. MPP Charge has **never** settled a payment on mainnet | Stated plainly; MPP mainnet is untested |
| Payers | All payments came from the team's own throwaway testnet keys | Internal tests, not usage |

## Settlements

All payments are USDC (SAC) transfers to the endpoint's testnet receiving account `GC4Q5TWW…UOPC`.

**MPP Charge.** Client `@stellar/mpp` 0.7.1 with `mppx` 0.6.31, throwaway buyer `GB6J4FME…UXSIC`. The first three used the library directly, the last two the published `nirium` 0.16.1 client.

| UTC (2026-10-15) | Mode | Amount | Transaction |
| --- | --- | --- | --- |
| 01:43:02 | pull | 0.02 | [`4d20d864…0236`](https://stellar.expert/explorer/testnet/tx/4d20d8648ce001c4e8e894725ad08a8dbcd3904e0024ca440728bc91e6c10236) |
| 01:43:27 | push | 0.05 | [`e70cfcb8…f167`](https://stellar.expert/explorer/testnet/tx/e70cfcb884c17d946210a000a627b7a9c316737a61a66ad2e42d905df679f167) |
| 01:43:37 | pull | 0.05 | [`be7865f1…ff03`](https://stellar.expert/explorer/testnet/tx/be7865f19b4daf9b60993eb4df900098663a55861a357fd7b67a059cc228ff03) |
| 01:48:22 | pull | 0.02 | [`b8c4ed56…e3ed`](https://stellar.expert/explorer/testnet/tx/b8c4ed568f6eb41321b97d251d7f3727c57570a226d187e3675a9df03496e3ed) |
| 01:48:27 | push | 0.02 | [`96d43d6f…242c`](https://stellar.expert/explorer/testnet/tx/96d43d6f4cf38fbd3a4df2c2bb6625e650c0d0509184d86c1e42c7e6ebdf242c) |

**x402 (`exact`).** Client `nirium` 0.16.1, throwaway payer `GC42S3UC…4X2Y`.

| UTC (2026-10-21) | Amount | Transaction |
| --- | --- | --- |
| 19:31:42 | 0.05 | [`7f406669…0710`](https://stellar.expert/explorer/testnet/tx/7f4066691cdce82b61d1ebf177fd48edad193c9dc220879e72a7db1cd1e50710) |
| 19:32:07 | 0.05 | [`c649c462…7c36`](https://stellar.expert/explorer/testnet/tx/c649c462c4f16f5a2c526d2ffe02dbd0080900a9575b086b51b7fee0f1857c36) |

## Who Submits, Who Pays the Fee

Read directly from the transactions:

| Rail | Transaction source account | What the payer signed |
| --- | --- | --- |
| MPP Charge, pull | The payer (`GB6J4FME…`) | The full transaction; the server broadcast it |
| MPP Charge, push | The payer (`GB6J4FME…`) | The full transaction, which the payer broadcast itself, then sent the hash |
| x402 `exact` | A different account (`GBYJPJ…`), on the facilitator side | Only the authorization for the transfer |

MPP here ran without a fee sponsor, so the payer's account is the source and pays the network fee. In x402 the payer signs an authorization and another account submits the transaction. Per the `@stellar/mpp` documentation, push mode is not compatible with a sponsored fee payer.

## The Failure Mode That Matters for Escrow

On 2026-09-29, before the setup above worked, a **push-mode** payment on the same testnet endpoint was confirmed on-chain and the server still answered with a verification failure: the payer paid and received nothing (transaction prefix `41f12900ac27…`; the full hash was not kept). The root cause was not determined at the time. By 2026-10-15 the same endpoint verified and delivered in both modes, as the table shows.

That is the shape of the problem escrow exists for. In push mode the payer moves the money first and asks for the goods second. If the server's verification fails, the transfer is already final and nothing on-chain links it to an undelivered request. In pull mode, and in x402, the server verifies first and only then submits, so a failed verification leaves the payer uncharged. Delivery can still fail after the server submits, but then the failure sits on the server's side of the line, after it chose to take the money.

For a small, immediate digital purchase, this is the boundary the [direct x402 use case](../use-cases/nirium-direct-x402-payment.md) already describes: direct payment is fine when the amount is small and delivery is immediate. This note adds that **the submission mode changes the failure**: "paid and not delivered" is possible in push mode in a way that it is not when the server submits.

## Research Lessons

1. **Ask who submits, not only who pays.** The same amount, asset and endpoint fail differently depending on whether the client or the server broadcasts.
2. **"Settled" and "delivered" are separate states.** An agent needs both, plus a way to reconcile a settled payment with an undelivered response, which is the "ambiguous settlement" case in the recovery-semantics note.
3. **A rail that works on testnet has not been shown on mainnet.** MPP Charge settles on testnet; on mainnet it has no settled payment. A 402 challenge alone does not prove that a payment can be collected.

## Relation to Existing Artifacts

| Artifact | How this note relates |
| --- | --- |
| [Payment rail and auth-capture primitives](payment-rail-and-auth-capture-primitives-2026-09-21.md) | Settled evidence for the "Machine Payments Protocol" and "x402" rows, plus the submission-mode distinction |
| [Agent-facing interface recovery semantics](agent-facing-interface-recovery-semantics-2026-09-21.md) | The push failure is a concrete "settled but not delivered" case, which needs its own state |
| [Nirium direct x402 payment](../use-cases/nirium-direct-x402-payment.md) | Same endpoint family; this note adds MPP and the push-mode failure |
| [RFC 0003](../rfcs/0003-x402-escrow-composition.md) | Supports deciding when a direct rail is enough and when escrow should hold the funds |

## What This Does Not Establish

- Nothing about mainnet MPP. It is untested there.
- Nothing about volume or real usage: every payment here came from the team's own throwaway keys.
- The September push failure is described from notes kept at the time; only a hash prefix survives, so it cannot be re-checked on-chain from this note.

## Open Questions

- Should an escrow-backed flow refuse push-style direct payments for anything above a small amount, since the payer cannot get a link between money and delivery?
- Is there a common "settled, not delivered" state that direct rails and escrow could both report, so an agent can reconcile it the same way?

## Supports

RFC 0003, RFC 0004, the payment-rail note and the recovery-semantics note.
