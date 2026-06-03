# ❓ FAQ

> Straight answers to the questions people ask before and while adopting DDT.

**On this page:** [About the product](#about-the-product) · [Beta & pricing](#beta--pricing) · [Using it](#using-it) · [Data safety](#data-safety)

---

## About the product

### How is DDT different from dbt, Flyway, Liquibase, or Terraform?

DDT is **state-based**: you author each Unity Catalog object as a `.sql` file describing the desired end state (using `CREATE OR REPLACE`, the Databricks idiom), and DDT computes the deltas. You never hand-write forward/backward migration scripts. Every alternative differs on one of those points:

| Tool | What it is | How DDT differs |
|---|---|---|
| **dbt** | Transformation framework | dbt explicitly does **not** own DDL — your `CREATE TABLE` / `ALTER TABLE` changes happen outside dbt. DDT is the tool for that piece, and sits happily next to dbt. |
| **Flyway / Liquibase** | Migration-based change management | You write the forward and backward SQL by hand, and the adapters are generic SQL rather than Unity-Catalog-native. DDT diffs desired state and classifies each change's safety per finding. |
| **Terraform / Databricks SDK / CLI** | Building blocks for automation | General-purpose tools, not a schema compare/deploy engine. No 4-tier safety classification of DDL changes; no reviewable migration script; no UC-specific failure-mode awareness. |

DDT understands UC-specific risks that generic tools miss — dropping a streaming table loses its checkpoint cursor; dropping a managed table deletes its files. It does not replace dbt or your lakehouse — it covers the schema compare/deploy/safety layer those tools leave out. For migration paths from other tools, see [Migrating from other tools](migrating.md).

### Is DDT affiliated with Databricks?

No. DDT is an **independent** tool. It is not affiliated with, endorsed by, or sponsored by Databricks Inc. "Databricks", "Unity Catalog", and "Delta Lake" are trademarks of Databricks Inc., used here only to describe what DDT works with.

### Does my SQL leave my machine?

No. DDT runs entirely on your machine and talks directly to your Databricks workspace. Your SQL, identifiers, catalog/schema/table/column names, data, and credentials never leave your computer. Telemetry is opt-in and heavily sanitized, and AI features are bring-your-own-key — see [Data safety](#data-safety) and [PRIVACY.md](../PRIVACY.md).

---

## Beta & pricing

### What's free during the beta?

**Everything.** DDT is in a 30-day public beta, and during it every feature is free — the full deterministic engine plus all AI write-side features. See [Beta program](../README.md#beta-program) for the dates and details.

### What happens after the beta?

The deterministic core stays **free forever** — compare, safe deploy, extract, lint, format, lineage, Object Explorer, schema diagram, and the MCP server. After the beta, AI write-side features and some ownership features move to a paid Pro tier; if you keep using a Pro feature without a license, the CLI tells you clearly rather than failing silently. Free-tier commands never regress. The current tier breakdown is in the [README pricing section](../README.md#beta-program); release-time specifics land in [RELEASES.md](../RELEASES.md).

### Do AI features cost extra?

The AI features themselves follow the tier model above, but there is **no AI markup from us**. AI is **bring-your-own-key**: your prompts go directly from your machine to the AI provider you configure (Anthropic, OpenAI, Azure OpenAI, or an OpenAI-compatible / self-hosted endpoint), under your own account. DDT never proxies, resells, or stores that traffic, so you pay your provider directly at their rates and nothing to us for the tokens.

---

## Using it

### Do I need both the VS Code extension and the CLI?

No — either one works on its own. They run the same engine and share the same project format (`.ddtproj` + one `.sql` file per object). Databricks users predominantly drive DDT from the CLI and CI: the project lifecycle (init / build / publish / extract) is CLI-first, while the VS Code surface focuses on browsing, comparing, diagramming, and reviewing. Install whichever fits your workflow, or both.

### Can I use DDT in CI?

Yes. The CLI is built for it — compare, dry-run publish, lint, and (with a license) apply all run headless, and OAuth machine-to-machine (M2M) auth is designed for service principals. Telemetry is automatically disabled when `CI=true`. See [CI/CD](ci-cd.md) for wiring patterns.

### What permissions does DDT need on my workspace?

DDT needs the principal behind your PAT or service principal to have:

- **`CAN USE`** on the SQL warehouse it connects through.
- **`USE CATALOG`** and **`USE SCHEMA`** on the namespaces it reads.
- For deploy, the per-object privileges (`MODIFY`, `CREATE`, `SELECT`, …) on the targets you change.

Treat the connection profile like a CI/CD secret: its blast radius is everything that principal can see and change.

### Does DDT work with my Databricks cloud?

Yes — DDT connects to any workspace with **Unity Catalog enabled**, on AWS, Azure, or GCP. Personal Access Token (PAT) and OAuth machine-to-machine (M2M) auth are supported today; OAuth U2M, Azure AD, and Google Identity are on the roadmap. See [Connecting to Databricks](connections.md).

> [!NOTE]
> DDT requires Unity Catalog. Workspaces provisioned before UC (legacy `hive_metastore`) aren't supported — enable UC at the account console or point DDT at a UC-enabled workspace.

---

## Data safety

![Safe deploy demo](../assets/demo-deploy.gif)

### Can DDT drop my tables?

Only when you explicitly tell it to. Destructive operations are blocked by default — they come out of a deploy as `-- WARNING:` comments, not executed SQL. To run one, you opt in with the matching gate (for example `--allow-unrecoverable-drop` for a streaming-table drop). This is by design — "no silent destruction" is a core principle, and DDT is especially careful with UC-specific risks: dropping a **managed table** deletes its files, and dropping a **streaming table** loses its source offset and refresh checkpoint. See [Safe deploy](safe-deploy.md) and the [Safety classifier](safety-classifier.md).

### What does error reporting send?

If you keep automatic error reporting on (opt-out, asked on first run), a report contains the error class, a **sanitized** message and stack (quoted identifiers and string literals are replaced before the report ever leaves your machine), the error kind, a fingerprint, and your OS / version info. It includes an email address **only** if you explicitly provide one.

It **never** sends your SQL text, identifier names, credentials, environment variables, file contents, or project structure. The sanitizer runs on your machine, so unsanitized data never reaches the network. Turn it off any time with `ddt telemetry off`, `DDT_TELEMETRY=0`, or `DO_NOT_TRACK=1`. Full detail: [PRIVACY.md](../PRIVACY.md).

---

**Next:** [Troubleshooting](troubleshooting.md) · **Up:** [Documentation home](README.md)
