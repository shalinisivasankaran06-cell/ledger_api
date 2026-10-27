# Walkthrough (for you — not part of the submission)

This file is listed in `.gitignore`, so it will NOT be pushed to GitHub.
Read it once top to bottom and you can explain every part of the project.

---

## 1. The big picture in one paragraph

A client sends HTTP requests. **FastAPI** receives them, **Pydantic** checks the body/query
is valid, the **router** calls a **service** function, the service talks to **PostgreSQL**
through **SQLAlchemy**, and the result goes back as JSON. Every error goes through one
place (`errors.py`) so they all look the same. **Alembic** creates the tables. **pytest**
proves every rule works.

Request flow for "debit 30":

```
POST /accounts/{id}/transactions
  -> schemas.TransactionCreate validates type/amount/reference_id (else 422)
  -> routers/accounts.add_transaction
  -> services/transaction_service.create_transaction
       1. SELECT account ... FOR UPDATE   (lock the row; 404 if missing)
       2. reference_id already used here? -> 409
       3. balance < amount?               -> 400
       4. balance -= 30, INSERT transaction row
       5. COMMIT  (any error -> ROLLBACK, nothing saved)
  -> 201 + the transaction as JSON
```

---

## 2. File by file

**`app/config.py`** — settings read from environment variables (or `.env`) using
`pydantic-settings`: the database URL and pagination limits. Nothing secret is hard-coded.

**`app/db.py`** — creates the SQLAlchemy *engine* (a pool of DB connections) and
`SessionLocal` (a factory for sessions). `get_db()` is a FastAPI *dependency*: each request
gets its own session, and it's always closed at the end (which also rolls back anything
uncommitted). `expire_on_commit=False` means objects keep their values after commit so we
can return them without another query.

**`app/models.py`** — the two tables as Python classes.
- `Account`: id (UUID), name, balance (`NUMERIC(18,2)`, default 0), created_at.
- `Transaction`: id, account_id (foreign key), type, amount, description, reference_id,
  balance_after (balance right after this txn), transfer_id (links the 2 legs of a
  transfer), created_at.
- Constraints: `CHECK balance >= 0`, `CHECK amount > 0`, `type IN ('credit','debit')`,
  `UNIQUE(account_id, reference_id)`, and an index on `(account_id, created_at)` for history.

**`app/schemas.py`** — Pydantic models = the API contract.
- `AccountCreate` only has `name`. Because of `extra="forbid"`, sending `balance` is a 422.
  That's how "client cannot set the initial balance" is enforced.
- `PositiveAmount` = Decimal, `> 0`, max 2 decimals; `30` becomes `30.00`.
- `TransactionCreate`, `TransferCreate` (with a check that from ≠ to), and the response
  models (`AccountOut`, `BalanceOut`, `TransactionOut`, `TransactionPage` with `pagination`).

**`app/errors.py`** — custom exceptions (`AccountNotFound` 404, `InsufficientBalance` 400,
`DuplicateTransaction` 409) and *exception handlers* that convert every error — including
FastAPI's own validation errors, 404/405s and unexpected crashes — into
`{"error": {"code", "message", "details"}}`. It also turns bad `page`/`limit` into
`400 INVALID_PAGINATION` instead of FastAPI's default 422.

**`app/services/account_service.py`** — create account (balance hard-set to 0), get account
(404 if missing), and `get_account_for_update`, which runs `SELECT ... FOR UPDATE`.

**`app/services/transaction_service.py`** — the heart of the project.
- `apply_transaction(...)`: duplicate check → balance check → update balance → insert row →
  `flush()`. It does NOT commit, so it can be reused inside a bigger transaction (transfer).
- `create_transaction(...)`: lock, apply, `commit()`; on any exception `rollback()`.
  It also catches `IntegrityError` from the UNIQUE constraint as a last safety net and
  turns it into a 409.
- `list_transactions(...)`: count + one page, filtered by type, `ORDER BY created_at DESC, id DESC`,
  `OFFSET (page-1)*limit LIMIT limit`.

**`app/services/transfer_service.py`** — locks both accounts **sorted by id**, debits the
source, credits the destination (both with `apply_transaction`), then one `commit()`.
Any failure → `rollback()` → neither account changes.

**`app/routers/accounts.py`, `transfers.py`** — the endpoints. Thin: take validated input,
call the service, return a response model. `responses={...}` only documents error codes in Swagger.

**`app/main.py`** — creates the FastAPI app, registers error handlers and routers, adds `/health`.

**`alembic/versions/0001_...py`** — the migration that creates the tables, constraints and
indexes. `alembic upgrade head` runs it. (`alembic/env.py` makes Alembic use `DATABASE_URL`.)

**`tests/`** — 72 tests against a real Postgres.
- `conftest.py`: points the app at `ledger_test`, runs the migration once, empties the tables
  before every test, gives a `TestClient`.
- `test_accounts.py`, `test_transactions.py`, `test_history.py`, `test_transfers.py`: every rule
  in the brief, plus error shapes.
- `test_transfers.py::test_failure_after_debit_rolls_back_everything` fakes a crash after the
  debit and checks nothing was saved — proof of atomicity.
- `test_concurrency.py`: real threads, separate DB connections. 20 parallel debits of 30
  on a balance of 100 → exactly 3 succeed, balance 10. (I checked: if you delete
  `.with_for_update()`, these tests fail.)

**`Dockerfile` / `docker-compose.yml`** — `docker compose up --build` starts Postgres + the
API; the container runs the migration then uvicorn.

**`scripts/export_openapi.py`** — regenerates `docs/openapi.json` from the code.

---

## 3. Likely interview questions (short answers)

**Why FastAPI?** Fast to build, type hints drive validation (Pydantic), and you get
Swagger docs automatically from the code.

**Why PostgreSQL and not MongoDB/SQLite?** The brief is all about consistency: atomic
transactions, row locks, CHECK/UNIQUE constraints. That's what a relational DB guarantees.
SQLite has no row-level `FOR UPDATE`, so the concurrency tests wouldn't be meaningful.

**How do you make the transaction + balance update atomic?** Both happen in one DB
transaction and are committed together. Any exception → rollback. There's a test that
crashes between the two steps of a transfer and checks nothing was saved.

**What happens if two debits hit the same account at the same time?** `SELECT ... FOR UPDATE`
locks the account row. The second request waits until the first commits, then sees the
updated balance. Without the lock both could read 100 and both debit 80 → −60. The DB
`CHECK (balance >= 0)` is a second line of defence.

**What's a deadlock and how do you avoid it in transfers?** Transfer 1 locks A then waits
for B, transfer 2 locks B then waits for A — both stuck forever. We always lock the two
accounts in the same order (sorted by id), so that cycle can't happen.

**How do you handle duplicate requests?** `reference_id` is unique per account. We check
after taking the lock and return 409 with the existing transaction; the
`UNIQUE(account_id, reference_id)` constraint is the backup if anything slips through.
It's what makes retries safe (idempotency).

**Why 409 for duplicates and 400 for insufficient balance?** 409 Conflict = the resource
already exists. 400 = the request is well-formed but the business rule rejects it. Malformed
input is always 422 (FastAPI's default). It's documented in ASSUMPTIONS.md.

**Why Decimal and not float?** Floats are binary approximations: 0.1 + 0.2 = 0.30000000000000004.
Money must be exact, so `NUMERIC(18,2)` in the DB and `Decimal` in Python. Returned as a
string for the same reason.

**Why store the balance instead of calculating SUM every time?** Reading is O(1) and it's
what we lock on. It always matches the sum because it's updated in the same atomic
transaction as the insert (a test checks the sum). We also store `balance_after` on each
transaction as an audit trail.

**How is "client can't set the balance" enforced?** The request schema has only `name`, and
`extra="forbid"` rejects any other field with 422. The service also hard-codes 0.

**How does pagination work?** `COUNT(*)` for total records, `OFFSET (page-1)*limit LIMIT limit`
for the page, `total_pages = ceil(total/limit)`. Sorted `created_at DESC, id DESC` so the
order is stable. *Follow-up "what about millions of rows?"* → keyset/cursor pagination
(`WHERE created_at < last_seen`) avoids slow large OFFSETs; I kept page/limit because the
brief asked for it.

**How do you make transactions immutable?** No update/delete endpoints exist (405 if tried).
In production I'd also revoke UPDATE/DELETE on the table for the app's DB user or add a trigger.

**Why UUIDs?** Unique without a central counter, safe to expose, and don't leak how many accounts exist.

**What would you add for production?** Authentication, rate limiting, structured logging,
monitoring, an `Idempotency-Key` header, keyset pagination, and DB-level permissions to make
the transactions table append-only.

**How did you test it?** pytest against a real Postgres: every rule, every error code,
pagination metadata, ordering, a forced crash mid-transfer, and real parallel requests.

**Why does the transfer reuse the same reference_id on both accounts?** The rule is
"unique per account", so it's allowed, and it makes the two legs easy to find. They're also
linked by `transfer_id`.

---

## 4. Pushing it from your GitHub

```bash
cd ledger_api
git init
git add .
git commit -m "Transaction Ledger API"
git branch -M main
git remote add origin https://github.com/<your-username>/transaction-ledger-api.git
git push -u origin main
```

Create the empty repo on GitHub first (no README, so there's no conflict). `.gitignore`
already keeps out `.venv`, `.env`, caches and this walkthrough.

To run it on your laptop: follow README "Option B" (or "Option A" if you have Docker).
Run `pytest` once before submitting so you've seen it pass yourself.
