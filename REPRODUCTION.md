# Issue #129 reproduction

## What I am reproducing

Issue #129 is a missing CI safeguard rather than a user-facing bug. PathReview has
an Alembic migration history, but the current GitHub Actions workflow never runs
that history or checks the resulting database schema against the SQLAlchemy
models. This means a pull request can pass the checks that currently exist
without proving that a database can be upgraded safely.

## Steps and observations

1. I inspected `.github/workflows/ci.yml`. It defines lint, typecheck, unit-test,
   integration-test, and frontend jobs. The integration job starts PostgreSQL,
   but it only runs `pytest tests/integration`; there is no `alembic upgrade
   head`, `alembic check`, or dedicated migration-validation job.
2. I ran `.venv/bin/alembic heads` and observed one current head: `002`.
3. I ran `.venv/bin/alembic history` and observed the ordered history
   `<base> -> 001 -> 002`.
4. I ran `.venv/bin/alembic upgrade head --sql`. Alembic successfully generated
   88 lines of PostgreSQL DDL for both revisions, showing that the migration
   path exists. Because `--sql` is offline mode, this only generates SQL; it
   does not prove that the statements execute successfully against PostgreSQL
   or that the resulting schema matches `Base.metadata`.
5. I searched the CI workflow for `alembic` and `migration` and received no
   matches. Therefore, none of the current automatic checks exercises the
   migration path found in steps 2–4.

## Expected vs. actual

**Expected:** On every pull request, CI should create an isolated PostgreSQL
database, run every migration from base to head, and fail if the resulting
schema differs from the SQLAlchemy models.

**Actual:** CI can start PostgreSQL and run integration tests without ever
executing Alembic. A green build currently means the existing checks passed; it
does not mean the migration history is executable or free of schema drift.

## Current limitation

My machine does not currently have Docker, `psql`, or a PostgreSQL server, so
this reproduction does not claim a live database result. The first
implementation step will use the disposable PostgreSQL service in GitHub
Actions (or an equivalent local container) to run `alembic upgrade head` and
`alembic check` end to end.
