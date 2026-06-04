# 🧩 VS Code extension reference

> Everything the Databricks Data Tools extension adds to VS Code — activity-bar views, every Command Palette command, right-click menus, settings, and the project editor.

**On this page:** [Browse/compare focus](#browsecompare-focus) · [Activity bar views](#activity-bar-views) · [Command Palette](#command-palette) · [Context menus](#context-menus) · [Settings](#settings) · [Keybindings](#keybindings) · [Walkthrough](#walkthrough) · [Project editor](#project-editor)

---

## Browse/compare focus

The DDT extension activates when you open a `.sql` file, a `.ddtproj`, or a `.ddtsuite` workspace. It adds a dedicated activity-bar container, **DDT — Databricks Data Tools**.

> [!IMPORTANT]
> The DDT extension is **browse / compare / review-focused**. Project lifecycle commands — `ddt init`, `ddt build`, `ddt publish`, `ddt extract` — are **CLI-first**: run them from a terminal (see the [CLI reference](cli-reference.md)). The extension surfaces compare, schema diagram, Object Explorer, AI assists, formatting, and deploy history; it does not embed init/build/publish/extract buttons.

---

## Activity bar views

Click the DDT icon in the activity bar to open three tree views.

| View | What it shows |
|---|---|
| **Suites** | Every `.ddtsuite` (multi-project Suite) in the workspace, with a refresh action. |
| **Object Explorer** | The live catalog → schema → object tree for a connected workspace, read from the local catalog cache. Right-click objects for refactors, lineage, schema diagram, and compare. |
| **Deploy History** | Past deploys with status-badge icons; refreshable. |

> [!TIP]
> Object Explorer reads a cached snapshot, so it never issues live queries while you browse. Refresh the cache from the terminal with `ddt catalog refresh --connection <name>`.

---

## Command Palette

Open the Command Palette (<kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>P</kbd>) and type **DDT:** to see every command.

### Compare & explore

| Command | What it does |
|---|---|
| **DDT: Open Interactive Schema Diagram** | Open the interactive schema diagram. |
| **DDT: Open Compare Profiles** | Manage saved source → target compare profiles. |
| **DDT: Open connection setup docs** | Open the workspace connection-setup guide. |
| **DDT: Validate Suite** | Validate a multi-project `.ddtsuite`. |
| **DDT: Refresh Suites** | Reload the Suites view. |

### Object Explorer

| Command | What it does |
|---|---|
| **DDT: Refresh Object Explorer** | Reload the live object tree from the cache. |
| **DDT: Copy FQN** | Copy an object's fully-qualified name. |
| **Rename Column…** | Record a column rename refactor on a table. |
| **Change Column Type…** | Record a column type-change refactor. |
| **Drop Column…** | Record a column drop refactor. |

### AI

| Command | What it does |
|---|---|
| **DDT: Sketch Object from Description (AI)** | Scaffold idiomatic UC DDL from a prose description. |
| **DDT: Suggest Safer Alternative (AI)** | Propose a safer DDL alternative for a dangerous change. |
| **DDT: Show Column Lineage** | Trace column-level lineage for a view or table. |
| **DDT: Open Ask DDT chat panel (AI)** | Open the "Ask DDT" chat panel. |
| **DDT: Apply chat-proposed refactor ops** | Apply refactor operations proposed in the chat panel. |
| **DDT: Open Visual Table Designer** | Open the visual table designer. |

### Formatting

| Command | What it does |
|---|---|
| **DDT: Format SQL Document** | Format the active `.sql` file with the DDT formatter. |

### Discovery & support

| Command | What it does |
|---|---|
| **DDT: Find a Feature** | Search every DDT feature (<kbd>Ctrl</kbd>+<kbd>K</kbd> <kbd>F</kbd>). |
| **DDT: Open Query Results panel** | Open the query-results pane. |
| **DDT: Sign in with SSO (Enterprise)** | Sign in via your org's license server. |
| **DDT: Sign out (SSO)** | Sign out of SSO. |
| **DDT: Show License Status** | Show the current license status. |
| **DDT: Report a Bug…** | Open the bug-report flow. |
| **DDT: Refresh Deploy History** | Reload the Deploy History view. |

---

## Context menus

DDT adds right-click actions that are context-aware — they appear only where they apply.

| Where you right-click | Available actions |
|---|---|
| **A table in Object Explorer** (MANAGED_TABLE, EXTERNAL_TABLE, STREAMING_TABLE) | Rename Column…, Change Column Type…, Drop Column…, Suggest Safer Alternative (AI) |
| **A view in Object Explorer** (VIEW, MATERIALIZED_VIEW, STREAMING_TABLE) | Show Column Lineage, Suggest Safer Alternative (AI) |
| **A catalog or schema in Object Explorer** | Open Interactive Schema Diagram, Open Compare Profiles |
| **A connection in Object Explorer** | Refresh Object Explorer, Refresh Deploy History |
| **In a `.sql` editor** | Format SQL Document, Suggest Safer Alternative (AI), Show Column Lineage, Open Ask DDT chat panel (AI) |

---

## Settings

Configure DDT under **Settings → Extensions → DDT — Databricks Data Tools**, or edit `settings.json` directly.

| Setting | Default | What it does |
|---|---|---|
| `ddt.defaultProfile` | `""` | Default connection profile name used by the Object Explorer. |
| `ddt.formatOnSave` | `true` | Auto-format `.sql` files on save with the DDT formatter. |
| `ddt.discoverability.level` | `standard` | Coarse discoverability knob: `minimal` suppresses hints/tooltips/CodeLens/context surfaces; `standard` is default; `verbose` adds inline Try-it buttons. |
| `ddt.discoverability.hints` | `auto` | Per-channel override for post-command hints. `auto` follows the level. |
| `ddt.discoverability.tooltips` | `auto` | Per-channel override for rich tooltips on Object Explorer nodes. |
| `ddt.discoverability.codeLens` | `auto` | Per-channel override for DDT CodeLens hints. |
| `ddt.discoverability.contextMenu` | `auto` | Per-channel override for feature-aware right-click entries. |
| `ddt.discoverability.emptyStates` | `auto` | Per-channel override for actionable empty-state placeholders (stays on even at `minimal`). |
| `ddt.discoverability.suppressedHintIds` | `[]` | Hint IDs silenced via "Don't show this again"; managed automatically. |
| `ddt.licenseServer.url` | `""` | Base URL of your org's DDT On-prem License Server (Enterprise SSO). |
| `ddt.licenseServer.oidcClientId` | `""` | OAuth2 client ID for desktop sign-in (Enterprise SSO). |
| `ddt.licenseServer.oidcAuthorizeUrl` | `""` | Identity-provider authorization endpoint (Enterprise SSO). |
| `ddt.licenseServer.oidcTokenUrl` | `""` | Identity-provider token endpoint (Enterprise SSO). |
| `ddt.licenseServer.scopes` | `[]` | OAuth scopes for SSO. Empty uses `openid profile email`. |

---

## Keybindings

| Keybinding | Command |
|---|---|
| <kbd>Ctrl</kbd>+<kbd>K</kbd> <kbd>F</kbd> (mac: <kbd>Cmd</kbd>+<kbd>K</kbd> <kbd>F</kbd>) | **DDT: Find a Feature** |

> [!TIP]
> "Find a Feature" is the in-editor equivalent of `ddt find` — when you can't remember a command name, press <kbd>Ctrl</kbd>+<kbd>K</kbd> <kbd>F</kbd> and search.

---

## Walkthrough

The first time you open VS Code with DDT installed, **Get started with DDT** appears on the Welcome page (also reachable via **Help → Welcome**). It walks you through:

1. **Configure a Databricks workspace connection** — add a PAT or OAuth-M2M profile to `~/.ddt/profiles.json`.
2. **Verify the workspace** — run `ddt connection test <name>` in a terminal to confirm Unity Catalog is reachable and your profile resolves.
3. **Author a Suite** — if you manage several `.ddtproj` projects, wrap them in a `.ddtsuite` for cross-project validation.

> [!NOTE]
> The walkthrough reflects the CLI-first model — connection setup and validation happen in the terminal, then the extension takes over for browsing, comparing, and reviewing.

---

## Project editor

`.ddtproj` files open in a dedicated **DDT Project** editor rather than as raw JSON, giving you a structured view of project scope, object folders, and deployment settings. SQL files get the built-in snippet library, live lint squiggles, and column-validation diagnostics inside any `.ddtproj`.

---

**Next:** [AI features](ai-features.md) · **Up:** [Documentation home](README.md)
