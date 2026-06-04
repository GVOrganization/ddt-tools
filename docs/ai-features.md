# ✨ AI features

> Layer optional AI on top of the deterministic engine — narrate a diff, suggest a safer change, scaffold an object from prose, or grade a deploy before you apply it.

> [!NOTE]
> AI is **optional**. The deterministic engine — extract, compare, safe deploy, the safety classifier — is fully useful without it. DDT (Databricks Data Tools) never ships its own inference: you bring your own API key, and DDT calls your provider directly with **no markup**. During the public beta, all AI features are unlocked.

**On this page:** [How AI fits in](#how-ai-fits-in) · [Set up a provider](#set-up-a-provider) · [Assist features](#assist-features) · [Advanced features](#advanced-features) · [Foundation Models](#foundation-models-passthrough) · [Privacy](#privacy)

---

## How AI fits in

DDT is a state-based schema-change engine. AI is leverage on top of that engine, not a replacement for it. Every AI feature reads the same diff, safety findings, and project model the deterministic commands already produce.

The AI surface splits into two buckets:

| Bucket | What it does | Example |
|---|---|---|
| **AI Assist** | Augments a single command — annotate, scaffold, or narrate while you stay in the driver's seat | `compare --explain`, "Ask DDT" chat, `sketch` |
| **AI Advanced** | Runs multi-step workflows that gate or generate work — preflight grading, senior-DBA review, AI rollback | `publish --ai-preflight`, `review --senior-dba` |

---

## Set up a provider

DDT speaks to each provider over its public REST endpoint — there's no separate SDK to install. You set a provider and supply its key via an environment variable.

Supported providers:

| Provider | Notes |
|---|---|
| Anthropic | Default recommendation |
| OpenAI | |
| Azure OpenAI | For Azure-hosted deployments |
| OpenAI-compatible | Any gateway speaking the OpenAI API |
| Self-hosted | A local or on-prem OpenAI-compatible endpoint (Ollama, vLLM, LM Studio) via the `openai-compatible` provider plus an `endpoint` override — works air-gapped |

### CLI

```sh
ddt ai set-provider anthropic   # anthropic | openai | azure-openai | openai-compatible
ddt ai status                   # show the current provider config
ddt ai test                     # round-trip a hello-world prompt to confirm the key works
```

Set your key as an environment variable before running any AI command:

```sh
export ANTHROPIC_API_KEY=sk-ant-...   # or OPENAI_API_KEY, etc.
```

Config is read from `~/.config/ddt/ai.json` and environment variables. Keep keys in env vars or a gitignored config file — never commit them.

> [!TIP]
> Cap spend with `--ai-max-spend $X` on any AI command — it refuses to start if the call would exceed the cap. Every AI call logs its token count and model to a local usage file; review it with `ddt ai usage`.

### VS Code

Set the provider in the extension's settings, and view or switch it from the status-bar AI entry, which shows the active provider and who's paying for inference.

---

## Assist features

Augment-a-command features. The AI annotates, scaffolds, or narrates; you decide what to keep.

| Feature | What it does | How to invoke |
|---|---|---|
| **`--explain` narration** | Appends a plain-English summary of what changes, what's safe, and what to look at first | Add `--explain` to `compare`, `diagnose`, `review`, `impact`, `cost-estimate`, or `lineage` |
| **Safety explainer** | Expands a safety finding into *why* it's UNRECOVERABLE and what cannot be reversed, citing the platform behavior | `ddt safety explain <code>` |
| **Ask DDT chat panel** | Chat sidebar with context from your project model, the connected target (read-only), and the most recent compare result | VS Code sidebar; type natural language |
| **Natural-language → DDL edit** | Describe a change in the chat panel; DDT proposes file edits with apply/reject preview | VS Code Ask DDT panel |
| **Sketch from description** | Turn prose ("a streaming pipeline from a raw table to a cleansed table") into an object DDL scaffold for the right Databricks object types | `ddt sketch <kind>`; **DDT: Sketch Object** in the Command Palette |
| **Safer-alternative CodeLens** | On an UNRECOVERABLE / DESTRUCTIVE finding, proposes a less-risky path (e.g. soft-delete a column via a view layer) and edits the file in place | CodeLens above the finding in VS Code |
| **Auto-comment generator** | One-shot batch: fills in a `COMMENT` for each object missing one, read from its DDL | `ddt docs --augment-comments` |
| **Schema inference** | Infers types and a `CREATE TABLE` from a sample data file | `ddt schema infer-csv` / `infer-parquet` |
| **Lineage diff narration** | Narrates how a lineage graph changed between two states | `ddt graph --compare-to <ref> --explain` |
| **Changelog generation** | Turns commit history into a release-note paragraph | `ddt changelog --from <ref> --to <ref>` |

The `--explain` output drops in right after the structured result:

```sh
ddt compare --source my-project.ddtpac --target prod.ddtpac --explain
```

You'll see:

```
... diff ...

=== AI summary ===
This deploy adds 2 new tables (customers_v2, orders_v2) and modifies
1 view (v_active_customers). The view change widens a STRING column —
safe. No destructive findings.
```

---

## Advanced features

Autonomous-workflow features. The AI proposes, grades, or generates across multiple steps — often gating a deploy or producing recovery SQL the deterministic engine can't synthesize.

| Feature | What it does | How to invoke |
|---|---|---|
| **Pre-apply preflight grading** | Before `--apply`, you type the impact in your own words; the AI grades whether your understanding matches the actual diff + safety findings | `ddt publish --ai-preflight` (with `--strict-ai-preflight` / `--confirm-after-preflight-mismatch` overrides) |
| **Senior-DBA review** | Reads the diff + project + target metadata and comments on best-practice violations (missing partitioning / clustering, unbounded retention); exits non-zero on `request_changes` so it gates CI | `ddt review --senior-dba` |
| **PII detector** | Scans column names and comments for likely PII; suggests masking policies | `ddt pii scan` / `ddt pii mask --ai` |
| **Rollback generator** | Generates reverse SQL even when the deterministic plan can't invert a structural change | `ddt rollback-suggest` |
| **Cost estimator** | Estimates the DBUs a deploy will consume, calibrated against the target's recent query patterns | `ddt cost-estimate --calibrate-from <ref>` |
| **Test advisor** | Recommends regression tests for the change set | `ddt advise-tests` |
| **Compliance scanner** | Flags compliance-relevant findings | `ddt diagnose --category compliance` |
| **MCP AI tools** | Exposes `narrate_diff`, `detect_pii`, and `suggest_safer` to an MCP agent (Claude Desktop, Cursor, Continue), which calls them with its own key | `ddt mcp` |

> [!TIP]
> `review --senior-dba` is built for CI: it exits with a non-zero code when the AI requests changes, so a pull-request check can block a merge on its verdict. See [CI/CD](ci-cd.md).

---

## Foundation Models passthrough

Instead of an external provider, you can route inference through **Databricks Foundation Models** so it bills against your existing Databricks workspace and your data never leaves it. DDT calls a Model Serving endpoint using your connection profile.

This is the best privacy and procurement story for Databricks-resident teams — no new AI vendor, inference on your existing platform spend (billed in DBUs), no markup from DDT. Foundation Models passthrough is an Enterprise feature.

---

## Privacy

- **Your data goes from your machine to your provider, directly.** With a BYO key, prompts travel from the CLI or VS Code extension to the provider you configured under your own provider agreement — DDT never proxies them.
- **What's in a prompt depends on the feature.** Narration and review send the diff JSON and safety findings; the chat panel can include your project model, a read-only view of the connected target, and a small selection of project files; auto-comment and sketch send the relevant DDL. AI commands are opt-in per invocation.
- **Foundation Models keeps data in-workspace.** Routing through Foundation Models keeps prompts inside your Databricks workspace.
- **Self-hosted is air-gapped.** Point the `openai-compatible` provider at a local endpoint and nothing leaves your network.

---

**Next:** [CI/CD](ci-cd.md) · **Up:** [Documentation home](README.md)
