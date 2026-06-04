# 🔁 CI/CD integration

> Run DDT in your pipeline so every pull request previews Unity Catalog changes and every merge deploys them safely.

**On this page:** [The CI pattern](#the-ci-pattern) · [Exit codes for gating](#exit-codes-for-gating) · [GitHub Actions](#github-actions) · [GitLab CI](#gitlab-ci) · [Azure DevOps](#azure-devops) · [Secrets in CI](#secrets-in-ci) · [Scheduled drift gate](#scheduled-drift-gate) · [PR diff comments](#pr-diff-comments) · [Docker image](#docker-image)

---

## The CI pattern

DDT — Databricks Data Tools fits the standard four-stage pipeline. The same `.ddtpac` artifact you build in CI is the artifact you deploy, so what you reviewed is exactly what ships.

| Stage | Command | Runs on |
|---|---|---|
| **Build** | `ddt build` — compile `.ddtproj` to `.ddtpac` | every push + PR |
| **Compare** | `ddt drift` — diff the project against the live workspace | every PR |
| **Review gate** | `ddt publish --dry-run` — show the migration without running it | every PR |
| **Publish** | `ddt publish --apply --yes` — deploy on merge | merge to `main` |

The rule of thumb: **dry-run in the PR, apply on merge.** A PR shows the diff and fails the build if it carries unapproved destructive changes; the merge job applies the same plan.

> [!IMPORTANT]
> Always build the pac once and pass it forward as a pipeline artifact. Re-building between the review job and the deploy job risks deploying something the reviewer never saw.

> [!WARNING]
> On Unity Catalog, dropping a **managed table** deletes its underlying files, and altering a **streaming table** can lose its checkpoint. The review gate exists to catch exactly these — never let a PR merge with an unapproved destructive finding.

---

## Exit codes for gating

Every `ddt` command returns a stable exit code. Gate your pipeline on these.

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | User error (bad args, missing file, syntax error) |
| 2 | System error (network, permission, internal failure) |
| 3 | Compare diff found (drift detected in CI, flag-controlled) |
| 4 | Destructive changes detected and not approved |
| 5 | Drift detected (`ddt drift` in CI mode) |
| 6 | Locked feature attempted without a Pro/Team/Enterprise license |

In any non-interactive shell — when `stdin` is not a TTY, or `CI=true` — DDT never prompts and never opens a browser. Pass `--yes` (or `--confirm`) explicitly, and use `--output json` for machine-readable results.

> [!TIP]
> Exit code 4 is your destructive-change tripwire. A PR job that runs `ddt publish --dry-run --fail-on-destructive` fails the build before anyone can merge a change that would delete managed-table files.

---

## GitHub Actions

Install the published CLI and drive it directly — this works on any runner today, with no extra Marketplace dependency. **OAuth M2M (a service principal, not a personal token) is the recommended CI auth method.**

### PR validation — build, dry-run, fail on destructive

```yaml
name: Validate DDT project on PR
on:
  pull_request:

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm install -g @ddt-tools/cli

      # Build the artifact once.
      - run: ddt build --project ./catalogs/MyProject.ddtproj --out ./bin/MyProject.ddtpac

      # Register the workspace connection from env-var secrets (see "Secrets in CI").
      - run: ddt connection add --name staging
               --host "$DATABRICKS_HOST"
               --token env:DATABRICKS_TOKEN
               --warehouse-id "$DATABRICKS_WAREHOUSE_ID"
        env:
          DATABRICKS_HOST: ${{ vars.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
          DATABRICKS_WAREHOUSE_ID: ${{ vars.DATABRICKS_WAREHOUSE_ID }}

      # Snapshot the live workspace into a pac to diff against (compare/publish are pac ↔ pac).
      - run: ddt extract --connection staging --out-pac ./bin/live.ddtpac

      # Dry-run the deploy; exit code 4 fails the build on unapproved destructive change.
      - run: ddt publish --source ./bin/MyProject.ddtpac --target ./bin/live.ddtpac
               --connection staging --dry-run
```

### Deploy on merge

```yaml
name: Deploy DDT project
on:
  push: { branches: [main] }

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm install -g @ddt-tools/cli
      - run: ddt build --project ./catalogs/MyProject.ddtproj --out ./bin/MyProject.ddtpac
      - run: ddt connection add --name prod
               --host "$DATABRICKS_HOST" --token env:DATABRICKS_TOKEN
               --warehouse-id "$DATABRICKS_WAREHOUSE_ID"
        env:
          DATABRICKS_HOST: ${{ vars.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
          DATABRICKS_WAREHOUSE_ID: ${{ vars.DATABRICKS_WAREHOUSE_ID }}
      # Snapshot the live workspace into a pac to diff against (publish is pac ↔ pac).
      - run: ddt extract --connection prod --out-pac ./bin/live.ddtpac
      # Real deploy. The safety classifier runs first and blocks unapproved destructive change.
      - run: ddt publish --source ./bin/MyProject.ddtpac --target ./bin/live.ddtpac
               --connection prod --apply --yes
```

> [!NOTE]
> A dedicated `GVOrganization/ddt-action` GitHub Action is planned but not yet published. Until it ships, use the CLI-in-CI pattern above — it is the supported way to run DDT in GitHub Actions today.

---

## GitLab CI

```yaml
stages: [build, deploy-dev, deploy-prod]
image: node:20

variables:
  DATABRICKS_HOST: $DATABRICKS_HOST
  DATABRICKS_WAREHOUSE_ID: $DATABRICKS_WAREHOUSE_ID

build:
  stage: build
  script:
    - npm install -g @ddt-tools/cli
    - ddt build --project ./catalogs/MyProject.ddtproj --out ./bin/MyProject.ddtpac
  artifacts: { paths: [./bin/*.ddtpac] }

deploy_dev:
  stage: deploy-dev
  script:
    - npm install -g @ddt-tools/cli
    - ddt connection add --name dev --host "$DATABRICKS_HOST" --token env:DATABRICKS_TOKEN --warehouse-id "$DATABRICKS_WAREHOUSE_ID"
    - ddt extract --connection dev --out-pac ./bin/live.ddtpac
    - ddt publish --source ./bin/MyProject.ddtpac --target ./bin/live.ddtpac --connection dev --apply --yes
  environment: dev

deploy_prod:
  stage: deploy-prod
  when: manual           # manual approval gate before production
  script:
    - npm install -g @ddt-tools/cli
    - ddt connection add --name prod --host "$DATABRICKS_HOST" --token env:DATABRICKS_TOKEN --warehouse-id "$DATABRICKS_WAREHOUSE_ID"
    - ddt extract --connection prod --out-pac ./bin/live.ddtpac
    - ddt publish --source ./bin/MyProject.ddtpac --target ./bin/live.ddtpac --connection prod --apply --yes
  environment: prod
```

`ddt publish --apply` runs the safety classifier first; an UNRECOVERABLE / DESTRUCTIVE finding blocks the deploy unless the matching `allow*` deploy option is set in `.ddtproj`.

---

## Azure DevOps

A multi-stage pipeline with a production approval gate. Install the CLI in each stage and pass the pac forward as a pipeline artifact.

```yaml
trigger:
  branches: { include: [main] }

stages:
  - stage: Build
    jobs:
      - job: BuildPac
        steps:
          - task: NodeTool@0
            inputs: { versionSpec: '20.x' }
          - script: npm install -g @ddt-tools/cli
          - script: ddt build --project ./catalogs/MyProject.ddtproj --out ./bin/MyProject.ddtpac
          - publish: ./bin
            artifact: pac

  - stage: DeployProd
    dependsOn: Build
    jobs:
      - deployment: DeployProd
        environment: prod    # an AzDO Environment with an approval check
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: pac
                - script: npm install -g @ddt-tools/cli
                - script: >
                    ddt connection add --name prod --host "$(DATABRICKS_HOST)"
                    --token env:DATABRICKS_TOKEN --warehouse-id "$(DATABRICKS_WAREHOUSE_ID)"
                  env:
                    DATABRICKS_TOKEN: $(DATABRICKS_TOKEN)
                - script: >
                    ddt extract --connection prod
                    --out-pac $(Pipeline.Workspace)/pac/live.ddtpac
                - script: >
                    ddt publish --source $(Pipeline.Workspace)/pac/MyProject.ddtpac
                    --target $(Pipeline.Workspace)/pac/live.ddtpac
                    --connection prod --apply --yes
```

Attach an **Approval and check** to the `prod` Environment so a human signs off before the deploy step runs.

> [!TIP]
> `ddt lint --format sarif` emits SARIF 2.1.0 — Azure DevOps and GitHub code-scanning both ingest it, so risky-deploy findings appear inline on the PR.

---

## Secrets in CI

Never inline credentials. Connection profiles resolve `env:<VAR_NAME>` placeholders at runtime, so the secret stays in your CI secret store.

```bash
# Personal access token (or M2M token) referenced via env var — never written to the profile file.
ddt connection add --name prod --host "$DATABRICKS_HOST" \
  --token env:DATABRICKS_TOKEN --warehouse-id "$DATABRICKS_WAREHOUSE_ID"
```

The profile store records the placeholder, not the secret value.

> [!IMPORTANT]
> Use an OAuth M2M service principal for CI, not a personal access token tied to an individual. A service principal survives staff changes and scopes cleanly to the deploy workspace.

---

## Scheduled drift gate

Run `ddt drift-gate` on a schedule to catch a workspace that has wandered from its declared state — including replica workspaces drifting from a primary.

```yaml
name: Nightly drift gate
on:
  schedule:
    - cron: '0 6 * * *'   # 06:00 UTC daily

jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm install -g @ddt-tools/cli
      # Fail (exit 5) if any replica drifts from the primary by more than 0 differences.
      - run: ddt drift-gate --primary prod-us --replicas prod-eu,prod-apac --threshold 0
```

For a single workspace, `ddt drift --source ./MyProject.ddtproj --connection prod` reports drift against the project and exits 5 when drift is found.

---

## PR diff comments

Post the schema diff straight onto the pull request. `ddt pr-comment` renders the project diff, lint findings, and impact analysis as a GitHub-flavoured Markdown comment (Team tier).

```yaml
- run: npm install -g @ddt-tools/cli
- run: ddt pr-comment --base main --head HEAD --format md > comment.md
- uses: actions/github-script@v7
  with:
    script: |
      const fs = require('fs');
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: fs.readFileSync('comment.md', 'utf8'),
      });
```

`ddt install-hooks` installs a matching pre-commit hook that runs `ddt validate` + `ddt lint` on changed `.sql` files, so the same checks fire before the commit ever reaches CI.

---

## Docker image

> [!NOTE]
> A published DDT Docker image (`sdtddttools/ddt`) is planned but not yet available in the current beta. Until it lands, install the CLI in your pipeline with `npm install -g @ddt-tools/cli` as shown in the examples above.

When published, the image will run as a non-root user with `tini` as the init process, ship multi-arch (`linux/amd64`, `linux/arm64`), and target under 100 MB compressed.

---

**Next:** [Migrating from other tools](migrating.md) · **Up:** [Documentation home](README.md)
