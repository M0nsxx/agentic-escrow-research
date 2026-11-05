# Agentic Escrow Research

Open-source research on conditional settlement, bounded authority, and dispute-aware payments for the agentic economy.

## Thesis

AI agents will increasingly discover, negotiate, purchase, hire, and coordinate economic activity. Stablecoins are a strong settlement medium for that world, but direct payments and agent wallets alone do not answer the core trust question:

> Under what conditions should a payment become final?

A wallet spending limit constrains how much an agent can spend. Agentic escrow constrains when, why, and under what verified conditions committed funds can be released.

Trustless Work is exploring escrow as a machine-readable trust layer for humans, agents, merchants, autonomous services, and platforms.

## What This Repository Is

This repository is a public research initiative. It is meant to host:

- a living whitepaper;
- open RFCs (Requests for Comments);
- use-case analysis;
- diagrams and trust models;
- reference experiments;
- research notes and implementation evidence;
- contributor research questions.

It is not a production protocol commitment. Research conclusions should be validated before they are promoted into Trustless Work's core product, API, SDK, MCP, or smart-contract roadmap.

## Why Open Research

The goal is to build a public learning base for agentic settlement: real use cases, implementation evidence, failed assumptions, safety patterns, integration lessons, and protocol comparisons.

By collecting this work in the open, the project can:

- help builders understand when conditional settlement is useful;
- make integration lessons reusable instead of private and scattered;
- attract contributors who are already building agentic commerce, wallets, payment rails, marketplaces, and automation tools;
- identify where Trustless Work should compose with other protocols instead of replacing them;
- learn which features belong above existing contracts, in APIs or SDKs, and which would require deeper architecture review;
- make Trustless Work the natural integration surface for outcome-based agentic transactions.

The repository should therefore preserve both positive evidence and negative evidence. A case where escrow adds no value is as useful as a case where escrow is essential.

## How to Read This Repo

This repository separates evidence, proposals, and product commitments.

| Artifact | Meaning |
| --- | --- |
| Use case | A concrete scenario or implementation example that helps test the research thesis. |
| Research note | Non-normative evidence, operational observations, version comparisons, or source material that may inform later proposals. |
| RFC | Request for Comments: a structured proposal meant for review, critique, and iteration. |
| Accepted RFC | A research output strong enough to guide experiments or implementation thinking, but not automatically a production roadmap item. |
| Production commitment | A separate Trustless Work decision involving product, engineering, security, compliance, and roadmap approval. |

An RFC is therefore not the same thing as an approved product change. It is a disciplined way to expose a proposal to public review before anything becomes canonical.

## Starting Points

| File | Purpose |
| --- | --- |
| [CHARTER.md](CHARTER.md) | Initiative scope, principles, and governance |
| [WHITEPAPER.md](WHITEPAPER.md) | Living whitepaper v0 |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How contributors can participate |
| [rfcs/0001-agentic-escrow-flow.md](rfcs/0001-agentic-escrow-flow.md) | Baseline agentic escrow flow |
| [rfcs/0002-agent-authority-levels.md](rfcs/0002-agent-authority-levels.md) | Human/agent autonomy model |
| [rfcs/0003-x402-escrow-composition.md](rfcs/0003-x402-escrow-composition.md) | x402 and escrow composition |
| [rfcs/0004-agent-facing-tool-schema.md](rfcs/0004-agent-facing-tool-schema.md) | Candidate tool/API schema for agents |
| [research-notes/](research-notes/) | Non-normative implementation evidence and version-aware research notes |
| [use-cases/](use-cases/) | Use-case library and implementation evidence |
| [use-cases/nirium-bounded-milestone-payouts.md](use-cases/nirium-bounded-milestone-payouts.md) | Real-world bounded-authority milestone payout example |
| [use-cases/nirium-treasury-missing-parameter.md](use-cases/nirium-treasury-missing-parameter.md) | Authority-by-missing-parameter treasury example |
| [use-cases/nirium-direct-x402-payment.md](use-cases/nirium-direct-x402-payment.md) | Verified direct-payment case where escrow adds no value |
| [use-cases/open-stellar-skills-marketplace.md](use-cases/open-stellar-skills-marketplace.md) | Agent skills marketplace boundary between direct payment and escrow-backed outcomes |
| [research-notes/open-stellar-agentic-payment-evidence-2026-09-21.md](research-notes/open-stellar-agentic-payment-evidence-2026-09-21.md) | Open Stellar evidence for x402 skill payments, spend caps, receipts, and evaluator-assisted release reasoning |

## Research Tracks

1. Agentic commerce: agents buying goods, services, software, data, or compute.
2. Agent-to-agent work: agents hiring other agents or autonomous service providers.
3. Enterprise procurement: bounded agents coordinating vendor payments, invoices, and approvals.
4. Bounties and grants: agents matching work, evidence, approvals, and milestone payments.
5. Machine-native APIs: x402-style direct payments combined with conditional escrow for larger or higher-risk commitments.
6. Safety and authorization: signing policy, key custody, approval separation, disputes, recovery, and recourse.
7. Bounded control of owned capital: recurring automated actions where capability design constrains what economic outcomes an agent can express.

## Core Questions

- What should an agent be allowed to prepare, create, fund, approve, dispute, and release?
- Which economic actions require human approval in early implementations?
- How should order, invoice, evidence, and escrow state bind together?
- When is a direct machine payment enough?
- When does conditional settlement provide materially better safety?
- What tool schema should make invalid authority obvious before transaction submission?
- What failure paths are required if an agent, merchant, oracle, verifier, or user disappears?
- Which dangerous economic actions can be made structurally inexpressible to an automated actor?
- How should we distinguish exfiltration authority from economic-decision authority?
- How should payment-infrastructure/dependency risk be modeled separately from counterparty/fulfillment risk?
- What recovery semantics do unattended agents need when settlement is ambiguous?

## Current Working Model

The recommended starting architecture keeps the escrow primitive narrow and composes agentic behavior through APIs, SDKs, MCP-style tools, policies, and orchestration.

```text
Human intent
  -> buyer agent negotiation
  -> merchant/agent confirmation
  -> protected escrow commitment
  -> human or policy-controlled funding
  -> fulfillment evidence
  -> approval / dispute / fallback
  -> release or refund
```

Implementation evidence is beginning to refine this model. The [Nirium bounded-authority milestone payout use case](use-cases/nirium-bounded-milestone-payouts.md) demonstrates a concrete pattern in which a human fixes the economic commitment up front while automation is limited to state transitions inside already-created, allowlisted escrows.

> **Human defines the economic boundary; automation operates inside that boundary.**

A second Nirium implementation adds another form of bounded authority: [authority by missing parameter](use-cases/nirium-treasury-missing-parameter.md). In this pattern, the automated actor's callable operation does not expose an arbitrary beneficiary parameter.

> **Make unauthorized economic actions inexpressible, not merely disallowed.**

Together, these examples suggest that safe agentic finance depends both on **pre-authorized objects an agent may act upon** and on **capabilities the agent is never given in the first place**.

A third Nirium contribution documents the boundary where escrow should be bypassed: [a verified direct x402 settlement](use-cases/nirium-direct-x402-payment.md) for a small, immediate digital purchase.

> **Do not introduce conditional settlement when there is no meaningful condition to protect.**

That evidence also separates **counterparty/fulfillment risk** from **payment-infrastructure/dependency risk**. Escrow addresses the former; facilitator health, retry safety, reconciliation and ambiguous settlement require their own controls.

A fourth contribution focuses on the machine interface itself: [production evidence for the agent-facing tool schema](rfcs/0005-agent-facing-tool-schema-production-evidence.md). It is classified as a **Research Note supporting RFC 0004**, not as a new normative RFC.

Its V1-to-V2 comparison reinforces that agent tools must distinguish chain submission, read-model convergence, signer mode, retry safety, caller capability, and next valid action.

> **Version-specific workarounds are evidence; durable agent-facing abstractions should encode the underlying state and authority problem instead.**

A fifth contribution adds adjacent Stellar ecosystem evidence from Open Stellar: a [skills marketplace use case](use-cases/open-stellar-skills-marketplace.md) and [research note](research-notes/open-stellar-agentic-payment-evidence-2026-09-21.md) showing how direct x402-style skill payments, receipts, spend caps, and typed evaluators inform the boundary between paid access and escrow-backed outcomes.

> **Use direct payment for access, usage, attempts, and immediate responses; use escrow when the agent is buying an outcome that needs evidence, review, dispute, or recovery.**

## Relationship to Trustless Work

Trustless Work provides stablecoin escrow infrastructure. This research explores how that infrastructure could support agentic commerce without giving agents unchecked financial authority.

The research should prefer:

- non-custodial user authority;
- explicit role separation;
- deterministic escrow state;
- narrow smart-contract primitives;
- machine-readable workflows;
- bounded agent authorization;
- recoverable human control;
- clear dispute and fallback paths.

## Status

Early public research. Contributions, critiques, diagrams, use cases, and experiments are welcome.
