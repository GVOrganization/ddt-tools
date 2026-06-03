# 🔍 Schema compare

> Diff any two of your project, a built artifact, or a live Databricks workspace — and see exactly what a deploy would change before it runs.

**On this page:** [The compare model](#the-compare-model) · [Reading results](#reading-results) · [VS Code flow](#vs-code-flow) · [CLI flow](#cli-flow) · [Compare audit trail](#compare-audit-trail) · [Drift check](#drift-check)

---

## The compare model

![Schema compare demo](../assets/demo-compare.gif)

DDT's compare engine diffs two **sources** in any direction. A source is one of:

| Source | What it is |
|---|---|
| Project (`.ddtproj`) | Your working tree of `.sql` files — the desired state. |
| Build artifact (`.ddtpac`) | A compiled, frozen snapshot from `ddt build`. |
| Live workspace (`databricks://<profile>[/<catalog>[/<schema>]]`) | A connected Databricks workspace, read live. |

Any combination works. The everyday ones:

- **Project ↔ live** — does the workspace match what I authored?
- **Project ↔ pac** — what changed since the last build?
- **Pac ↔ pac** — audit two tagged releases against each other.

The same engine powers `ddt publish` and `ddt drift`, so the diff you review in compare is the diff a deploy acts on.

---

## Reading results

Every object is classified by **status**:

| Status | Meaning |
|---|---|
| Added | Present in the source, absent from the target. |
| Removed | Present in the target, absent from the source. |
| Modified | Present in both, but differs — drill into field-level changes. |
| Unchanged | Identical on both sides. |

On top of status, every change carries a **safety category** so you know which diffs are dangerous before you act on them:

| Category | What it signals |
|---|---|
| UNRECOVERABLE | Cannot be undone — data or an object is lost with no rollback. |
| DESTRUCTIVE | Drops or rebuilds that lose data unless explicitly allowed. |
| EXPENSIVE | Safe, but costly — a full table rebuild or large rewrite. |
| WARNING | Worth a look, but not destructive. |

> [!WARNING]
> On Unity Catalog, dropping a **managed table** deletes its underlying files, and dropping or rebuilding a **streaming table** loses its checkpoint — re-ingestion starts from scratch. Both surface as UNRECOVERABLE or DESTRUCTIVE findings. Read the safety report before deploying.

See [Safety classifier](safety-classifier.md) for the full category breakdown and the stable finding codes, and [Safe deploy](safe-deploy.md) for how the gates turn into deploy options.

---

## VS Code flow

The DDT VS Code extension is **browse / review-focused** — running a compare is CLI-first (`ddt compare`, above). The extension complements it: the **Object Explorer** browses live workspace state side-by-side with your project files, **DDT: Open Compare Profiles** manages saved compare configurations, and the interactive schema diagram visualizes what you extracted.

The view shows:

- A tree of every object grouped by type, with **added / removed / modified / unchanged** status badges.
- Per-object, field-level drill-down for modified rows.
- Safety findings inline — every diff classified UNRECOVERABLE / DESTRUCTIVE / EXPENSIVE / WARNING.
- Include/exclude selection per row to narrow what a generated script would cover.

> [!NOTE]
> Project lifecycle — build, publish, extract, compare — is CLI-first in DDT. Run the compare in a terminal, then review the output and run the deploy from there.

---

## CLI flow

`ddt compare` diffs two sources, passed explicitly as `--source` and `--target`.

```sh
# Project vs live workspace
ddt compare --source ./MyProject.ddtproj \
            --target 'databricks://prod/MY_CATALOG'

# Pac vs pac — audit two builds, as Markdown
ddt compare --source ./bin/v1.ddtpac --target ./bin/v2.ddtpac --format markdown

# Project vs live with AI narration (Pro)
ddt compare --source ./MyProject.ddtproj \
            --target 'databricks://prod' --explain
```

| Flag | What it does | Notes |
|---|---|---|
| `--source <spec>` | The desired-state side. | A `.ddtproj`, `.ddtpac`, or `databricks://` URL. |
| `--target <spec>` | The side being compared against. | Same accepted forms as `--source`. |
| `--format <fmt>` | Output format: `table`, `json`, `yaml`, `sql`, `markdown`. | Defaults to `table`. `--json` is an alias for `--format json`. |
| `--ignore-case` | Compare object names case-insensitively. | — |
| `--explain` | Add AI narration of the diff. | Pro tier; requires a configured AI provider. |
| `--color <when>` | `always` / `never` / `auto`. | `auto` honors the TTY and `NO_COLOR`. |
| `--report-html <path>` | Write a self-contained HTML compare report. | — |
| `--map <src=tgt>` | Rewrite source-side names so differently-named catalogs compare semantically. | Repeatable; `--map-file <path>` reads a list. |
| `--type-safe` | Run impact analysis after the compare and gate the exit code. | Pair with `--break-on`. |
| `--break-on <sev>` | Threshold for `--type-safe`: `error` (default) or `warning`. | `warning` is strict CI mode. |
| `--no-history` | Skip writing the compare audit record. | For read-only CI environments. |

Severity colors: UNRECOVERABLE renders bold red, DESTRUCTIVE red, EXPENSIVE yellow, WARNING cyan, SAFE/OK green.

### Exit codes

`ddt compare` returns a meaningful exit code so it can gate CI:

| Code | Meaning |
|---|---|
| 0 | Success — no gate tripped. |
| 1 | User error (bad args, missing file, syntax error). |
| 2 | System error, **or** a `--type-safe` gate tripped at its `--break-on` threshold. |
| 3 | Compare diff found (when used to detect drift in CI). |
| 6 | A locked feature was attempted without the required license tier. |

```sh
# CI gate: fail the build when a change ripples a type error
ddt compare --source ./MyProject.ddtproj --target ./bin/last.ddtpac \
            --type-safe --format json

# Strict mode + impact file the editor reads back as squiggles
ddt compare --source ./MyProject.ddtproj --target ./bin/last.ddtpac \
            --type-safe --break-on warning --write-impact
```

### Comparing differently-named environments

When dev and prod use different catalog or schema names, every object reads as added/removed. Map the names so the compare is semantic:

```sh
# Compare a dev pac against prod naming without false diffs
ddt compare --source ./dev.ddtpac --target ./prod.ddtpac --map dev_cat=prod_cat
```

`--map` is repeatable, and `--map-file <path>` loads a list. The same mapping works on `ddt script`, `ddt drift`, and `ddt publish`.

---

## Compare audit trail

Every compare run records a history entry under `.ddt/history/compare/` next to the source — capturing the outcome (`NO_CHANGES` / `CHANGES_FOUND` / `BLOCKED`), the source and target labels, and a diff summary. Pass `--no-history` to skip the write in read-only environments.

These records export alongside deploy history via `ddt audit-log emit`, giving you one ordered stream of "what was compared and what was deployed."

---

## Drift check

Compare answers "how do these two sources differ?" **Drift** answers a narrower, recurring question: "has the live workspace drifted from what my project expects?" It's compare in report-only mode, purpose-built for a scheduled or pre-deploy check.

```sh
# Report drift between the project and the live workspace
ddt drift --project ./MyProject.ddtproj --connection prod
```

For multi-region or replica fleets, `ddt drift-gate` compares each replica against a primary and **fails** the gate when any replica drifts beyond a threshold — drop it into CI:

```sh
ddt drift-gate \
  --primary prod-us \
  --replicas prod-eu,prod-apac \
  --threshold 0
```

When `ddt drift` runs in CI mode, drift detection returns exit code 5, so a pipeline can act on it automatically.

See [CI/CD](ci-cd.md) for wiring drift and type-safe gates into a pipeline.

---

**Next:** [Safe deploy](safe-deploy.md) · **Up:** [Documentation home](README.md)
