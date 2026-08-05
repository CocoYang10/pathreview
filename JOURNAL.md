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

**Reproduction commit link:** https://github.com/CocoYang10/pathreview/commit/a25a659

**Reproduction summary:**
I reproduced this as a missing CI safeguard: the repository has an ordered
Alembic history (`001 -> 002`), but `.github/workflows/ci.yml` never runs it. On
a fresh local PostgreSQL 16 database, both migrations applied successfully,
but `alembic check` then detected an extra `uq_users_email` constraint, proving
that the migrated schema already drifts from the SQLAlchemy models while the
current CI has no check that reports it.

**PLAN.md link:** https://github.com/CocoYang10/pathreview/blob/feat/129-database-migration-validation/PLAN.md

**Walkthrough video (recommended):** Not recorded.

**Blockers or open questions:**
The live database reproduction is complete. The remaining implementation
question is how narrowly to correct the confirmed drift: the new revision must
remove only the redundant `uq_users_email` constraint while preserving the
unique `ix_users_email` index that still enforces email uniqueness. The same
successful-failure sequence must then be reproduced in GitHub Actions to prove
the new CI job is using the intended async PostgreSQL connection.

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:** I traced the drift to migration `001`, which creates both
a unique constraint and a unique index for `users.email`, while the current
SQLAlchemy metadata expects only the unique index. I added a new corrective
migration instead of modifying published migration history, and started a
reusable validation script plus a separate PostgreSQL-backed CI job.

**Next steps:** Add focused tests, recreate a clean PostgreSQL database, run the
full migration chain and schema comparison, then self-review the final diff.

**Blockers:** The repository's existing test and lint suites are not green, so I
need to compare before-and-after results and demonstrate that this change adds
no new failures rather than claiming to fix unrelated baseline problems.

---

### Check-in 2 (end of week)

**PR link:** To be added after the pull request is opened.

**Branch:** `feat/129-database-migration-validation`

**What you built:** I added migration `003` to remove the redundant
`uq_users_email` constraint without removing the unique index, a fail-fast shell
script that runs `alembic upgrade head` followed by `alembic check`, and a new
GitHub Actions job that executes this workflow against fresh PostgreSQL 16.

**Tests added or updated:** I added four unit tests for the complete revision
chain, the corrective migration's upgrade and downgrade operations, and the
validation script's missing-`DATABASE_URL` behavior. All four pass. I also ran
the real workflow against a recreated local database: migrations `001` through
`003`, rollback and reapplication of `003`, and the final schema comparison all
passed. The full unit-suite baseline remained unchanged at 52 failures and 31
errors, with passing tests increasing from 345 to 349.

**Self-review confirmation:** [ ] `make check` passes [ ] `make test-unit`
passes. Both commands still fail on documented pre-existing repository issues;
focused Ruff, Black, shell syntax, and the four new tests pass, and no new full
suite failures were introduced.

**Draft PR feedback received from:** None yet.
