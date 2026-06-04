# 📁 Projects & suites

> Author a Unity Catalog schema as a folder of `.sql` files, build it into a portable artifact, and stitch multiple projects into one deploy bundle.

**On this page:** [The project file](#the-project-file) · [Folder layout](#folder-layout) · [Project scope](#project-scope) · [Variables & deployment profiles](#variables--deployment-profiles) · [Pre/post deploy scripts](#prepost-deploy-scripts) · [Building a `.ddtpac`](#building-a-ddtpac) · [Slices](#slices--owning-part-of-a-shared-metastore) · [Suites](#suites--grouping-projects)

---

## The project file

A DDT (Databricks Data Tools) project is a folder of `.sql` files — one object per file — described by a single JSON manifest with the `.ddtproj` extension. The manifest declares the project's identity, what part of Unity Catalog it owns, and how it deploys to each environment.

> [!NOTE]
> The VS Code extension is browse / compare / review focused. The full project lifecycle — `init`, `build`, `publish`, `extract` — is CLI-first with the `ddt` command.

A minimal `my-project.ddtproj`:

```jsonc
{
  "$schema": "https://ddt.dev/schemas/ddtproj/v1.json",
  "name": "MyProject",
  "version": "0.1.0",
  "targetPlatform": { "platform": "Databricks", "edition": "Premium" },
  "scope": { "type": "metastore" }
}
```

That's enough for `ddt validate` to accept. A fuller project with per-environment profiles and deploy options:

```jsonc
{
  "$schema": "https://ddt.dev/schemas/ddtproj/v1.json",
  "name": "MyProject",
  "version": "0.1.0",
  "targetPlatform": { "platform": "Databricks", "edition": "Premium" },
  "scope": { "type": "catalog", "catalog": "main" },
  "include": ["**/*.sql", "**/*.json"],
  "exclude": ["bin/**"],
  "defaults": { "warehouseName": "etl_wh", "defaultCatalog": "main" },
  "preDeployScripts": ["scripts/pre/*.sql"],
  "postDeployScripts": ["scripts/post/*.sql"],
  "deploymentProfiles": {
    "dev":  { "connection": "dev-workspace",  "variables": { "ENV_PREFIX": "dev_" } },
    "prod": { "connection": "prod-workspace", "production": true, "variables": { "ENV_PREFIX": "" } }
  }
}
```

> [!TIP]
> The `$schema` field is a forward-looking hint for editors. Once the schema is
> hosted at that URL, your editor will offer hover-help and JSON validation as
> you type; the field is otherwise inert and safe to keep.

### Fields

| Field | What it does | Notes / default |
|---|---|---|
| `$schema` | URL of the JSON schema | Advisory; enables IDE autocomplete once the schema is hosted. `https://ddt.dev/schemas/ddtproj/v1.json` |
| `name` | Project name | Required. Stamped into `.ddtpac` manifests and error messages |
| `version` | SemVer string | Required. Bumped per release |
| `targetPlatform` | `{ platform: 'Databricks', edition?, minRuntime? }` | Required. `edition` is `Standard` / `Premium` / `Enterprise` |
| `scope` | What the project owns — see [Project scope](#project-scope) | Required |
| `include` | Glob list of SQL/JSON files | Default `['**/*.sql', '**/*.json']` |
| `exclude` | Globs excluded from `include` | Default `['bin/**', 'node_modules/**', '.ddt-cache/**']` |
| `defaults` | `{ warehouseName?, defaultCatalog? }` | Optional |
| `preDeployScripts` | Globs run before the body | Optional |
| `postDeployScripts` | Globs run after the body | Optional |
| `deploymentProfiles` | Per-environment overlays — see [Variables & deployment profiles](#variables--deployment-profiles) | Optional |
| `deployOptions` | Project-wide compare + deployment options | Optional |
| `references` | Cross-project references `[{ path, alias? }]` | Optional |
| `codeAnalysis` | `{ enabled, rules? }` — drives `ddt lint` | Optional |
| `slice` | Project Slice — see [Slices](#slices--owning-part-of-a-shared-metastore) | Optional |

The schema is strict — unknown fields are rejected at load time, so typos surface immediately.

---

## Folder layout

The folder layout is a convention: the project loader maps each file's path to an object's identity (catalog / schema / type / name) rather than reading file contents first. Keep files in their canonical folders.

```
my-project/
├── my-project.ddtproj
├── catalogs/
│   └── main/
│       ├── catalog.sql              -- CREATE CATALOG
│       └── schemas/
│           └── gold/
│               ├── schema.sql
│               ├── tables/
│               │   └── fact_sales.sql
│               └── views/
│                   └── v_sales.sql
├── seeds/                           -- optional declarative seed data
└── bin/                             -- gitignored; ddt build output
    └── my-project.ddtpac
```

Each `.sql` file contains **exactly one** object definition, using `CREATE OR REPLACE ...` — the Databricks idiom for Delta / Iceberg tables, views, and UC functions.

> [!WARNING]
> Replacing or dropping a **managed table** deletes its underlying data files. Editing a **streaming table** so it drops and recreates loses the checkpoint. The safety classifier flags both as UNRECOVERABLE — read the finding before applying. See the [Safety classifier](safety-classifier.md).

---

## Project scope

`scope` is a hard fence: `extract` only walks objects inside it, and `compare` ignores anything outside it. A tighter scope means a faster compare and a safer deploy.

| Scope `type` | What DDT operates on | Required fields |
|---|---|---|
| `metastore` | All catalogs the connection can see — the largest blast radius | `type` only |
| `catalog` | One catalog and every schema inside it | `catalog` |
| `schema` | One schema — the single-team default | `catalog` + `schema` |

```jsonc
{ "scope": { "type": "schema", "catalog": "main", "schema": "gold" } }
```

---

## Variables & deployment profiles

A **deployment profile** pairs a connection name with variables and optional deploy-option overrides for one environment. Profiles live under `deploymentProfiles` in the `.ddtproj`; the `connection` name resolves to a configured connection in `~/.ddt/profiles.json`.

```jsonc
{
  "deploymentProfiles": {
    "dev": {
      "connection": "dev-workspace",
      "variables": { "ENV_PREFIX": "dev_", "RETENTION_DAYS": "1" },
      "deployOptions": { "deployment": { "allowDropTable": true } }
    },
    "prod": {
      "connection": "prod-workspace",
      "production": true,
      "variables": { "ENV_PREFIX": "", "RETENTION_DAYS": "30" },
      "deployOptions": {
        "deployment": { "blockOnPossibleDataLoss": true, "treatWarningsAsErrors": true }
      }
    }
  }
}
```

Each profile has:

- `connection` (required) — a connection name in `~/.ddt/profiles.json`.
- `variables` (optional) — `Record<string, string>` for substitution.
- `production` (optional) — when `true`, applies the production safety floor regardless of profile name. The classifier treats a profile as production when its name matches `prod` / `prd` / `production` / `live`, or when `production: true` is set.
- `deployOptions` (optional) — a partial overlay merged on top of the project-wide `deployOptions`.

### `$(VAR)` substitution

In raw `.sql` files, write `$(VAR_NAME)` and the value is substituted into the generated migration script at deploy time. Substitution is textual — wrap quoted strings yourself when needed.

```sql
USE CATALOG $(CATALOG);

CREATE OR REPLACE TABLE $(ENV_PREFIX)orders (
  id BIGINT,
  ts TIMESTAMP
)
USING DELTA
TBLPROPERTIES (
  'delta.logRetentionDuration' = 'interval $(RETENTION_DAYS) days'
);
```

`$(VAR)` is wired into `ddt script --variables KEY=VALUE,...` and `ddt promote` (which passes the profile's variables automatically).

### Resolution order

Later sources win:

1. Built-in variables (below).
2. Project-level `defaults`.
3. Deployment-profile `variables`.
4. `--variables KEY=VALUE,...` on `ddt script`.

### Built-in variables

| Name | Value |
|---|---|
| `DDT_PROFILE` | The active profile name |
| `DDT_PLATFORM` | Always `databricks` for DDT |
| `DDT_USER` | The Databricks user / service principal that ran the deploy |
| `DDT_TIMESTAMP` | ISO 8601 UTC at deploy start |
| `DDT_PROJECT_NAME` | From `.ddtproj.name` |
| `DDT_PROJECT_VERSION` | From `.ddtproj.version` |

---

## Pre/post deploy scripts

`preDeployScripts` run before the generated migration body; `postDeployScripts` run after it. Both take glob lists. Use them for grants, warehouse setup, or data backfills that bracket the schema change.

```jsonc
{
  "preDeployScripts": ["scripts/pre/*.sql"],
  "postDeployScripts": ["scripts/post/*.sql"]
}
```

`$(VAR)` substitution applies inside these scripts just as it does in object files.

---

## Building a `.ddtpac`

A `.ddtpac` is a self-contained, deterministic build artifact of your `.ddtproj` — a ZIP container you can hand to CI, sign, archive, or deploy without the original source tree.

```sh
ddt build -p ./my-project.ddtproj
# Built ./bin/my-project.ddtpac
```

A `.ddtpac` holds four sections:

| Entry | What it contains |
|---|---|
| `manifest.json` | Pac metadata + build provenance — name, version, scope, target platform, `builtAt`, `builtBy`, object count, format version |
| `model.json` | The extracted/parsed model array, each entry tagged with `objectType` and an `fqn` |
| `source/` | Verbatim copies of every authored `.sql` / `.json`, preserving the folder layout |
| `checksums.json` | A SHA-256 per source file and per model object |

Why it exists: the pac is the unit of deployment in CI/CD. `ddt publish --source <file>.ddtpac --target <current>.ddtpac` deploys the diff and `ddt verify --pac <file>` audits it — re-hashing every source and model entry against `checksums.json` to confirm the pac wasn't tampered with after build.

The build is **deterministic**: file entries are sorted by path, the model array is sorted by `(objectType, fqn)`, and ZIP timestamps are pinned. Two builds of the same project produce byte-identical pacs apart from the `builtAt` timestamp — which is what makes signed-pac distribution feasible.

> [!NOTE]
> The pac format is versioned with a strict-equality check. A reader rejects a pac whose format version it doesn't understand rather than guessing — rebuild it with a matching version of the CLI.

---

## Slices — owning part of a shared metastore

When several teams share one metastore, a full-scope project is dangerous: `publish` would propose dropping anything on the target that isn't in your source. A **Slice** narrows a project to only the objects it owns.

Add a `slice` block to the `.ddtproj`:

```jsonc
{
  "slice": {
    "owns": ["main.gold.*", "main.marketing_*"],
    "reads": ["main.finance.dim_account", "governance.tag.*"]
  }
}
```

| Field | What it does |
|---|---|
| `owns` | Glob patterns matched against the FQN. Matching objects are in scope for diff and deploy |
| `reads` | Read-only references the project may mention (foreign keys, view dependencies) but never mutate |
| `caseSensitive` | When `false` (the DDT default), matches case-insensitively |

Glob rules: `*` matches within one identifier segment, `**` matches across segments, plain identifiers match literally.

With a Slice in place:

- Objects on the target that don't match `owns` are partitioned out of the diff — they never show as added, removed, or modified.
- Objects matching `reads` but not `owns` are read-only — the engine refuses to emit any DDL touching them.
- `ddt compare` prints an informational footer listing what was left untouched.

| Command | Slice behavior |
|---|---|
| `ddt compare` | Honors `slice` by default. Pass `--no-slice` to treat the project as full-scope |
| `ddt publish` | Honors `slice` by default. `--no-slice` works the same way |
| `ddt validate` | Compiles the slice and flags any overlap between `owns` and `reads` patterns |

When a `.ddtpac` is built from a sliced project, the slice is captured in its `manifest.json`, so `ddt publish` honors the same boundary.

---

## Suites — grouping projects

A **Suite** is a `.ddtsuite` file that aggregates multiple Slice-aware projects into one deployable unit and validates that they fit together cleanly.

```json
{
  "$schema": "https://ddt.dev/schemas/ddtsuite/v1.json",
  "suiteName": "Lakehouse Platform",
  "version": "1.0.0",
  "platform": "Databricks",
  "projects": [
    { "alias": "core", "path": "./projects/Core/Core.ddtproj" },
    { "alias": "product", "path": "./projects/Product/Product.ddtproj" }
  ],
  "deployOrder": "auto"
}
```

Scaffold and validate a suite:

```sh
ddt suite init --output ./Platform.ddtsuite       # scaffold a new suite
ddt suite validate --suite ./Platform.ddtsuite    # run cross-project checks
```

`ddt suite validate` runs five integrity checks across the projects:

1. **Ownership coverage** — every Unity Catalog object is owned by exactly one project's Slice; overlapping `owns` patterns are an error.
2. **Reference resolution** — every `reads` edge points at a real project + object.
3. **Cycle detection** — no `reads` cycle across projects.
4. **Deploy ordering** — a topological sort across all projects so dependencies deploy first.
5. **Coverage** — every project file is reached from at least one Slice.

> [!TIP]
> Both Slices and Suites are Pro-tier features. During the public beta, all paid features are unlocked.

### In VS Code

The extension ships a Suite tree view in the activity bar showing every open `.ddtsuite` and the projects inside. Run **DDT: Validate Suite** from the Command Palette to run the checks and see findings inline in the Problems panel. Project entries carry a `(scoped)` badge when they have a `slice`, or `(full)` when they own the entire target.

---

**Next:** [Extract](extract.md) · **Up:** [Documentation home](README.md)
