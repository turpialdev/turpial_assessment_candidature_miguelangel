# Loan servicing — assessment repo

<!-- Maintainers: ADR-0013. NOT verified in the last 6 months = DO NOT SEND. -->
**Repo Owner:** Carlos Rodríguez <carlos@turpialdev.com> (primary) · Jacobo Martínez <jacobo@turpialdev.com> (deputy)
**Last verified (Dry Run):** 2026-08-18

A small back office for an invented lender: it issues fixed-term loans and services
the repayments. No real company, no real data.

**Read [`GLOSSARY.md`](./GLOSSARY.md) before the code.** It defines the domain — the
double-entry ledger, the repayment waterfall, holds, and interest accrual — and it's
about fifteen minutes. The code is meant to follow that model; how faithfully it does
is part of what you're here to find out.

## Running it

### With Docker (recommended)

```bash
docker compose up
```

- API: http://localhost:8000/api/loans/
- Frontend: http://localhost:5173

This brings up PostgreSQL 16, migrates, seeds a couple of loans with history, and
serves both the Django API and the Vue app.

### Without Docker

**Backend** (Python 3.13, Django 6):

```bash
cd backend
python -m venv .venv && . .venv/bin/activate
pip install -r requirements-dev.txt
python manage.py migrate
python manage.py seed
python manage.py runserver         # http://localhost:8000
pytest                             # run the test suite
```

Runs on SQLite out of the box (no database to install). Set the `POSTGRES_*` env
vars to point at Postgres instead.

**Frontend** (Node 22, Vue 3.5 + Vite):

```bash
cd frontend
npm install
npm run dev                        # http://localhost:5173 (proxies /api to :8000)
npm test                           # run the test suite (Vitest)
```

## The API

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/loans/` | All loans, with a quick `outstanding`. |
| GET | `/api/loans/<id>/` | One loan. |
| GET | `/api/loans/<id>/statement/` | A loan's transaction/posting history, ordered. |

## Your task

You've been given one of two tasks. Both build one vertical slice on top of what's
already here.

**Back-end** — implement `record_repayment` in
`backend/bank/services/repayment.py`. A repayment arrives against a loan; accept it,
allocate it across the loan's balances in the order the domain requires, and record
the result so the accounting stays consistent. The waterfall allocation
(`bank/money.py`) and the way this codebase writes balanced postings
(`bank/services/transactions.py`) are already here.

Scope: the service function is the task. You do **not** need to add an HTTP endpoint
for it — exposing it over the API is out of scope.

**Front-end** — build the loan statement view in
`frontend/src/components/LoanStatement.vue`. Given a loan, show its statement: the
running balance as it changes over the loan's history, reconciled from
`/api/loans/<id>/statement/`. Routing and fetching patterns are in `LoanList.vue`
and `api/loans.ts`.

You decide what "consistent" / "useful" means and what happens at the edges — some
of them aren't specified, on purpose. When you hand back your pull request, tell us
what you built **and** what you made of the code you inherited: what you'd raise with
the team, what you'd change, what you'd leave. Nothing here is above criticism.

## Pull Request

### Resumen

Se construyó la vista de estado de cuenta en `LoanStatement.vue`: consume el endpoint `/api/loans/<id>/statement/`, concilia los apuntes (*postings*) de cuentas por cobrar para generar un saldo pendiente corrido, y muestra el historial reflejando el cambio por línea y el saldo actual. También gestiona la paginación de la API del estado de cuenta (cuyo tamaño de página predeterminado es `2`) para usar todo el historial disponible, y lanza una advertencia cuando el saldo pendiente en caché del detalle del préstamo no coincide con el saldo derivado del estado de cuenta.

### Notas de revisión

- **Paginación en el endpoint**: el `page_size=2` por defecto es fácil de pasar por alto; un cliente que trate la respuesta como un arreglo plano se quedará con un historial incompleto. Yo lo señalaría al equipo, subiría el valor predeterminado o al menos lo documentaría explícitamente.
- **Saldo en caché vs. libro mayor**: el campo de lista/detalle usa saldos contables cacheados, mientras que el estado de cuenta es la vista derivada directamente de los apuntes. Con los datos de prueba (*seeded data*) no coinciden (la caché se queda estancada con el saldo pendiente inicial). Habría que plantear si la caché está mal, desactualizada, o si directamente la API debería exponer la cifra derivada en lugar de la cacheada.
