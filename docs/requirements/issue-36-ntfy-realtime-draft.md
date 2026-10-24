# Issue #36 — ntfy RequirementsAnalysis Revision Report

**GRACE ID:** `RA36-001`  
**Scope:** `bogachenko/bfe-chat-service` only  
**Engineering Orchestrator disposition:** `REWORK REQUIRED BEFORE HUMAN APPROVAL`  
**Effective baseline:** `ISSUE32`  
**Implementation:** `BLOCKED`

## Decision and boundary

The Project Owner's direction is to replace Centrifuge with ntfy as the sole target chat realtime notification transport, not to add ntfy alongside Centrifuge. This report covers only requirements and architecture artifacts in `bogachenko/bfe-chat-service`. No other repository, client application, or external infrastructure is modified.

The approved ISSUE32 artifacts remain effective. This proposal does not authorize implementation, dependency changes, database operations, production configuration changes, merge to the effective baseline, or deployment.

## Artifacts in this proposal

| Artifact | Current blob |
|---|---|
| `RequirementsAnalysis.xml` | `8ae7b8e5ff334ff49dc8cbed98c262a1594bf192` |
| `docs/architecture/c4/workspace.dsl` | `f7f7875af8e8a23ecec849b9ad5c252b7e57b593` |

The proposal defines `REQ36-RT-01` through `REQ36-RT-10`, acceptance criteria, invariants, and the realtime business process. The 30-second active-subscription revocation bound is proposed, not approved. This service-only revision does not impose a client-side periodic reconciliation cadence. The C4 workspace shows ntfy as the proposed service-side publication dependency; it does not claim that external subscriber implementations are part of this change.

## Blocking finding: active ntfy subscription revocation is unproven

The existing service validates BFE Drive sessions through the established Internal Auth UDS + DPoP boundary. Current membership removal explicitly calls Centrifuge `Unsubscribe` in `internal/issue25/http.go`; this is an existing transport-specific revocation mechanism.

The reviewed official ntfy documentation describes long-lived HTTP/SSE/WebSocket subscriptions and per-topic ACL/authentication, but does not establish that changing an ACL or token terminates an already-open subscription stream:

- [ntfy Subscribe API](https://docs.ntfy.sh/subscribe/api/)
- [ntfy access control and configuration](https://docs.ntfy.sh/config/)
- [ntfy publishing and authentication](https://docs.ntfy.sh/publish/)

This absence of documented behavior is not proof that the bound is impossible. It means the current proposal does not yet contain evidence that it is achievable. In particular, the service must not assume it can validate another user's BFE session using the sender's credentials, and it must not change the existing BFE Drive Internal Auth/DPoP contract to solve this.

The updated `AC36-RT-03` therefore requires a reproducible service-side test that keeps a subscriber connection open, revokes the authoritative session or membership epoch, and proves that no protected event is delivered after the approved bound and that reconnect/new subscription is denied. An ACL/token update alone is insufficient unless the test demonstrates that already-active streams stop receiving protected events.

**Gate:** `REWORK_REQUIRED_BEFORE_HUMAN_APPROVAL`. Before requesting Human Approval, document a concrete service-only revocation mechanism compatible with the existing BFE Drive Internal Auth/DPoP contract and reproducible evidence for the bound. If it cannot be demonstrated, revise the proposed requirement with the Project Owner before approval. Do not modify `Technology.xml`, `DevelopmentPlan.xml`, `Contracts.xml`, runtime code, dependencies, or production configuration until the requirements are approved.

## Other requirements retained

- PostgreSQL-backed backend state and authorized history remain authoritative.
- Sender is excluded from recipients.
- The application-supplied ntfy body contains only `version` and opaque `wakeToken`; it contains no message/document content or identifiers.
- Topic access or notification receipt never authorizes Chat HTTP/history requests or grants BFE Drive document rights.
- The existing DPoP-only Chat HTTP contract and Internal Auth UDS integration must remain unchanged.
- Publication is best-effort after durable commit; notification failure cannot undo a committed message.
- Lost, duplicate, delayed, and reordered signals cannot change durable state; recovery remains through the existing authorized history API.
- Consumer-side scheduling, lifecycle, and reconciliation cadence are outside this service-only requirements revision.

## Verification status

Validation passed on proposal source commit `2d62bd18b5b53c84b59be94083b3db8ea4312647`:

- [Requirements and C1 validation, run 40121792957](https://github.com/bogachenko/bfe-chat-service/actions/runs/40121792957): `success`.
- [Quality Gates, run 40121792928](https://github.com/bogachenko/bfe-chat-service/actions/runs/40121792928): `success`.
- XML parser: `PASS`; 76 IDs, all unique; 33 local references resolved; one historical external approval reference retained.
- Structurizr C1 parser: `PASS`, exit code 0.
- Exact checked blobs: `RequirementsAnalysis.xml` = `8ae7b8e5ff334ff49dc8cbed98c262a1594bf192`; `workspace.dsl` = `f7f7875af8e8a23ecec849b9ad5c252b7e57b593`.
- Validation workflow blob: `4af6d18a48d422724fe4f92316343eda9501a340`.

These checks establish XML syntax, ID uniqueness, local-reference integrity, C4 validity, and the configured quality gates. They do not establish that active-stream revocation is feasible or that the proposed requirements have been approved. The report update does not alter the validated requirements or C4 blobs.

## Remaining gates

1. Resolve the active-stream revocation feasibility gap within `bfe-chat-service` scope without changing the existing BFE Drive Internal Auth/DPoP contract.
2. If the proposed bound is feasible, attach reproducible evidence and request Independent QA and Human Approval; otherwise revise the bound with the Project Owner before approval.
3. Keep PR #38 unmerged until the required human gates are satisfied.
4. Only after Human Approval, update and independently verify `Technology.xml`, `DevelopmentPlan.xml`, contracts, mental scenarios, C1, and quality checks in GRACE order.
5. Implementation, dependency changes, production configuration changes, and deployment remain blocked until their respective GRACE gates pass.
