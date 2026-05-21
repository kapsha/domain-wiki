> **AI-GENERATED — review before relying on this page.**

# Expense Approval Service

## Purpose

The Expense Approval Service enforces a mandatory human-review gate for high-value expenses. Any expense exceeding **$500** must pass through this service before being recorded in the downstream [expense-service](https://github.com/kapsha/expense-service). Expenses at or below the threshold must be submitted directly to the expense tracker and are rejected by this service with a `400` error.

## Key Business Rules

- The approval threshold is **strictly greater than $500**; an amount of exactly $500 is not routed for approval.
- A newly submitted request always enters the **`pending`** state.
- Once a decision has been recorded (`approved` or `rejected`), subsequent calls to the decision endpoint for the same request are silently ignored — the existing status is returned unchanged.
- On **approval**, the service synchronously creates an expense record in the expense tracker via `POST /expenses/`. If the expense tracker returns a non-2xx response, the decision endpoint returns `502` and the approval record is **not** updated.
- On **rejection**, no call is made to the expense tracker; the request is marked `rejected` and `expense_id` remains `null`.

## API Surface

### `GET /health/`
Returns a simple liveness check.

**Response `200`:**
```json
{ "status": "ok", "service": "expense-approval-service" }
```

---

### `POST /approvals/`
Submit an expense for manager approval. Rejects with `400` if `amount <= 500`.

**Request body:**
| Field | Type | Constraints |
|---|---|---|
| `title` | string | required, max 100 chars |
| `amount` | float | required, > 0 |
| `category` | enum | `food`, `travel`, `utilities`, `entertainment`, `other` |
| `date` | date (ISO 8601) | defaults to today |
| `notes` | string \| null | optional |
| `submitted_by` | string | required, max 100 chars |

**Response `201`:** `ApprovalResponse` (see [Data Model](#data-model))

---

### `GET /approvals/`
Returns all approval requests as a list of `ApprovalResponse` objects.

---

### `GET /approvals/{id}`
Returns a single `ApprovalResponse`. Returns `404` if not found.

---

### `POST /approvals/{id}/decision`
Record an approval or rejection decision.

**Request body:**
| Field | Type | Constraints |
|---|---|---|
| `approved` | bool | required |
| `reviewer` | string | required, max 100 chars |
| `reason` | string \| null | optional |

**Response `200`:** Updated `ApprovalResponse`.  
**Error `404`:** Request not found.  
**Error `502`:** Expense tracker rejected the forwarded expense (approval not committed).

## Data Model

The `ApprovalResponse` schema represents an approval request record:

| Field | Type | Notes |
|---|---|---|
| `id` | int | Auto-assigned, sequential |
| `title` | string | |
| `amount` | float | |
| `category` | `Category` enum | |
| `date` | date | |
| `notes` | string \| null | |
| `submitted_by` | string | |
| `status` | `ApprovalStatus` enum | `pending`, `approved`, `rejected` |
| `reviewer` | string \| null | Set on decision |
| `reason` | string \| null | Set on decision |
| `expense_id` | int \| null | ID returned by expense tracker on approval |

**Storage note:** Records are held in an in-process Python dictionary. **All data is lost on process restart** — there is no persistent database.

## Dependencies

| Service | Interaction | Configuration |
|---|---|---|
| [expense-service](https://github.com/kapsha/expense-service) | `POST /expenses/` — called synchronously when an expense is approved | Base URL configured via `EXPENSE_TRACKER_URL` env var |

The HTTP client uses a 10-second timeout for calls to the expense tracker.

## Notable Constraints and Limits

- **No persistence:** The in-memory store means approval history does not survive restarts or horizontal scaling. Multiple instances will have isolated, inconsistent state.
- **No authentication or authorization:** Any caller can submit requests or record decisions. Access control must be enforced externally.
- **Sequential integer IDs** are generated from a module-level counter; they reset to `1` on restart and are not safe across multiple instances.
- The `category` field is a closed enum — submissions with unlisted categories will fail Pydantic validation before reaching business logic.