> **AI-GENERATED — review before relying on this page.**

# Expense Service

## Purpose

The Expense Service is a REST API for submitting and tracking employee expenses. It enforces per-category monthly spending caps and routes high-value expenses (> $1,000) to the [expense-approval-service](https://github.com/kapsha/expense-approval-service) for manager sign-off before recording.

---

## Key Business Rules

1. **High-value expenses require notes.** Any expense with `amount > $1,000` must include a non-blank `notes` field; the request is rejected at validation time (HTTP 422) if this is missing.
2. **Monthly category caps.** Each category has a hard monthly cap. A create or update that would push the running monthly total for that category over its cap is rejected with HTTP 400 (`CapExceededError`). Caps are:

   | Category      | Monthly Cap |
   |---------------|-------------|
   | Food          | $450.00     |
   | Travel        | $1,350.00   |
   | Utilities     | $900.00     |
   | Entertainment | $270.00     |
   | Other         | $900.00     |

3. **Cap enforcement on update.** When updating an existing expense, the record's own current amount is excluded from the running total before checking the cap, so an in-place edit does not double-count the original value.

---

## API Surface

All expense endpoints are prefixed with `/expenses`.

### `GET /health`
Returns service liveness.

**Response `200`**
```json
{ "status": "ok" }
```

---

### `POST /expenses/`
Submit a new expense.

**Request body**
```json
{
  "title":    "string (max 100 chars, required)",
  "amount":   "float (> 0, required)",
  "category": "food | travel | utilities | entertainment | other",
  "date":     "YYYY-MM-DD (defaults to today)",
  "notes":    "string | null (required when amount > $1,000)"
}
```

**Response `201`** — `ExpenseResponse` (see Data Model)  
**Response `400`** — monthly cap would be exceeded  
**Response `422`** — validation failure (e.g. missing notes for large expense)

---

### `GET /expenses/`
List all expenses, with optional filters. Multiple filters combine (logical AND).

**Query parameters**

| Parameter   | Type   | Description                                                                 |
|-------------|--------|-----------------------------------------------------------------------------|
| `category`  | string | Optional. One of the `Category` enum values.                                |
| `date_from` | date   | Optional. ISO 8601 date (e.g. `2026-01-01`). Returns only expenses where `expense.date >= date_from`. Inclusive. |

**Response `200`** — array of `ExpenseResponse`

---

### `GET /expenses/{id}`
Retrieve a single expense by integer ID.

**Response `200`** — `ExpenseResponse`  
**Response `404`** — expense not found

---

### `PUT /expenses/{id}`
Full replacement update of an existing expense. Accepts the same fields as `POST` (all required, no optional `date` default).

**Response `200`** — updated `ExpenseResponse`  
**Response `400`** — monthly cap would be exceeded  
**Response `404`** — expense not found  
**Response `422`** — validation failure

> **Note:** The README documents this endpoint as `PATCH`, but the implementation registers it as `PUT` (full update, all fields required).

---

### `DELETE /expenses/{id}`
Remove an expense record.

**Response `204`** — deleted  
**Response `404`** — expense not found

---

## Data Model

All persistence is currently **in-memory** (a module-level dictionary `_store`). There is no database backend; data does not survive a process restart. SQLAlchemy models exist in `src/models/` but are not yet wired up.

### `ExpenseResponse`

| Field      | Type           | Notes                            |
|------------|----------------|----------------------------------|
| `id`       | `int`          | Auto-incrementing integer key    |
| `title`    | `string`       | Max 100 characters               |
| `amount`   | `float`        | Positive, in USD                 |
| `category` | `Category`     | Enum — see Business Rules        |
| `date`     | `date`         | ISO 8601 (`YYYY-MM-DD`)          |
| `notes`    | `string\|null` | Mandatory when amount > $1,000   |

---

## Dependencies

| Dependency | Purpose |
|---|---|
| [expense-approval-service](https://github.com/kapsha/expense-approval-service) | Receives high-value expenses (> $1,000) for manager sign-off before they are recorded |

The external approval service call is referenced in the README but the integration code is not yet visible in the source; treat this as a planned or in-progress dependency.

---

## Notable Constraints & Limits

- **No persistence.** The in-memory store is reset on every process restart. All stored expenses are lost when the service stops.
- **Monthly caps are hardcoded.** Cap values are defined as constants in `src/services/expense_service.py` and cannot be changed at runtime or via configuration.
- **IDs are ephemeral.** The auto-increment counter resets with the process, so IDs from a previous run will conflict or be reused.
- **No authentication or authorisation.** All endpoints are publicly accessible as implemented.
- **Single-process only.** Because state lives in a module-level dict, running multiple worker processes would result in inconsistent state across requests.