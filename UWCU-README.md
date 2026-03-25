# MCP Gateway — UWCU Fork

Fork of [microsoft/mcp-gateway](https://github.com/microsoft/mcp-gateway) customized for UWCU's NKP deployment.

## Fork Changes

### 1. Redis instead of Cosmos DB (storage backend)

**Files:** `dotnet/Microsoft.McpGateway.Service/src/Program.cs`, `dotnet/Microsoft.McpGateway.Tools/src/Program.cs`

Upstream requires Azure Cosmos DB for production mode. We replaced Cosmos DB with Redis in all environments (Development and Production), keeping Entra ID auth intact in Production mode.

**Why:** UWCU runs on-prem NKP (no Azure Cosmos DB). Redis is already deployed alongside the gateway.

### 2. Configurable adapter namespace

**Files:** `dotnet/Microsoft.McpGateway.Service/src/Program.cs`, `dotnet/Microsoft.McpGateway.Service/src/Routing/AdapterKubernetesNodeInfoProvider.cs`

Upstream hardcodes `"adapter"` as the Kubernetes namespace for managed MCP server pods. We made it configurable via the `AdapterNamespace` config value (defaults to `"adapter"` for backwards compatibility).

**Why:** UWCU deploys to `mcp-gateway-np`, not `adapter`.

### 3. Dockerfiles + ADO pipeline

**Files:** `dotnet/Microsoft.McpGateway.Service/Dockerfile`, `dotnet/Microsoft.McpGateway.Tools/Dockerfile`, `azure-pipelines.yml`

Upstream uses `dotnet publish` with publish profiles (no Dockerfiles). We added multi-stage Dockerfiles and an Azure Pipelines definition for building on NKP-Agents and pushing to Harbor.

### 4. Kustomize manifests

**Directory:** `base/`, `overlays/`

K8s deployment manifests following the ai-gateway Kustomize pattern. FluxCD syncs from `overlays/dev/` to `mcp-gateway-np`.

## Architecture

```
Client (VS Code, Claude Code, uChat)
  ↓
mcp-gateway.nkp-np.uwcu.org (Traefik Ingress)
  ↓
mcpgateway pod (auth + session routing)
  ↓                          ↓
/adapters/{name}/mcp         /mcp (tool router)
  ↓                          ↓
mcp-proxy pod             toolgateway pod
  ↓                          ↓
actual MCP server         routes to registered tools
(in mcp-servers-np)
```

## How to Register MCP Servers

The gateway manages MCP servers as "adapters". Our servers run in `mcp-servers-np` and are proxied through the `mcp-proxy` image.

### Register a server

```bash
curl -sk -X POST https://mcp-gateway.nkp-np.uwcu.org/adapters \
  -H "Content-Type: application/json" \
  -d '{
    "name": "nutanix-mcp",
    "imageName": "mcp-gateway/mcp-proxy",
    "imageVersion": "latest",
    "environmentVariables": {
      "MCP_PROXY_URL": "http://nutanix-mcp.mcp-servers-np.svc.cluster.local:8080/mcp"
    },
    "description": "Nutanix Prism Central MCP server"
  }'
```

### List registered servers

```bash
curl -sk https://mcp-gateway.nkp-np.uwcu.org/adapters
```

### Connect an MCP client

VS Code `.vscode/mcp.json`:
```json
{
  "servers": {
    "nutanix": {
      "url": "https://mcp-gateway.nkp-np.uwcu.org/adapters/nutanix-mcp/mcp"
    },
    "all-tools": {
      "url": "https://mcp-gateway.nkp-np.uwcu.org/mcp"
    }
  }
}
```

### All MCP server registrations

| Name | Proxy URL |
|------|-----------|
| nutanix-mcp | `http://nutanix-mcp.mcp-servers-np.svc.cluster.local:8080/mcp` |
| openrouter-image-mcp | `http://openrouter-image-mcp.mcp-servers-np.svc.cluster.local:3001/mcp` |
| exa-search-mcp | `http://exa-search-mcp.mcp-servers-np.svc.cluster.local:8000/mcp` |
| ivanti-mcp | `http://ivanti-mcp.mcp-servers-np.svc.cluster.local:8090/mcp` |
| ado-mcp | `http://ado-mcp.mcp-servers-np.svc.cluster.local:8055/mcp` |
| logicmonitor-mcp | `http://logicmonitor-mcp.mcp-servers-np.svc.cluster.local:3000/mcp` |

## Syncing with Upstream

```bash
git fetch upstream
git merge upstream/main
# Resolve conflicts in Program.cs and AdapterKubernetesNodeInfoProvider.cs if needed
git push origin main
git push ado main
```

## Environment

- **Cluster:** fit-nkp-nonprod
- **Gateway namespace:** mcp-gateway-np
- **MCP servers namespace:** mcp-servers-np
- **Ingress:** mcp-gateway.nkp-np.uwcu.org
- **Harbor:** nkp-mgmt.uwcu.org:5000/mcp-gateway/*
- **Auth:** Development mode (no auth) — switch to Production for Entra ID
