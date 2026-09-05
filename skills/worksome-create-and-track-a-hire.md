---
name: worksome-create-and-track-a-hire
description: Create a draft hire for an existing trusted contact in Worksome, then follow it through contract acceptance to an active engagement using webhooks and the hires query.
api: Worksome GraphQL API
endpoint: https://api.worksome.com/graphql
operations:
  - accounts
  - trustedContacts
  - createDraftHire
  - hire
  - hires
  - cancelHire
  - terminateHire
generated: '2026-09-04'
method: generated
source: graphql/worksome-introspection.json + https://docs.worksome.com/graphql/ + https://docs.worksome.com/webhooks/reference/
---

# Create and track a hire

Every operation named here was read from the live introspected schema. Do not invent field names — if something is missing, introspect `https://api.worksome.com/graphql` and check.

## Before you start

- Authenticate with `Authorization: Bearer {token}` (Personal Access Token or OAuth access token). See `authentication/worksome-authentication.yml`.
- Send `Content-Type: application/json` on every POST. Without it the Apollo gateway rejects the request as CSRF with `extensions.code: BAD_REQUEST` — the only error you will see as a real HTTP 400.
- Every other failure arrives over **HTTP 200**. Always parse `errors[]` even when `data` is present.

## 1. Resolve the company you are acting for

Most write operations need a `Company` id. `accounts` returns the `Account` interface, so fragment it.

```graphql
{
  viewer { id name email }
  accounts {
    id
    name
    ... on Company { market }
  }
}
```

Keep the `Company` id. The `Account` interface is also implemented by `Organisation`, `Partner`, `Recruiter` and `Worker` — passing one of those where a `Company` is expected produces `"The selected {field} is invalid."` in the validation map.

> **Docs/schema discrepancy.** The Getting Started page lists the implementations as "`Company`, `Organisation`, `Partner`, `StaffingAgency`, and `Worker`". Introspection returns `Company`, `Organisation`, `Partner`, `Recruiter`, `Worker` — there is no `StaffingAgency` type in the schema. Fragment on `... on Recruiter`, not `... on StaffingAgency`, or the query fails schema validation with `GRAPHQL_VALIDATION_FAILED`.

## 2. Find the trusted contact

A worker cannot be hired unless they are already a trusted contact of the hiring company. Workers have no top-level query; reach them through `trustedContacts`.

```graphql
query FindContact($company: ID!, $search: String) {
  trustedContacts(accounts: [$company], search: $search, first: 25) {
    data { id status worker { id name email } }
    paginatorInfo { total hasMorePages }
  }
}
```

If nobody comes back, create the relationship first with `createTrustedContact` and wait for `trustedContactUpdated`. Do not attempt the hire.

## 3. Create the draft hire

Use `createDraftHire`. The `hire` mutation still exists but is deprecated — the schema says so and the changelog dated 2026-01-15 says so.

```graphql
mutation CreateDraftHire($input: HireInput!) {
  createDraftHire(input: $input) { id activeStatus }
}
```

`HireInput` fields, from the schema. Only `company` is non-null, but a hire without `trustedContact`, dates and a rate will fail business validation:

| Field | Type | Notes |
|---|---|---|
| `company` | `ID!` | The only required field. Company id from step 1. |
| `trustedContact` | `ID` | The contact from step 2. |
| `job` | `ID` | Optional. If omitted, a job is created automatically from the details you pass. |
| `name` / `description` / `hireDescription` | `String` | |
| `rateType` | `RateType` | e.g. `FIXED`, `HOURLY`. |
| `rate` | `Float` | Must be numeric or validation returns "The rate must be a number." |
| `startDate` / `endDate` | `Date` | `YYYY-MM-DD`. Anything else returns "The start date is not a valid date." |
| `locationPreference` | `LocationPreferenceInput` | e.g. `{preference: REMOTE_ONLY}`. |
| `includeStandardContract` | `Boolean` | |
| `purchaseOrderNumber` | `String` | |
| `externalIdentifier` | `String` | Your own id for this hire. Carry it — it comes back to you. |
| `owners` | `[ID!]` | User ids. |
| `customFieldValues` | `[CustomFieldTypeValueInput!]` | **Trap:** passing `[]` skips custom-field syncing entirely. If the company has required custom fields configured, omitting values fails validation. |

**There is no idempotency key on this mutation.** If the request times out, do not blind-retry — query `hires` filtered by your `externalIdentifier` first to see whether one was already created.

A draft hire is not live. Worksome's docs state it must be completed in the Worksome UI before it becomes active.

## 4. Track it

Poll, or better, subscribe to webhooks.

```graphql
query TrackHire($company: ID!) {
  hires(accounts: [$company], status: [OFFERED, READY, ACTIVE], first: 25) {
    data {
      id
      activeStatus
      latestContract { id startDate endDate rate currency }
      worker { id name }
    }
    paginatorInfo { currentPage lastPage total hasMorePages }
  }
}
```

Status ladder: `DRAFT` → `OFFERED` → `READY` → `ACTIVE` → `ENDED` | `CANCELLED` | `TERMINATED`. `READY` means accepted but the start date has not passed; `ACTIVE` means it has.

For the webhook path, subscribe to `contractAccepted`, not `hireAccepted` — `hireAccepted` is subscribable but **never delivered**, and Worksome documents this. See `asyncapi/worksome-webhooks.yml`.

Read `data.contract.hireStatus` from the payload rather than inferring state from which event arrived: `contractAccepted`, `hireUpdated` and `hireEnded` can all fire around the same moment. Webhook payloads carry the status lowercase (`active`); the API returns it uppercase (`ACTIVE`). Compare case-insensitively.

Key on `data.contract.hireId`, not `contract.id` — revising terms creates a new contract with a new id while `hireId` stays constant.

## 5. Reversal, and its limits

```graphql
mutation CancelHire($input: CancelHireInput!) { cancelHire(input: $input) { id activeStatus } }
mutation TerminateHire($input: TerminateHireInput!) { terminateHire(input: $input) { id activeStatus } }
```

- `CancelHireInput` = `{ hire: ID!, message: String! }`. Cancellation is for a hire that has **not yet become active** — the webhook reference defines `hireCancelled` as firing when "a hire is cancelled before it became active". No time window is published.
- `TerminateHireInput` = `{ hire: ID!, reason: ContractTerminationReason!, date: Date!, comments: String, message: String }`. This ends an active engagement early. It is a forward state change with financial consequences, **not an undo** — work already performed still flows into payment requests and invoicing, and there is no un-terminate.

Confirm any window with Worksome before relying on one. None is documented.

## Error handling

Drive control flow from `extensions.code`, never the message — except for rate limiting, where the message is the only signal.

- `extensions.validation` present → fix the named input paths, do not retry.
- `extensions.guards: ["api"]` with `"Unauthenticated."` → token missing, expired (PATs last 6 months) or revoked.
- No validation map, no guards, `"You are not authorized to perform this action."` → wrong role or wrong company scope. There is no machine-readable authorization tag; this is the documented way to tell.
- Message matches `/too many requests/i` → throttled. 60 requests/minute per token by default. **No `Retry-After` or `X-RateLimit-*` header reaches you** — the gateway strips them. Back off exponentially with jitter (1s, 2s, 4s) and count requests client-side.

Full catalogue: `errors/worksome-error-codes.yml`.
