# 🩹 Troubleshooting

> Find the symptom you're hitting, see the likely cause, and apply the fix.

**On this page:** [Connection failures](#connection-failures) · [Extract issues](#extract-issues) · [Compare surprises](#compare-surprises) · [Deploy failures](#deploy-failures) · [Extension issues](#extension-issues) · [Still stuck?](#still-stuck)

---

## Connection failures

Run `ddt connection test <profile>` (or `ddt validate --profile <profile>`) first — it authenticates, lists visible catalogs, and probes the SQL warehouse, so it tells you exactly which step failed.

| Symptom | Cause | Fix |
|---|---|---|
| `401 Unauthorized` on test | PAT expired, or the wrong OAuth client secret. | Regenerate the PAT (default lifetime 90 days), or re-check the service-principal client secret. |
| `403 Forbidden` on `SHOW CATALOGS` | The principal lacks `USE CATALOG` in Unity Catalog. | Grant `USE CATALOG` (and `USE SCHEMA`) to the user / service principal. |
| `403 forbidden on the SQL Warehouse` | The principal owning the PAT doesn't have `CAN USE` on that warehouse. | Grant it from the SQL warehouse Permissions panel. |
| `Unity Catalog is not enabled` | The workspace was provisioned before UC. | Enable UC at the account console, or point DDT at a UC-enabled workspace. |
| `WORKSPACE_DOES_NOT_EXIST` | Wrong host. Hosts are workspace-scoped, not account-scoped. | Use the workspace host form `adb-<id>.<n>.azuredatabricks.net` (Azure) / `dbc-…cloud.databricks.com` (AWS/GCP). |
| `Could not find SQL warehouse` | Wrong workspace host, or the warehouse is in a different workspace. | Re-copy the warehouse ID from SQL → SQL Warehouses → Connection Details. |
| `503 Service Unavailable` on the first query | The SQL warehouse is cold-starting. | Retry in ~30s, or pre-warm the warehouse. If only the warehouse probe fails, it's probably stopped — start it from the SQL Warehouses page. |
| `redirect_uri_mismatch` (OAuth U2M) | The loopback port is in use by another process. | Restart DDT to pick a new port. |

> [!IMPORTANT]
> Secrets must use the `env:VAR_NAME` placeholder in the profile — raw secrets are rejected. If a connection can't find its credential, confirm the named environment variable is set in the shell that runs `ddt`. Full setup: [Connecting to Databricks](connections.md).

---

## Extract issues

| Symptom | Cause | Fix |
|---|---|---|
| A streaming table or MV is missing its refresh schedule | `SHOW CREATE TABLE` returns the `AS SELECT` body but **not** the refresh schedule metadata. | The extractor stitches in `DESCRIBE TABLE EXTENDED` for that metadata — make sure your principal can run it on the object. |
| A Lakeflow pipeline's config isn't in the `.sql` files | Pipeline config (clusters, schedule, notifications) lives in the workspace REST API, not in SQL DDL. | DDT extracts pipelines as JSON-like sidecars rather than DDL — that's expected; the sidecar is the source of truth for those fields. |
| Re-extracting wiped a manual edit | An object's live DDL changed, so extract rewrote that file. | Commit your project to git before re-extracting so you can review and merge changes. |

---

## Compare surprises

![Schema Compare demo](../assets/demo-compare.gif)

These are the "but they look the same!" cases. Unity Catalog's own behavior is usually the cause.

| Symptom | Cause | Fix |
|---|---|---|
| Names differ only by case and still flag | UC identifiers are case-insensitive in practice but **case-preserving in storage**; `SHOW CREATE TABLE` returns whichever case was used at create time. DDT treats case as significant by default. | Use `--ignore-case` (or `compare.ignoreCase: true`) — useful when porting from UPPER-cased output. |
| A `CLUSTER BY AUTO` table flags as drifted | `CLUSTER BY AUTO` (Predictive Optimization auto-selecting clustering keys) is a valid table state, not drift. | Current versions preserve `CLUSTER BY AUTO` and won't flag the auto-managed key set; upgrade if you still see it. |
| A liquid-clustering change shows as a warning, not an error | A change to **liquid clustering** columns is metadata-only (the next OPTIMIZE rewrites files). | Expected — it's flagged as a non-destructive rebuild warning, not a blocking change. |
| A partition-column change is blocked as a rebuild | A change to **partition columns** physically re-partitions the data files, requiring a full table rebuild. | Expected and gated by design — see Deploy failures below. |
| A materialized-view refresh mode flags as changed | DDT treats refresh-mode (`MANUAL` / `SCHEDULED` / `CONTINUOUS`) changes as a diff signal by default. | Set `compare.ignoreRefreshMode: true` if you manage the schedule outside the SQL body. |
| Comparing two environments whose catalog/schema names differ flags everything | The names genuinely differ. | Use logical-name mapping: `--map src=tgt` (add `--map-rewrite-strings` to also rewrite fully-qualified names inside object bodies). |
| Table properties / storage location always flag | Those legitimately differ between environments. | Use the matching ignore option (e.g. `ignoreTableProperties`, `ignoreStorageLocation`). |

---

## Deploy failures

Most "failures" here are DDT doing its job — refusing to destroy data silently. DDT is especially strict around UC's stateful objects.

| Symptom | Cause | Fix |
|---|---|---|
| `publish --apply` refuses, leaving destructive ops as `-- WARNING:` comments | **By design.** Destructive and unrecoverable changes are gated. | Review the safety report, then opt in with the matching gate. See the [Safety classifier](safety-classifier.md). |
| Dropping / rebuilding a streaming table is blocked as `UNRECOVERABLE` | A streaming table carries **two** pieces of unrecoverable state: the source offset (how far it's read) and the refresh checkpoint. Recreating it re-reads from earliest. | Only if you accept the loss, pass `--allow-unrecoverable-drop`. Otherwise rework the change to avoid a rebuild. |
| Dropping a Lakeflow pipeline is blocked as `UNRECOVERABLE` | `DROP PIPELINE` invalidates every downstream streaming table's checkpoint, even when no single table looks broken. | Same gate applies; confirm you intend to lose every downstream checkpoint first. |
| A managed-table drop warns about file deletion | For a **managed** table, `DROP TABLE` deletes the underlying files (an **external** table leaves them). | Confirm the table is meant to go; recover within the Delta retention window via `RESTORE TABLE` if you dropped one in error. |
| Changing a partition column is blocked as a rebuild | Partition columns are physical, so the change requires a full table rebuild. | Opt in with the table-rebuild gate only if a rebuild is acceptable. |
| You want one-command rollback after a deploy | — | Publish with `--manifest <path>` to capture an audit trail, then `ddt revert --manifest <path>`. Rollback of a column drop / type-narrow relies on Delta Time Travel, so the change must be within the table's retention window (`delta.logRetentionDuration`, default 30 days). |

See [Safe deploy](safe-deploy.md) for dry-run, apply, and rollback in depth.

---

## Extension issues

> [!NOTE]
> The DDT VS Code extension is browse / compare / review-focused. The project lifecycle (init / build / publish / extract) is CLI-first — if you're looking for those, run them with the `ddt` CLI.

| Symptom | Cause | Fix |
|---|---|---|
| The extension doesn't activate | VS Code version too old, or it needs a reload. | Requires VS Code 1.90+. Reload the window (**Developer: Reload Window** from the Command Palette). |
| A command isn't found in the Command Palette | The extension didn't finish activating, or the command is CLI-only. | Reload the window; confirm **DDT — Databricks Data Tools** is installed and enabled in the Extensions panel. For lifecycle commands, use the CLI. |
| You need diagnostic logs for a bug report | — | Open **View → Output** and select the DDT channel; include the relevant lines (with anything sensitive removed) in your report. |

---

## Still stuck?

Report it — bug reports directly shape the beta. There are three channels, all triaged from the same place:

1. **VS Code** — run **DDT: Report a Bug…** from the Command Palette (<kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>P</kbd>). Use this for wrong-output bugs that don't throw an error.
2. **CLI** — run `ddt feedback "<what happened>"` from any terminal. Add `ddt feedback --contact you@example.com` to be emailed when the bug is fixed.
3. **GitHub Issues** — open one at the [issue tracker](https://github.com/GVOrganization/ddt-tools/issues/new/choose) and pick the **Bug report** template. Search [existing issues](https://github.com/GVOrganization/ddt-tools/issues) first.

**What to include:**

- DDT version (`ddt --version`, or the extension version from the Extensions panel)
- Your OS and, for the CLI, your Node.js version
- Your cloud (AWS / Azure / GCP) and auth method (PAT / OAuth M2M), if relevant
- What you did, what you expected, and what actually happened
- Any error output, with anything sensitive removed

What you type via `feedback` / **Report a Bug** is sanitized before sending. For private reports, security issues, or data-deletion requests, email **sdt.ddt.tools@gmail.com**. Full guidance: [SUPPORT.md](../SUPPORT.md).

---

**Up:** [Documentation home](README.md)
