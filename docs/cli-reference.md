# ⌨️ CLI reference

> The complete `ddt` command catalog, organized by the workflow you're in — author a project, connect, deploy safely, analyze, and ship through CI.

**On this page:** [Discovery](#discovery) · [Global flags](#global-flags) · [Exit codes](#exit-codes) · [Output formats](#output-formats) · [Project lifecycle](#project-lifecycle) · [Connections](#connections) · [Safety & rollback](#safety--rollback) · [Code quality](#code-quality) · [Analysis](#analysis) · [Data](#data) · [CI/CD](#cicd) · [AI](#ai) · [Migration & interop](#migration--interop) · [Utilities](#utilities)

---

The `ddt` command surface mirrors `sdt` by design — same lifecycle phases, same flags, same exit codes. The deltas are the project extension (`.ddtproj`) and the build artifact (`.ddtpac`). (A couple of commands differ in shape today — `ddt compare` is `.ddtpac ↔ .ddtpac` only, and `ddt publish` takes `--source`/`--target` rather than `--pac`; both are called out below.)

## Discovery

You never have to memorize this page. Every command, flag, and config key is searchable from the terminal.

```sh
ddt explain "drop table"        # search the options catalog by topic
ddt explain --list              # dump every option
ddt find "drift detection"      # "what can DDT do for X?"
ddt <command> --help            # per-command usage + flags
```

> [!TIP]
> `ddt explain <anything>` and `ddt <command> --help` are the fastest path to discovery — start there before scanning the tables below.

> [!NOTE]
> A few commands are still being wired in the current beta. `ddt exec` is a headless-path stub that today directs you to the VS Code query window; `ddt search` directs you to run `ddt extract` first. `ddt format` defaults to the battle-tested `legacy` keyword engine; pass `--engine v2` to opt into the tokenizer-based engine. Connection auth beyond PAT and OAuth M2M (`oauth-u2m`, `azure-ad`, `google-idc`) is still landing. These are marked **(beta)** in the tables below.

---

## Global flags

These apply to most commands.

| Flag | What it does |
|---|---|
| `--output table\|json\|yaml\|sql` | Output format; defaults to `table` for humans. |
| `--explain` | Narrate the result through your configured AI provider (on supported commands). |
| `--quiet` / `--verbose` / `--debug` | Log verbosity. |
| `--no-color` | Disable ANSI color (also honors `NO_COLOR` / `DDT_NO_COLOR`). |
| `--yes` / `--confirm` | Required for write operations in non-interactive mode (CI or piped stdin). |

Structured output goes to stdout; logs go to stderr. In non-interactive mode (no TTY or `CI=true`) DDT never prompts and never opens a browser — pass `--yes` explicitly.

---

## Exit codes

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | User error (bad args, missing file, syntax error) |
| 2 | System error (network, permission, internal failure) |
| 3 | Compare diff found (CI drift detection, flag-controlled) |
| 4 | Destructive changes detected and not approved |
| 5 | Drift detected (`ddt drift` in CI mode) |
| 6 | Locked feature attempted without a Pro/Team/Enterprise license |

---

## Output formats

`--output table | json | yaml | sql` controls the global format. Several commands also take `--format text | json | markdown | sarif` (lint, diagnose) or `--format table | json | markdown` (cost-estimate). SARIF output is consumable by GitHub code-scanning and Azure DevOps.

---

## Project lifecycle

The everyday loop: author a `.ddtproj`, build it to a `.ddtpac`, deploy it, observe what changed. See [Projects & suites](projects.md), [Extract](extract.md), [Schema compare](schema-compare.md), [Safe deploy](safe-deploy.md).

| Command | What it does | Key flags |
|---|---|---|
| `ddt init` | Scaffold a new project (`<name>.ddtproj` + folder layout). | `--name`, `--scope metastore\|catalog\|schema`, `--catalog`, `--schema`, `--dir` |
| `ddt validate` | Schema-shape check; with `--references` runs the build-time semantic resolver. | `--project`, `--references`, `--columns`, `--check-variables`, `--min-severity`, `--format`, `--out` |
| `ddt extract` | Reverse-engineer a live Unity Catalog workspace into per-object `.sql` files. | `--connection`, `--catalog`, `--output`, `--out-pac`, `--project-name` |
| `ddt build` | Compile a `.ddtproj` to a `.ddtpac` build artifact. | `-p, --project` (required), `-o, --out` |
| `ddt publish` | Deploy a `.ddtpac` to a connection; honors every safety gate. | `--source`, `--target`, `--connection`, `--dry-run`, `--apply`, `--yes`, `--manifest`, `--profile`, `--variables`, `--restore-from-snapshot`, `--report-html`, `--webhook`, `--changelog`, `--no-lint`, `--require-reversible`, `--no-rollback`, `--map`, `--map-file` |
| `ddt compare` | Diff two `.ddtpac` build artifacts (`pac ↔ pac`; project/live compare pending v0.3 — `ddt build` / `ddt extract --out-pac` each side first). | `--source`, `--target`, `--format`, `--json`, `--explain`, `--color`, `--ignore-case`, `--no-slice`, `--type-safe`, `--break-on`, `--write-impact`, `--report-html`, `--map`, `--map-file`, `--no-history` |
| `ddt drift` | Compare the live workspace to what the project expects; report only. | `--source`, `--connection`, `--catalog`, `--schema` |
| `ddt script` | Generate the migration SQL without executing it (always offline). | `--source`, `--target`, `-o, --out`, `--profile`, `--variables`, `--format`, `--banner`, `--color`, `--ignore-case`, `--report-html`, `--map`, `--map-file` |
| `ddt promote` | Branch-per-env deploy automation; diff two live envs, emit a bundle, optionally open a PR. | `--from`, `--to`, `--open-pr`, `--repo`, `--base` |
| `ddt refresh` | Generate a post-deploy REFRESH script (streaming tables + MVs + pipelines). | `--source`, `-o`, `--no-streaming-tables`, `--no-materialized-views`, `--no-pipelines` |
| `ddt exec` **(beta)** | Run a `.sql` script against one or more profiles in parallel. | `--profiles`, `--yes`, `--timeout` |

---

## Connections

Manage Databricks workspace connection profiles in `~/.ddt/profiles.json`. See [Connections](connections.md).

| Command | What it does | Key flags |
|---|---|---|
| `ddt connection add --name <name>` | Register a profile (a positional `<name>` is also accepted). | `--name`, `--host`, `--warehouse-id`, `--auth pat\|oauth-m2m\|oauth-u2m\|azure-ad\|google-idc`, `--token`, `--client-id`, `--client-secret`, `--tenant-id`, `--service-account-key-path`, `--catalog`, `--schema` |
| `ddt connection list` | List configured profiles. | — |
| `ddt connection get <name>` | Print a profile as JSON with secrets redacted. | — |
| `ddt connection test <name>` | Verify a profile connects. | — |
| `ddt connection remove <name>` | Delete a profile. | — |

> [!IMPORTANT]
> Secrets are never inlined. `--token` and `--client-secret` accept `env:VAR_NAME` placeholders that resolve at runtime. PAT and OAuth M2M work today; `oauth-u2m`, `azure-ad`, and `google-idc` throw clear "not yet implemented" messages.

---

## Safety & rollback

No silent destruction: destructive changes are gated, snapshots are captured, and reverts replay reverse SQL. See [Safety classifier](safety-classifier.md) and [Safe deploy](safe-deploy.md).

> [!WARNING]
> On Unity Catalog, dropping a **managed table** deletes its underlying data files, and rebuilding a **streaming table** loses its checkpoint. The safety classifier flags both as unrecoverable — never override those gates without a snapshot.

| Command | What it does | Key flags |
|---|---|---|
| `ddt safety` | Inspect the safety-finding catalog. | `list --category`, `explain <code>`, `--format` |
| `ddt revert` | Roll back a previous `ddt publish` using its captured manifest. | `--manifest`, `--connection`, `--yes`, `--dry-run`, `--continue-on-error` |
| `ddt snapshot` | Inspect and prune the pre-deploy snapshot registry (Delta SHALLOW CLONE). | `list`, `show <id>`, `prune`, `--apply`, `--yes`, `--out`, `--root`, `--registry-only`, `--expired-only` |
| `ddt replay` | Re-execute the forward SQL from deploy manifests against a target. | `--manifest`, `--connection`, `--yes`, `--filter`, `--from-step`, `--to-step`, `--continue-on-error`, `--dry-run` |
| `ddt approval` | File-based multi-approver gate for production deploys. *(Team)* | `add`, `list`, `clear`, `check`, `--required`, `--allowed`, `--digest`, `--root` |
| `ddt approval-chain` | Cryptographically-signed M-of-N approval workflow (Ed25519). *(Team)* | `keygen`, `sign`, `verify`, `--deploy-id`, `--env`, `--key`, `--config`, `--records` |
| `ddt verify` | Recompute checksums on a `.ddtpac` and confirm every object matches the manifest. | `--pac`, `--json` |
| `ddt purge` | Generate a DROP script for every object in a project (review-only). | `--source`, `--confirm-production`, `-o`, `--allow-drop-table`, `--allow-unrecoverable-drop`, `--allow-narrowing-types`, `--allow-table-rebuild` |
| `ddt data-fit` | Generate read-only probes confirming no rows overflow a narrowing type change. *(Pro)* | `--source`, `--target`, `-o`, `--format` |
| `ddt scan-secrets` | Scan DDL bodies for hard-coded credentials. | `--source`, `--strict`, `--format`, `-o` |
| `ddt rollback-suggest` | AI-propose a reverse-SQL skeleton for a forward DDL the deterministic revert couldn't invert (review-only). *(Team)* | `--forward-sql`, `--fqn`, `--object-type`, `--finding`, `--intent`, `--format` |

---

## Code quality

Lint, format, and enforce team SQL standards. See [Configuration reference](configuration.md).

| Command | What it does | Key flags |
|---|---|---|
| `ddt lint` | Lint a SQL script for risky deploy patterns. | `--script`, `--rules`, `--max-warnings`, `--format text\|json\|markdown\|sarif`, `--rules-dir`, `--no-custom-rules` |
| `ddt format` | SQL formatter (`legacy` keyword engine by default; `--engine v2` is tokenizer-based). | `-p, --project`, `--check`, `--engine legacy\|v2`, `--style compact\|expanded\|wide`, `--dbt`, `--no-dbt` |
| `ddt standards` | Team SQL standards: scaffold, lint, explain. | `init --preset`, `check`, `--fix`, `--ai-fix`, `--explain <rule>`, `--json` |
| `ddt install-hooks` | Install a pre-commit hook running `ddt validate` + `ddt lint` on changed `.sql` files. | — |

---

## Analysis

Understand what depends on what, where data flows, and what a change costs. See [Schema compare](schema-compare.md).

| Command | What it does | Key flags |
|---|---|---|
| `ddt lineage` | Data-flow lineage from view bodies and Lakeflow Declarative Pipeline references. | `--source`, `--fqn`, `--direction`, `--depth`, `--format mermaid\|json\|dot` |
| `ddt graph` | Emit the dependency graph as Mermaid / DOT. | `--project`, `--format`, `-o` |
| `ddt impact` | Estimate blast radius of a pending change. | `--project`, `--object` |
| `ddt diagnose` | Project-wide health report (lint + lineage + smell + cost + dim-modeling). | `--source`, `--min-severity`, `--category`, `--format` |
| `ddt review` | AI-assisted senior-DBA review of a project. *(Pro)* | `--project`, `--format` |
| `ddt cost-estimate` | Estimate the SQL-warehouse DBU cost of a planned migration. | `--script`, `--warehouse-size`, `--calibrate-from`, `--explain`, `--format` |
| `ddt optimize` | Ranked heuristic optimization recommendations (Z-ORDER, narrowing, PKs, refresh schedules). | `--source`, `--severity`, `--explain`, `--format`, `-o` |
| `ddt docs` | Generate HTML schema docs from a project or pac. | `--source`, `-o`, `--title` |
| `ddt erd` | Generate an ER diagram from a project or pac. | `--source`, `-o`, `--title` |
| `ddt profile` | Per-column profile statistics for a table. | `--rows`, `--live`, `--row-limit`, `--top-n`, `--format` |
| `ddt suggest-constraints` | Heuristic PK / UK / FK / CHECK candidates from a table profile. | `--profile`, `--parents`, `--include-low`, `--ai`, `--apply-to`, `--apply-min` |

---

## Data

Row-level data comparison, seeds, and the declarative test framework.

| Command | What it does | Key flags |
|---|---|---|
| `ddt data-compare` | Diff two tables by PK; emit the INSERT / UPDATE / DELETE script. | `--table`, `--key`, `--source`, `--target`, `--source-live`, `--target-live`, `--columns`, `--row-limit`, `--max-changes`, `--no-transaction`, `--format` |
| `ddt data-fit` | Read-only probes for narrowing-type changes (see Safety & rollback). *(Pro)* | `--source`, `--target`, `-o`, `--format` |
| `ddt seed` | Manage seed reference data. *(Pro)* | `list`, `render`, `apply`, `--project`, `--connection`, `--yes`, `--dry-run`, `--delete-not-in-source`, `--out` |
| `ddt test` | Declarative SQL test framework. *(Pro)* | `--project`, `--connection` |
| `ddt preview` | Read-only sample-data inspector (`SELECT * … LIMIT n`). | `--connection`, `--limit`, `--where`, `--sql-only`, `--format` |

---

## CI/CD

Gate pull requests, comment on diffs, and inspect resumable deploys. See [CI/CD](ci-cd.md).

| Command | What it does | Key flags |
|---|---|---|
| `ddt pr-comment` | Render a diff / lint / impact analysis as a PR Markdown comment. *(Team)* | `--base`, `--head`, `--format` |
| `ddt drift-gate` | Multi-region/replica drift gate; fail when any replica drifts past `--threshold`. *(Team)* | `--primary`, `--replicas`, `--threshold` |
| `ddt deploy-status` | Show or list resumable deploy checkpoints. | `[<deployId>]`, `--format`, `--state-dir`, `--remove` |
| `ddt advise-tests` | PR-time regression-test advisor (matches changed files to the feature catalog). | `--changed-files`, `--diff-summary`, `--existing-tests`, `--ai-max-spend`, `--format`, `-o` |

---

## AI

BYO-key AI features, spend-capped locally. See [AI features](ai-features.md).

| Command | What it does | Key flags |
|---|---|---|
| `ddt ai` | Show + test the AI provider configuration. | `status`, `test --prompt`, `usage` |
| `ddt sketch` | Scaffold idiomatic Databricks UC DDL from a prose description. *(Pro)* | `<kind>`, `--description`, `--target`, `--out` |
| `ddt suggest` | AI-assisted modeling suggestions (PKs, FKs). *(Pro)* | `--project`, `--kind`, `--min-confidence` |
| `ddt design` | AI-assisted generative schema design. *(Pro)* | `star`, `snowflake-schema`, `--description`, `--target`, `--fact`, `--dims`, `--refinement`, `--ai-max-spend` |
| `ddt safer-alternative` | Propose a safer DDL alternative for a flagged migration step. *(Pro)* | `--code`, `--fqn`, `--object-type`, `--reason`, `--gate`, `--sql`, `--format`, `--ai-max-spend` |
| `ddt pii` | Detect likely-PII columns and render Unity-Catalog column-mask DDL. | `scan`, `mask`, `--source`, `--ai`, `--unmasked-group`, `--mask-function-schema`, `--verify-at-or-below`, `-o`, `--format` |

> [!NOTE]
> The `--explain` flag also adds AI narration to `compare`, `diagnose`, `review`, `impact`, `cost-estimate`, `lineage`, and `pr-comment`. *(Pro)*

---

## Migration & interop

Bring schemas in from other tools, export to other formats, and translate dialects.

| Command | What it does | Key flags |
|---|---|---|
| `ddt import` | Parse a SQL script and import its objects into a `.ddtproj` tree. | `--project`, `--script`, `--dry-run`, `--force`, `--ignore-errors` |
| `ddt migrate` | Migrate from another tool into DDT (v1: `from-dbt`). | `from-dbt`, `--output`, `--manifest`, `--identifier-case`, `--default-database`, `--default-schema` |
| `ddt migrate-platform` | Translate Databricks `.sql` to another dialect (v1: `snowflake`). | `--to`, `--output`, `--warn-on-unsupported` |
| `ddt schema` | Bidirectional schema interop (import / export / infer / validate-star). | `import-ddl\|import-dbml\|import-jsonschema\|import-dbt`, `export-dbml\|export-drawio\|export-jsonschema`, `infer-csv`, `infer-parquet`, `validate-star`, `suggest-snowflake-schema`, `--source`, `--out`, `--root`, `--name`, `--using` |
| `ddt xcompare-ir` | Cross-platform IR compare (Databricks ↔ Snowflake); IR-level only. | `--source`, `--tier`, `--format`, `--allow-destructive`, `--allow-unmappable` |

---

## Utilities

Authoring helpers, observability, catalog cache, licensing, and meta commands.

### Authoring helpers

| Command | What it does | Key flags |
|---|---|---|
| `ddt suite` | Create or validate a multi-project Suite (`.ddtsuite`). *(Pro)* | `init`, `validate`, `--out`, `--name` |
| `ddt refactor` | Record a rename so compares treat it as a rename, not drop + add. | `rename`, `list`, `--object`, `--new-name` |
| `ddt template` | Schema cookbook generator (SCD1–6, star, fact, time-series, audit, …). | `<kind>`, `--dims`, `--source`, `--out` |
| `ddt generate` | Code-gen for ingest + load scaffolds (PySpark / SparkSQL). | `notebook`, `scd`, `star-pipeline`, `--table`, `--natural-key`, `--type`, `--from-json`, `--from-design`, `--runtime`, `--format`, `--out` |
| `ddt snippets` | Browse the built-in SQL snippet catalog. | `list --category`, `show --format` |
| `ddt branch` | Branching for Unity Catalog via SHALLOW CLONE. *(Pro)* | `create`, `drop`, `--from`, `--catalog`, `--connection`, `-o` |
| `ddt anonymize` | Sanitized copy of a `.ddtproj` for bug reports / demos. | `--project`, `--output` |

### Observability & history

| Command | What it does | Key flags |
|---|---|---|
| `ddt history` | Show recent deploys from the local history store. | `--limit` |
| `ddt bisect` | Find the commit where an FQN first changed (pure-git). | `<fqn>`, `--good`, `--bad`, `--object-type`, `--format` |
| `ddt changelog` | Generate a Markdown CHANGELOG from Conventional Commits. | `--from`, `--to`, `--version`, `-o`, `--append` |
| `ddt audit-log` | Export deploy + compare records to an enterprise sink. | `emit`, `--sink file\|syslog\|splunk\|datadog\|otlp`, `--workspace`, `-o`, `--dry-run` |
| `ddt telemetry` | Manage the local telemetry opt-in (also governs error reporting). | `on`, `off`, `show`, `clear`, `config`, `upload` |
| `ddt query-log` | Browse the local query log written by the VS Code query window. | `list`, `show`, `add`, `clear`, `--profile`, `--limit`, `--yes` |
| `ddt error-lookup` | Look up a deploy/adapter failure in the known-error catalog. | `--code`, `--fingerprint`, `--message`, `--list`, `--format` |

### Catalog cache & explorer

| Command | What it does | Key flags |
|---|---|---|
| `ddt catalog` | Manage the per-connection catalog cache (feeds Object Explorer + intellisense). | `refresh`, `show`, `clear`, `--connection`, `--catalogs`, `--concurrency`, `--include-builtins`, `--no-fingerprint-skip`, `--json` |
| `ddt explorer` | ASCII tree of the live catalog/schema/object hierarchy (reads the cache). | `--connection`, `--filter`, `--depth`, `--json` |
| `ddt compare-profiles` | Manage saved compare profiles (source → target + mapping rules). *(Pro)* | `list`, `show`, `save`, `remove`, `preview`, `--json-file`, `--json`, `--examples` |
| `ddt search` **(beta)** | Find objects matching a pattern across profiles, flagging presence drift. | `--profiles`, `--exact`, `--type`, `--no-drift`, `--format` |

### Licensing, meta & integration

| Command | What it does | Key flags |
|---|---|---|
| `ddt license` | Activate / show / deactivate a license. | `activate --key`, `status`, `deactivate` |
| `ddt trial` | Manage the local 30-day Pro trial (no account needed). | `start`, `status` |
| `ddt pilot` | Join or check the pilot program (90-day Pro access). | `register --email`, `status` |
| `ddt features` | Inspect the visible-but-locked feature catalog. | `list`, `show`, `upgrade` |
| `ddt find` | Search the feature catalog by keyword. | `--limit`, `--tier`, `--shipped`, `--ai`, `--ai-threshold` |
| `ddt explain` | Search the options catalog for a flag / config key / behavior. | `--tag`, `--safety`, `--limit`, `--list` |
| `ddt discover` | Personalized feature suggestions from your usage history. | `--reset` |
| `ddt hosts` | Inspect the host-adapter catalog (every CLI/IDE/CI surface). | `list`, `show <id>` |
| `ddt mcp` | Run the MCP server (stdio JSON-RPC) so AI agents can invoke DDT as a tool. | — |
| `ddt completion` | Emit a shell-completion script. | `bash\|fish\|zsh\|powershell` |
| `ddt feedback` | Send a feedback message to the support inbox. | `"<message>"`, `--category` |
| `ddt savings` | Estimated AI-token savings from deterministic surfaces. | `--top`, `--format`, `--reset` |
| `ddt bookmarks` | Saved pointers to queries, files, or sections. | `add`, `remove`, `get`, `list`, `--type`, `--target`, `--label`, `--format` |
| `ddt perf` | CLI performance diagnostics (cold-start latency). | `cold-start`, `--json`, `--budget` |
| `ddt watch` | Diff a Databricks release-notes file against a snapshot. | `bulletin`, `--current`, `--prior`, `--save-snapshot`, `--format` |
| `ddt --version` | Print the installed version. | — |

---

**Next:** [VS Code extension reference](vscode-extension.md) · **Up:** [Documentation home](README.md)
