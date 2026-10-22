# Pre-Sign Policy Hook: Decision States and Fail-Closed Signing - 2026-10-21

Status: Research note  
Scope: policy-constrained signing on the agent side of an x402 payment  
Normative status: non-normative; does not rewrite or approve any RFC

## Purpose

The authority note lists "policy-constrained signing" as its own interaction model, between human signing and autonomous signing. This note documents one implementation of it: a hook that an x402 client asks before it signs a Stellar payment authorization, and the states it can end in.

It is relevant here for two reasons. First, the hook's outcomes are a concrete answer to the recovery-semantics question "which failures are retryable, terminal, ambiguous, or dependency-related?". Second, it shows where an agent-side check stops, which is the point where escrow or account-level enforcement has to take over.

## Evidence Classification

| Kind | What | Where |
| --- | --- | --- |
| Implementation | Hook source, 463 lines | [`packages/sdk/src/x402-policy.ts`](https://github.com/nirium-protocol/nirium/blob/1e22d6c552529a4b77ed933c6575151b088ba5f4/packages/sdk/src/x402-policy.ts) |
| Tests | 57 unit tests + 8 fetch-path tests | [`x402-policy.test.ts`](https://github.com/nirium-protocol/nirium/blob/1e22d6c552529a4b77ed933c6575151b088ba5f4/packages/sdk/test/x402-policy.test.ts), [`x402-policy-fetch.test.ts`](https://github.com/nirium-protocol/nirium/blob/1e22d6c552529a4b77ed933c6575151b088ba5f4/packages/sdk/test/x402-policy-fetch.test.ts) |
| Documentation | Behavior and limits as published | [SDK README, "Pre-sign policy hook (experimental)"](https://github.com/nirium-protocol/nirium/blob/1e22d6c552529a4b77ed933c6575151b088ba5f4/packages/sdk/README.md#pre-sign-policy-hook-experimental) |
| Design discussion | The agent-side cases the hook was built against, and its known limits | [nirium-protocol/nirium#96](https://github.com/nirium-protocol/nirium/issues/96) |
| Production | None claimed | The hook is experimental since `nirium` 0.16.0 (current 0.16.1). This note does not claim production use |

Links are pinned to commit `1e22d6c5` so line references stay valid.

## What the Hook Sees

The policy is asked after the Stellar authorization entry is built and before it is signed. It receives the authorization decoded from the bytes that would be signed: amount, destination, asset, nonce, expiration ledger and network, plus a `contextHash` that identifies this exact authorization.

Before the policy is even called, the client refuses anything that is not a single `transfer(from, to, amount)` matching the selected payment requirements: same asset contract, same destination, same amount, same network, no sub-invocations, and `from` equal to the signer. These refusals use the outcome `CONTEXT_MISMATCH`. The point is that the policy reasons about what will actually be signed, not about what the server's 402 response claimed.

## Decision States

The policy answers `ALLOW`, `DENY` or `WAIT`. Only one path reaches the signer: an `ALLOW` that echoes this authorization's `contextHash` and is still valid. Every other result signs nothing.

The full outcome set, from the source (`X402PolicyOutcome`):

| Outcome | Meaning | Recovery reading |
| --- | --- | --- |
| `ALLOWED` | The policy's `ALLOW` was accepted. A notice, not a final result | Not terminal: final checks still follow |
| `SIGNED` | The signer returned a signature | Terminal success for the agent side |
| `SIGNER_ERROR` | The signer rejected, threw, or returned no signed entry | Signer dependency failure |
| `DENY` | The policy refused | Terminal unless the policy changes |
| `WAIT` | The policy wants a later decision (for example, a human approval) | Retryable later, nothing signed now |
| `TIMEOUT` | No answer before the deadline (default 5 s, at most half the payment's `maxTimeoutSeconds`) | Dependency failure, retry safe |
| `ENGINE_ERROR` | The policy threw | Dependency failure |
| `MALFORMED` | The answer did not have the expected shape | Policy bug, terminal |
| `UNBOUND` | An `ALLOW` that did not echo this `contextHash` | Terminal for this authorization |
| `EXPIRED` | An `ALLOW` past its `expiresAt`, or with no `expiresAt` when `requireExpiry` is set | Retry with a fresh decision |
| `STALE` | The `ALLOW`'s `policyVersion` no longer matches the version in force | Retry with a fresh decision |
| `VERSION_UNAVAILABLE` | The version check could not be read | Dependency failure, fail closed |
| `SIGNER_CHANGED` | The signer's address changed while the policy was being asked | Terminal for this authorization |
| `CLOCK_INVALID` | The clock did not return a finite epoch time | Environment failure, fail closed |
| `CONTEXT_MISMATCH` | Authorization did not match the payment requirements (see above) | Terminal |

The "recovery reading" column is this note's interpretation, not a field the SDK returns.

## Three Design Choices Worth Comparing

1. **The approval is bound to the exact bytes.** Echoing `contextHash` ties an `ALLOW` to one authorization. An approval cannot be replayed onto a different amount, destination or nonce.
2. **Approvals expire and go stale.** `expiresAt` limits how long an `ALLOW` is usable, and with `currentVersion` set, the policy version is read again right before signing. A changed policy refuses an older `ALLOW` with `STALE`.
3. **Receipts say what happened, in order.** The `onDecision` observer receives `ALLOWED` before the final checks and `SIGNED` only after a signature exists. `ALLOWED` does not mean "paid" or even "signed". The observer cannot change the outcome and cannot delay the payment.

## What This Does Not Establish

These limits are stated in the SDK README and in #96. Neither is solved:

- **L02, same-process signer access.** This is a check on the agent side. Code in the same process that holds the raw signer can sign without asking the policy. Nothing on-chain enforces it.
- **L01, pending-capacity reservation.** Two concurrent requests can each fit a limit that together they exceed. An aggregate cap has to be held by the policy itself.
- **The version check is not atomic with signing.** It narrows the stale-approval window to one synchronous step; it does not close it.
- **The hook does not verify the signature itself.** That check comes from `@stellar/stellar-sdk` (`authorizeEntry`), and a test in the package pins it.

## Relation to Existing Artifacts

| Artifact | How this note relates |
| --- | --- |
| [Authority, intent, and delegation](authority-intent-and-delegation-2026-09-21.md) | A concrete case of "policy-constrained signing". Matches the principle "make unauthorized economic actions inexpressible": `CONTEXT_MISMATCH` refuses anything but the one expected transfer |
| [Agent-facing interface recovery semantics](agent-facing-interface-recovery-semantics-2026-09-21.md) | The outcome table is one real vocabulary for retryable, terminal and dependency failures, at the signing step |
| [RFC 0002](../rfcs/0002-agent-authority-levels.md) | Sits between Level 1 (human signs) and full autonomy: the agent signs, but only after a policy that can say `WAIT` |
| [Nirium direct x402 payment](../use-cases/nirium-direct-x402-payment.md) | The same payment flow, with a policy step before the signature |

## Open Questions

- For escrow-backed flows, should a release signer expose the same split between `ALLOWED` (accepted) and `SIGNED` (signature exists)? Agents that read only "approved" can treat a not-yet-signed action as done.
- Is `WAIT` the right primitive for human-in-the-loop approval, or should escrow model it as its own state with an owner and a deadline?
- L01 and L02 are exactly the places where an agent-side check is not enough. Is account-level or contract-level enforcement (for example, a spending cap held by the escrow) the right answer for those cases?

## Supports

RFC 0002, RFC 0004, the authority note and the recovery-semantics note.
