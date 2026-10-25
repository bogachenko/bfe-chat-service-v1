# Issue #4 / backend Issue #36 — ntfy realtime requirements draft

**Status:** DRAFT_NOT_APPROVED. Human Approval: NOT GRANTED. Technology/implementation/deployment: BLOCKED.

## Source revisions
- Backend main: f4d626ddaee4c691ef4c3033062f95e7953e0821.
- Approved RequirementsAnalysis blob: 976c8cd8dfc41a0d938f350ee46db8b0bad4d79d.
- Approved C1 DSL blob: 1db2f96a299a733ec064fba197ced623bd7bcb73.
- Project Owner direction: bfe-chat-flutter Issue #4.
- Decision record: Project Owner comment https://github.com/bogachenko/bfe-chat-service/issues/36#issuecomment-6385730270 directs this GRACE revision to target replacing Centrifuge with ntfy. The original Issue #36 body remains historical context; the approved baseline stays effective until Human Approval.

## Traceability matrix

| Draft requirement | Existing requirement trace | AAG scenario | Business process | Invariant | Acceptance | Mental scenario |
|---|---|---|---|---|---|---|
| REQ36-RT-01: ntfy replaces Centrifuge as target realtime transport | REQ32-MESSAGE, REQ32-HISTORY, REQ32-READ | UC36-REALTIME-RECOVERY | BP36-NTFY-REALTIME | INV36-NTFY-NO-AUTHORITY | AC36-RT-01 | MS36-NTFY-04 |
| REQ36-RT-02: backend-controlled topics and subscription isolation | REQ32-AUTH, REQ32-ISOLATION, REQ32-MEMBERSHIP | UC36-REALTIME-RECOVERY | BP36-NTFY-REALTIME | INV36-NTFY-SESSION | AC36-RT-02 | MS36-NTFY-01 |
| REQ36-RT-03: active revocation and membership intervals | REQ32-MEMBERSHIP, REQ32-AUTH, REQ32-ISOLATION | UC36-REALTIME-RECOVERY | BP36-NTFY-REALTIME | INV36-NTFY-SESSION | AC36-RT-03 | MS36-NTFY-02 |
| REQ36-RT-04: missing/duplicate/reordered signals reconcile from history | REQ32-HISTORY, REQ32-MESSAGE, REQ32-READ | UC36-REALTIME-RECOVERY | BP36-NTFY-REALTIME | INV36-NTFY-NO-AUTHORITY | AC36-RT-04 | MS36-NTFY-04 |
| REQ36-RT-05: minimum privacy-preserving payload | REQ32-DOCUMENT, REQ32-ISOLATION | UC36-REALTIME-RECOVERY | BP36-NTFY-REALTIME | INV36-NTFY-PRIVACY, INV36-NTFY-DRIVE-ACL | AC36-RT-05 | MS36-NTFY-06 |
| REQ36-RT-06: post-commit failure/retry/dedupe | REQ32-MESSAGE, REQ32-API | UC36-REALTIME-RECOVERY | BP36-NTFY-REALTIME | INV36-NTFY-POST-COMMIT | AC36-RT-06 | MS36-NTFY-05 |
| REQ36-RT-07: account/session isolation | REQ32-AUTH, REQ32-ISOLATION, REQ32-HISTORY | UC36-REALTIME-RECOVERY | BP36-NTFY-REALTIME | INV36-NTFY-SESSION | AC36-RT-07 | MS36-NTFY-03 |
| REQ36-RT-08: document authorization remains independent | REQ32-DOCUMENT, REQ32-ISOLATION | UC36-REALTIME-RECOVERY | BP36-NTFY-REALTIME | INV36-NTFY-DRIVE-ACL | AC36-RT-08 | MS36-NTFY-06 |
| REQ36-RT-09: production boundary and release compatibility | REQ32-PRESERVE | UC36-RELEASE-READINESS | BP36-NTFY-RELEASE-READINESS | INV36-NTFY-PRODUCTION-BOUNDARY | AC36-RT-09 | MS36-NTFY-08 |
| REQ36-RT-10: supported Web/Android transport and lifecycle | REQ32-AUTH, REQ32-ISOLATION, REQ32-HISTORY | UC36-REALTIME-RECOVERY | BP36-NTFY-REALTIME | INV36-NTFY-SESSION | AC36-RT-10 | MS36-NTFY-07 |
| REQ36-RT-11: BFE-owned Android system notification is distinct from ntfy app notification; best-effort lifecycle and history recovery | REQ32-AUTH, REQ32-HISTORY, REQ36-RT-05, REQ36-RT-10 | UC36-ANDROID-SYSTEM-NOTIFICATION | BP36-ANDROID-NOTIFICATION | INV36-NTFY-ANDROID-OWNERSHIP, INV36-NTFY-NO-AUTHORITY | AC36-RT-11 | MS36-NTFY-09/10/11 |

## Mental scenarios

- **MS36-NTFY-01:** cross-account or cross-conversation topic access is denied.
- **MS36-NTFY-02:** member removal/rejoin while connected respects the approved revocation bound and hides the membership gap.
- **MS36-NTFY-03:** logout/account switch clears the old subscription and account-scoped state before new data appears.
- **MS36-NTFY-04:** missing, duplicate, delayed and reordered signals converge through authoritative HTTP history.
- **MS36-NTFY-05:** post-commit ntfy failure does not reverse the committed send and does not create duplicate messages.
- **MS36-NTFY-06:** a chat signal/document reference does not bypass Drive ACLs or expose protected content.
- **MS36-NTFY-07:** unsupported browser auth or Android background delivery fails safely and reconciles on resume.
- **MS36-NTFY-08:** unknown deployed integration/rollback compatibility keeps implementation and rollout blocked.

## Android system notification lifecycle — requirements draft

**Status: PROPOSED_NOT_APPROVED. XML and Structurizr C1 parser gates: PASS on the validated artifact blobs below; Independent QA and Human Approval remain pending.** The ntfy Android app's own notification is not a BFE notification. BFE Android Client must post its own generic system notification from the opaque `wakeToken` while its eligible stream/service is active. The current Flutter source constraints still prohibit FCM and a required separate app, so a foreground-service-based ntfy subscription is the only current-constraint-preserving candidate; it is not cleared for Google Play or production.

| Android state | Draft behavior | Limit |
|---|---|---|
| App foreground | ntfy HTTP JSON/SSE stream is used as a wake-up signal; refresh chat via authorized HTTP history | A missing event never proves that no message exists |
| Background after explicit user enablement | A BFE-owned native FGS may keep the stream active and post generic BFE notifications; best effort only | Start the service while the app is visible; persistent accurate notification and user stop control are required |
| Background without an active FGS / user stops service | No live-stream or notification delivery is promised | Do not try to start the service from background without a documented Android exemption |
| Screen off / Doze | Delivery may continue only while the service/connection survives and Android permits network access | No universal delivery or latency guarantee; validate on supported physical devices |
| Network unavailable | Reconnect when possible, then reconcile through authoritative history | Stream absence is not proof of message absence |
| OS process death | Service restart and delivery are not guaranteed | Reconcile after manual launch/resume |
| Device reboot | Automatic service restart is not assumed | Reconcile after the user launches/resumes BFE |
| User force-stop | No delivery is promised until manual relaunch | Reconcile history after the user reopens BFE |
| POST_NOTIFICATIONS denied / channel disabled | No ordinary visible BFE notification; Android controls how the required FGS notification is surfaced | Respect Android 13+ runtime permission and user channel settings |
| Logout / account switch | Stop old subscription/service and clear prior-session state before new-session data | Revalidate the new session before history catch-up |
| Chat administratively disabled | Existing subscription may receive only wakeToken; no new operations/history/subscriptions | After re-enable, use authorized history catch-up; do not add periodic Drive-session checks |

**Architecture candidate under the fixed constraints:** use ntfy's long-lived HTTP JSON/SSE stream for in-app realtime. For optional best-effort Android background notifications, BFE itself owns a native foreground service that is started from a visible activity only after explicit user enablement, displays a persistent accurate notification with a stop action, and turns opaque wakeToken events into generic BFE notifications. Android 12+ restricts starting FGS from the background. Android 14+ requires the service type and corresponding permission. For a continuous stream, `dataSync` is unsuitable for Android 15+ targets because its background use is limited to six hours per 24 hours; `remoteMessaging` is not assumed to apply because its documented purpose is text-message transfer/continuity between devices. `specialUse` is only a conditional candidate if no other type accurately fits; it needs `FOREGROUND_SERVICE_SPECIAL_USE`, a manifest subtype property and Google Play declaration/review. Play policy requires user benefit, user awareness/control and a justified duration; approval is not guaranteed. Android 13+ ordinary notifications require runtime `POST_NOTIFICATIONS` permission and channels may be disabled. Do not request battery-optimization exemption by default.

FCM/Firebase and UnifiedPush are excluded target alternatives under the current Project Owner constraints; do not implement or substitute either. If no FGS type/policy path is accepted, background delivery remains a blocking product limitation to return to the Project Owner, not a reason to silently change the constraints. Publisher-controlled BFE application data is a JSON object with exactly one field, `wakeToken`, distinct from the ntfy transport JSON envelope (`id`, `time`, `event`, `topic`, etc.). Envelope metadata is never BFE message identity/order/cursor or authorization. The target history contract mandates stable persisted message identity, monotonic conversation-scoped order and server-issued cursor pagination; current API conformance is unverified and blocking. Project Owner approved exact-conversation notification routing: a separate current-session/DPoP-authorized lookup must resolve opaque wakeToken, re-check current membership, then open the corresponding conversation. wakeToken contains no conversation/user/message identifiers and grants no authority. The proposed backend lookup contract is documented below. Project Owner approval of the proposed 30-day token lifetime and reusable/non-consuming lookup remains blocking; the existing schema also lacks a unique constraint on wake_token and explicit expiry storage/enforcement. FGS/Google Play eligibility remains a separate blocker.


## Backend notification-target lookup contract — OQ36-NTFY-12

**Status: PROPOSED_NOT_APPROVED; implementation blocked.** Source review was limited to this repository's PR branch and existing backend files. No endpoint, migration, source code, or tests were changed.

### Existing backend mechanisms verified

- `internal/issue25/http.go` already enforces the protected HTTP boundary: exactly one `Authorization: DPoP <access-token>` header and one `DPoP` proof header, validates the caller through the existing `Validator.ValidateCaller` / Internal Auth UDS path, and binds the proof to the HTTP method and public HTTPS path. Query strings are rejected.
- `internal/authentication/authentication.go` revalidates the session through Internal Auth and verifies DPoP key binding, `htm`, `htu`, `ath`, proof freshness and replay. The lookup must reuse this boundary, not create a parallel auth mechanism.
- `internal/issue25/notify.go` creates one server-side notification attempt per active recipient and message. `internal/issue25/realtime.go` publishes the opaque `wakeToken` only; `WakeToken()` uses the existing UUIDv4 generator.
- `issue25_publish_attempts` already stores `wake_token`, `viewer_id`, `membership_epoch`, `conversation_id` and `created_at`. Membership history is stored in `issue25_memberships`; the current active membership row has `leave_boundary IS NULL`. The existing schema does **not** declare `wake_token` unique and does not store an explicit expiry/revocation timestamp.
- `internal/issue25/history.go` checks current membership and visibility intervals when reading history. The lookup must return only a conversation route; it must not return or expose message data.
- The reviewed HTTP router has no `/conversations/notification-target` route today; it must be added as a fixed route before the existing one-segment direct/group dispatch and dynamic conversation-ID parsing. In the reviewed request path, an explicit Chat-disabled enforcement hook was not established; the backend owner must identify/prove that policy before implementing this route.

### Proposed HTTP contract

| Item | Requirement |
|---|---|
| Method/path | `POST /conversations/notification-target`; fixed route must precede dynamic conversation path parsing |
| Request | `Content-Type: application/json`; exactly `{"wakeToken":"opaque-token"}`; no query string or additional fields |
| Authentication | Existing `Authorization: DPoP <access-token>` plus one `DPoP` proof; validate the current BFE session through Internal Auth and bind DPoP to the exact POST path |
| Server lookup | Match exactly one `issue25_publish_attempts.wake_token` record; require `viewer_id == authenticated account` and an unexpired token |
| Current authorization | Re-check current Chat availability and current active membership; the active `membership_epoch` must equal the record's `membership_epoch`. Repeat on every lookup; never trust cached ACLs or client-supplied IDs |
| Success | `200` with only `{"conversationId":"uuid"}`; set `Cache-Control: no-store` and `X-Content-Type-Options: nosniff` |
| Invalid body/token syntax | `400 VALIDATION_ERROR` |
| Missing/invalid/revoked session or DPoP | Existing `401` authentication error mapping |
| Unknown, expired, foreign-account, removed/rejoined epoch, ambiguous mapping or unavailable conversation | Same non-disclosing `404 NOT_FOUND`; no conversation ID or existence distinction |
| Internal Auth/database unavailable | `503`; do not return routing data |

The client then fetches history through the existing authorized `GET /conversations/{conversationId}/history` endpoint. History applies current membership and membership-interval visibility rules; `wakeToken` does not identify a message and does not authorize history.

### Token lifetime, reuse and revocation — proposed policy

- **Lifetime proposal:** 30 days from the server-recorded `created_at`, with absolute expiry not extended by lookup. This is a proposal, not a Project Owner decision.
- **Reuse:** lookup is repeatable and non-consuming until expiry. Repeated taps for the same still-authorized recipient return the same conversation ID; no lookup changes token state or extends lifetime. Security comes from current-session DPoP, recipient binding and current membership checks, not one-time-token semantics.
- **Membership revocation:** removal/rejoin changes the active membership epoch; the old attempt's epoch no longer matches, so lookup returns the same 404 as any invalid token. Re-joining does not restore access to messages in the membership gap.
- **Logout/account switch:** each request revalidates the current session through existing Internal Auth. The client must clear notification-routing state on logout/switch; a new account cannot resolve a token whose `viewer_id` differs. The BFE auth owner must confirm that logout revokes the old session in Internal Auth.
- **Token-specific revocation:** no separate bearer capability or public revoke endpoint is proposed. Access is denied by session revocation, recipient mismatch, membership-epoch change, Chat availability and expiry. Because `wake_token` is not authority, knowing it alone never grants access.
- **Schema prerequisite:** add a database uniqueness constraint/index for `wake_token` and deterministic 30-day expiry enforcement in a separately approved downstream contract/migration. The current schema alone does not prove uniqueness or expiry enforcement; no migration is part of this requirements task.
- Never put `wakeToken` in a URL or application logs. Do not return message ID, message content, document metadata, user IDs, membership details or token metadata in the success response.

### Negative scenarios and acceptance

1. Missing/invalid DPoP, expired/revoked session, or a DPoP proof bound to another method/path: reject before lookup; no routing data.
2. Account B submits account A's valid token: return the same 404 as unknown/expired token.
3. Recipient removed or removed then rejoined: active epoch mismatch; same 404 and no conversation ID.
4. Expired, unknown, malformed or ambiguous (multiple-row) token mapping: 400 only for malformed syntax; otherwise the same 404, with no existence leak.
5. Token tapped twice before expiry by the same currently authorized account: same conversation ID both times; no consumption, expiry extension or side effect.
6. Session revoked after the notification was delivered: revalidation fails; a token cannot keep the old session alive.
7. Chat unavailable/disabled or database/Internal Auth unavailable: no conversation ID; deny or return 503 according to the error mapping above.
8. After a successful lookup, history still independently filters current membership intervals and document access remains governed by BFE Drive.

### Remaining approval/evidence gates

- **OQ36-NTFY-12 remains BLOCKING.** Project Owner must approve or change the proposed 30-day lifetime and repeat-use policy.
- BFE auth owner must confirm logout/session revocation behavior through existing Internal Auth.
- Backend owner must identify/prove how Chat administrative disablement is enforced on the new route; the existing reviewed route path does not establish that gate.
- Downstream approved schema work must enforce unique token mapping and deterministic expiry before implementation.
- The contract remains a GRACE Requirements Analysis proposal; no endpoint implementation, schema change, migration or test change is authorized.

Official evidence: [ntfy subscription API](https://docs.ntfy.sh/subscribe/api/), [Android Doze](https://developer.android.com/training/monitoring-device-state/doze-standby), [foreground-service types](https://developer.android.com/develop/background-work/services/fgs/service-types), [background FGS start restrictions](https://developer.android.com/develop/background-work/services/fgs/restrictions-bg-start), [Android 15 FGS timeout](https://developer.android.com/develop/background-work/services/fgs/timeout), [Google Play FGS declaration/review](https://support.google.com/googleplay/android-developer/answer/13392821?hl=en-GB), [notification permission](https://developer.android.com/develop/ui/compose/notifications/notification-permission), [ntfy instant delivery](https://docs.ntfy.sh/subscribe/phone/), [ntfy self-hosted FCM](https://docs.ntfy.sh/config/), [UnifiedPush Android spec](https://unifiedpush.org/developers/spec/android/), [FCM force-stop behavior](https://firebase.google.com/docs/cloud-messaging/flutter/receive-messages).

## Blocking questions
| ID | Owner | Required evidence and closure |
|---|---|---|
| OQ36-NTFY-01 | BFE Drive/Internal Auth and backend owners | Auth flow, capability lifetime, replay/expiry/account-switch/revocation tests; session-bound auth is fail-closed |
| OQ36-NTFY-02 | Backend and ntfy infrastructure owners | Endpoint/topic/ACL ownership and negative cross-topic tests; server-controlled isolation proven |
| OQ36-NTFY-03 | Backend and ntfy infrastructure owners | Active subscriber removal/rejoin tests; approved revocation bound demonstrated |
| OQ36-NTFY-04 | Flutter Web and backend owners | Browser matrix, CORS and credential-leak tests; supported auth flow works without URL credentials |
| OQ36-NTFY-05 | Flutter Android owner and Project Owner | Evidence for the exact FGS type, manifest permissions/subtype, visible/stoppable notification, Google Play declaration/review, and device matrix including Doze, process death, reboot and force-stop |
| OQ36-NTFY-06 | Backend and Flutter owners | Confirm stable persisted message identity/order and cursor pagination in the current HTTP history API; if missing, approve an upstream contract requirement before implementation; never infer cursor/order from ntfy metadata |
| OQ36-NTFY-07 | Backend owner and Project Owner | Failure analysis, retry/outbox and telemetry tests; persisted send independent of ntfy |
| OQ36-NTFY-08 | Project Owner, Release Manager and client maintainers | Deployed dependency inventory and rollout/rollback plan; production impact evidenced |
| OQ36-NTFY-09 | Project Owner | RESOLVED — target-direction decision recorded in [Issue #36 comment](https://github.com/bogachenko/bfe-chat-service/issues/36#issuecomment-6385730270); draft approval remains blocked by other gates |

- **OQ36-NTFY-10:** RESOLVED by Project Owner on 2026-10-25: best-effort BFE-owned Android notifications are accepted. The exact FGS type, consuming-host target SDK/manifest and Google Play eligibility remain blocking evidence gates.
- **OQ36-NTFY-11:** RESOLVED by Project Owner on 2026-10-25: tapping a notification must open the corresponding conversation. The separately authorized lookup contract remains blocking as OQ36-NTFY-12.
- **OQ36-NTFY-12:** BLOCKING — proposed contract is in the section “Backend notification-target lookup contract — OQ36-NTFY-12”. Project Owner decision is required for the proposed 30-day absolute TTL and reusable/non-consuming lookup; BFE auth owner must confirm logout/session revocation; an approved downstream schema change must enforce unique wake_token mapping and deterministic expiry. No implementation until these and the GRACE gates are approved.

## Production boundaries
- bfe-drive is production and strictly READ-ONLY.
- bfe-drive-client is production; main and production/android must not change.
- Existing chat package integration in deployed clients is not assumed.
- No code, config, database, migration, permission, ntfy infrastructure or deployment change is authorized.
- Technology.xml, DevelopmentPlan.xml and Contracts.xml remain unchanged; their Centrifuge assumptions are downstream blockers after RequirementsAnalysis approval.
- Issue #36 is not edited or closed.

## Verification status

**Parser gates: PASS on the exact current artifact blobs.** Validation used the permanent GitHub Actions workflow `.github/workflows/requirements-c1-validation.yml`, the Python standard-library XML parser and Structurizr `2026.09.19`.

- Validation commit: `81a546d04ca5ee8f6ce25cc98c63f85794cc4862`.
- `RequirementsAnalysis.xml` blob: `cd15d699a1782627c603290f7a15d494d763a0ff`.
- C1 `workspace.dsl` blob: `a4e303085b83c61c10409ce63cc83518cdb62efb`.
- XML parser: **PASS**, exit code 0; root `REQUIREMENTS_ANALYSIS`; 107 IDs, 107 unique; 318 local references resolved; one historical external provenance reference (`APPROVAL25-REQUIREMENTS`).
- Structurizr parser: **PASS**, exit code 0; image `structurizr/structurizr:2026.09.19`, digest `sha256:721136283c2f9cf1ba69037bc9de136c579d66fdcb2d771cb60546ec68def1a5`.
- Workflow run and logs: [40191845919](https://github.com/bogachenko/bfe-chat-service/actions/runs/40191845919).
- The report-only commit does not change these validated XML/DSL blobs.
- Independent QA: `NOT RUN`; Human Approval: `NOT GRANTED`; Technology, implementation and deployment remain **BLOCKED**.
- No production code/configuration, database, migrations, permissions or infrastructure were changed; no tests were added or modified. `bfe-drive` remains READ-ONLY; `bfe-drive-client` production branches remain untouched.

## Cross-component semantic consistency review (2026-10-24)

**Result: PARTIAL ALIGNMENT — REQUIREMENTS APPROVAL BLOCKED.** This is an orchestrator content review, not the formal Independent QA gate. No RequirementsAnalysis XML, C1 DSL, production artifact or implementation was changed by this review.

Reviewed against Flutter draft HEAD `4b8d70e5ef7e59ed253d0d04b8150dbd481eff10`, including its `RequirementsAnalysis.xml` blob `8d8912fd701da001a326ff2e3aacc2f45b662433`, and the current Backend Issue #36 and Flutter Issue #4 text.

### Cross-component consistency — corrected draft

**Result: PARTIAL ALIGNMENT; requirements approval remains blocked.** The Project Owner's target direction is now explicit in [Backend Issue #36](https://github.com/bogachenko/bfe-chat-service/issues/36#issuecomment-6385730270): the GRACE target is to replace Centrifuge with ntfy. This resolves the target-direction conflict for requirements work only; it does not approve the draft or authorize implementation.

| Backend draft | Flutter draft | Current result |
|---|---|---|
| `REQ36-RT-01` | `NTFY-RT-01` | Target agrees: ntfy replaces Centrifuge; HTTP/PostgreSQL remain authoritative. |
| `REQ36-RT-02` | `NTFY-RT-02` | Server-side recipient/topic authorization; sender is explicitly excluded from recipients and acceptance tests. |
| `REQ36-RT-03`, `REQ36-RT-07` | `NTFY-RT-03` | Membership/session revocation and account-switch isolation aligned; actual revocation bound remains unresolved. |
| `REQ36-RT-04` | `NTFY-RT-04` | Authoritative-history reconciliation aligned; event identity/cursor, dedupe and catch-up trigger remain unresolved. |
| `REQ36-RT-05`, `REQ36-RT-08` | `NTFY-RT-05` | Clarified: document ID/title remain in authorized backend history/internal events as required by `REQ32-DOCUMENT`, but are excluded from ntfy payloads. |
| `REQ36-RT-06` | `NTFY-RT-06` | Post-commit failure behavior aligned; retry/outbox/dedupe mechanism remains unresolved. |
| `REQ36-RT-09` | `NTFY-RT-07` | Production-boundary invariant and direct traceability added; production impact and release evidence remain required. |
| `REQ36-RT-10` | `NTFY-RT-08` | Explicit acceptance now requires supported-browser and Android-lifecycle evidence. |

### Remaining blockers

1. **ntfy authorization and revocation:** exact session-bound authentication, topic/ACL ownership, and active-subscription revocation bound require evidence and negative tests.
2. **Event and failure contract:** payload schema/version, event ID/cursor, retry/outbox, deduplication and catch-up behavior remain unspecified.
3. **Production compatibility:** actual deployed-client dependency inventory, release coordination, rollback and migration/coexistence needs remain unproven.
4. **Validation and independent acceptance:** XML and Structurizr parser gates passed on the exact final XML/C1 blobs recorded above; Independent QA has not run.

### Orchestrator pre-QA consistency correction

The draft's supersession note now explicitly distinguishes the historical wording in the original Issue #36 body from the Project Owner's decision comment that resolves the target direction for this GRACE revision. The XML parser gate was reset before validation and returned PASS on the final XML blob above. The C1 DSL blob was unchanged and passed in the same run.

### Gate status

- Backend XML parser: `PASS` on blob `cd15d699a1782627c603290f7a15d494d763a0ff`, validated on HEAD `81a546d04ca5ee8f6ce25cc98c63f85794cc4862` by [GitHub Actions run 40191845919](https://github.com/bogachenko/bfe-chat-service/actions/runs/40191845919): 107 IDs, 107 unique, 318 local references, one historical external provenance reference.
- Backend Structurizr parser: `PASS` on C1 blob `a4e303085b83c61c10409ce63cc83518cdb62efb`, same run.
- Flutter XML parser: `PASS` on blob `64f5dde91020fda36cddb68cb930450e850e1527`, validated on HEAD `06bf808289ca8386f4e5fa60403cc53a7a24757d` by [GitHub Actions run 40191852479](https://github.com/bogachenko/bfe-chat-flutter/actions/runs/40191852479): 305 IDs, 305 unique, 1,288 local references, zero historical external references.
- Flutter Structurizr parser: `PASS` on C1 blob `26c3de3f5f624f21fc0b5b04599b73d6882ea552`, same run.
- Independent QA: `NOT RUN`.
- Human Approval: `NOT GRANTED`.
- Technology, implementation and deployment: **BLOCKED**.

The report-only follow-up does not change the validated XML or C1 blobs. No production code, configuration, database, migrations, permissions or infrastructure were changed; no tests were added or modified.

## Administrative Chat disablement — cross-component decision (2026-10-24)

The existing subscription may continue receiving only the opaque `wakeToken` while Chat is administratively disabled. No new Chat operations, history requests or subscriptions are allowed while disabled. Re-enabling Chat triggers authorized history catch-up. This administrative state is distinct from membership-epoch/session revocation and does not add periodic BFE-session checks or Drive callbacks.


## Latest validation evidence (2026-10-25)

- Repository/branch: `bogachenko/bfe-chat-service` / `docs/issue-36-ntfy-realtime-requirements-draft`.
- PR: [#37](https://github.com/bogachenko/bfe-chat-service/pull/37), remains open and draft.
- **Validation commit/HEAD:** `ba8f4cb0aa191bd323bd7f58267ca2e8c431670a`.
- `RequirementsAnalysis.xml` blob: `9b7dec7a337878267a2d7d567f05c7a05725725a`.
- C1 `docs/architecture/c4/workspace.dsl` blob: `a1c87d50ddd2ad139be7d9e395fc044c2327f682` (unchanged).
- GitHub Actions: [run 40208796934](https://github.com/bogachenko/bfe-chat-service/actions/runs/40208796934) — **success** on the exact validation commit and XML/C1 blobs above.
- XML parser: **PASS**, exit code 0; root `REQUIREMENTS_ANALYSIS`; 117 IDs, 117 unique, 0 duplicates; 360 local references resolved; one historical external provenance reference retained.
- Structurizr C1 parser: **PASS**, exit code 0; pinned image digest `sha256:721136283c2f9cf1ba69037bc9de136c579d66fdcb2d771cb60546ec68def1a5`.
- Independent QA: **NOT RUN**. Human Approval: **NOT GRANTED**.
- Technology, implementation, migration and deployment: **BLOCKED**. `OQ36-NTFY-12` remains blocking pending Project Owner approval of the proposed 30-day TTL/reuse policy, Internal Auth logout-revocation confirmation, and approved downstream uniqueness/expiry enforcement.
- This section is a report-only update after validation. It does not change the validated XML/C1 blobs; the PR head may advance for this Markdown-only commit. Historical validation records above remain preserved and are not presented as current evidence.
