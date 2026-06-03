# ⚙️ Configuration reference

> Every option you can set — in your `.ddtproj`, in VS Code, and through environment variables.

**On this page:** [Where options live](#where-options-live) · [Compare options](#compare-options) · [Deployment options](#deployment-options) · [Safety gates](#safety-gates) · [VS Code settings](#vs-code-settings) · [Environment variables](#environment-variables)

---

## Where options live

`.ddtproj` deploy options live in two places, with the more specific one winning:

1. **Project-level** — under `deployOptions` in the `.ddtproj`. Applies to every profile.
2. **Profile-level** — under `deploymentProfiles.<env>.deployOptions`. Merges on top of project-level.

Defaults are tuned so a fresh project running against a `prod`-named profile refuses every dangerous operation until you explicitly opt in.

```jsonc
{
  "deployOptions": {
    "compare": { "ignoreComments": true, "ignoreCase": true },
    "deployment": { "blockOnPossibleDataLoss": true }
  },
  "deploymentProfiles": {
    "dev": {
      "connection": "dev",
      "deployOptions": {
        "deployment": { "blockOnPossibleDataLoss": false, "allowDropTable": true }
      }
    }
  }
}
```

> [!TIP]
> Every field below also has a richer catalog entry behind `ddt explain <field>` — covering why the option exists and which flag pairs with it.

---

## Compare options

These live under `deployOptions.compare` and decide which field changes get surfaced.

| Field | Default | What it does |
|---|---|---|
| `ignoreComments` | `false` | Don't flag `COMMENT` changes. |
| `ignoreCase` | `false` | Compare FQNs case-insensitively (UC is case-insensitive in practice). |
| `ignoreQuotedIdentifiers` | `false` | Strip backticks before comparing identifiers. |
| `ignoreWhitespace` | `false` | Collapse whitespace before comparing SQL bodies (views, functions, MV SELECTs). |
| `ignoreSemicolons` | `false` | Strip trailing `;` before comparing. |
| `ignoreFormattingDifferences` | `false` | Superset of the three above — canonicalize before comparing. |
| `ignoreColumnOrder` | `false` | Don't flag column-reordering changes. |
| `ignoreColumnDefaults` | `false` | Don't flag `DEFAULT` clause changes. |
| `ignoreGeneratedColumns` | `false` | Don't flag generated-column expression changes. |
| `ignoreColumnPatterns` | `[]` | Wildcard list (`*.gold.*.audit_*`) — matched columns are skipped entirely. |
| `ignoreTableProperties` | `false` | Don't flag `TBLPROPERTIES` changes. |
| `ignoreClusterBy` | `false` | Don't flag clustering / liquid-clustering changes. |
| `ignorePartitioning` | `false` | Don't flag partition-column changes. |
| `ignoreStorageLocation` | `false` | Don't flag `LOCATION` changes on external tables. |
| `ignoreTagAttachments` | `false` | Don't flag tag-attachment changes. |
| `ignoreColumnMasksAndRowFilters` | `false` | Don't flag inline column-mask / row-filter changes. |
| `ignoreGrants` | `true` | Don't flag direct-grant changes. |
| `ignoreOwner` | `true` | Don't flag owner changes. |
| `ignoreRefreshMode` | `false` | Don't flag refresh-mode changes on streaming tables / Lakeflow pipelines. |
| `ignoreSchedule` | `false` | Don't flag schedule changes on jobs / pipelines / streaming tables. |
| `excludeObjectTypes` | `[]` | Filter by object type — e.g. `["JOB", "PIPELINE"]`. |
| `excludeObjectPatterns` | `[]` | Wildcard FQN list — matched objects are skipped entirely. |
| `includeObjectPatterns` | `[]` | When non-empty, ONLY matched objects are compared. |
| `ignoreFieldPaths` | `[]` | Field-level skip list — e.g. `columns[*].comment`. |

---

## Deployment options

These live under `deployOptions.deployment` and decide what SQL the emitter is allowed to produce.

### Top-level safety

| Field | Default | What it does |
|---|---|---|
| `blockOnPossibleDataLoss` | `true` | Refuse to emit any DESTRUCTIVE change unless the per-op gate is set. |
| `blockWhenDriftDetected` | `true` | Refuse to deploy when extracted target state has drifted from the last build. |
| `treatWarningsAsErrors` | `false` | Promote WARNING-category findings to blockers. |
| `verifyDeployment` | `true` | Re-extract + diff after `--apply` to confirm zero residual drift. |

### Drop policy

| Field | Default | What it does |
|---|---|---|
| `preserveTargetOnlyObjects` | `false` | Refuse to drop any object on target but not in source. |
| `doNotDropObjectTypes` | `[]` | Per-type whitelist — never drop these even if on target only. |
| `doNotDropObjectPatterns` | `[]` | Per-FQN-pattern whitelist. |
| `dropTargetOnlyPermissions` | `false` | Drop direct grants on target that aren't in source. |
| `dropTargetOnlyTagAttachments` | `false` | Drop tag attachments on target that aren't in source. |
| `doNotAlterStreamingObjects` | `true` | Refuse drops/alters on objects feeding streaming tables / Lakeflow pipelines. |
| `doNotAlterShares` | `true` | Refuse drops/alters on Delta Sharing shares. |

### Pre / post-deploy hooks

| Field | Default | What it does |
|---|---|---|
| `preDeployScriptPatterns` | `[]` | Glob list — `.sql` files run before the body. Idempotent expected. |
| `postDeployScriptPatterns` | `[]` | Glob list — `.sql` files run after the body. Idempotent expected. |

### Databricks-specific

| Field | Default | What it does |
|---|---|---|
| `refreshAfterDeploy` | `false` | After `--apply`, run `REFRESH` on every affected MV / streaming table. |
| `stopPipelinesBeforeDeploy` | `true` | Stop running Lakeflow pipelines that feed objects being modified. |
| `startWarehouseBeforeDeploy` | `true` | Start a STOPPED SQL warehouse before running the deploy. |
| `generateSmartDefaults` | `true` | When adding a NOT NULL column to a non-empty table, emit a typed default. |
| `scriptBanner` | `''` | Banner text prepended to every generated migration script. |
| `strategyOverrides` | `{}` | Per-`DatabricksObjectType` overrides (`{ rebuild, modify }`). |

---

## Safety gates

These per-operation gates live under `deployOptions.deployment` and unlock blocked operations. All default to `false`.

| Field | CLI flag | Unlocks |
|---|---|---|
| `allowDropTable` | `--allow-drop-table` | `DROP TABLE` on objects flagged DESTRUCTIVE. |
| `allowDropColumn` | `--allow-drop-column` | `ALTER TABLE … DROP COLUMN`. |
| `allowNarrowingTypes` | `--allow-narrowing` | Narrowing type changes (`DECIMAL(38,9)` → `DECIMAL(10,2)`). |
| `allowTableRebuild` | `--allow-table-rebuild` | Full DROP+CREATE rebuilds for clustering/partition changes. |
| `allowUnrecoverableDrop` | `--allow-unrecoverable-drop` | Drops of streaming tables / Lakeflow pipelines / jobs. |
| `allowStorageLocationChange` | (deployOptions) | MANAGED ↔ EXTERNAL storage transitions. |
| `noRebuildObjectPatterns` | (deployOptions) | Wildcard FQN list — matched objects are exempt from rebuilds. |

See [The safety classifier](safety-classifier.md) for how these interact with the four risk categories.

> [!NOTE]
> Pattern fields (`ignoreColumnPatterns`, `excludeObjectPatterns`, `includeObjectPatterns`, `doNotDropObjectPatterns`, `noRebuildObjectPatterns`) use a wildcard grammar where `*` matches any run of identifier characters and `.` is literal — e.g. `main.gold.*` or `*.gold.*.audit_*`.

---

## VS Code settings

Configure the DDT extension under **Settings → Extensions → DDT** (or in `settings.json`). All keys are prefixed `ddt.`.

| Setting | Default | What it does |
|---|---|---|
| `ddt.defaultProfile` | `""` | Default Databricks connection profile name used by the Object Explorer. |
| `ddt.formatOnSave` | `true` | Auto-format `.sql` files on save with the DDT formatter. Disable to use VS Code's built-in format-on-save. |
| `ddt.discoverability.level` | `standard` | Coarse discoverability knob. **minimal** suppresses hints, rich tooltips, CodeLens, and context-menu surfaces; **standard** is the default; **verbose** adds inline Try-it buttons. |
| `ddt.discoverability.hints` | `auto` | Per-channel override for post-command hints. `auto` follows the level; `on`/`off` force it. |
| `ddt.discoverability.tooltips` | `auto` | Per-channel override for rich Markdown tooltips on Object Explorer nodes. |
| `ddt.discoverability.codeLens` | `auto` | Per-channel override for DDT CodeLens hints. |
| `ddt.discoverability.contextMenu` | `auto` | Per-channel override for feature-aware right-click menu entries. |
| `ddt.discoverability.emptyStates` | `auto` | Per-channel override for actionable empty-state placeholders. |
| `ddt.discoverability.suppressedHintIds` | `[]` | Hint IDs silenced via "Don't show this again". Managed automatically. |
| `ddt.licenseServer.url` | `""` | Base URL of your organization's On-prem License Server (Enterprise SSO). |
| `ddt.licenseServer.oidcClientId` | `""` | OAuth2 client ID for desktop sign-in (Enterprise SSO). |
| `ddt.licenseServer.oidcAuthorizeUrl` | `""` | Identity provider authorization endpoint (Enterprise SSO). |
| `ddt.licenseServer.oidcTokenUrl` | `""` | Identity provider token endpoint (Enterprise SSO). |
| `ddt.licenseServer.scopes` | `[]` | OAuth scopes for SSO sign-in. Empty uses `openid profile email`. |

---

## Environment variables

| Variable | Effect |
|---|---|
| `NO_COLOR` | Disables ANSI colour in CLI output (per the [NO_COLOR convention](https://no-color.org/)). |
| `CI` | When `CI=true`, the CLI runs non-interactively — it never prompts (require `--yes`), never opens a browser, and disables spinners. |
| `DDT_NO_HINTS` | `DDT_NO_HINTS=1` maps to the `minimal` discoverability level. |

Connection profiles also support `env:VAR_NAME` placeholders so secrets stay out of `profiles.json`.

---

**Next:** [CLI reference](cli-reference.md) · **Up:** [Documentation home](README.md)
