> **AI-GENERATED — review before relying on this page.**

# Expense Service

## Purpose

The Expense Service is a REST API for submitting and tracking employee expenses. It enforces per-category monthly spending caps and routes high-value expenses (> $500) to the [expense-approval-service](https://github.com/kapsha/expense-approval-service) for manager sign-off before recording.

## Key Business Rules

- **Notes required for large expenses:** Any expense with `amount > $500` must include a non-empty `notes` field; the request is rejected at validation with HTTP 422 otherwise.
- **Monthly category caps:** Each expense category has a hard monthly spending ceiling. Submitting or updating an expense that would push the running monthly total over the cap returns HTTP 400 (`CapExceededError`). Caps are:

  | Category | Monthly Cap |
  |---|---|
  | `food` | $450.00 |
  | `travel` | $1,350.00 |
  | `utilities` | $900.00 |
  | `entertainment` | $270.00 |
  | `other` | $900.00 |

- Cap enforcement is evaluated against the calendar month of the expense's `date` field, not the submission date.
- Updates (PUT) exclude the expense being updated from the cap calculation to avoid false positives on re-submission.

## API Surface

Base path: `/`  
Interactive docs: `http://localhost:8000/docs`

| Method | Path | Description | Success Status |
|---|---|---|---|
| GET | `/health` | Liveness check | 200 |
| POST | `/expenses/` | Submit a new expense | 201 |
| GET | `/expenses/` | List all expenses (filterable by category) | 200 |
| GET | `/expenses/{id}` | Retrieve a single expense | 200 |
| PUT | `/expenses/{id}` | Full update of an expense | 200 |
| DELETE | `/expenses/{id}` | Delete an expense | 204 |

> **Note:** The README documents `PATCH /{id}` but the implementation uses `PUT /{id}` (full replacement, all fields required).

### Request / Response Shapes

**POST /expenses/** — `ExpenseCreate`
```json
{
  "title": "Team lunch",          // string, max 100 chars, required
  "amount": 45.50,                // float, > 0, required
  "category": "food",             // enum, required (see categories above)
  "date": "2024-06-15",           // ISO date, optional (defaults to today)
  "notes": "Optional context"     // string | null; required when amount > $500
}
```

**PUT /expenses/{id}** — `ExpenseUpdate`  
Same fields as `ExpenseCreate` but `date` is required (no default).

**ExpenseResponse** (all read endpoints)
```json
{
  "id": 1,
  "title": "Team lunch",
  "amount": 45.50,
  "category": "food",
  "date": "2024-06-15",
  "notes": null
}
```

**GET /expenses/?category=food** — optional `category` query parameter filters the list to a single category.

**Error responses:**
- `400` — monthly cap would be exceeded (message includes cap, current total, attempted amount).
- `404` — expense ID not found.
- `422` — request validation failure (e.g., missing notes on a >$500 expense, invalid category).

**GET /health** response:
```json
{ "status": "ok" }
```

## Data Model

Data is stored entirely **in-memory** in a module-level dictionary (`_store: dict[int, dict]`) with an auto-incrementing integer ID. There is no database; all data is lost on restart. SQLAlchemy models exist in `src/models/` but are not yet wired up. The effective schema per expense record is:

| Field | Type | Constraints |
|---|---|---|
| `id` | `int` | Auto-assigned, immutable |
| `title` | `str` | Max 100 characters |
| `amount` | `float` | > 0 |
| `category` | `Category` (enum) | One of: food, travel, utilities, entertainment, other |
| `date` | `datetime.date` | ISO 8601 date |
| `notes` | `str \| None` | Required when amount > $500 |

## Dependencies on Other Services

| Service | Role | Trigger |
|---|---|---|
| [expense-approval-service](https://github.com/kapsha/expense-approval-service) | Manager sign-off for high-value expenses | `amount > $500` |

> **Warning:** The approval-service integration is described in the README but is **not implemented** in the current source code. Expenses over $500 are stored directly after passing local validation. Treat this as a planned dependency.

## Notable Constraints and Limits

- **No persistence:** In-memory store only. A service restart clears all expense records.
- **No authentication or authorization:** All endpoints are publicly accessible; there is no user identity or role enforcement.
- **Monthly cap scope:** Caps are per-category per-calendar-month and are computed at request time from the in-memory store. They reset implicitly when the month rolls over (no active reset mechanism).
- **Approval-service integration is unimplemented:** Despite the README stating high-value expenses are routed for approval, the code stores them immediately. Engineers should not rely on any approval workflow being enforced.
- **Full replacement on update:** The `PUT /{id}` endpoint replaces the entire record; partial updates are not supported despite the README listing `PATCH`.