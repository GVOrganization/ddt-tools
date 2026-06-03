# 🚦 The safety classifier

> Understand exactly how DDT decides a change is safe, expensive, or dangerous — and which opt-in unlocks each blocked operation.

**On this page:** [What it does](#what-it-does) · [The categories](#the-categories) · [Unity Catalog risks](#unity-catalog-specific-risks) · [Opt-in gates](#opt-in-gates) · [Where findings appear](#where-findings-appear) · [Reversibility grouping](#reversibility-grouping)

---

## What it does

The safety classifier is the heart of DDT — Databricks Data Tools, and is byte-identical in shape to the one in its Snowflake sibling. Every change DDT plans is run through it before any SQL executes. It reads two inputs — the compare result and your resolved deploy options — and refuses to deploy unsafe operations unless you have explicitly opted in.

This is the *no silent destruction* guarantee: a destructive operation is never executed by accident. It is either unlocked by you, or it stays commented out and blocked.

---

## The categories

DDT classifies every change into one of four categories.

| Category | What it means | Example operations | What unlocks it |
|---|---|---|---|
| `UNRECOVERABLE` | State that cannot be reconstructed even from a `SHALLOW CLONE`. | Streaming-table checkpoint state, Lakeflow Declarative Pipeline schedules, registered-model versions, secret-scope secrets; dropping streaming tables / pipelines / jobs / warehouses. | `allowUnrecoverableDrop: true` |
| `DESTRUCTIVE` | Drops persistent rows or narrows column semantics. | `MANAGED_TABLE` / `EXTERNAL_TABLE` / `STREAMING_TABLE` drops, column drops, type-narrowing (`DECIMAL(38,9)` → `DECIMAL(10,2)`), `BLOCKED`-rebuild changes. | `allowDropTable`, `allowDropColumn`, `allowNarrowingTypes`, `allowTableRebuild` (per operation); blocked while `blockOnPossibleDataLoss: true` |
| `EXPENSIVE` | Full rewrites — costly but not data-losing. | Materialized-view rebuilds, `OPTIMIZE` on large Delta tables, Z-ORDER re-clustering. | Allowed by default; surfaced in the safety report. |
| `WARNING` | Adds of state-bearing objects, and drops skipped by `doNotDropObjectTypes`. | New streaming tables (they start running immediately on deploy). | Allowed by default; `treatWarningsAsErrors: true` blocks. |

---

## Unity Catalog-specific risks

Two Databricks risks deserve special attention, because the data loss is invisible until something downstream breaks.

> [!WARNING]
> **Managed-table file deletion.** Dropping a `MANAGED_TABLE` (or a `MANAGED` volume) deletes the underlying files in cloud storage. An `EXTERNAL_TABLE` drop only removes the Unity Catalog registration and leaves the files. Toggling MANAGED ↔ EXTERNAL storage is gated behind `allowStorageLocationChange`.

> [!WARNING]
> **Streaming-table checkpoint loss.** Dropping a `STREAMING_TABLE` is `UNRECOVERABLE`: the checkpoint holds the offset into the source, and it cannot be reconstructed without re-running the pipeline from scratch. The same applies to Lakeflow Declarative Pipelines, which hold materialization state for every downstream streaming table — which is why DDT alters pipelines in place rather than recreating them.

---

## Opt-in gates

A blocked operation reports which gate it needs. Set the gate as a `deployOptions` key, or pass the matching `ddt publish` flag.

| `deployOptions` key | CLI flag | What it unlocks |
|---|---|---|
| `allowDropTable` | `--allow-drop-table` | `DROP TABLE` on DESTRUCTIVE objects. |
| `allowDropColumn` | `--allow-drop-column` | `ALTER TABLE … DROP COLUMN`. |
| `allowNarrowingTypes` | `--allow-narrowing` | Narrowing type changes (`DECIMAL(38,9)` → `DECIMAL(10,2)`). |
| `allowTableRebuild` | `--allow-table-rebuild` | Full DROP+CREATE rebuilds (clustering / partition changes). |
| `allowUnrecoverableDrop` | `--allow-unrecoverable-drop` | Drops of streaming tables / Lakeflow pipelines / jobs. |
| `allowStorageLocationChange` | (deployOptions) | MANAGED ↔ EXTERNAL storage transitions. |

Two top-level options control the overall posture:

| Option | Default | Effect | CLI equivalent |
|---|---|---|---|
| `blockOnPossibleDataLoss` | `true` | Refuse any deploy with a destructive change. | `--fail-on-destructive` |
| `treatWarningsAsErrors` | `false` | Promote every WARNING finding to a blocker. | `--strict` |

These confirmation gates layer on top:

| Gate | Satisfied by |
|---|---|
| `REQUIRE_YES_FLAG` | `--yes` on the CLI. |
| `REQUIRE_CONFIRM_PRODUCTION` | `--confirm-production` (required for prod-named profiles). |

> [!WARNING]
> Unlocking a gate does not make the operation safe — it only tells DDT you accept the consequence. A `DROP TABLE` on a managed table with `allowDropTable: true` still deletes the underlying files.

---

## Where findings appear

The same classification surfaces in three places.

**In compare results** — `ddt compare` prints findings inline, color-coded by category: UNRECOVERABLE in bold red, DESTRUCTIVE in red, EXPENSIVE in yellow, WARNING in cyan, SAFE/OK in green.

**In migration scripts** — any non-`SAFE` drop is emitted as a commented-out statement with a `-- WARNING:` note, and blocked operations carry a `-- BLOCKED:` line naming the gate that would enable them:

```sql
-- WARNING: dropping a STREAMING TABLE is unrecoverable — checkpoint state is lost.
-- BLOCKED: set allowUnrecoverableDrop=true to enable.
-- DROP TABLE main.gold.events_stream;
```

**In CLI exit codes** — for CI gating:

| Code | Meaning |
|---|---|
| 0 | Success |
| 3 | Compare diff found (flag-controlled) |
| 4 | Destructive changes detected and not approved |
| 5 | Drift detected (`ddt drift` in CI mode) |

---

## Reversibility grouping

Category answers *how dangerous?* Reversibility answers a different question: *if this goes wrong, can I undo it?* The two axes don't always agree — a DESTRUCTIVE change can be reversible (Delta Time Travel within retention), and a WARNING change can be irreversible (a streaming table once it is running).

Both `ddt compare` and `ddt publish --dry-run` print a reversibility summary after the per-object diff:

| Bucket | Glyph | Contents |
|---|---|---|
| Unrecoverable | 🛑 | UNRECOVERABLE findings — `DROP STREAMING_TABLE`, `DROP PIPELINE`, etc. |
| Data-impacting | ⚠ | DESTRUCTIVE + EXPENSIVE findings — can lose or rewrite data. |
| Reversible | ℹ | WARNING findings — visible but recoverable. |

```text
🛑 UNRECOVERABLE (1)
  • DROP STREAMING TABLE main.gold.events_stream
    → checkpoint state lost; cannot resume from offset

⚠ DATA-IMPACTING (2)
  • ALTER TABLE main.gold.orders DROP COLUMN memo
    → column data lost; revert requires Time Travel within
      delta.logRetentionDuration
  • REBUILD main.gold.customer_agg (Z-ORDER columns changed)
    → 4.3M rows × DBU rewrite cost

ℹ REVERSIBLE (1)
  • ALTER TABLE main.gold.orders ADD COLUMN status STRING
    → safe; can be dropped if needed
```

---

**Next:** [Configuration reference](configuration.md) · **Up:** [Documentation home](README.md)
