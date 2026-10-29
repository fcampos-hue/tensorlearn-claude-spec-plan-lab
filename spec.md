# Specification

Scope: the results-ready notification for the fictional clinic, derived from the approved intent in `intent.md` (I-1 to I-4, OPEN-1, OPEN-2) and the constraints in `inputs/current-policy.md`. The archived proposal in `inputs/archived-design.md` was never approved and is rejected in full; none of its advice is adopted.

## Requirements

- **R1 (I-1) Release gate.** A patient notification is created only after a doctor has reviewed and released a result version; a result that has not been released never produces a notification.
- **R2 (I-1) Trigger point.** The release event is the only trigger; notifying on upload, as the archived proposal suggested, is rejected.
- **R3 (I-2) Content-free notification.** The notification contains only a generic "a result is ready" message and a link to the existing authenticated portal route; clinical content stays out of notifications and logs.
- **R4 (I-2) Privacy-safe logging.** Operational logs record only event IDs and delivery states, never patient identifiers or clinical payloads.
- **R5 (I-3) Reuse the existing portal route.** The link is built from `resultRoute` (`/portal/results/:resultId`) exported by the existing file `src/portal/result-route.mjs`, used unchanged; the archived portal redesign is rejected and out of scope.
- **R6 (I-4) Idempotency key.** The idempotency key is the released result version (result ID plus release version), so one released version is never delivered twice; a newly released version gets a new key.
- **R7 (I-4) Provider throughput.** Sends never exceed 50 requests per second or 20 messages per batch.
- **R8 (I-4) Bounded retries.** Provider failures are retried a bounded number of times, always with the same idempotency key, so retries cannot create duplicate deliveries; when the bound is reached the delivery is marked failed and only its event ID and state are logged.
- **R9 (OPEN-1, OPEN-2) Recipients and channel.** Only the patient is notified; the channel is not fixed in code until OPEN-1 is approved, and referring-doctor notifications are not built until OPEN-2 is approved.

## Design

- **Flow:** release event → release-gate check (R1, R2) → build idempotency key from the released result version (R6) → skip if the key was already delivered or is in flight → build the content-free message with the portal link (R3, R5) → enqueue through `enqueueNotification` in the existing file `src/service/queue.mjs` → sender drains the queue in batches of at most 20 messages and at most 50 requests per second (R7) → bounded retries with the same key (R8) → record delivery state.
- **New code:** one new file `src/service/notify.mjs` holds the release gate, key derivation, message builder, rate and batch limiter and retry policy. It imports the two existing modules and does not modify them.
- **Reused code:** the existing file `src/portal/result-route.mjs` (route constant) and the existing file `src/service/queue.mjs` (queue adapter) are reused as they are.
- **Rejected alternatives:** the archived helper is rejected and not used (it was never implemented); sending every uploaded result immediately is rejected; including result details in messages is rejected; rebuilding the portal is rejected.
- **Message template:** a fixed, generic text such as "A new result is available in your portal" plus the portal link; no result values, test names, diagnoses or patient identifiers.
- **Logging:** a log entry contains only `eventId` and `deliveryState` (queued, sent, failed, skipped-duplicate).
- **Channel:** the sender talks to a channel interface; the concrete channel is configured once OPEN-1 is decided, not chosen in this spec.

## Open decisions and owners

| ID | Decision | Owner | Status and effect on this spec |
| --- | --- | --- | --- |
| OPEN-1 | Which channel to use for the patient notification (for example SMS, email or app push). | Product owner with privacy officer approval | Open, no approval recorded. R9 keeps the channel behind an interface; nothing ships to a channel until approved. |
| OPEN-2 | Whether referring doctors are also notified. | Clinical lead with privacy officer approval | Open and out of scope. R9 limits recipients to the patient; no referring-doctor path is built. |

## Traceability

| Intent | Requirement | Test |
| --- | --- | --- |
| I-1 notify after clinical review and release | R1, R2 | `release_gate_blocks_unreleased_result`: an unreleased or unreviewed result enqueues nothing, and releasing it enqueues exactly one job. |
| I-2 clinical content stays in the portal | R3, R4 | `notification_and_logs_exclude_clinical_content`: the message body and every log entry contain no clinical payload or patient identifier, only the generic text, link, event ID and delivery state. |
| I-3 reuse the current portal route | R5 | `link_uses_existing_portal_route`: the notification link equals `resultRoute` from `src/portal/result-route.mjs` with the result ID filled in. |
| I-4 no duplicates, provider contract | R6, R7, R8 | `duplicate_retry_delivers_once`: re-releasing or retrying the same released version delivers once. `provider_throughput_boundary`: 50 requests per second and 20 messages per batch pass, the 51st request waits and a 21st message goes into a new batch. `bounded_retry_on_provider_failure`: failures retry up to the bound with the same key, then the delivery is marked failed with no duplicate. |
