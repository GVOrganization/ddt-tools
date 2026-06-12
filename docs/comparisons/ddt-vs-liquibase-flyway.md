# DDT vs Liquibase / Flyway on Databricks

Liquibase and Flyway are the canonical migration runners — proven over two decades, across 60+ databases. Both support Databricks. The comparison comes down to two things: **migration-based vs state-based**, and **generic-SQL vs platform-native**.

## TL;DR

- Pick **Liquibase / Flyway** if: your org already runs them across many databases and wants one uniform process — or you need their compliance-grade audit trails and enterprise approval surfaces.
- Pick **DDT** if: Databricks is the platform that matters, and you want desired-state files, a compare engine, and a safety classifier that knows what a generic runner can't — that dropping a *managed* table deletes files, and that replacing a *streaming* table destroys its checkpoint.

## The two differences

**1. Migration-based vs state-based.** Flyway (and classic Liquibase) apply hand-written, versioned change scripts in order; the edit history is the source of truth. DDT inverts this: each object is one `.sql` file describing its desired state, and `ddt compare` computes the migration. You never hand-write (or hand-review) forward/backward scripts, and "what should prod look like?" is answered by reading the repo, not replaying the chain. (Liquibase Secure's recent *modeled change control* is a real step toward state-based — on Snowflake. On Databricks, it remains generic SQL.)

**2. Generic-SQL vs platform-native.** A multi-database tool necessarily treats DDL generically — a `DROP TABLE` is a `DROP TABLE`. On Databricks the same statement spans three different risk classes, and the difference isn't in the SQL text:

| Statement | What actually happens | DDT classification |
|---|---|---|
| `DROP TABLE` (external) | metadata removed, files retained | `DESTRUCTIVE` |
| `DROP TABLE` (managed) | **files deleted** after the UNDROP window | `UNRECOVERABLE` — refused without a gate |
| `CREATE OR REPLACE` (streaming table) | checkpoint destroyed; re-reads from earliest | `UNRECOVERABLE` — refused without a gate |

A generic runner executes all three identically. Liquibase's risk-pattern scanning is lint-style text matching; it doesn't consult the catalog to know managed from external.

## Honest feature comparison

| Capability | Liquibase / Flyway | DDT |
|---|---|---|
| Maturity, ecosystem, track record | ✅ ~20 years | ❌ public beta |
| One process across 60+ database engines | ✅ | ❌ Databricks only (Snowflake sibling: [SDT](https://github.com/GVOrganization/sdt-tools)) |
| Compliance-grade audit trails, approval workflows | ✅ (paid tiers) | partial — deploy manifests + webhooks |
| Open-source core | ✅ | ❌ (free core tier; artifact Apache-2.0) |
| State-based desired-state authoring | ❌ on Databricks | ✅ |
| Compare repo ↔ live catalog, any direction | ❌ | ✅ |
| Catalog-aware safety classification | ❌ | ✅ four tiers + reversibility analysis |
| Drift detection | paid feature | ✅ `ddt drift`, offline |
| Extract existing catalog to files | ❌ | ✅ `ddt extract` |
| Hand-written rollback scripts required | yes | no — manifest-based revert |

## Migrating

The importer replays your scripts in version order and emits the final declarative state — bulk DML and unclassifiable operations are surfaced as warnings with file + line:

```sh
ddt import --from flyway    --source-path ./sql        --output ./Lakehouse
ddt import --from liquibase --source-path ./changelog  --output ./Lakehouse
```

Your existing history tables are untouched; run both side by side while you evaluate. See [Migrating from other tools](../migrating.md).

---

*Independent comparison written by the DDT maintainer — corrections welcome via [issues](https://github.com/GVOrganization/ddt-tools/issues). "Liquibase" and "Flyway" are trademarks of their respective owners.*
