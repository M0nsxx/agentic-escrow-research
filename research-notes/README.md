# Research Notes

Research notes preserve implementation evidence, operational observations, comparisons, and maintainer interpretation that inform the RFCs without becoming normative proposals themselves.

A research note may:

- document production or testnet evidence;
- compare V1 and V2 behavior;
- record failure modes and recovery paths;
- map external implementations to an existing RFC;
- preserve contributor evidence while adding maintainer flags or version context.

Research notes do **not** establish a production commitment, approved API, smart-contract change, or accepted RFC by themselves.

## Current Research Notes

| Note | Supports | Interpretation |
| --- | --- | --- |
| [Production evidence for an agent-facing tool schema](../rfcs/0005-agent-facing-tool-schema-production-evidence.md) | [RFC 0004](../rfcs/0004-agent-facing-tool-schema.md) | V1 production evidence mapped against V2 beta/testnet architecture, plus Nirium and historical Trustless Work MCP implementation evidence |
| [Merged community evidence map - 2026-09-21](merged-community-evidence-map-2026-09-21.md) | RFC 0002, RFC 0003, RFC 0004, use-case library | Maps the first four merged community contributions to the repository artifacts and research questions they now inform |
| [Conditional settlement taxonomy - 2026-09-21](conditional-settlement-taxonomy-2026-09-21.md) | RFC 0003, whitepaper, use-case library | Separates usage escrow, authorization/capture escrow, and outcome/agreement escrow before any RFC rewrite |
| [Payment rail and auth-capture primitives - 2026-09-21](payment-rail-and-auth-capture-primitives-2026-09-21.md) | RFC 0003, RFC 0004 | Reference scaffold for x402, MPP, CPP, payment channels, Stellar agentic payments, and AgentCore payment execution |
| [Outcome agreement protocols - 2026-09-21](outcome-agreement-protocols-2026-09-21.md) | RFC 0001, RFC 0003, RFC 0004 | Reference scaffold for ERC-8183, VCAP, ASP, Sinetti/FNA, ENACT, and agreement-level settlement comparisons |
| [Authority, intent, and delegation - 2026-09-21](authority-intent-and-delegation-2026-09-21.md) | RFC 0002, RFC 0004 | Reference scaffold for Verifiable Intent, bounded authority, missing-parameter authority, signer modes, and capability limits |
| [Verification and evidence envelope - 2026-09-21](verification-and-evidence-envelope-2026-09-21.md) | RFC 0001, RFC 0004 | Reference scaffold for evidence, verifier identity, attestation, and release reasoning |
| [Agent-facing interface recovery semantics - 2026-09-21](agent-facing-interface-recovery-semantics-2026-09-21.md) | RFC 0004 | Reference scaffold for transaction state, read-model state, retry safety, dependency state, and next valid actions |
| [Open Stellar agentic payment evidence - 2026-09-21](open-stellar-agentic-payment-evidence-2026-09-21.md) | RFC 0002, RFC 0003, RFC 0004, verification note, use-case library | Adjacent implementation evidence for Stellar x402 skill payments, ZK spend-cap authorization, direct-payment vs escrow boundaries, and evaluator-assisted release reasoning |
| [Fail-closed payment and delivery gates - 2026-10-05](fail-closed-payment-and-delivery-gates-2026-10-05.md) | RFC 0002, RFC 0004, recovery semantics note | Two fixed production failure modes where a payment gate and a delivery step defaulted to allow on internal error |

### Provenance note

The first research note retains its original `rfcs/0005-...` path and title because it arrived as a community contribution under that name and the maintainer review intentionally preserved rather than deleted contributor material.

For repository governance, it is classified here as a **Research Note supporting RFC 0004**, not as a normative RFC 0005.
