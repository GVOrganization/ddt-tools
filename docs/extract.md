# 📥 Extract a live workspace

> Reverse-engineer a connected Databricks Unity Catalog workspace into a project tree of `.sql` files you can version, diff, and deploy.

**On this page:** [What extract does](#what-extract-does) · [CLI usage](#cli-usage) · [VS Code usage](#vs-code-usage) · [What gets extracted](#what-gets-extracted) · [Re-extract and refresh](#re-extract-and-refresh)

---

## What extract does

![Extract demo](../assets/demo-extract.gif)

`ddt extract` connects to a live Databricks workspace and writes one `.sql` file per Unity Catalog object at a canonical path. The result is a working project tree — the same shape `ddt init` scaffolds — that you can build into a `.ddtpac`, compare against the workspace, and deploy.

It is the fastest way to bring an existing workspace under source control. Point it at a connection profile, choose the scope, and DDT walks every catalog, schema, and object in range.

Each object lands at:

```
<output>/catalogs/<CAT>/schemas/<SCHEMA>/<plural>/<NAME>.sql
```

Object types the path registry doesn't recognize fall into `<output>/_unsorted/` — the writer never refuses an object.

> [!TIP]
> Run extract into a fresh, empty directory the first time. You get a complete, round-trippable project you can immediately `ddt build`.

---

## CLI usage

Project lifecycle in DDT is **CLI-first** — `ddt extract` is the entry point. Register a connection first (see [Connections](connections.md)), then extract:

```sh
# Capture one catalog into a new project tree
ddt extract --connection prod \
            --catalog main \
            --output ./MyProject

# Extract straight into a build artifact (extract + build in one step)
ddt extract --connection prod \
            --catalog main \
            --out-pac ./bin/MyProject.ddtpac \
            --project-name MyProject
```

| Flag | What it does | Notes |
|---|---|---|
| `--connection <name>` | The Databricks workspace connection profile to read from. | Required. Register with `ddt connection add`. |
| `--catalog <catalog>` | Limit extraction to one Unity Catalog catalog. | The DDT equivalent of SDT's `--db`. |
| `--output <dir>` | Where to write the project tree. | Round-trippable with `ddt build`. |
| `--out-pac <file>` | Write a `.ddtpac` build artifact directly, instead of (or alongside) a tree. | Combines extract + build in one step. |
| `--project-name <name>` | Stamps the pac manifest when `--out-pac` is used. | — |

After a tree extract, build it to confirm it round-trips:

```sh
ddt build --project ./MyProject.ddtproj --out ./bin/MyProject.ddtpac
```

---

## VS Code usage

The DDT VS Code extension is **browse / compare / review-focused** — project lifecycle (init, build, publish, extract) is driven from the CLI. To capture a workspace, run `ddt extract` from a terminal, then open the resulting project folder in VS Code to browse and edit it.

Once the project is open, the editor flags missing-column references inline as you edit any `.sql` file inside the `.ddtproj`.

---

## What gets extracted

DDT models Unity Catalog objects. Each object type is captured one of two ways:

- **Fully modeled** — DDT understands the object's structure and captures it with a hand-tuned extractor (table classification, columns, properties).
- **Extracted as-is (DDL)** — DDT captures the object's DDL verbatim. Compare is DDL-string equality; this is enough until a richer model is needed.

Coverage by object family:

| Object family | Examples | Coverage |
|---|---|---|
| Tables | `MANAGED_TABLE`, `EXTERNAL_TABLE` | Fully modeled (table classification from the UC Tables REST API); diff and migration via DDL |
| Foreign tables | `FOREIGN_TABLE` | Fully modeled, read-only — the source system owns the schema |
| Streaming & MV | `STREAMING_TABLE`, `MATERIALIZED_VIEW` | Fully modeled (refresh-state-aware); diff and migration via DDL |
| Compute policies | `CLUSTER_POLICY` | Fully modeled (REST descriptor); built-in default policies are never dropped |

> [!NOTE]
> Many Unity Catalog types — `VIEW`, `VOLUME`, `FUNCTION`, `ROUTINE`, catalogs, schemas, Delta Sharing objects, and account-level identities — are modeled but not yet extracted. See the full per-type status table in the source repository for an exact breakdown.

> [!IMPORTANT]
> Secret values are never extracted. For secret scopes, only scope metadata is captured — the values themselves stay in the workspace.

---

## Re-extract and refresh

Re-run `ddt extract` against an existing project to refresh it from the live workspace. Re-extraction overwrites object files with the workspace's current DDL.

The safer refresh path once a project is under source control: **compare first** to see exactly what changed, then decide what to pull.

```sh
# Snapshot the live workspace into a pac, build your project into a pac,
# then compare the two (compare is pac ↔ pac).
ddt build -p ./MyProject.ddtproj                                  # → ./bin/MyProject.ddtpac
ddt extract --connection prod --out-pac ./bin/prod-live.ddtpac
ddt compare --source ./bin/MyProject.ddtpac \
            --target ./bin/prod-live.ddtpac
```

> [!IMPORTANT]
> Re-extracting into a git-tracked project, then reviewing the diff, is the safest refresh path — you see every change before committing it.

See [Schema compare](schema-compare.md) for the full compare workflow.

---

**Next:** [Schema compare](schema-compare.md) · **Up:** [Documentation home](README.md)
