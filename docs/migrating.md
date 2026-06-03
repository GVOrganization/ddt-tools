# 🚚 Migrating from other tools

> Convert an existing setup — a dbt project, a folder of raw SQL, or a live workspace — into a DDT project.

**On this page:** [How import works](#how-import-works) · [dbt](#dbt) · [Flat SQL files](#flat-sql-files) · [DDL, DBML & JSON Schema](#ddl-dbml--json-schema) · [Live workspace](#live-workspace) · [Run both in parallel](#run-both-in-parallel)

---

## How import works

DDT — Databricks Data Tools is state-based: you describe the desired Unity Catalog state and DDT computes the deltas. Migrating means turning your current setup into a `.ddtproj` where each object is its own `.sql` file, then authoring declaratively from there.

Every importer **only reads** the source — it writes a new project tree and never modifies what you migrate from. Objects land under the catalog-based layout `catalogs/<catalog>/schemas/<schema>/`.

---

## dbt

A dedicated migration command reads a dbt project and reports how it maps into DDT.

```bash
# Read dbt_project.yml + target/manifest.json + sources.yml, emit a migration report.
ddt migrate from-dbt ./my-dbt-project --output report.json

# Normalize identifier case and fill in unpinned catalog/schema names.
ddt migrate from-dbt ./my-dbt-project --identifier-case upper --default-database analytics
```

| Flag | What it does |
|---|---|
| `--manifest <path>` | Override the `target/manifest.json` path |
| `--identifier-case` | `preserve` (default) / `upper` / `lower` |
| `--default-database` / `--default-schema` | Fill in entries the manifest left unpinned |

You can also import a compiled dbt manifest directly as schema:

```bash
ddt schema import-dbt ./manifest.json --out ./MyProject
```

> [!IMPORTANT]
> dbt models are templated SQL. The migration reports what maps cleanly and what needs manual attention — Jinja-heavy models, incremental logic, and macros generally need a human pass.

---

## Flat SQL files

Already keep your schema as `.sql` scripts? `ddt import` parses a SQL script and writes each object it finds into the project tree.

```bash
# Import objects from a script into an existing project.
ddt import --project ./MyProject.ddtproj --script ./schema.sql

# Preview the planned files without writing anything.
ddt import --project ./MyProject.ddtproj --script ./schema.sql --dry-run
```

| Flag | What it does |
|---|---|
| `--project <path>` | Target `.ddtproj` to import into (required) |
| `--script <path>` | SQL script to parse (required) |
| `--dry-run` | Show the planned files without writing |
| `--force` | Overwrite existing object files |
| `--ignore-errors` | Skip statements the parser can't handle |

> [!TIP]
> Run with `--dry-run` first to see exactly which files would be created, then re-run without it once the layout looks right.

---

## DDL, DBML & JSON Schema

Model definitions in another format import through `ddt schema`. Each subcommand reads a foreign format and writes a `.ddtproj` tree.

```bash
ddt schema import-ddl ./schema.sql --out ./MyProject          # raw CREATE statements
ddt schema import-dbml ./model.dbml --out ./MyProject         # a DBML diagram model
ddt schema import-jsonschema ./schema.json --out ./MyProject  # JSON Schema definitions
```

**After import:** the importers translate structure, not data-loading logic. Review the generated objects, then `ddt build` and compare against your workspace before you deploy anything.

---

## Live workspace

The fastest start: reverse-engineer an existing Unity Catalog workspace directly into a project with `ddt extract`.

```bash
ddt extract --connection prod --catalog main --output ./MyProject
```

This writes the standard DDT folder structure populated from your live workspace, round-trippable with `ddt build`. Pass `--out-pac <file>` to write a `.ddtpac` build artifact in the same step. See [Extract](extract.md) for the full set of options.

---

## Run both in parallel

You don't have to cut over in a single step. The recommended transition:

1. **Import into a fresh project** — the importers never touch your source, so both setups can coexist.
2. **Keep deploying with your existing tool** while you validate the DDT project: `ddt build` then `ddt drift --source ./MyProject.ddtproj --connection prod` should report no unexpected diffs.
3. **Switch reads first** — use DDT for `drift`, `lint`, and `review` against production before you let it write anything.
4. **Cut over writes** once the diff is clean and your team is comfortable, and retire the old tool.

> [!WARNING]
> Before the first DDT-driven deploy, confirm the diff is clean. On Unity Catalog a destructive change can delete managed-table files or drop a streaming table's checkpoint — the parallel-validation phase exists to catch those before DDT writes to production.

---

**Next:** [FAQ](faq.md) · **Up:** [Documentation home](README.md)
