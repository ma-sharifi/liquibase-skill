---
name: liquibase-changeset
description: >-
  Creates and reviews Liquibase YAML changesets for Spring Boot projects.
  Use when the user asks to add, modify, or roll back a database schema change,
  create a Liquibase changeset or changelog, add a table/column/index/constraint
  via migration, tag a release, or review an existing changeset. Do NOT use for
  JPA/Hibernate auto-DDL, Flyway migrations, or raw SQL scripts outside Liquibase.
---

# Liquibase Changeset Skill

## Purpose

Generate Liquibase changesets that follow this project's conventions: YAML format, one logical change per changeset, explicit rollback for every changeset, and release tagging. Never modify a changeset that has already been applied to any shared environment — always add a new one.

## File and directory conventions

- Changelogs live under `src/main/resources/db/changelog/`
- Master changelog: `db.changelog-master.yaml` — only contains `include` entries, never changesets directly
- Migration files: `db/changelog/changes/<version>/<NNN>-<short-description>.yaml`
  - Example: `db/changelog/changes/v1.4/003-add-index-customer-email.yaml`
- New files must be appended to the master changelog via `include` with `relativeToChangelogFile: true`
- Never reorder existing `include` entries

## Changeset rules

1. **One change type per changeset.** A `createTable` and its indexes are separate changesets. This keeps rollbacks atomic and avoids partial failures with auto-commit DDL.
2. **ID convention:** `<version>-<sequence>` (e.g. `v1.4-003`). Never reuse an ID.
3. **Author:** use the developer's git username, not "liquibase" or a team name.
4. **Comment is mandatory:** one line explaining *why* the change is made, referencing the ticket (e.g. `JIRA-1234: support customer e-mail lookup`).
5. **Explicit rollback for every changeset**, even when Liquibase could auto-generate one. For destructive or non-reversible changes (`dropColumn`, data migration), write a rollback that restores structure, and add a comment noting data is not restored.
6. **Preconditions** for idempotency-sensitive changes: use `onFail: MARK_RAN` with `not: tableExists` / `not: columnExists` / `not: indexExists` when there is any chance the object already exists in some environments. Always scope the check with `tableName` (and `columnName`/`indexName`) so it is unambiguous across databases.
7. **No raw SQL** unless the change type doesn't exist in Liquibase. If `sql` is unavoidable, pair it with an explicit `rollback` block and a precondition.
8. **Column definitions:** always specify `nullable` explicitly; new NOT NULL columns on existing tables must include a `defaultValue*` or be added in two steps (add nullable → backfill → add constraint).

## Template

```yaml
databaseChangeLog:
  - changeSet:
      id: v1.4-003
      author: mahdi
      comment: "JIRA-1234: add index for customer e-mail lookup"
      preConditions:
        - onFail: MARK_RAN
        - not:
            - indexExists:
                tableName: customer
                indexName: idx_customer_email
      changes:
        - createIndex:
            tableName: customer
            indexName: idx_customer_email
            columns:
              - column:
                  name: email
      rollback:
        - dropIndex:
            tableName: customer
            indexName: idx_customer_email
```

### Adding a NOT NULL column to an existing table (two-step)

```yaml
databaseChangeLog:
  - changeSet:
      id: v1.4-004
      author: mahdi
      comment: "JIRA-1250: add status column (nullable first, backfilled next)"
      preConditions:
        - onFail: MARK_RAN
        - not:
            - columnExists:
                tableName: orders
                columnName: status
      changes:
        - addColumn:
            tableName: orders
            columns:
              - column:
                  name: status
                  type: varchar(20)
                  constraints:
                    nullable: true
      rollback:
        - dropColumn:
            tableName: orders
            columnName: status

  - changeSet:
      id: v1.4-005
      author: mahdi
      comment: "JIRA-1250: backfill status then enforce NOT NULL"
      changes:
        - update:
            tableName: orders
            columns:
              - column:
                  name: status
                  value: "NEW"
            where: "status IS NULL"
        - addNotNullConstraint:
            tableName: orders
            columnName: status
            columnDataType: varchar(20)
      rollback:
        - dropNotNullConstraint:
            tableName: orders
            columnName: status
            columnDataType: varchar(20)
```

## Release tagging

At the end of each release's changes, add a tag changeset so environments can be rolled back to a known point:

```yaml
  - changeSet:
      id: v1.4-tag
      author: mahdi
      comment: "Tag release v1.4"
      changes:
        - tagDatabase:
            tag: v1.4
```

## Validation (run before finishing)

After creating or editing changesets, always validate:

```bash
# Syntax + changelog validation
liquibase validate --changelog-file=src/main/resources/db/changelog/db.changelog-master.yaml

# Preview generated SQL without applying
liquibase update-sql --changelog-file=src/main/resources/db/changelog/db.changelog-master.yaml

# Verify the rollback actually works (against a local/H2 database only)
liquibase update-testing-rollback --changelog-file=src/main/resources/db/changelog/db.changelog-master.yaml
```

If a `liquibase.properties` or a Maven/Gradle plugin config exists in the repo, prefer `./mvnw liquibase:validate` / `./gradlew validate` with the project's configuration instead of the bare CLI.

## Review checklist (when asked to review a changeset)

- [ ] One logical change per changeset
- [ ] ID follows `<version>-<sequence>` and is unique
- [ ] Comment references a ticket and explains intent
- [ ] Explicit rollback present and correct
- [ ] NOT NULL additions handle existing rows
- [ ] No modification of an already-applied changeset (check git history)
- [ ] File included in master changelog, appended at the end
- [ ] Destructive changes (`drop*`, `delete`) flagged for extra human review

## Never do

- Never edit a changeset that exists on `main` or has been deployed — checksums will break (`DATABASECHANGELOG.MD5SUM` mismatch)
- Never use `runAlways: true` for schema changes
- Never put credentials or environment-specific values in changelogs — use property substitution
- Never combine DDL and data (DML) in one changeset
