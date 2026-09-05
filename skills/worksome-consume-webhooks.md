---
name: worksome-consume-webhooks
description: Register a Worksome webhook, verify its HMAC-SHA256 signature, route the 17 event types, and recover missed deliveries with the webhook event log and retry mutation.
api: Worksome GraphQL API
endpoint: https://api.worksome.com/graphql
operations:
  - createWebhook
  - updateWebhook
  - deleteWebhook
  - webhooks
  - webhook
  - webhookEvents
  - webhookEventLogs
  - retryWebhookEvent
generated: '2026-09-04'
method: generated
source: https://docs.worksome.com/webhooks/ + https://docs.worksome.com/webhooks/guides/handle-webhooks/ + https://docs.worksome.com/webhooks/reference/ + graphql/worksome-introspection.json
---

# Consume Worksome webhooks

Worksome sends 17 event types. Payloads are deliberately minimal — ids plus a few key fields — and the expectation is that you call the GraphQL API with those ids for anything more. The ids are the same Global IDs the API uses.

## 1. Register the endpoint

```graphql
mutation CreateWebhook($input: CreateWebhookInput!) {
  createWebhook(input: $input) { id title url isActive }
}
```

`CreateWebhookInput`, from the schema:

| Field | Type | Notes |
|---|---|---|
| `company` | `ID!` | Required. |
| `title` | `String!` | Required. |
| `url` | `String!` | Required. Must be HTTPS. |
| `description` | `String` | |
| `secret` | `String` | The shared secret used to sign deliveries. |
| `isActive` | `Boolean` | |
| `subscribedEvents` | `[WebhookEventType!]` | The events you want. |
| `clientId` / `clientUrl` | `String` | |

The CLI does the same: `worksome webhooks create`, `worksome webhooks list`, `worksome webhooks delete`.

## 2. Verify the signature before parsing

Every delivery carries a `Signature` header: an HMAC-SHA256 digest of the **raw request body**, signed with your shared secret.

Read the raw body first. Do not parse, re-serialise, or let a framework normalise it — any of those changes the bytes and the digest will not match.

```python
import hmac, hashlib

def verify(raw_body: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(secret.encode(), raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature or "")
```

Always compare in constant time. Reject with 401 on a mismatch, and fail closed on a missing or malformed header rather than crashing.

**Know what this signature does not give you.** There is no timestamp and no version prefix in the signed material, so there is no bounded replay window — a captured request stays valid indefinitely. Your defences are HTTPS, keeping the secret out of source control, rotating it periodically with Worksome support, and idempotent handling.

## 3. Route on `event`

```json
{ "event": "contractAccepted", "data": { "contract": {}, "worker": {}, "trustedContact": {}, "customFieldValues": [] } }
```

**Hires and contracts:** `contractAccepted`, `hireUpdated`, `hireCancelled` (adds `cancelReason`), `hireEnded`, `hireTerminated` (adds `terminatedReason`).

**Talent pool:** `trustedContactUpdated`. Note the payload explicitly does **not** say what changed — you must re-query and diff.

**Payment requests:** `paymentRequestIssued`, `paymentRequestApproved`, `paymentRequestRejected`, `paymentRequestPaid`, `paymentRequestCancelled`, `paymentRequestWorkerPaidOut`, `paymentRequestRecruiterPaidOut`. All seven share an identical payload shape (`paymentRequest`, `worker`), so one handler serves all of them — switch on `event`.

**Invoicing:** `invoiceCreated`, `invoicePaid` (both wrap under `invoice`), `creditNoteCreated` (same shape, wrapped under `creditNote`).

**Do not subscribe to `hireAccepted`.** It is subscribable and **never sent** — nothing in the platform emits it. When a worker accepts, the handler sends `contractAccepted` instead. Worksome documents this explicitly.

## 4. Do not infer state from which event arrived

`contractAccepted`, `hireUpdated` and `hireEnded` can fire around the same moment, because accepting a contract also changes the hire. Read `data.contract.hireStatus` instead.

Values: `draft`, `offered`, `ready`, `active`, `ended`, `cancelled`, `terminated`. A hire the worker has accepted is `ready` until its start date passes, and only then `active`.

The webhook carries these **lowercase**; the GraphQL `HireActiveStatus` enum returns them **uppercase**. Compare case-insensitively when matching payloads against API responses.

Key your records on `data.contract.hireId`, not `contract.id`. Revising contract terms creates a new contract with a new id; `hireId` is stable across revisions and is the real engagement reference.

## 5. Make handlers idempotent

This is required, not advisory. Delivery is at-least-once with up to 5 retries, and overlapping lifecycle events describe the same state change more than once. Worksome's instruction: "Use the entity IDs in the payload to detect duplicates."

Keep a processed-event table keyed on `(event, hireId or paymentRequest id, hireStatus)` and no-op on a repeat.

## 6. Respond correctly

- Return any **2XX**. Worksome does not need a body on success.
- **Return 200 even for event types you do not recognise.** This stops Worksome retrying events you do not handle, and lets Worksome add new event types without breaking you.
- On failure, return a non-2XX **with a JSON body containing an error message** — Worksome's support team reads it.
- Respond within **60 seconds** or the delivery counts as failed. Acknowledge first, process asynchronously.

Retry schedule on failure or timeout: 10s, then 100s, then 1000s, then 10000s — 5 attempts total, roughly a 3.1-hour window. After the fifth, Worksome stops.

## 7. Recover what you missed

Every attempt is logged and is queryable:

```graphql
query WebhookHealth($webhookId: ID!) {
  webhookEvents(webhookId: $webhookId, first: 50) {
    data { id }
    paginatorInfo { total hasMorePages }
  }
}
```

`webhookEventLogs` exposes the individual delivery attempts. To force a redelivery:

```graphql
mutation RetryWebhookEvent($input: RetryWebhookEventInput!) {
  retryWebhookEvent(input: $input) { id }
}
```

Only companies can retry webhook events. If a whole run was lost beyond the retry window, contact Worksome customer service — they can restart sending for a specific event.

## Reconciliation

Because payloads are minimal and `trustedContactUpdated` does not say what changed, treat webhooks as **invalidation signals**, not as a data feed. The reliable pattern is: receive event → verify signature → return 200 → enqueue → re-query the GraphQL API with the ids for current truth.
