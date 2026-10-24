# Issue #36 — ntfy RequirementsAnalysis Revision Report

**GRACE ID:** `RA36-001`  
**Scope:** `bogachenko/bfe-chat-service` only  
**Status:** `PROPOSED / PENDING_HUMAN_APPROVAL`  
**Effective baseline:** `ISSUE32`  
**Implementation:** `BLOCKED`

## Decision and boundary

Issue #36 is titled **Replace Centrifuge with ntfy**. The Project Owner's direction is to replace Centrifuge, not add ntfy alongside it. This report covers only the Backend requirements and architecture artifacts in this repository. No other repository is modified by this work.

The approved ISSUE32 artifacts remain effective until the Project Owner grants Human Approval for RA36-001. No implementation, dependency change, database operation, production configuration change, merge to the effective baseline, or deployment is authorized by this proposal.

## Artifacts in this proposal

| Artifact | Proposed blob |
|---|---|
| `RequirementsAnalysis.xml` | `5afefcbffd1fbeab2a6a9bb085e2cece2d20e6d2` |
| `docs/architecture/c4/workspace.dsl` | `f7f7875af8e8a23ecec849b9ad5c252b7e57b593` |

The requirements proposal defines `REQ36-RT-01` through `REQ36-RT-10`, acceptance criteria, invariants, and the realtime business process. The 30-second service-side revocation bound is proposed and requires Human Approval; this service-only revision does not impose a client-side periodic reconciliation cadence.

## Corrections against Backend-relevant Independent QA findings

| QA finding | Disposition in this revision |
|---|---|
| `NTFY-QA-001` — Flutter baseline provenance | Out of scope for this Backend-only change. No Flutter artifact, branch, PR, or Issue is changed or treated as a dependency of this Backend revision. |
| `NTFY-QA-002` — session-bound authorization | `REQ36-RT-02` preserves the existing BFE Drive Internal Auth/DPoP UDS validation and DPoP-only Chat HTTP contract. Any ntfy-specific access capability is separate from HTTP/history authorization and cannot replace or bypass the BFE Drive session. The concrete ntfy access design is deferred to Technology and must prove compliance without changing the existing Drive contract. |
| `NTFY-QA-003` — active subscriber revocation | `REQ36-RT-03` proposes a measurable maximum 30-second revocation bound and specifies the revocation triggers and old-session cleanup. The value requires Human Approval. |
| `NTFY-QA-004` — silent event loss | `REQ36-RT-04` defines a versioned opaque wake token and treats notifications as best-effort invalidation hints. Durable state and ordering remain authoritative in the backend; catch-up remains the existing authorized history API. Consumer-side scheduling and reconciliation cadence are outside this service-only revision. |
| `NTFY-QA-005` — post-commit publish failure | `REQ36-RT-06` defines best-effort publication after durable commit, records the attempt outcome, forbids blind retry of ambiguous/UNKNOWN outcomes, and preserves recovery through the existing authorized history API. |
| `NTFY-QA-006` — administrative UI in subscriber set | C1 separates Flutter chat clients from the administrative UI. Only chat clients subscribe to ntfy; the administrative UI has no ntfy relationship. |
| `NTFY-QA-007` — conflicting verification evidence | This report is the current verification summary for this proposal branch. It does not reuse old parser runs as proof for new source blobs. The earlier draft report remains historical evidence on its original draft branch and is not the current report for PR #38. |
| `NTFY-QA-008` — distinct human QA identity | Not resolved by Engineering Orchestrator edits. A distinct human reviewer must acknowledge the final Backend revision before the formal Independent QA gate is considered satisfied. |

## Security and recovery contract

- PostgreSQL-backed backend state and authenticated history remain authoritative.
- Sender is excluded from recipients.
- Each recipient/membership epoch has an opaque topic; client-supplied topic selection does not grant authority.
- The application-supplied ntfy body contains only `version` and opaque `wakeToken`; no message/document content or identifiers are included.
- The administrative UI is not an ntfy subscriber.
- Session and membership revocation must terminate active subscriptions within the proposed bound.
- Lost, duplicate, delayed, and reordered signals cannot change durable state; recovery remains available through the existing authorized history API. Consumer-side reconciliation scheduling is outside this service-only revision.
- Document ID/title may remain in authorized backend history, but not in ntfy payload or metadata; Drive authorization remains independent.

## Verification status

The latest requirements/C1 parser run passed on PR source commit `99f612f7cfc1f992820f9acc3236a66852579c41`:

- [GitHub Actions — Requirements and C1 validation, run 40120905177](https://github.com/bogachenko/bfe-chat-service/actions/runs/40120905177): `success`.
- [GitHub Actions — Quality Gates, run 40120905244](https://github.com/bogachenko/bfe-chat-service/actions/runs/40120905244): `success`.
- XML parser: `PASS`; 76 IDs, all 76 unique; 33 local references resolved; one historical external approval reference retained.
- Structurizr C1 parser: `PASS`, exit code 0.
- Exact checked blobs: `RequirementsAnalysis.xml` = `5afefcbffd1fbeab2a6a9bb085e2cece2d20e6d2`; `workspace.dsl` = `f7f7875af8e8a23ecec849b9ad5c252b7e57b593`.
- Validation workflow blob: `4af6d18a48d422724fe4f92316343eda9501a340`.

These successful checks apply to the exact requirements and C1 blobs above. Parser success establishes syntax, ID uniqueness, local-reference integrity, and C1 validity only; it does not prove that the proposed revocation bound is implementable or approved. Independent human acknowledgement and Human Approval remain pending. This report-only update does not alter the validated requirements or C1 source blobs. Parser success establishes syntax, ID uniqueness, and local-reference integrity only; it does not prove that the proposed security/recovery contract is implementable or accepted. Independent human acknowledgement and Human Approval remain pending.

Official ntfy documentation relevant to later Technology analysis:
- [Subscribe API and stream/replay semantics](https://docs.ntfy.sh/subscribe/api/)
- [Publish and message-cache behavior](https://docs.ntfy.sh/publish/)
- [Server configuration and access control](https://docs.ntfy.sh/config/)

## Remaining gates

1. XML and Structurizr parsers must pass on the exact proposal blobs.
2. PR #38 must remain unmerged pending Human Approval.
3. A distinct human reviewer must acknowledge the final Backend revision.
4. Project Owner must approve or revise the proposed 30-second service-side revocation bound.
5. Only after Human Approval may Technology be revised; dependent GRACE artifacts must then be checked and approved before implementation.
