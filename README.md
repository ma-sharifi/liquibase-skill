# liquibase-changeset skill

A reference [Agent Skill](https://code.claude.com/docs/en/skills) for creating and
reviewing **Liquibase YAML changesets** in Spring Boot projects.

The skill teaches Claude a consistent, safe migration workflow:

- YAML changelogs under `src/main/resources/db/changelog/`
- One logical change per changeset with a stable `<version>-<sequence>` ID
- An **explicit rollback** for every changeset
- Preconditions for idempotency, two-step NOT NULL additions, and release tagging
- A validation step (`validate`, `update-sql`, `update-testing-rollback`) before finishing

The skill definition lives in [`SKILL.md`](./SKILL.md).

## Installing

Copy the skill into a skills directory Claude Code discovers:

```bash
# Personal (all projects)
mkdir -p ~/.claude/skills/liquibase-changeset
cp SKILL.md ~/.claude/skills/liquibase-changeset/

# Or project-scoped (checked into a repo)
mkdir -p .claude/skills/liquibase-changeset
cp SKILL.md .claude/skills/liquibase-changeset/
```

Claude activates it automatically when a request matches the `description` in the
frontmatter (adding a table/column/index, rolling back a change, tagging a release,
or reviewing an existing changeset).

## When it applies

Use it for Liquibase migrations. It intentionally does **not** cover
JPA/Hibernate auto-DDL, Flyway migrations, or raw SQL scripts run outside Liquibase.
