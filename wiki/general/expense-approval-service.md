# Expense Approval Service

> **AI-GENERATED — review before relying on this page.**

## Purpose

The Expense Approval Service gates high-value expenses behind a human review step before they are recorded in the [expense tracker](https://github.com/kapsha/pr-test-agent-learn-personal). Any expense submission exceeding $500 must pass through this service; a manager explicitly approves or rejects it, and only approved expenses are forwarded to the expense tracker for final recording.

## Key Business Rules

- Expenses with `amount > $500` (exclusive — amounts of exactly $500 are **not** routed here) require approval. Submissions at or below the threshold are rejected with HTTP 400, instructing the caller to submit directly to the expense tracker.
- An approval record is created in `PENDING` status. Once a decision is posted, the status transitions to either `APPROVED` or `REJECTED` and does not change again (subsequent decisions on an already-decided record are silently ignored and the existing record is returned).
- On approval, the service synchronously calls the expense tracker to create the expense and stores the resulting `expense_id` on the approval record. If the expense tracker returns an error, the caller receives HTTP 502.

## API Surface

### Health

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health/` | Returns `{"status": "ok", "service": "expense-approval-service"}` |

### Approvals

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/approvals/` | Submit an expense for approval (201 Created) |
| `GET` | `/approvals/` | List all approval records |
| `GET` | `/approvals/{id}` | Fetch a single approval record |
| `POST` | `/approvals/{id}/decision` | Record an approve/reject decision |

**Submit for approval — request body (`ApprovalRequest`)**

```json
{
  "title": "Team offsite flights",      // string, max 100 chars, required
  "amount": 1200.00,                    // float, > 0, required
  "category": "travel",                 // enum: food | travel | utilities | entertainment | other
  "date": "2024-06-15",                 // ISO date, defaults to today
  "notes": "Q3 planning offsite",       // string | null
  "submitted_by": "alice@example.com"   // string, max 100 chars, required
}
```

**Record a decision — request body (`ApprovalDecision`)**

```json
{
  "approved": true,                     // bool, required
  "reviewer": "bob@example.com",        // string, max 100 chars, required
  "reason": "Within budget allocation"  // string | null
}
```

**Response shape (`ApprovalResponse`)** — returned by all approval endpoints

```json
{
  "id": 1,
  "title": "Team offsite flights",
  "amount": 1200.00,
  "category": "travel",
  "date": "2024-06-15",
  "notes": "Q3 planning offsite",
  "submitted_by": "alice@example.com",
  "status": "pending",                  // pending | approved | rejected
  "reviewer": null,                     // populated after decision
  "reason": null,                       // populated after decision
  "expense_id": null                    // populated on approval, ID from expense tracker
}
```

## Data Model

Approval records are stored entirely **in-process memory** (`_store: dict[int, ApprovalResponse]`) with a monotonically incrementing integer ID (`_next_id`). There is no database or persistent storage. All records are lost on process restart. The schema fields are defined in `src/schemas/approval.py` using Pydantic models.

## Dependencies

| Dependency | Type | Details |
|---|---|---|
| **Expense Tracker Service** | Runtime HTTP | Called synchronously via `POST /expenses/` when an expense is approved. Base URL configured via `EXPENSE_TRACKER_URL` env var (default: `http://localhost:8000`). HTTP timeout: 10 seconds. |

## Notable Constraints & Limits

- **No persistence** — in-memory store only; a service restart clears all pending approvals.
- **Synchronous expense creation** — approval decisions block on the expense tracker HTTP call; a slow or unavailable expense tracker will delay responses and surface as HTTP 502 to the caller.
- **Single-instance only** — the in-memory store is not shared across multiple process instances; horizontal scaling is not supported without adding an external data store.
- **No authentication or authorization** — any caller can submit expenses or record decisions. The `reviewer` and `submitted_by` fields are free-text strings with no identity verification.
- Field length limits: `title` and `submitted_by` max 100 characters; `reviewer` max 100 characters.