# Transaction Ledger API

A small backend for a transaction ledger: create accounts, post credit/debit transactions,
read balances and paginated transaction history, and (bonus) transfer money between accounts
atomically.

- **Stack:** Python 3.11 · FastAPI · Pydantic v2 · SQLAlchemy 2 · PostgreSQL · Alembic · pytest
- **API docs:** Swagger UI at `/docs`, ReDoc at `/redoc`, exported spec in [`docs/openapi.json`](docs/openapi.json),
  written guide with examples in [`docs/API.md`](docs/API.md)
- **Assumptions:** [`ASSUMPTIONS.md`](ASSUMPTIONS.md)

---

## Project structure

```
app/
  main.py                    FastAPI app: routers + error handlers + /health
  config.py                  Settings from environment variables / .env
  db.py                      Engine, session factory, get_db() dependency
  models.py                  SQLAlchemy tables (Account, Transaction) + DB constraints
  schemas.py                 Pydantic request/response models (validation lives here)
  errors.py                  Domain errors + one consistent JSON error format
  routers/
    accounts.py              /accounts endpoints (accounts, balance, transactions, history)
    transfers.py             /transfers endpoint (bonus)
  services/
    account_service.py       Create/get accounts, row locking (SELECT ... FOR UPDATE)
    transaction_service.py   Apply credit/debit atomically, duplicate check, history query
    transfer_service.py      Atomic transfer: lock both accounts in id order, debit + credit
alembic/versions/0001_...py  Migration that creates the tables, constraints and indexes
tests/                       pytest suite (runs against real PostgreSQL)
scripts/export_openapi.py    Regenerates docs/openapi.json
docs/                        API.md + openapi.json
Dockerfile, docker-compose.yml, requirements.txt, .env.example
```

Routers are thin (HTTP in/out), services hold the business rules, models hold the schema.

---

## Running it

### Option A: Docker Compose (easiest)

```bash
docker compose up --build
```

This starts PostgreSQL 16 and the API, runs the migration automatically and serves on
http://localhost:8000 (Swagger UI: http://localhost:8000/docs). Postgres is exposed on host
port 5433 (so it doesn't clash with a local Postgres). With the older standalone tool the
command is `docker-compose up --build`.

Run the tests inside the container (the compose setup also creates a `ledger_test` database):

```bash
docker compose exec api pytest
```

### Option B: Without Docker

Requirements: Python 3.11+ and a running PostgreSQL (14+).

```bash
# 1. Create the databases
createdb -U postgres ledger
createdb -U postgres ledger_test          # only needed for the tests

# 2. Install dependencies
python -m venv .venv
source .venv/bin/activate                 # Windows: .venv\Scripts\activate
pip install -r requirements.txt

# 3. Configure
cp .env.example .env                      # edit DATABASE_URL if your user/password differ

# 4. Create the tables
alembic upgrade head

# 5. Start the API
uvicorn app.main:app --reload
```

Open http://localhost:8000/docs.

### Environment variables

| Variable | Default | Meaning |
|---|---|---|
| `DATABASE_URL` | `postgresql+psycopg://postgres:postgres@localhost:5432/ledger` | Database used by the app and by Alembic |
| `TEST_DATABASE_URL` | `postgresql+psycopg://postgres:postgres@localhost:5432/ledger_test` | Disposable database for the tests (tables are dropped and re-created) |
| `DEFAULT_PAGE_SIZE` | `10` | Default `limit` for transaction history |
| `MAX_PAGE_SIZE` | `100` | Largest allowed `limit` |
| `SQL_ECHO` | `false` | Log every SQL statement |

### Running the tests

```bash
pytest            # or: pytest -v
```

72 tests covering: account creation rules, credit/debit maths, insufficient balance,
duplicate `reference_id` (same account and across accounts), validation errors, pagination
metadata and ordering, type filter, immutability, DB constraints, transfers (including a
simulated crash between debit and credit that must roll back), and concurrency
(20 parallel debits never overdraw, 15 parallel duplicates are applied once, opposite
transfers don't deadlock).

The tests use a real PostgreSQL database on purpose: row locks, constraints and
transactions are exactly what is being tested, and SQLite would not behave the same way.

---

## Endpoints

| Method | Path | Purpose | Success |
|---|---|---|---|
| `POST` | `/accounts` | Create an account (`{"name": "..."}`); balance always starts at 0 | 201 |
| `GET` | `/accounts/{account_id}` | Account details | 200 |
| `GET` | `/accounts/{account_id}/balance` | Current balance | 200 |
| `POST` | `/accounts/{account_id}/transactions` | Add a credit/debit | 201 |
| `GET` | `/accounts/{account_id}/transactions` | History: `?type=credit\|debit&page=1&limit=10`, newest first | 200 |
| `POST` | `/transfers` | Bonus: move money A → B atomically | 201 |
| `GET` | `/health` | Liveness + DB check | 200 |

There are deliberately **no** PUT/PATCH/DELETE endpoints for transactions: they are
immutable (those methods return `405 METHOD_NOT_ALLOWED`).

### Quick example

```bash
# create
curl -X POST localhost:8000/accounts -H 'Content-Type: application/json' -d '{"name":"Shalini"}'
# -> 201 {"id":"c6f5b8f6-...","name":"Shalini","balance":"0.00","created_at":"2026-10-26T11:38:48.458450Z"}

# credit 100, then debit 30
curl -X POST localhost:8000/accounts/<id>/transactions -H 'Content-Type: application/json' \
     -d '{"type":"credit","amount":100,"reference_id":"T1","description":"Salary"}'
curl -X POST localhost:8000/accounts/<id>/transactions -H 'Content-Type: application/json' \
     -d '{"type":"debit","amount":"30","reference_id":"T2"}'

# balance
curl localhost:8000/accounts/<id>/balance
# -> 200 {"account_id":"c6f5b8f6-...","balance":"70.00"}

# history, debits only
curl 'localhost:8000/accounts/<id>/transactions?type=debit&page=1&limit=10'
```

Full request/response examples for every endpoint and error: [`docs/API.md`](docs/API.md).

### Errors

Every error has the same shape:

```json
{"error": {"code": "INSUFFICIENT_BALANCE", "message": "Insufficient balance for this debit",
           "details": {"account_id": "…", "balance": "70.00", "requested_amount": "500.00"}}}
```

| Situation | HTTP | `code` |
|---|---|---|
| Account does not exist | 404 | `ACCOUNT_NOT_FOUND` |
| Body field invalid (missing name, bad type, amount ≤ 0, missing reference_id, unknown field such as `balance`) | 422 | `VALIDATION_ERROR` |
| Body is not valid JSON | 422 | `INVALID_REQUEST_BODY` |
| Malformed account id in the URL (not a UUID) | 422 | `VALIDATION_ERROR` |
| Debit larger than the balance | 400 | `INSUFFICIENT_BALANCE` |
| `reference_id` already used on this account | 409 | `DUPLICATE_TRANSACTION` (existing transaction in `details`) |
| Bad `page` / `limit` | 400 | `INVALID_PAGINATION` |
| Bad `type` filter in history | 422 | `VALIDATION_ERROR` |
| Method not supported (e.g. DELETE a transaction) | 405 | `METHOD_NOT_ALLOWED` |
| Unexpected server error | 500 | `INTERNAL_ERROR` |

---

## Design decisions

**Atomicity.** Adding a transaction is one database transaction: update `accounts.balance`
and insert the `transactions` row, then `COMMIT`. If anything fails (validation, low balance,
duplicate, constraint, crash) the session is rolled back, so the balance can never change
without its transaction row, or the other way round.

**Locking / concurrency.** Before changing a balance the service runs
`SELECT … FROM accounts WHERE id = :id FOR UPDATE`. A second request for the same account
waits until the first commits, then reads the *new* balance. This prevents two parallel
debits from both seeing "100" and overdrawing (the "lost update" problem). Transfers lock
**both** accounts, always in ascending id order, so two opposite transfers (A→B and B→A)
can't deadlock each other. The concurrency tests prove this (and fail if the lock is removed).

**Idempotency / duplicates.** `reference_id` is unique per account, enforced twice:
an explicit check after taking the row lock (gives a clean 409 with the existing transaction),
and a `UNIQUE(account_id, reference_id)` constraint as the final guard. A retried request
therefore can't double-charge. The same `reference_id` may be reused on a different account.
A *rejected* transaction is not stored, so its `reference_id` stays free for a retry.

**Database constraints.** The database refuses bad data even if the code had a bug:
`CHECK (balance >= 0)`, `CHECK (amount > 0)`, `CHECK (type IN ('credit','debit'))`,
`UNIQUE (account_id, reference_id)`, a foreign key from transactions to accounts, and
`NOT NULL` on required columns.

**Money as Decimal.** Amounts use `NUMERIC(18,2)` in PostgreSQL and `Decimal` in Python —
never `float` (0.1 + 0.2 must be 0.30, not 0.30000000000000004). Amounts with more than
2 decimal places are rejected. Money is returned as a JSON string (`"70.00"`) so no client
loses precision.

**Stored balance + transaction log.** The balance is stored on the account (fast reads) and
every change is recorded as an immutable transaction with `balance_after` (a running balance
for auditing). Because both are written in the same atomic transaction, the stored balance
always equals the sum of credits minus debits; the tests check this.

**Validation.** Pydantic validates the body before any DB work. Request models use
`extra="forbid"`, which is how a client is stopped from setting `balance`, `id` or `created_at`.

**History.** Sorted by `created_at DESC` with `id` as a tie-breaker (stable pages), using
`OFFSET/LIMIT` and a `COUNT(*)` for the metadata. Index on `(account_id, created_at)`.
