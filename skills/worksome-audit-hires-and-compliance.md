---
name: worksome-audit-hires-and-compliance
description: Read-only audit of a company's external workforce in Worksome — walk hires, contracts, classifications, compliance gates, payment requests and invoices with correct pagination and query-complexity discipline.
api: Worksome GraphQL API
endpoint: https://api.worksome.com/graphql
operations:
  - accounts
  - hires
  - hire
  - contracts
  - classifications
  - compliance
  - paymentRequests
  - invoices
  - projects
generated: '2026-09-04'
method: generated
source: graphql/worksome-introspection.json + https://docs.worksome.com/graphql/guides/pagination/ + https://docs.worksome.com/graphql/guides/rate-limiting/
---

# Audit hires and compliance

A read-only sweep. Nothing here mutates, so it is the safe way for an agent to get oriented in a Worksome account. The constraint is not permissions — it is **query complexity and the 60 requests/minute budget**.

## Pagination

Worksome uses Lighthouse offset pagination, not Relay cursors.

- Request `first` (page size) and `page` (1-based).
- Read items from `data` and metadata from `paginatorInfo`.
- `paginatorInfo` gives `currentPage`, `lastPage`, `total`, `count`, `hasMorePages`, `perPage`, `firstItem`, `lastItem`.
- Stop when `hasMorePages` is `false`.

Use `first: 25`, not `first: 100`. Worksome's own guidance is explicit, and page size feeds directly into the complexity budget.

## Query complexity — the real limit

Each field has a cost. Deeply nested queries and large related collections consume budget fast, and **a query over the maximum is rejected before execution**. The budget is not published, so you cannot compute it — you have to stay well inside it.

Rules that work:

1. Select only the fields you need. Never take every field on a type.
2. Do not chain collections. `company → workers → contracts → invoices` in one query is the documented anti-pattern.
3. Fan out across several shallow queries rather than one deep one.

## Step 1: accounts

```graphql
{
  viewer { id name email }
  accounts { id name ... on Company { market } }
}
```

## Step 2: hires, shallow

```graphql
query AuditHires($company: ID!, $page: Int!) {
  hires(accounts: [$company], status: [ACTIVE, READY], first: 25, page: $page) {
    data {
      id
      activeStatus
      worker { id name }
      latestContract { id startDate endDate rate currency rateType status }
    }
    paginatorInfo { currentPage lastPage total hasMorePages }
  }
}
```

`hires` also accepts `search`, `workers`, `recruiters`, `companies` and date-range filters. Filter server-side; do not page the whole set and filter locally.

## Step 3: classification, per hire

Classification is what Worksome actually sells — IR35 in the UK, 1099/W-2 in the US, and local rules across 150+ countries. It is a separate query keyed on a hire.

```graphql
query HireClassification($hire: ID!) {
  classifications(hire: $hire, first: 10) {
    data { id status result { __typename } }
    paginatorInfo { total }
  }
}
```

Do **not** read `Hire.classificationResult`, `Hire.classificationLabel`, `Hire.classificationPdfUrl` or `Hire.wcr` — all four are deprecated, with `wcr` pointing at `classification` and the other three flagged "Decoupling classification from hire in future versions."

In `ClassificationResult`, prefer `EMPLOYEE` and `INDEPENDENT`. `LIKELY_EMPLOYEE` and `LIKELY_INDEPENDENT` are deprecated in favour of the definite forms.

## Step 4: compliance gates

```graphql
query HireCompliance($hire: ID!) {
  compliance(id: $hire) { id action { __typename } }
}
```

`compliance(id: ID!, names: [ComplianceName!], cached: Boolean)` — pass `names` to narrow to specific checks (the enum includes `IR35`, `IR35_COMPANY_SETTINGS`, `GLOBAL_CONTRACT_TYPE_HIRE` and others), and `cached: true` where a slightly stale answer is acceptable, to save budget.

Note `ComplianceName.COMMON_BUSINESS_ENTITY` is deprecated — use `COMPANY_COMMON_BUSINESS_ENTITY` or `FREELANCER_COMMON_BUSINESS_ENTITY`.

## Step 5: the money trail

```graphql
query AuditSpend($company: ID!, $page: Int!) {
  paymentRequests(accounts: [$company], first: 25, page: $page) {
    data { id worker { id name } hire { id } invoice { id } }
    paginatorInfo { currentPage lastPage total hasMorePages }
  }
}
```

```graphql
query AuditInvoices($company: ID!, $page: Int!) {
  invoices(accounts: [$company], first: 25, page: $page) {
    data { id originalInvoice { id } }
    paginatorInfo { currentPage lastPage total hasMorePages }
  }
}
```

Credit notes are `Invoice` records reachable through `Invoice.creditNotes`, with `originalInvoice` pointing back — there is no separate `CreditNote` type in the schema, even though `creditNoteCreated` is a webhook event.

Chain: `Job → JobCandidate → Bid → Hire → Contract → Timesheet → PaymentRequest → Invoice`. See `data-model/worksome-data-model.yml`.

## Fee fields

Read `fees`, not `recruiterFee`. `Hire.recruiterFee` and `CompanyRecruiter.recruiterFee` were deprecated on 2026-04-22 because they only support percentage-based fees and return `null` for hourly, daily, weekly or monthly bases — so a `null` there is ambiguous between "no fee" and "a fee this field cannot express". `fees` returns the full set and supports every basis.

## Rate limiting during a sweep

An audit is exactly the workload that trips the limit: 60 requests/minute per token, default tier.

You will get **no header telling you where you stand**. The gateway does not propagate `X-RateLimit-*` and drops the platform's `Retry-After`. Throttling reaches you as an HTTP **200** carrying `extensions.code: DOWNSTREAM_SERVICE_ERROR` with the message `"Too Many Requests"`.

So:

- Count your own calls in a rolling 60-second window and slow down at ~55.
- Spread a batch job evenly rather than bursting.
- On a throttle, back off exponentially with jitter (1s, 2s, 4s).
- Cache anything that does not change often. Prefer a webhook over a poll.

## Partial results are normal

Field-level authorization means a field you may not read comes back `null` inside `data` with a matching entry in `errors[]` carrying its `path`. That is a successful response with a hole in it, not a failure. Record the hole and carry on — do not abort the sweep.
