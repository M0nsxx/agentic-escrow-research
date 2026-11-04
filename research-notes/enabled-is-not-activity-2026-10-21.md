# "Enabled" Is Not Activity: Liveness Evidence for an Autonomous Signer - 2026-10-21

Status: Research note  
Scope: how an autonomous signer reports that it is allowed to act, that it is running, and when it last acted  
Normative status: non-normative; does not rewrite or approve any RFC

## Purpose

The [treasury missing-parameter use case](../use-cases/nirium-treasury-missing-parameter.md) showed an automated signer whose interface cannot express a withdrawal. This note follows the same signer two months later and records a reporting failure, not a security one: a status field said `enabled` while the chain showed no activity since August, and from the outside the two readings looked like a contradiction.

The lesson is general for agent-facing interfaces. "This actor is authorized", "this actor is running" and "this actor did something, and here is when" are three different facts. Reporting only the first invites readers, human or agent, to infer the other two.

## Evidence Classification

| Kind | What | Status |
| --- | --- | --- |
| Mainnet, on-chain | Four `rebalance` calls by the RebalanceManager account on 2026-08-06 | Public, re-checked on Horizon 2026-10-21 |
| Mainnet, live endpoint | Vault state and signer heartbeat | Public, read 2026-10-21 |
| Implementation | The reporting fix | Private repository; described by behavior, private commit `b72af7f2` (2026-10-21) |
| Funds | The vault is owned by the team's own account | Own funds, not a third party's |

## What Was Observed

The vault [`CAMDXG6L…57MH`](https://stellar.expert/explorer/public/contract/CAMDXG6L4LXLXXV675KZSHM3BMSETZ4NVMC7JYIQCZ2JTG54OMSK57MH) names a separate signing account as its RebalanceManager, [`GCEAC7KK…7MNI`](https://stellar.expert/explorer/public/account/GCEAC7KKHLSPNHEO3N662N57ABRQZQ7QDVIZWQSQAUJZDMHU6ZEH7MNI). Manager, EmergencyManager and FeeReceiver are the owner's account. That account's only contract calls are four successful `rebalance` invocations on 2026-08-06:

| UTC | Transaction |
| --- | --- |
| 01:54:25 | [`b7bf6d70…40a9`](https://stellar.expert/explorer/public/tx/b7bf6d7079aa128081ac2a5091cbff2da2b1e9f2157232209cf03eab00f840a9) |
| 02:02:59 | [`9f0e7e03…f033`](https://stellar.expert/explorer/public/tx/9f0e7e03cf3f17655b7e1b755c6a3fb5e65c499835424e440e601ecc8b68f033) |
| 02:21:07 | [`d8bec2b4…30b1`](https://stellar.expert/explorer/public/tx/d8bec2b44a0b053d45f843aed21f87e217a2fe87d2761c1e2d3b04318f6330b1) |
| 02:29:05 | [`ccae5347…7f45`](https://stellar.expert/explorer/public/tx/ccae5347739e1f65d319f4ce4c2b586661365ffc919e572a758fd2e11a9b7f45) |

Since then the account has only received unsolicited dust payments.

The public vault endpoint returned `"autonomousRebalancing": "enabled"`. In the code, `enabled` meant one thing: the vault is on the signer's allowlist. It did not mean the signer was running, and it said nothing about when it last moved funds.

## Why Nothing Moved

The signer was running the whole time. It writes a heartbeat every 10 minutes, readable at a public endpoint. On 2026-10-21 it read:

```json
{"ok":true,"lastBeatAgeSec":156,"staleAfterSec":1800,
 "lastMessage":"[rebalancer] vuelta ok · tasa declarada 5.57 · 1 bóveda(s) · 0 movimiento(s)"}
```

("round ok · declared rate 5.57 · 1 vault · 0 moves".)

The vault had 0 idle and `9999997` invested in its single strategy, and the declared rate (5.57%) was above the vault's exit threshold (2.0%). The decision rule had nothing to do: no idle balance to invest, and no reason to unwind. Two months without a transaction was the correct behavior. The thresholds are written per vault by its owner, so the signer executes an instruction rather than deciding on its own when to move.

## The Fix

The status flag stays, so existing clients do not break, and two pieces of evidence sit next to it:

```json
"autonomousRebalancing": "enabled",
"rebalanceEvidence": {
  "lastRebalance": null,
  "signer": { "lastBeatAgeSec": 22, "alive": true }
}
```

- `signer` is the heartbeat age. `alive` turns false after 30 minutes of silence.
- `lastRebalance` is the last move the signer **confirmed on-chain**. It is written only after the transaction is confirmed, never on submission.

The `null` above is real output from the live endpoint on 2026-10-21, and it shows the remaining trap: the recorder only exists from 2026-10-21, so the four August moves are not in it. `null` means "none recorded", not "never moved". The chain is still the source of truth; a one-time backfill from Horizon is prepared but had not been applied when this note was written.

## Research Lessons

1. **Separate permission, liveness and last action.** "Allowed to act", "running" and "acted, at this time" need separate fields. An agent reading one boolean will guess the others.
2. **Silence is ambiguous without a heartbeat.** Zero transactions looks the same whether the signer is dead or correctly idle. The heartbeat turns that silence into a measurement.
3. **Record actions only after confirmation.** A submitted but unconfirmed transaction is not a move.
4. **Recorded history has a start date.** A field that is filled from some date on must say so, or `null` gets read as "never".

## Relation to Existing Artifacts

| Artifact | How this note relates |
| --- | --- |
| [Treasury missing-parameter use case](../use-cases/nirium-treasury-missing-parameter.md) | Same vault and signer; this note adds what happened afterwards |
| [Agent-facing interface recovery semantics](agent-facing-interface-recovery-semantics-2026-09-21.md) | Evidence for `dependencyState`: a signer's liveness is a dependency an unattended client needs to see |
| [Authority, intent, and delegation](authority-intent-and-delegation-2026-09-21.md) | The allowlist is the "allowlisted callable surface" pattern; this note shows that being allowlisted says nothing about activity |
| [RFC 0004](../rfcs/0004-agent-facing-tool-schema.md) | Argues for exposing authorization, liveness and last confirmed action as distinct fields |

## What This Does Not Establish

- It is one vault, owned by the team, with a single strategy. It is not evidence about third-party vaults.
- The heartbeat proves the process runs and completes a round. It does not prove the decision rule is correct.
- The fix is in a private repository; this note describes its behavior and shows its live output.

## Open Questions

- For escrow, is there an equivalent of this split for an automated approver or release signer: "is a signer configured", "is it running", "when did it last act"?
- Should a heartbeat be signed, so a reader can tell it came from the signer and not from the service that reports it?

## Supports

RFC 0004, the recovery-semantics note and the authority note.
