# falcon-mcp-aca

Deploy the **CrowdStrike Falcon MCP server** (`falcon-mcp`) as an **Azure Container App** in `streamable-http` mode, ready to plug into **Microsoft Copilot Studio** (or any MCP-capable agent) as a remote tool server.

This repo ships a single **ARM template** (`azuredeploy.json`) that provisions everything needed end to end:

- a **Log Analytics workspace** (container logs / telemetry)
- a **Container Apps managed environment**
- the **`falcon-mcp` Container App** itself, exposed over HTTPS at `/mcp`

The MCP endpoint is gated by a self-generated API key (separate from your Falcon API credentials), so only your agent can reach the tools.

---

## Architecture

```
SOC Analyst ──► Agentic layer (Copilot Studio / LangChain + CrewAI)
                         │  HTTPS + MCP API key
                         ▼
        Azure Container App  ──  falcon-mcp  (streamable-http, /mcp)
                         │  FALCON_CLIENT_ID / SECRET
                         ▼
                CrowdStrike Falcon API  (detections, incidents, intel, hosts)
```

The Container App exposes one public HTTPS endpoint. The agent connects to that endpoint over the **streamable-http** MCP transport, presenting the MCP API key. The server then calls the Falcon API on the backend using your Falcon OAuth2 client credentials, which never leave Azure (they are stored as Container App secrets).

---

## What gets deployed

| Resource | Type | Purpose |
|---|---|---|
| `law-falcon-mcp` | `Microsoft.OperationalInsights/workspaces` | Centralized logs for the Container App |
| `cae-falcon-mcp` | `Microsoft.App/managedEnvironments` | Container Apps environment |
| `ca-falcon-mcp` | `Microsoft.App/containerApps` | The `falcon-mcp` server, HTTPS ingress on port 8000 |

Defaults: `0.5` vCPU / `1 GiB` per replica, scaling `1 → 3` replicas. Stateless HTTP is enabled (`FALCON_MCP_STATELESS_HTTP=true`), so running more than one replica is safe.

---

## Prerequisites

- An **Azure subscription** and a resource group
- **Azure CLI** (`az`) or access to the Azure Portal
- A **CrowdStrike Falcon API client** (Client ID + Secret) scoped to least privilege — grant only the scopes matching the modules you enable (`detections`, `incidents`, `intel`, `hosts`)
- Your correct **Falcon API base URL** for your tenant region:

  | Region | Base URL |
  |---|---|
  | US-1 | `https://api.crowdstrike.com` |
  | US-2 | `https://api.us-2.crowdstrike.com` |
  | EU-1 | `https://api.eu-1.crowdstrike.com` |
  | GovCloud | `https://api.laggar.gcw.crowdstrike.com` |

---

## Step 1 — Generate the MCP API key

The **MCP API key** is a secret *you* create. It is **not** a CrowdStrike credential. It gates access to the MCP HTTP endpoint — it is the credential your agent (Copilot Studio) presents to prove it is allowed to call the server.

Generate a strong 256-bit random key:

```bash
# Linux / macOS / WSL — 64 hex chars (256 bits)
openssl rand -hex 32
```

```powershell
# Windows PowerShell
[Convert]::ToHexString((1..32 | ForEach-Object { Get-Random -Maximum 256 }))
```

```bash
# Alternative, if openssl is not available
python3 -c "import secrets; print(secrets.token_hex(32))"
```

Copy the output. You will pass it as the `mcpApiKey` parameter at deploy time, and reuse the **same value** when you configure the connection in Copilot Studio.

> **Treat this like a password.** Don't commit it. Rotate it by redeploying with a new value (or updating the `mcp-api-key` secret on the Container App). Anyone holding this key can reach your Falcon tools.

---

## Step 2 — Deploy the template

### Option A — Azure CLI

```bash
# 1. Create a resource group (skip if you already have one)
az group create \
  --name rg-falcon-mcp \
  --location canadacentral

# 2. Deploy
az deployment group create \
  --resource-group rg-falcon-mcp \
  --template-file azuredeploy.json \
  --parameters \
      falconClientId="<YOUR_FALCON_CLIENT_ID>" \
      falconClientSecret="<YOUR_FALCON_CLIENT_SECRET>" \
      falconBaseUrl="https://api.crowdstrike.com" \
      mcpApiKey="<PASTE_THE_KEY_FROM_STEP_1>"
```

To restrict the exposed tools, override the modules (least privilege):

```bash
      falconMcpModules="detections,incidents"
```

### Option B — Portal

Use **Deploy a custom template** → **Build your own template in the editor**, paste `azuredeploy.json`, and fill in the parameters.

---

## Step 3 — Get the endpoint

After deployment, read the outputs:

```bash
az deployment group show \
  --resource-group rg-falcon-mcp \
  --name azuredeploy \
  --query properties.outputs.mcpServerUrl.value -o tsv
```

You'll get something like:

```
https://ca-falcon-mcp.<random>.canadacentral.azurecontainerapps.io/mcp
```

That `/mcp` URL is what you give to Copilot Studio.

---

## Step 4 — Connect from Copilot Studio

1. In Copilot Studio, add a **custom connector / MCP server** pointing at the `/mcp` URL from Step 3.
2. Set the transport to **streamable-http**.
3. Add the **MCP API key** from Step 1 as the auth credential (header), so requests are authenticated.
4. Save and test — the Falcon tools (per the modules you enabled) should now be callable by your agent.

---

## Parameters reference

| Parameter | Required | Default | Notes |
|---|---|---|---|
| `falconClientId` | ✅ | — | CrowdStrike API client ID |
| `falconClientSecret` | ✅ | — | CrowdStrike API client secret (securestring) |
| `falconBaseUrl` | ✅ | `https://api.crowdstrike.com` | Must match your tenant region |
| `mcpApiKey` | ✅ | — | Self-generated key (Step 1), securestring |
| `falconMcpModules` | | `detections,incidents,intel,hosts` | Comma-separated; trim to least privilege |
| `containerImage` | | `quay.io/crowdstrike/falcon-mcp:latest` | Pin a tag for production |
| `containerAppName` | | `ca-falcon-mcp` | |
| `environmentName` | | `cae-falcon-mcp` | |
| `logAnalyticsName` | | `law-falcon-mcp` | |
| `location` | | resource group location | |
| `minReplicas` / `maxReplicas` | | `1` / `3` | Stateless HTTP, safe to scale out |

---

## Security notes

- **Secrets stay in Azure.** Falcon client ID/secret and the MCP API key are stored as Container App secrets and surfaced to the container via `secretRef` env vars — they are not baked into the image and never returned in template outputs.
- **Least privilege.** Scope the Falcon API client to only what the enabled modules need, and trim `falconMcpModules` accordingly.
- **Pin the image.** For production, replace `:latest` with a specific digest/tag.
- **Rotate the MCP key** periodically by redeploying or updating the `mcp-api-key` secret.
- Consider fronting the Container App with IP restrictions or private networking if your agent runtime supports it.

---

## License

MIT — see `LICENSE`.
