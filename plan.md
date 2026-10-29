# Implementation plan

This plan implements `spec.md` (R1 to R9). It reuses the existing files `src/portal/result-route.mjs` and `src/service/queue.mjs` without modifying them, and adds one new file, `src/service/notify.mjs`. The archived helper is rejected and not used anywhere in this plan.

## Ordered steps

1. Confirm the inputs: check with `ls` that the existing files `src/portal/result-route.mjs` and `src/service/queue.mjs` are present, and confirm that the archived helper does not exist and is not used (R5).
2. Write the tests named in *Risks and verification* first, so each requirement has a failing test before code exists (R1 to R8).
3. Create the new file `src/service/notify.mjs` with the release gate: accept only a result version that a doctor has reviewed and released, and return without enqueueing otherwise (R1, R2).
4. In the new file `src/service/notify.mjs`, derive the idempotency key from the released result version and skip any key already delivered or in flight (R6).
5. Build the content-free message from a fixed generic template and a link made from `resultRoute` in the existing file `src/portal/result-route.mjs` (R3, R5).
6. Enqueue the job through `enqueueNotification` in the existing file `src/service/queue.mjs`, then add the sender's rate and batch limiter: at most 50 requests per second and at most 20 messages per batch (R7).
7. Add bounded retries that reuse the same idempotency key and mark the delivery failed when the bound is reached; log only the event ID and delivery state (R4, R8).
8. Keep the channel behind an interface and the recipient fixed to the patient until OPEN-1 and OPEN-2 are approved by their owners (R9).

## Paths and dependencies

- Existing file `src/portal/result-route.mjs`: exports `resultRoute`; reused read-only by step 5.
- Existing file `src/service/queue.mjs`: exports `enqueueNotification(queue, job)`; reused read-only by step 6.
- New file `src/service/notify.mjs` (proposed, does not exist yet): depends on both existing files; created in step 3 and extended in steps 4 to 8.
- Dependency order: tests (step 2) → release gate (step 3) → idempotency key (step 4) → message and link (step 5) → enqueue and throughput limits (step 6) → retries and logging (step 7) → channel and recipient guard (step 8). Step 6 needs the key from step 4 so that the queue never holds two jobs for one released version.
- No new dependencies are installed, and nothing depends on the archived helper, which is rejected.
- Open decisions block only the final channel choice (OPEN-1, owner: product owner with privacy officer approval) and any referring-doctor path (OPEN-2, owner: clinical lead with privacy officer approval).

## Risks and verification

Risks:

- Notifying before release would expose unreviewed results; the release gate test fails if any unreleased version enqueues a job.
- Clinical content or patient identifiers could leak through message text or logs; the privacy test inspects both the message body and every log entry.
- Retries after a provider timeout could deliver twice; the idempotency key is the released result version and the duplicate test checks for exactly one delivery.
- Bursts could exceed the provider limit of 50 requests per second or 20 messages per batch; the throughput test checks the exact boundary.
- Reintroducing archived advice (portal redesign, notifying on upload, the archived helper) is rejected; review checks that none of it appears in code.

| Requirement | Step | Existing path | Test |
| --- | --- | --- | --- |
| R1, R2 release gate | 3 | - | `release_gate_blocks_unreleased_result`: unreleased or unreviewed results enqueue nothing; one release enqueues one job. |
| R3, R4 privacy | 5, 7 | - | `notification_and_logs_exclude_clinical_content`: message and logs hold only generic text, link, event ID and delivery state. |
| R5 portal route reuse | 5 | src/portal/result-route.mjs | `link_uses_existing_portal_route`: link equals `resultRoute` with the result ID filled in. |
| R6 idempotency | 4, 6 | src/service/queue.mjs | `duplicate_retry_delivers_once`: the same released version is delivered once across retries and repeat events. |
| R7 throughput | 6 | src/service/queue.mjs | `provider_throughput_boundary`: 50 requests per second and 20 per batch pass; the 51st request waits and the 21st message starts a new batch. |
| R8 bounded retries | 7 | - | `bounded_retry_on_provider_failure`: retries stop at the bound with the same key and mark the delivery failed. |
| R9 recipients and channel | 8 | - | `recipient_is_patient_only`: no referring-doctor or unapproved channel send is possible while OPEN-1 and OPEN-2 are open. |
