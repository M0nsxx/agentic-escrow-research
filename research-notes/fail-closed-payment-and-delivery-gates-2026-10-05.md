# Fail-Closed Payment and Delivery Gates - 2026-10-05

Status: Research note  
Scope: two fixed failure modes in one production agent-payment service (Nirium)  
Normative status: non-normative; informs RFC 0002 and RFC 0004, does not propose an RFC change  
Contributor: Nirium (Eras256)

## Purpose

The merged use cases so far describe how bounded authority is supposed to work. This note records two places where it did not: a payment gate and a delivery step that each defaulted to "allow" when something inside them failed. Both were fixed. They are documented here because the failure direction is easy to miss in review. Nothing was over-permissive by design. The service fell open on its own internal errors.

## Evidence Classification

| Case | Environment | Public verifiability | Status |
| --- | --- | --- | --- |
| A. Payment gate fell open when the payment client was not initialized | Production service. The x402 half gated a mainnet route; the MPP half gated a route that, as later found, never settled a mainnet payment | Fix commits are in a private repo, cited by hash for provenance only. The mainnet settlement on the affected x402 route is public | Fixed 2026-08-28 |
| B. Delivery step returned success when on-chain submission failed | Testnet only, no real funds at risk | Public issue, public fix commit, reporter credited | Fixed 2026-09-01 |

## Case A: The Gate Side

**What failed.** Two Express middlewares in the payment service, one for x402 and one for MPP, called `next()` when their payment client was `null`. A `null` client happened whenever a required environment value was missing or the client threw during initialization. The paid route then answered normally, for free. The only signal was a startup log line. For the caller, a paid route that fell open looked identical to a route that was never priced.

**How it got there.** The x402 pass-through was added on 2026-04-03 as a graceful fallback for local development, three months before the service had any mainnet route. It was a reasonable choice for a testnet-only codebase. It stayed in place after the route started settling real mainnet USDC (first confirmed settlement: [`3134a51c…58bc`](https://stellar.expert/explorer/public/tx/3134a51c66091fd7fbd85b38a4a6ec6cd432bb92c2450eac84ea7855cb7558bc), 2026-07-09, ledger 63403784).

**A compounding cause.** On the MPP side, a configuration value was passed as raw hex where the library expects a Stellar address or a keypair. The library threw at construction time. A surrounding `try/catch` turned that exception into a generic "init failed" log line, and the `null` client then hit the fail-open path above. Two independently survivable decisions, a swallowed init error and a permissive fallback, combined into free access.

**How it was found.** An automated review comment on a public documentation PR that described this service's production patterns ([stellar/stellar-dev-skill#97](https://github.com/stellar/stellar-dev-skill/pull/97), opened 2026-08-15). The team checked the claim against the real code before changing anything.

**Fix.** Both middlewares now return `503` when the payment client is not ready, instead of falling through. The hex value is converted to an address before use, and the recipient address is format-validated. Private commits `c54c707d` (MPP) and `e88b2360` (x402), both 2026-08-28.

**Version context.** After this fix, a separate review on 2026-10-01 found that the MPP path had never settled a payment on mainnet. That means the MPP half of this bug had no realized billing impact. The x402 half gated a route with real mainnet settlements.

## Case B: The Delivery Side

**What failed.** In a marketplace route that sells premium agent skills, the `catch` block for a failed `submitTransaction` computed the local transaction hash, which never reached the chain, and still returned `success: true` with the premium payload. Any syntactically valid XDR that failed on submission received the paid content. A second bug in the same route charged a hardcoded `0.01` instead of the computed `0.05`.

**How it was found.** Responsible disclosure by an external researcher, `chenshj73`, in a public issue ([Eras256/Nirium#1](https://github.com/Eras256/Nirium/issues/1), 2026-08-28). The full exploit mechanism was public in that issue for about three and a half days before the fix.

**Fix.** A failed submission is now a failed payment. Destination, asset and exact amount are verified before the transaction is submitted and before anything is delivered. The pricing mismatch is fixed, and a best-effort replay check was added. Public commit [`2660a365`](https://github.com/Eras256/Nirium/commit/2660a365), 2026-09-01. The reporter is credited in the issue.

## Research Lessons

1. **Fail-open on internal error is a distinct authority failure.** Most authority review asks whether a signer can do too much. These two bugs were about what happens when the gate itself cannot run. A payment gate that cannot evaluate its condition should deny. For an agent caller it should also return a machine-readable reason, so the agent does not read silence as success.
2. **Verifying payment and gating delivery are separate steps.** Case B's payment check was never wrong. The delivery step acted without confirming its own precondition. Each step needs its own fail-closed rule.
3. **Development fallbacks outlive their context.** The Case A pass-through was correct when it was written. It became a bypass once the route carried real value. Fallbacks added before mainnet are worth re-reviewing when a route starts settling real funds.
4. **Swallowed exceptions turn configuration errors into authority errors.** Case A's MPP chain shows a `try/catch` that converted a loud misconfiguration into a silent permissive state.

## Relation to Existing Artifacts

- [RFC 0002 - agent authority levels](../rfcs/0002-agent-authority-levels.md): authority levels assume the gate evaluates. This note covers what happens when it cannot.
- [RFC 0004 - agent-facing tool schema](../rfcs/0004-agent-facing-tool-schema.md) and [agent-facing interface recovery semantics](agent-facing-interface-recovery-semantics-2026-09-21.md): a `503` with a stated reason is a recovery signal an agent can act on. A silent pass-through is not.
- [Verification and evidence envelope](verification-and-evidence-envelope-2026-09-21.md): Case B is a delivery step that trusted a locally computed hash instead of a confirmed one.

## Open Questions

- Should "fails closed on internal error" become a standard review item for agent-facing payment surfaces in this repository's contribution guidance, next to allowlist and authority review?
- For an escrow release step, what is the equivalent of Case B's mistake? A plausible candidate is releasing on a submission that was built but not confirmed. This is not verified against any escrow implementation here.

## Limits of This Note

- The Case A fix commits live in a private repository. They are cited by hash for provenance and cannot be checked by readers.
- Both cases come from a single service. They are evidence that these failure modes occur, not evidence of how common they are.
