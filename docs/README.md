# 📚 DDT Documentation

> Everything you need to use **DDT — Databricks Data Tools**, from first install to CI/CD automation.

![Databricks Data Tools](../assets/hero.png)

## Start here

| | Page | What you'll learn |
|---|---|---|
| 🚀 | **[Getting started](getting-started.md)** | Install → connect → first project → first compare → first deploy, in ~15 minutes |
| 🔌 | **[Connecting to Databricks](connections.md)** | PAT and OAuth M2M auth, profiles, workspace validation, troubleshooting |
| 📁 | **[Projects & suites](projects.md)** | The `.ddtproj` format, folder layout, variables, deployment profiles, `.ddtpac` artifacts |

## The core workflow

| | Page | What you'll learn |
|---|---|---|
| 📥 | **[Extract a live workspace](extract.md)** | Reverse-engineer Unity Catalog into git-ready `.sql` files |
| 🔍 | **[Schema compare](schema-compare.md)** | Diff project ↔ pac ↔ live workspace in any direction, read the results |
| 🛡️ | **[Safe deploy](safe-deploy.md)** | Generate migration scripts, opt-in gates, DEEP CLONE rollback, deploy manifests |
| 🚦 | **[The safety classifier](safety-classifier.md)** | What DESTRUCTIVE / UNRECOVERABLE / EXPENSIVE / WARNING mean — including UC-specific risks like managed-table file deletion |

## Reference

| | Page | What you'll learn |
|---|---|---|
| ⌨️ | **[CLI reference](cli-reference.md)** | Every `ddt` command, grouped by workflow, with flags and exit codes |
| 🧩 | **[VS Code extension reference](vscode-extension.md)** | Every command, view, setting, context menu, and keybinding |
| ⚙️ | **[Configuration reference](configuration.md)** | Every `.ddtproj` option and VS Code setting |

## Going further

| | Page | What you'll learn |
|---|---|---|
| ✨ | **[AI features](ai-features.md)** | Bring-your-own-key setup, sketch from prose, safer alternatives, Ask DDT |
| 🔁 | **[CI/CD integration](ci-cd.md)** | GitHub Actions, GitLab CI, Azure DevOps patterns; drift gates; PR comments |
| 🚚 | **[Migrating from other tools](migrating.md)** | Coming from dbt or raw SQL scripts |

## When something goes wrong

| | Page | What you'll learn |
|---|---|---|
| ❓ | **[FAQ](faq.md)** | How DDT compares to other tools, pricing, permissions, data safety |
| 🩹 | **[Troubleshooting](troubleshooting.md)** | Symptom → cause → fix, plus how to report a bug well |

---

## I want to…

| Goal | Go to |
|---|---|
| Get from zero to a deployed change | [Getting started](getting-started.md) |
| Put my existing Unity Catalog under version control | [Extract](extract.md) |
| See what changed between my project and production | [Schema compare](schema-compare.md) |
| Understand why my deploy was blocked | [The safety classifier](safety-classifier.md) → [Safe deploy](safe-deploy.md) |
| Understand why dropping a managed table is irreversible | [Safety classifier → UC-specific risks](safety-classifier.md) |
| Allow a column drop / type change on purpose | [Safety classifier → opt-in gates](safety-classifier.md) |
| Roll back a deploy | [Safe deploy → rollback](safe-deploy.md) |
| Run DDT in a GitHub Actions pipeline | [CI/CD integration](ci-cd.md) |
| Fail CI when production drifts from git | [CI/CD → drift gate](ci-cd.md) |
| Set up AI features with my own API key | [AI features](ai-features.md) |
| Find a command I can't remember | [CLI reference](cli-reference.md) — or run `ddt find <keyword>` / press <kbd>Ctrl</kbd>+<kbd>K</kbd> <kbd>F</kbd> in VS Code |
| Switch from dbt | [Migrating from other tools](migrating.md) |
| Report a bug | [Troubleshooting → Still stuck?](troubleshooting.md) or [SUPPORT](../SUPPORT.md) |

---

## Install

```sh
# VS Code extension
code --install-extension sdt-ddt-tools.ddt-vscode

# CLI (where the project lifecycle lives)
npm install -g @ddt-tools/cli
ddt --version
```

Or install **Databricks Data Tools** from the [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=sdt-ddt-tools.ddt-vscode).

> [!NOTE]
> DDT is in a **30-day public beta** — every feature is free during the beta. [What happens after?](../README.md#beta-program)

> [!TIP]
> DDT is **CLI-first**: project lifecycle (init, extract, build, publish, compare) runs in the terminal; the VS Code extension adds browsing, diagramming, and review on top.

---

**Sibling product:** working with Snowflake too? [**SDT — Snowflake Data Tools**](https://github.com/GVOrganization/sdt-tools) is the same design and CLI surface for Snowflake.
