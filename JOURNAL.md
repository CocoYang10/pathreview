## Week 7 - Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/129

**Issue title:** Add a database migration validation step to CI that checks all migrations can be applied cleanly

**Tier:** [ ] Tier 1 [ ] Tier 2 [x] Tier 3

**Problem summary:**
PathReview has SQLAlchemy models and Alembic migration files, but its CI workflow does not currently verify the database migration history. A migration can therefore be syntactically valid while failing against a clean PostgreSQL database or leaving the database schema inconsistent with the application models. The requested change will create a fresh database in CI, apply every migration in order, and compare the resulting schema with the SQLAlchemy metadata. This will catch migration failures and schema drift before a pull request is merged.

**Selection rationale:**
I can identify the relevant CI, Alembic, and SQLAlchemy files and explain the difference between application models, migration history, and the live database schema. The issue is a stretch because it crosses PostgreSQL, Alembic, shell scripting, and GitHub Actions, but the repository currently has a short two-migration history and a working PostgreSQL service pattern in the existing integration-test job. I chose it to build database and CI skills that complement my Python, SQL, statistics, and machine-learning background. The main scope risk is defining a reliable schema-consistency check and reproducing the same behavior locally and in GitHub Actions.

**Branch name:** feat/129-database-migration-validation

**Setup confirmation:** [ ] App runs locally at localhost:5173

**Cohort Ledger:** [x] Issue added to cohort ledger

## Week 8 - Reproduction & solution planning

**Reproduction commit link:** https://github.com/CocoYang10/pathreview/commit/9c902bc

**Reproduction summary:**
I reproduced this as a missing CI safeguard: the repository has an ordered
Alembic history (`001 -> 002`) and can generate the full PostgreSQL upgrade SQL,
but `.github/workflows/ci.yml` never runs Alembic. The integration job starts
PostgreSQL and then runs tests directly, so a green build currently does not
prove that migrations execute successfully or that their final schema matches
the SQLAlchemy models.

**PLAN.md link:** https://github.com/CocoYang10/pathreview/blob/feat/129-database-migration-validation/PLAN.md

**Walkthrough video (recommended):** Not recorded.

**Blockers or open questions:**
My local machine does not currently have Docker or PostgreSQL, so my
reproduction confirms the missing validation path and generates the migration
SQL offline, but does not claim a live database result. My first Week 9 step is
to run `alembic upgrade head` and `alembic check` against the disposable
PostgreSQL service in GitHub Actions. I also need to determine whether the
unique constraint and unique index created for `users.email` represent genuine
pre-existing schema drift before deciding whether a corrective migration is
in scope.
