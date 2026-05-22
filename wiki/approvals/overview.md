# Expense Approval Service

> **AI-GENERATED — review before relying on this page.**

## Purpose

The Expense Approval Service acts as a gating layer between expense submitters and the downstream [expense-service](../expenses/overview.md). Any expense exceeding $500 must pass through a manager approval workflow before it is recorded. Expenses at or below $500 are out of scope and should be submitted directly to the expense tracker.

## Key Business Rules

- **Approval threshold:** Only expenses with `amount > $500` are accepted. Submitting an amount ≤ $500 returns HTTP 400.
- **State machine:** An approval request starts as `PENDING`. A decision moves it to either `APPROVED` or `REJECTED`. Once a decision is recorded the status is final — subsequent calls to `/decision` on the same record are no-ops (the existing record is returned unchanged).
- **Downstream write on approval:** When a request is approved, the service immediately calls `POST /expenses/` on the expense tracker. If the expense tracker returns a non-2xx response, the decision endpoint returns HTTP 502 and the local record is **not** updated (the approval remains in its pre-decision state in memory, though the reviewer/reason fields will have been written — see constraints).
- **Expense linking:** On successful approval, the `expense_id` returned by the expense tracker is stored on the approval record.

## API Surface

### `GET /health/`
Returns a static liveness payload. No authentication required.

```json
{ "status": "ok", "service": "expense-approval-service" }
```

---

### `POST /approvals/` — Submit for approval
**Request body**

| Field | Type | Required | Constraints |
|---|---|---|---|
| `title` | string | ✓ | max 100 chars |
| `amount` | float | ✓ | > 0 and must be > 500 |
| `category` | enum | ✓ | `food`, `travel`, `utilities`, `entertainment`, `other` |
| `date` | date | — | defaults to today (`YYYY-MM-DD`) |
| `notes` | string | — | optional |
| `submitted_by` | string | ✓ | max 100 chars |

**Response** `201 Created` → `ApprovalResponse` (see [Data Model](#data-model))

---

### `GET /approvals/` — List all approvals
Returns an array of all `ApprovalResponse` objects held in memory.

---

### `GET /approvals/{id}` — Get single approval
Returns one `ApprovalResponse` or `404` if the ID is unknown.

---

### `POST /approvals/{id}/decision` — Record a decision
**Request body**

| Field | Type | Required | Constraints |
|---|---|---|---|
| `approved` | bool | ✓ | `true` to approve, `false` to reject |
| `reviewer` | string | ✓ | max 100 chars |
| `reason` | string | — | optional free text |

**Response** `200 OK` → updated `ApprovalResponse`, or `404` / `502` on failure.

## Data Model

All state is stored in an **in-process Python dictionary** (no database). Data is lost on restart.

### `ApprovalResponse`

| Field | Type | Notes |
|---|---|---|
| `id` | int | Auto-incrementing, service-local |
| `title` | string | — |
| `amount` | float | — |
| `category` | `Category` enum | — |
| `date` | date | — |
| `notes` | string \| null | — |
| `submitted_by` | string | — |
| `status` | `ApprovalStatus` enum | `pending` → `approved` or `rejected` |
| `reviewer` | string \| null | Set on decision |
| `reason` | string \| null | Set on decision |
| `expense_id` | int \| null | Set on successful approval; ID from expense tracker |

## Dependencies

| Dependency | Interface | Purpose |
|---|---|---|
| [expense-service](../expenses/overview.md) | `POST /expenses/` (HTTP) | Creates the expense record when an approval is granted |

The base URL is configured via the `EXPENSE_TRACKER_URL` environment variable. The HTTP client uses a 10-second timeout. If the expense tracker is unreachable or returns an error, the decision endpoint surfaces a `502` to the caller.

## Constraints and Limits

- **No persistence:** The approval store is an in-memory dict. A process restart wipes all pending and historical approvals.
- **No authentication/authorization:** Any caller can submit expenses or record decisions. There is no validation that `reviewer` is a legitimate manager.
- **Idempotency:** Re-submitting an identical expense creates a new approval record with a new ID; there is no deduplication.
- **Single instance only:** Because state is in-process memory, running multiple instances will result in split state and inconsistent responses.
- **Threshold is hardcoded:** `APPROVAL_THRESHOLD = 500.0` is a module-level constant; changing it requires a code deployment.