# 🔌 Connecting to Databricks

> Set up a connection profile so DDT can extract, compare, and deploy against a live Databricks workspace.

**On this page:** [The profiles file](#the-profiles-file) · [Pick an auth method](#pick-an-auth-method) · [PAT](#personal-access-token-pat) · [OAuth M2M](#oauth-machine-to-machine-m2m) · [OAuth U2M](#oauth-user-to-machine-u2m) · [Azure AD](#azure-ad) · [Google Identity](#google-identity) · [Env-var placeholders](#env-var-placeholders) · [Test the connection](#testing-the-connection) · [Troubleshooting](#troubleshooting)

---

## The profiles file

A connection profile carries your workspace host, auth credentials, SQL warehouse ID, and default catalog / schema so day-to-day commands don't repeat them. DDT reaches the workspace through two endpoints: the SQL Warehouse (for `SHOW` / `DESCRIBE` / `CREATE` / `ALTER` on Unity Catalog objects) and the Workspace REST API (for jobs, pipelines, and other workspace resources).

Profiles live at:

- `~/.ddt/profiles.json` (Linux/macOS)
- `%USERPROFILE%\.ddt\profiles.json` (Windows)

The file stores host, warehouse ID, catalog/schema defaults, and auth-method metadata — **never** the raw secret. A PAT profile looks like:

```json
{
  "version": 1,
  "profiles": [
    {
      "name": "dev",
      "platform": "Databricks",
      "auth": {
        "method": "PERSONAL_ACCESS_TOKEN",
        "host": "adb-1234567890123456.7.azuredatabricks.net",
        "token": "env:DATABRICKS_TOKEN"
      },
      "warehouseId": "abc123def456",
      "catalog": "main",
      "schema": "default"
    }
  ]
}
```

> [!TIP]
> You rarely write this file by hand — `ddt connection add` (below) writes it for you.

> [!IMPORTANT]
> Never commit credentials. Keep tokens and client secrets **outside the repo** and reference them with `env:VAR_NAME` placeholders (see [Env-var placeholders](#env-var-placeholders)). The profiles file holds metadata only; secrets resolve from env vars or the OS keychain at runtime.

### What you'll need

- A **workspace URL**, e.g. `https://adb-1234567890123456.7.azuredatabricks.net` (Azure) or `https://dbc-12345-abcd.cloud.databricks.com` (AWS/GCP).
- A **SQL warehouse**. In the UI: **SQL → SQL Warehouses →** your warehouse **→ Connection Details**. The ID looks like `1234abc5d678e9f0`.
- An **authentication method** (below).
- A **default catalog** and **schema** (optional, but recommended — otherwise every command takes them as flags).

---

## Pick an auth method

| Method | When to use | Notes |
|---|---|---|
| **Personal Access Token (PAT)** | Solo dev or interactive workstation | Simplest setup. Token expires (default 90 days); revoke if leaked. |
| **OAuth Machine-to-Machine (M2M)** | CI/CD, service-principal workloads | Recommended for automation. Requires a service principal + client secret. |
| **OAuth User-to-Machine (U2M)** | Interactive dev without a PAT on disk | Opens a browser. No long-lived secret. |
| **Azure AD** | Azure Databricks under an Entra ID tenant | Works with Managed Identities and Azure SP secrets. |
| **Google Identity** | Databricks on GCP | Requires a GCP service-account key. |

For a brand-new setup, start with **PAT** to validate end-to-end, then move to **OAuth M2M** before shipping the connection to CI.

> [!NOTE]
> PAT and OAuth M2M are wired today. OAuth U2M, Azure AD, and Google Identity are documented; the current beta returns a clear "not yet implemented" message for those methods.

---

## Personal Access Token (PAT)

### 1. Generate the token

1. In the workspace UI, click your avatar → **User Settings**.
2. **Developer → Access tokens → Generate new token**.
3. Comment it `ddt-cli`, set a lifetime, and copy the token immediately — Databricks shows it once.

### 2. Store it in an env var

```sh
export DATABRICKS_TOKEN="dapi-…"
export DATABRICKS_HOST="adb-1234567890123456.7.azuredatabricks.net"
export DATABRICKS_WAREHOUSE_ID="1234abc5d678e9f0"
```

### 3. Add the profile

```sh
ddt connection add --name prod-workspace \
  --host $DATABRICKS_HOST \
  --auth pat \
  --token env:DATABRICKS_TOKEN \
  --warehouse-id $DATABRICKS_WAREHOUSE_ID \
  --catalog main \
  --schema bronze
```

`--catalog` and `--schema` set the defaults for commands that don't pass them explicitly.

### 4. Test it

```sh
ddt connection test prod-workspace
```

You'll see:

```
workspace version, current user, and the warehouse name
```

---

## OAuth Machine-to-Machine (M2M)

### 1. Create a service principal

1. In the **Account console**, go to **User management → Service principals → Add service principal**.
2. **Generate secret → Add**. Copy the client ID and client secret.

### 2. Grant it access

1. Add the service principal to your workspace (**Settings → Admin Console → Identity and access**).
2. On the SQL warehouse, grant it at least **CAN USE**.
3. In Unity Catalog, grant `USE CATALOG`, `USE SCHEMA`, and the per-object privileges (`MODIFY`, `CREATE`, `SELECT`, …) it needs.

### 3. Add the profile

```sh
ddt connection add --name ci-workspace \
  --host $DATABRICKS_HOST \
  --auth oauth-m2m \
  --client-id env:DATABRICKS_CLIENT_ID \
  --client-secret env:DATABRICKS_CLIENT_SECRET \
  --warehouse-id $DATABRICKS_WAREHOUSE_ID \
  --catalog main \
  --schema gold
```

---

## OAuth User-to-Machine (U2M)

Best for interactive development without long-lived secrets. DDT opens a browser, you sign in, and DDT receives a refresh-token grant on the loopback callback. The refresh token is stored OS-keychain-encrypted — never written plaintext.

```sh
ddt connection add --name dev-workspace \
  --host $DATABRICKS_HOST \
  --auth oauth-u2m \
  --warehouse-id $DATABRICKS_WAREHOUSE_ID
```

> [!NOTE]
> This method is documented; the current beta returns a "not yet implemented" message until it is wired.

---

## Azure AD

For Azure Databricks running under an Entra ID tenant.

### With a service principal + secret

1. Azure Portal → **Microsoft Entra ID → App registrations → New registration**.
2. Copy the **Application (client) ID** and **Directory (tenant) ID**.
3. **Certificates & secrets → New client secret**. Copy the secret value.
4. Grant the SP access to the workspace and warehouse (same as the M2M flow).

```sh
ddt connection add --name azure-prod \
  --host $DATABRICKS_HOST \
  --auth azure-ad \
  --tenant-id env:DATABRICKS_AZURE_TENANT_ID \
  --client-id env:DATABRICKS_AZURE_CLIENT_ID \
  --client-secret env:DATABRICKS_AZURE_CLIENT_SECRET \
  --warehouse-id $DATABRICKS_WAREHOUSE_ID
```

### With a Managed Identity

Omit `--client-secret`; DDT discovers the managed identity via `DefaultAzureCredential`. Add the MI to the workspace and warehouse first.

> [!NOTE]
> This method is documented; the current beta returns a "not yet implemented" message until it is wired.

---

## Google Identity

For Databricks on GCP, using a GCP service-account key.

> [!NOTE]
> This method is documented; the current beta returns a "not yet implemented" message until it is wired.

---

## Env-var placeholders

Any secret-bearing flag accepts an `env:VAR_NAME` placeholder. DDT reads the value from the named environment variable **at runtime**, not when the profile is written, so the secret never lands in `profiles.json`:

```sh
--token env:DATABRICKS_TOKEN
--client-secret env:DATABRICKS_CLIENT_SECRET
```

For U2M, the refresh token is stored encrypted in the OS keychain rather than as an env var.

---

## Testing the connection

Once a profile is in place, verify it from the CLI:

```sh
ddt connection test prod-workspace   # auth + warehouse probe
ddt connection list                  # list profiles (secrets redacted)
```

`ddt connection test <name>` resolves the profile (env vars expanded), authenticates, and probes the SQL warehouse the profile references — a quick way to confirm the workspace is reachable before you run anything heavier. (`ddt validate` is a project-side schema/reference check and takes `-p <project>`, not a connection.)

---

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `401 Unauthorized` on test | PAT expired or revoked, or wrong client secret. Mint a new credential. |
| `403 Forbidden` on `SHOW CATALOGS` | The principal lacks `USE CATALOG` in Unity Catalog. Grant it. |
| `403 forbidden on the SQL Warehouse` | The principal lacks `CAN USE` on the warehouse. Grant it from the warehouse permissions panel. |
| `Unity Catalog is not enabled` | The workspace predates UC. Enable UC in the account console or pick a different workspace. |
| `503 Service Unavailable` on first query | Warehouse is cold-starting. Retry in ~30s, or pre-warm it. |
| `Could not find SQL warehouse` / `404 /api/2.0/sql/warehouses` | Wrong warehouse ID, or it lives in a different workspace than the host. |
| `WORKSPACE_DOES_NOT_EXIST` | Wrong host. Hosts are workspace-scoped (`adb-<id>.<n>.azuredatabricks.net`), not account-scoped. |
| `redirect_uri_mismatch` (OAuth U2M) | Loopback port in use by another process; restart DDT to pick a new port. |

---

**Next:** [Projects & suites](projects.md) · **Up:** [Documentation home](README.md)
