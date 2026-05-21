> **AI-GENERATED — review before relying on this page.**

# Expense Tracker API

## Purpose

The Expense Tracker API is a small REST service that records personal expenses and enforces per-category monthly spending caps. It exists primarily as the **target application** for the `kapsha/pr-test-agent-learn-personal` project, which explores automated wiki and test generation on pull requests. The API is intentionally simple — a handful of endpoints over an in-memory store — but contains real business rules that make it a meaningful test target.

## Key Business Rules

1. **Monthly spending caps** are enforced per category at create and update time. If adding (or updating) an expense would push the category's month-to-date total past its cap, the request is rejected with HTTP 400.

   | Category | Monthly Cap |
   |---|---|
   | `food` | $500.00 |
   | `travel` | $1,500.00 |
   | `utilities` | $1,000.00 |
   | `entertainment` | $300.00 |
   | `other` | $1,000.00 |

2. **Large expenses require a note.** Any expense with `amount > 500` must include a non-blank `notes` field; omitting it produces HTTP 422 (validation error, not a cap error).

3. **Caps are month-scoped, not rolling.** Two expenses in different calendar months never aggregate against each other. When updating, the record being replaced is excluded from the cap calculation so an in-place edit does not double-count itself.

## API Surface

Base path: `/`

### Health

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Liveness check |

**Response `200`**
```json
{ "status": "ok" }
```

---

### Expenses

| Method | Path | Description |
|---|---|---|
| `POST` | `/expenses/` | Create an expense |
| `GET` | `/expenses/` | List all expenses (optional category filter) |
| `GET` | `/expenses/{id}` | Fetch a single expense |
| `PUT` | `/expenses/{id}` | Replace an expense |
| `DELETE` | `/expenses/{id}` | Delete an expense |

#### Create — `POST /expenses/`

**Request body** (`ExpenseCreate`)

| Field | Type | Required | Constraints |
|---|---|---|---|
| `title` | `string` | ✓ | max 100 chars |
| `amount` | `float` | ✓ | > 0 |
| `category` | `Category` | ✓ | enum — see table above |
| `date` | `date` | — | defaults to today |
| `notes` | `string \| null` | — | required when `amount > 500` |

**Responses**

| Status | Condition |
|---|---|
| `201` | Created successfully; body is `ExpenseResponse` |
| `400` | Monthly cap would be exceeded (`CapExceededError`) |
| `422` | Validation failure (negative amount, invalid category, missing notes for large expense) |

#### List — `GET /expenses/`

Optional query parameter: `category=<Category>`. Returns a JSON array of `ExpenseResponse`. Returns `[]` when empty.

#### Get / Update / Delete

`PUT` accepts `ExpenseUpdate` (same fields as `ExpenseCreate`, all required — no partial update). `DELETE` returns `204 No Content`. All three return `404` when the ID does not exist; `PUT` additionally returns `400` on cap violation.

#### `ExpenseResponse` shape

```json
{
  "id": 1,
  "title": "Flight to NYC",
  "amount": 600.0,
  "category": "travel",
  "date": "2024-01-15",
  "notes": "Annual conference"
}
```

## Data Model

There is **no database** in the current implementation. State is held in a module-level in-memory dictionary (`_store: dict[int, dict]`) inside `src/services/expense_service.py`. IDs are assigned by a monotonically incrementing counter (`_next_id`). The store is wiped on process restart and can be reset programmatically via `expense_service.reset()` (used in tests).

Pydantic schemas in `src/schemas/expense.py` govern validation. SQLAlchemy models are scaffolded (`src/models/`) but unused — persistence is deferred to a later phase.

## Dependencies on Other Services

None. The API has no external runtime dependencies — no database, no cache, no third-party service calls. Development and test tooling dependencies are:

- **FastAPI / Uvicorn** — web framework and ASGI server
- **Pydantic v2** — request/response validation
- **pytest + httpx + FastAPI TestClient** — test execution
- **ruff** — linting/formatting
- **pip-audit** — dependency vulnerability scanning (run in CI)

The broader project will later add **AWS Bedrock (Claude)** for the test/wiki generation agent and **GitHub Actions** for orchestration — these are not dependencies of the API itself.

## Notable Constraints and Limits

- **In-memory store only.** All data is lost on restart. This is intentional for Phase 1; SQLite/SQLAlchemy is available in the stack for a future phase.
- **No authentication.** `src/deps.py` is an empty placeholder. Auth is deferred.
- **No partial update.** `PUT` requires all fields; there is no `PATCH` endpoint.
- **Cap enforcement is synchronous and non-atomic.** Concurrent writes to the same category/month could race past a cap; this is acceptable given the in-process, single-threaded learning context.
- **Self-hosted CI runner must be online** for test jobs to run. Actions are designed to be re-runnable to handle missed triggers.
- **Private repo required** when using a self-hosted runner — running a self-hosted runner on a public repository is a known RCE risk and is explicitly prohibited by project conventions.