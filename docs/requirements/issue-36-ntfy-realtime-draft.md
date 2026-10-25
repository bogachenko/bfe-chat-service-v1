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
| App foreground | ntfy wake-up refreshes the open chat through HTTP/PostgreSQL; notification rendering is a separate client UX choice | A missing event never proves that no message exists |
| Background / screen off | BFE-owned system notification is best-effort only while the client stream/service remains active | Foreground service needs an ongoing notification and must satisfy platform/policy requirements |
| Doze | Direct network streams can be suspended; ntfy's own app documents FGS-based instant delivery, but that behavior must be verified for BFE | No universal latency/delivery guarantee |
| OS process death | Delivery/restart is not guaranteed | Reconcile history on next launch/resume |
| User force-stop | No delivery is promised until manual relaunch | Reconcile history after the user reopens BFE |
| Notifications denied/channel disabled | No visible system notification | Respect POST_NOTIFICATIONS (Android 13+) and user channel settings |
| Chat administratively disabled | Existing subscription may receive only wakeToken; no new operations/history/subscriptions | After re-enable, use authorized history catch-up; do not add periodic Drive-session checks |

**Mechanism comparison:** FCM is Android's recommended route for chat notifications during Doze, but violates the current no-FCM requirement and needs an explicit Project Owner override. UnifiedPush avoids Firebase but requires a BFE connector and a configured distributor, conflicting with the no-extra-app requirement unless that requirement changes. A foreground service preserves current constraints only conditionally; Android 14+ requires service type/permission, specialUse requires a subtype property and Google Play review, and Android 12+ restricts background starts. wakeToken is non-identifying and non-authorizing, so notification tap opens the generic Chat entry point; exact conversation routing requires a separately approved backend lookup contract.

Official evidence: [Android Doze](https://developer.android.com/training/monitoring-device-state/doze-standby), [foreground-service types](https://developer.android.com/develop/background-work/services/fgs/service-types), [notification permission](https://developer.android.com/develop/ui/compose/notifications/notification-permission), [ntfy instant delivery](https://docs.ntfy.sh/subscribe/phone/), [ntfy self-hosted FCM](https://docs.ntfy.sh/config/), [UnifiedPush Android spec](https://unifiedpush.org/developers/spec/android/), [FCM force-stop behavior](https://firebase.google.com/docs/cloud-messaging/flutter/receive-messages).

## Blocking questions
| ID | Owner | Required evidence and closure |
|---|---|---|
| OQ36-NTFY-01 | BFE Drive/Internal Auth and backend owners | Auth flow, capability lifetime, replay/expiry/account-switch/revocation tests; session-bound auth is fail-closed |
| OQ36-NTFY-02 | Backend and ntfy infrastructure owners | Endpoint/topic/ACL ownership and negative cross-topic tests; server-controlled isolation proven |
| OQ36-NTFY-03 | Backend and ntfy infrastructure owners | Active subscriber removal/rejoin tests; approved revocation bound demonstrated |
| OQ36-NTFY-04 | Flutter Web and backend owners | Browser matrix, CORS and credential-leak tests; supported auth flow works without URL credentials |
| OQ36-NTFY-05 | Flutter Android owner and Project Owner | Current policy review and Doze/process-death tests; lifecycle limits documented |
| OQ36-NTFY-06 | Backend and Flutter owners | Versioned schema, event IDs/cursors, dedupe/gap examples; deterministic reconciliation approved |
| OQ36-NTFY-07 | Backend owner and Project Owner | Failure analysis, retry/outbox and telemetry tests; persisted send independent of ntfy |
| OQ36-NTFY-08 | Project Owner, Release Manager and client maintainers | Deployed dependency inventory and rollout/rollback plan; production impact evidenced |
| OQ36-NTFY-09 | Project Owner | RESOLVED — target-direction decision recorded in [Issue #36 comment](https://github.com/bogachenko/bfe-chat-service/issues/36#issuecomment-6385730270); draft approval remains blocked by other gates |

- **OQ36-NTFY-10:** Project Owner must choose whether to retain the current no-FCM/no-extra-app constraints with best-effort FGS delivery or approve a requirements change.
- **OQ36-NTFY-11:** Exact-conversation navigation is unresolved because wakeToken contains no identifying metadata; generic Chat entry is the current draft behavior.

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
