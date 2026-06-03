# 🛡️ Safe deploy

> Build your project into an artifact, preview the exact SQL, and apply it to Databricks Unity Catalog with destructive changes blocked unless you opt in.

**On this page:** [The deploy loop](#the-deploy-loop) · [Dry-run, then apply](#dry-run-then-apply) · [How blocked changes look](#how-blocked-changes-look) · [Opt-in gates](#opt-in-gates) · [Production safeguards](#production-safeguards) · [Rollback](#rollback)

---

## The deploy loop

![DDT safe deploy demo](../assets/demo-deploy.gif)

DDT — Databricks Data Tools deploys in four steps. Each step is its own command, so you always see what will happen before anything touches the workspace.

1. **Build** — compile the `.ddtproj` into a `.ddtpac` artifact.
2. **Dry-run** — generate the migration SQL and the safety assessment, execute nothing.
3. **Apply** — run the migration against the connection, capturing a manifest for revert.
4. **Roll back** — if needed, revert step-by-step from the manifest, or restore from a pre-deploy snapshot.

> [!IMPORTANT]
> DDT is state-based: you describe the desired state in `.sql` files and DDT computes the deltas. Re-running `ddt publish` against an up-to-date target is a no-op.

> [!NOTE]
> The DDT VS Code extension is browse / compare / review focused. The project lifecycle (init / build / publish / extract) is CLI-first — run it from a terminal.

---

## Dry-run, then apply

Always dry-run first. The dry-run prints the migration SQL exactly as it will run, plus the safety assessment.

```sh
# 1. Build the project into a .ddtpac artifact
ddt build --project ./MyProject.ddtproj --out ./bin/MyProject.ddtpac

# 2. Dry-run: generate the script + safety assessment, execute nothing
ddt publish --source ./bin/desired.ddtpac --target ./bin/current.ddtpac --dry-run

# 3. Apply for real, capturing a manifest you can revert from later
ddt publish --source ./bin/desired.ddtpac --target ./bin/current.ddtpac \
            --apply --connection prod --yes \
            --manifest ./deploy-2026-05-14.json
```

| Flag | What it does |
|---|---|
| `--dry-run` | Print the migration SQL; don't execute. |
| `--apply` | Execute the migration against the connection. |
| `--yes` | Confirm a destructive or non-interactive deploy. |
| `--manifest <path>` | Capture a deploy manifest so the deploy can be reverted. |
| `--require-reversible` | Refuse the deploy if any step has no captured reverse SQL. |
| `--no-lint` | Skip the pre-deploy lint gate. |

---

## How blocked changes look

DDT never silently destroys data. When the migration contains a destructive or unrecoverable operation that you have not unlocked, DDT emits the statement **commented out** with a `-- WARNING:` note explaining why, and the deploy is blocked.

```sql
-- WARNING: DROP TABLE is destructive — Delta data is lost.
-- BLOCKED: set allowDropTable=true (deployOptions) or pass --allow-drop-table to enable.
-- DROP TABLE main.gold.orders_old;
```

The dry-run output ends with a safety report grouped by reversibility:

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

For the full category model, see [The safety classifier](safety-classifier.md).

---

## Opt-in gates

Destructive operations stay blocked until you explicitly allow them. Set the gate in your project's `deployOptions` (per-environment, under `deploymentProfiles.<env>.deployOptions`), or pass the matching CLI flag.

| You want to allow | `deployOptions` key | `ddt publish` flag |
|---|---|---|
| `DROP TABLE` | `allowDropTable: true` | `--allow-drop-table` |
| Column drops | `allowDropColumn: true` | `--allow-drop-column` |
| Narrowing a column type | `allowNarrowingTypes: true` | `--allow-narrowing` |
| Full DROP+CREATE rebuilds | `allowTableRebuild: true` | `--allow-table-rebuild` |
| Dropping streaming tables / pipelines / jobs | `allowUnrecoverableDrop: true` | `--allow-unrecoverable-drop` |
| MANAGED ↔ EXTERNAL storage transitions | `allowStorageLocationChange: true` | (set in deployOptions) |

```jsonc
// In MyProject.ddtproj — loosen only the dev profile
"deploymentProfiles": {
  "dev": {
    "connection": "dev",
    "deployOptions": {
      "deployment": {
        "blockOnPossibleDataLoss": false,
        "allowDropTable": true,
        "allowTableRebuild": true
      }
    }
  }
}
```

> [!WARNING]
> `blockOnPossibleDataLoss` is `true` by default and refuses any deploy with a destructive change. Turning it off in a production profile removes the single biggest guard DDT gives you. Loosen gates per environment, never globally.

The full set of gates and exactly what each unlocks is documented in [The safety classifier](safety-classifier.md) and the [Configuration reference](configuration.md).

---

## Production safeguards

When the active profile name matches `prod`, `prd`, `production`, or `live` (or you pass `--production`, or the profile sets `production: true`), DDT requires both `--yes` **and** `--confirm-production` for any destructive operation.

```sh
ddt publish --source ./bin/desired.ddtpac --target ./bin/current.ddtpac \
            --apply --connection prod --yes \
            --confirm-production \
            --manifest ./deploy-2026-05-14.json
```

> [!WARNING]
> `--confirm-production` is a deliberate second confirmation, not a formality. DDT requires it precisely because the target is a profile that looks like production.

Set `treatWarningsAsErrors: true` (or pass `--strict`) to promote every WARNING finding to a blocker — a good CI-strict default.

---

## Rollback

Unlike Snowflake, Databricks Unity Catalog has **no zero-copy clone at the catalog or schema level**. DDT relies on per-table facilities and manifest-based revert instead.

### Per-table recovery facilities

| Facility | Scope | Cost | Recovery story |
|---|---|---|---|
| Delta `SHALLOW CLONE` | One table at a time | Metadata-only; cheap. | Snapshot a table before a risky `ALTER`. |
| Time Travel | One table at a time | Built-in (retention-window dependent). | `RESTORE TABLE x TO TIMESTAMP AS OF …`. |

> [!NOTE]
> Because there is no database-level clone, the DDT pre-deploy-clone facility is not the same as SDT's auto-clone-on-apply. The recommended rollback path on DDT is manifest-based revert, below.

### Revert from a manifest

Every `ddt publish --apply --manifest <path>` captures the forward and reverse SQL. `ddt revert` replays each step's reverse statement against the workspace in reverse order — the recommended way to roll back a DDT deploy.

```sh
# Preview what the revert would run
ddt revert --manifest ./deploy-2026-05-14.json --connection prod --yes --dry-run

# Apply the revert
ddt revert --manifest ./deploy-2026-05-14.json --connection prod --yes

# Keep going past a failing reverse statement (default: stop on first failure)
ddt revert --manifest ./deploy-2026-05-14.json --connection prod --yes --continue-on-error
```

> [!WARNING]
> Revert is itself destructive — `--yes` is mandatory. Steps with no captured reverse SQL (DROPs, for example) are inherently irreversible and are skipped with a warning; the exit code is non-zero when any step could not be reversed.

### Restore from a pre-deploy snapshot

`ddt publish --apply` writes a pre-deploy snapshot batch backed by `SHALLOW CLONE`. Restore from it with a single command — this is the command DDT prints when a deploy fails.

```sh
# Restore from the snapshot batch (dry-run by default)
ddt publish --restore-from-snapshot snap_lqz3k_4f8a2c \
            --connection prod --apply --yes
```

Manage the snapshot registry with `ddt snapshot list / show / prune`.

---

**Next:** [The safety classifier](safety-classifier.md) · **Up:** [Documentation home](README.md)
