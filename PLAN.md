# Solution plan

**Issue:** [#129 — Add a database migration validation step to CI that checks all migrations can be applied cleanly](https://github.com/ascherj/pathreview/issues/129)

## Understand

PathReview represents its database in three related places: SQLAlchemy models
describe the schema the application expects now, Alembic revisions describe how
an older database reaches that schema, and PostgreSQL holds the schema that
actually exists. The current CI workflow tests several parts of the application
and even starts PostgreSQL for integration tests, but it never runs Alembic.
Therefore, CI does not prove either that revisions `001` and `002` can be
applied in order or that their final schema matches `core.models.Base.metadata`.

The expected behavior is that each pull request gets a fresh database and CI
turns red for either of two failure classes: a migration that cannot execute, or
a migration that executes but produces schema drift. The actual behavior is
that both cases can be outside the checks currently run by CI.

## Map

Files I expect to change:

- `.github/workflows/ci.yml`: add an isolated migration-validation job with a
  temporary PostgreSQL service and an async database URL.
- `scripts/validate_migrations.sh`: add the reusable, fail-fast entry point that
  runs the two Alembic validations.
- `alembic/versions/003_*.py`: only if the new consistency check reveals a real
  mismatch that must be corrected without rewriting an already-applied
  revision.

Files I need to understand and use, but do not currently expect to change:

- `alembic/env.py`: connects Alembic through an async SQLAlchemy engine and sets
  `target_metadata = Base.metadata`.
- `alembic/versions/001_initial_schema.py` and
  `alembic/versions/002_add_error_message_to_reviews.py`: the complete migration
  history that CI must exercise.
- `core/models/*.py`: the expected schema used by `alembic check`.
- `core/config.py`: loads `DATABASE_URL`; its async driver requirements must
  match the CI environment.

## Plan

1. Create `scripts/validate_migrations.sh`. Make it fail clearly when
   `DATABASE_URL` is absent, then run `alembic upgrade head` followed by
   `alembic check`. Preserve each command's non-zero exit status so GitHub
   Actions marks the job failed.
2. Add a separate migration job to `.github/workflows/ci.yml`. Reuse the
   repository's PostgreSQL 16 service pattern, install the project on Python
   3.11, and pass a `postgresql+asyncpg://` URL because `alembic/env.py` creates
   an async engine.
3. Run the validation against a disposable, empty PostgreSQL database. Record
   the successful base-to-head path and deliberately introduce a temporary,
   controlled model/migration mismatch to verify the job becomes red. Remove
   that temporary change after the check is proven.
4. Investigate any mismatch reported by `alembic check`. In particular,
   migration `001` creates both `uq_users_email` and a unique
   `ix_users_email`, while the current model declares `unique=True,
   index=True`. Determine from the live comparison whether this is genuine
   drift. If it is, correct it with a new revision rather than editing `001`.
5. Run focused validation and review the CI diff for unrelated changes. Update
   `JOURNAL.md` with results, risks discovered, and links before opening the
   Week 9 pull request.

## Inputs & outputs

The validation takes:

- a `DATABASE_URL` that points to a fresh, disposable PostgreSQL database;
- the ordered files in `alembic/versions`;
- the SQLAlchemy metadata imported by `alembic/env.py`.

It produces no application data or UI change. Its primary output is a process
exit code: zero when all revisions apply and the final schema matches the
models, non-zero when migration execution or schema comparison fails. In
GitHub Actions, that exit code becomes a visible green or red pull-request
check, with Alembic's output explaining the failure.

## Risks & unknowns

- The current migration history may already differ from the models. A new check
  can correctly expose an old problem, so I need to distinguish a validation
  bug from genuine pre-existing drift.
- `alembic/env.py` uses `create_async_engine`; a plain `postgresql://` URL may
  select an incompatible synchronous driver. CI should use
  `postgresql+asyncpg://`.
- A reused or partially migrated database could produce misleading results.
  The CI job must receive a new isolated service database on every run.
- `alembic check` depends on all relevant model modules being registered in
  `Base.metadata`. An incomplete model import could create a false comparison.
- PostgreSQL may be started but not ready to accept connections. The service
  health check must gate the validation step.
- Existing unrelated CI failures must not be presented as failures introduced
  or fixed by this change.

## Edge cases

- A revision contains invalid SQL or refers to a missing table/column:
  `alembic upgrade head` should fail the job.
- A model changes without a matching revision: `alembic check` should fail the
  job after the existing revisions are applied.
- Two contributors create separate heads: validation should report the
  ambiguous history rather than silently choosing one branch.
- A revision references an unknown `down_revision`: the upgrade must fail with
  a useful Alembic error.
- `DATABASE_URL` is absent or points to the wrong driver: the script should exit
  early and explain the configuration problem.
- The database is already at head: validation should remain safe and
  repeatable, although CI will normally provide an empty database.
