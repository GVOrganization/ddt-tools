# DDT vs Databricks Declarative Automation Bundles

Databricks' **Declarative Automation Bundles** (formerly Asset Bundles) are the native way to deploy declared resources — jobs, pipelines, and Unity Catalog config — from files. DDT doesn't replace the bundle runtime and isn't trying to. This page is about which layer each owns.

## TL;DR

- Use **Bundles** for: deploying jobs, pipelines, and workspace resources as code. That's their home turf and DDT doesn't touch it.
- Use **DDT** for: the **table/column DDL layer** — what's actually different between your repo and the live catalog, which of those changes are dangerous, and refusing the unrecoverable ones by default.
- Most teams that adopt DDT keep their bundles exactly as they are.

## Honest feature comparison

| Capability | Bundles | DDT |
|---|---|---|
| Native, Databricks-supported, GA | ✅ | ❌ third-party, public beta |
| Price | included | free core tier; paid tiers for Pro features |
| Deploy jobs / pipelines / workspace resources | ✅ — the whole point | ❌ out of scope |
| UC `schema` resource | ✅ namespace level | ✅ plus the table/column layer beneath it |
| Table/column-granularity diff vs live catalog | ❌ | ✅ — the compare engine is the product |
| Static, reviewable plan | partial — config can include Python, so the full state isn't statically diffable | ✅ plain SQL files in, classified diff out |
| Safety classification | ❌ applies what you declared | ✅ `SAFE` / `EXPENSIVE` / `DESTRUCTIVE` / `UNRECOVERABLE`; unrecoverable changes refuse without an explicit gate |
| Knows managed-table drop deletes files / streaming-table replace loses its checkpoint | ❌ | ✅ encoded in the classifier |
| Runs offline (no workspace credentials) | ❌ | ✅ build/compare/classify/lint |
| Extract a live catalog into `.sql` files | ❌ | ✅ `ddt extract` |
| Drift detection as a CI gate | ❌ | ✅ `ddt drift` |
| Import from dbt / Flyway / Liquibase / Terraform state | ❌ | ✅ |

## The two differences that matter most

1. **Granularity.** A bundle declares "this schema exists, run this pipeline." It has no opinion on whether `orders.customer_id` changed type, whether that change rewrites files, or whether anything is consuming the table as a stream. DDT operates exactly there — object by object, column by column, against the live catalog.
2. **Safety is nobody's job in a bundle deploy.** Bundles faithfully apply your declaration — including the one that drops a managed table (files deleted) or replaces a streaming table (checkpoint gone, and no `RESTORE` brings a checkpoint back). DDT classifies every change *before* anything executes and refuses the unrecoverable ones unless you explicitly opt in.

## Using them together

A clean split that works today:

```text
bundles/            ← jobs, pipelines, workspace config   (databricks bundle deploy)
lakehouse-schema/   ← tables, views, volumes as .sql      (ddt publish, safety-gated)
```

CI runs `ddt compare` offline on every PR — the classified diff is the review artifact — then `bundle deploy` and `ddt publish` each deploy their own layer.

---

*Independent comparison written by the DDT maintainer — corrections welcome via [issues](https://github.com/GVOrganization/ddt-tools/issues). Not affiliated with or endorsed by Databricks Inc. Claims reflect Bundles as of mid-2026; verify against current Databricks docs.*
