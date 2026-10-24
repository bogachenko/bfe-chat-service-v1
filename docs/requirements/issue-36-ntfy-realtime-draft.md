# Issue #36 — ntfy RequirementsAnalysis Revision Report

**GRACE ID:** `RA36-001`  
**Scope:** `bogachenko/bfe-chat-service` only  
**Engineering Orchestrator disposition:** `REWORK REQUIRED BEFORE HUMAN APPROVAL`  
**Effective baseline:** `ISSUE32`  
**Implementation:** `BLOCKED`

## Decision and boundary

The target remains replacement of Centrifuge with ntfy as the sole Chat realtime notification transport, not an additive transport. This proposal changes requirements and architecture documentation only in `bfe-chat-service`. It does not authorize implementation, dependency changes, database migrations, production configuration changes, merge to the effective baseline, or deployment. BFE Drive, client applications, other repositories, and external infrastructure are not modified.

The key clarification in this revision is that **administrative Chat disablement is not BFE account/session revocation**. The existing BFE Drive Internal Auth/DPoP contract remains unchanged. No periodic BFE-session validation, new Drive callback, or session-revocation operation is introduced.

## Administrative Chat-disable flow

The required service-side flow is:

`Existing administrator UI → protected Chat Service API → administrative handler → Chat-specific access state in PostgreSQL bfe_chat`

1. An administrator uses the existing external administrative UI to request disablement or re-enablement of Chat access for a target BFE account UUID.
2. The Chat Service API uses the existing protected HTTP authentication path: Internal Auth over UDS plus DPoP, followed by server-side administrative authorization.
3. The handler persists the Chat-specific enabled/disabled state in Chat-owned PostgreSQL `bfe_chat`. The representation (for example, table or column) and exact API route are implementation details for a later approved stage.
4. After the disablement commit, new non-administrative Chat operations, history requests and new ntfy subscriptions are denied.
5. An already-active ntfy subscription is not forcibly terminated and may continue receiving only the minimal opaque wakeToken. Chat disablement alone does not require suppressing publication to an existing subscription or enforcing a delivery deadline. The wakeToken contains no message content or identifying metadata and does not authorize history/API access.
6. Re-enabling Chat changes only Chat-owned access state, subject to current membership. The client can then retrieve messages accumulated during disablement through the existing authorized history API. Re-enabling does not create, restore, or revoke a BFE Drive session.

The API must not report a successful completed disablement if the Chat state was not persisted. Existing Internal Auth/DPoP validation and history authorization remain authoritative. The requirement does not authorize a new external client implementation; this revision specifies the service-side contract and models the existing UI as an external caller.

## Requirements and acceptance

The updated `RequirementsAnalysis.xml` adds `UC36-ADMIN-CHAT-ACCESS`, `REQ36-ADMIN-ACCESS`, `AC36-ADMIN-ACCESS`, `BP36-ADMIN-CHAT-ACCESS`, and `INV36-CHAT-ACCESS-SEPARATION`.

Acceptance must demonstrate all of the following:

- Authorized administrator → API → handler → PostgreSQL state-change trace.
- Unauthorized administrative requests cannot change the target account's Chat state.
- After disablement commits, new Chat operations, history requests, and new subscriptions are denied.
- An already-active subscription may continue receiving only opaque wakeTokens; this does not authorize history/API access or expose message contents.
- After re-enablement, the client can retrieve messages accumulated during disablement through the authorized history API.
- A still-valid BFE session is not revoked or changed by the Chat-disable operation.
- The existing Internal Auth/DPoP contract and per-request session validation remain unchanged; no periodic session-check loop is added.
- No active-stream termination deadline applies to Chat-specific administrative disablement. The separate 30-second limit remains only for membership/epoch revocation under the approved ISSUE32 membership semantics.
- Re-enabling Chat does not create or revoke a BFE Drive session.
- The UI, BFE Drive, and other client applications are not modified by this PR.

The 30-second membership-revocation bound is measured from **t0 = successful PostgreSQL commit** of membership removal or membership-epoch revocation, not administrative Chat disablement. A reproducible test must keep a subscriber connection open, publish synthetic protected wake-up events continuously, record event and connection timestamps, and prove that delivery to the revoked membership epoch stops after **t0 + 30 seconds**. It must also verify that new subscriptions for the revoked membership are denied and that an old membership epoch remains unusable after rejoin. Chat disablement has a separate acceptance test: API/history requests and new subscriptions are denied, while an existing subscription may continue receiving only opaque wakeTokens.

## Active ntfy subscription revocation for membership changes

The 30-second active-stream requirement applies only to membership removal or membership-epoch revocation, preserving the approved ISSUE32 membership semantics. It does not apply to administrative Chat disablement: an existing subscription may continue receiving opaque wakeTokens while API/history access is denied. There is upstream implementation evidence that ntfy can forcibly terminate active subscriptions in a specific version:

- [ntfy v2.29.0 administrative handlers](https://github.com/binwiederhier/ntfy/blob/v2.29.0/server/server_admin.go#L133-L203): user deletion calls `killUserSubscriber` for all topics; access reset also calls the cancellation path for affected topics.
- [ntfy v2.29.0 active-stream test](https://github.com/binwiederhier/ntfy/blob/v2.29.0/server/server_admin_test.go#L428-L460): `TestAccess_AllowReset_KillConnection` verifies that an already-open stream is terminated after access reset.
- [ntfy v2.29.0 account deletion stream test](https://github.com/binwiederhier/ntfy/blob/v2.29.0/server/server_account_test.go#L529-L552): the test checks that deleting an account ends its active stream.

This is evidence of a capability in that upstream version, **not proof that the proposed bfe-chat-service integration meets the 30-second membership-revocation bound**. The target ntfy version, configuration, mapping between Chat memberships and ntfy identities, and error-handling behavior have not been selected and validated as a complete integration. The exact service-side integration test remains required for membership/epoch revocation. No equivalent stream-termination test or mechanism is required for administrative Chat disablement. An ACL/token update alone must not be treated as proof unless an already-open stream is observed to stop receiving protected events after membership revocation.

## Session and membership semantics

- Every new protected Chat HTTP request continues to use the existing Internal Auth UDS + DPoP path.
- Every new subscription continues to be checked against the existing session-validation boundary and current Chat membership/epoch.
- Administrative Chat disablement denies new Chat operations, history requests and new subscriptions but does not require terminating an already-active ntfy stream; opaque wakeTokens may continue to arrive. Membership/epoch revocation separately triggers active-stream termination within the proposed bound.
- Independent BFE-session expiry/revocation continues to be governed by the existing request-time Internal Auth/DPoP behavior. This requirements revision does not add a guarantee that an already-open ntfy stream is terminated solely because a BFE session expires, since that would require a new signal or periodic check not authorized by the current scope.
- Chat membership, Chat access state, and ntfy access never grant BFE Drive document permissions. Document open/download remains independently authorized by BFE Drive.

## C4 alignment

The C4 workspace models the existing administrative UI as an external caller and the proposed service-side path through the Chat API, administrative handler, and Chat-owned PostgreSQL state. It explicitly states that this is a requirements proposal, not an implemented component. The ntfy active-stream revocation edge applies to membership/epoch changes only and is marked as requiring target integration evidence. Chat-specific administrative disablement does not require that edge to be invoked. The existing BFE Drive Internal Auth/DPoP boundary is unchanged.

## Verification status

Validation of the updated requirements/C4 artifacts:

- [Requirements and C1 validation, run 40129413833](https://github.com/bogachenko/bfe-chat-service/actions/runs/40129413833): **success** on commit `aa8d76a39caf90ad72e3da62dc945467d848fcad`.
- XML parser: **PASS**; 82 IDs, all unique; 38 local references resolved; one historical external provenance reference retained.
- Structurizr C1 parser: **PASS**.
- Current exact artifact blobs: `RequirementsAnalysis.xml` = `7eca702eb8486717eaff98578fc9c5035480d13c`; `docs/architecture/c4/workspace.dsl` = `344e0d64ad4e49ff3cc5a9d465012acfa66e191f`.
- [Quality Gates, run 40129261343](https://github.com/bogachenko/bfe-chat-service/actions/runs/40129261343): **success** on the preceding requirements-content commit `ca4803975ec9ccbd28f2448a048ccaa16e56943e`. The later commit `aa8d76a39caf90ad72e3da62dc945467d848fcad` only adds the recorded Project Owner decision URL to the XML provenance; its Quality Gates rerun [40129413890](https://github.com/bogachenko/bfe-chat-service/actions/runs/40129413890) was still running at the time of this report update.
- No runtime ntfy integration test was executed because implementation is out of scope. Such a test is **not required for administrative Chat disablement**; active-stream termination evidence remains required only for membership/epoch revocation under the approved ISSUE32 semantics.

These checks validate XML structure, references, and C4 syntax. They do not prove active-stream termination after membership/epoch revocation. No runtime source, dependency, schema, production configuration, BFE Drive repository, or client application was changed.
## Remaining gates

1. Confirm XML well-formedness, unique IDs, internal reference integrity, and Structurizr C1 validity for the updated artifacts.
2. Keep Human Approval blocked until the target ntfy version/configuration and the bfe-chat-service integration demonstrate active-stream termination within 30 seconds after membership/epoch revocation only. Administrative Chat disablement is not subject to this bound.
3. If the membership-revocation bound cannot be demonstrated, obtain an explicit Project Owner decision before changing it.
4. Keep PR #38 unmerged until Human Approval and distinct Independent QA acknowledgement.
5. Only after requirements approval, update and independently verify dependent GRACE artifacts in order. Implementation, dependency changes, database operations, production configuration, and deployment remain blocked until their respective GRACE gates pass.
