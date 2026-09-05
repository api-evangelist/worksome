---
name: worksome-push-external-timesheets
description: Push timesheet registrations captured in an external time-tracking system into Worksome with the createCustomTimesheet mutation, validating payloads against Worksome's published JSON Schema and handling partial-success rejections.
api: Worksome GraphQL API
endpoint: https://api.worksome.com/graphql
operations:
  - hires
  - createCustomTimesheet
  - timesheets
  - paymentRequests
generated: '2026-09-04'
method: generated
source: https://docs.worksome.com/integrations/timesheet-integration/ + https://docs.worksome.com/schemas/timesheet-registration.json + graphql/worksome-introspection.json
---

# Push external timesheets into Worksome

For clients whose workers register time in an external system. Worksome turns submitted registrations into timesheets, auto-approves the resulting payment requests, pays the worker and invoices the client. **Money moves at the end of this flow** — treat it accordingly.

## What you need first

This is not a standalone integration. To match your time data to the right engagement you need, from Worksome:

- the **Global ID of each hire** (`hireId`), and
- its **contract period** (`startDate`, `endDate`).

Get them from the API and cache them:

```graphql
query HireContext($company: ID!) {
  hires(accounts: [$company], status: [ACTIVE, READY], first: 100) {
    data {
      id
      worker { id name }
      latestContract { id startDate endDate }
    }
    paginatorInfo { hasMorePages currentPage lastPage }
  }
}
```

Keep these fresh with the `hireUpdated`, `hireEnded` and `hireTerminated` webhooks. A contract whose dates changed will start rejecting registrations.

## Validate before you send

Worksome publishes a real JSON Schema (draft 2020-12) for the payload:

`https://docs.worksome.com/schemas/timesheet-registration.json` — also in this repo at `json-schema/worksome-timesheet-registration.json`.

Use it to validate outgoing payloads and to generate types. It accepts either one registration object or an array of them, and sets `additionalProperties: false`, so an unexpected key is a schema failure, not a silently ignored field.

A registration:

| Field | Type | Required | Notes |
|---|---|---|---|
| `hireId` | string | yes | Worksome Global ID of the hire. |
| `reportedDate` | string | yes | `YYYY-MM-DD`, enforced by `format: date` and a pattern. |
| `hours` | number | yes | Minimum 0. |
| `externalId` | string | yes | **Your** unique id. This is the upsert key — see below. |
| `reference` | string | no | Free text: project code, cost centre, PO number. |
| `isPayable` | boolean | no | Defaults to `true`. |
| `meta` | object | no | Arbitrary key/values, `additionalProperties: true`. |

## The one idempotent write in this API

`externalId` is documented as "used as the key for updates — sending the same `externalId` replaces the existing registration for that date."

That makes this the **only replay-safe mutation in the Worksome API**. Resubmitting the same payload converges instead of duplicating. Nothing else in the 113-mutation surface has an equivalent — there is no `Idempotency-Key` header anywhere. Use a stable, deterministic `externalId` derived from your own record, never a random one per attempt.

## Submit

```graphql
mutation CreateCustomTimesheet($input: CreateCustomTimesheetInput!) {
  createCustomTimesheet(input: $input) {
    providedRegistrations
    successfulRegistrations
    rejectedRegistrations { externalId reason message }
  }
}
```

`CreateCustomTimesheetInput` has exactly two fields, both `String!`:

- `schema` — must be the literal `"default-json"`.
- `data` — a **JSON string** containing the registration object or array. It is a string, not a JSON value: serialise your payload and pass it as text.

**Batch aggressively.** Worksome's guidance is explicit: submit many registrations per call rather than one call per registration or per worker. Large batches are easier to debug, monitor, re-send and reprocess. Thousands of small mutations also burn the 60 requests/minute per-token budget immediately.

## Handle partial success

This mutation does **not** use the GraphQL `errors` array for row failures. You get a 200 with a business-level body, and a single payload can produce both accepted and rejected rows. Always read `rejectedRegistrations`, even when `successfulRegistrations` is greater than zero.

Each rejected row carries one machine-readable `reason` (registrations stop at their first failure, so there is exactly one per row):

| `reason` | Meaning | What to do |
|---|---|---|
| `MISSING_REQUIRED_FIELD` | A required field is missing, empty, or unparseable (including bad dates). | Fix the payload; `message` names the field. Validating against the JSON Schema first prevents most of these. |
| `HIRE_NOT_FOUND` | The `hireId` does not resolve to a hire you may submit for. | Deliberately conflates "does not exist" with "you have no access" so access scopes are not leaked. Re-check the id against your cached hire context. |
| `DATE_OUTSIDE_CONTRACT_PERIOD` | `reportedDate` is before `startDate` or after `endDate`. | Correct the date, or stop submitting once the hire has ended. |

**US-payroll exception:** `DATE_OUTSIDE_CONTRACT_PERIOD` is not raised for hires on a US payroll scheme — contract dates can move after submission, and Worksome reconciles it server-side. Do not filter those client-side.

## After submission

```graphql
query CheckTimesheets($company: ID!) {
  timesheets(accounts: [$company], first: 25) {
    data { id hire { id } registrations(first: 50) { paginatorInfo { total } } }
    paginatorInfo { hasMorePages }
  }
}
```

Payment requests created by this integration are **auto-approved by default** — the worker gets paid and the client gets invoiced with no manual review inside Worksome. Watch `paymentRequestIssued`, `paymentRequestApproved`, `paymentRequestPaid` and `paymentRequestWorkerPaidOut`.

## Reversal

There is effectively no clean undo:

- Correction is by **resubmitting the same `externalId`**, which replaces the registration in place. This is the intended path.
- `deleteTimesheetRegistration` exists but the schema says **only workers** can call it. Your integration credential cannot.
- `approvePaymentRequest` has **no reversal operation at all**, and these are auto-approved.

Get the payload right before you send it. Validate against the JSON Schema, and rehearse in the sandbox — noting that sandbox processing of timesheets into payment requests requires coordination with your Worksome customer success manager and is not automatic there.
