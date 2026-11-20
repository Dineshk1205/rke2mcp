# Part 3: Chat with Your Rancher Clusters Using MCP and Open WebUI

This is Part 3 of the series **Building a GPU AI Platform with MAAS, RKE2 and Rancher**.
Read Parts 1 and 2 first to understand the setup this part builds on:

| Part | Topic | Link |
|---|---|---|
| 1 | Building an RKE2 Management Cluster with Rancher Manager Using MAAS | https://kdinesh.in/rke3mgmt/ |
| 2 | Deploying a GPU AI Workload Cluster with Rancher and Fleet | https://kdinesh.in/rke2gpu/ |
| 3 | Chat with your Rancher clusters using MCP and Open WebUI | this repo |

## How the parts connect

- **Part 1** uses MAAS to install the servers and builds the management cluster with Rancher
  Manager, Longhorn and monitoring.
- **Part 2** creates the `ai-prod` GPU cluster from Rancher and uses Fleet to deploy the platform
  and the AI apps: Ollama, vLLM, Open WebUI and Qdrant for RAG.
- **Part 3** (this repo) connects the AI stack back to the platform. A Rancher MCP server runs on
  `ai-prod`, and Open WebUI uses it as a tool. The local models can then answer questions about the
  clusters themselves, such as "which pods are not running?" or "show the vllm logs".

It is deployed the same way as Part 2: a separate Git repo, synced to `ai-prod` by Fleet.

## What is MCP

[Model Context Protocol](https://modelcontextprotocol.io) is an open standard that lets an AI
model use external tools. An MCP server exposes actions for one system (here: Rancher), and the
AI app (here: Open WebUI) calls them when the model needs live data.

Server used: [futuretea/rancher-mcp-server](https://github.com/futuretea/rancher-mcp-server)
(community project, not an official SUSE product).

```
Open WebUI (ns ai)  --MCP-->  rancher-mcp (ns ai)  --API-->  Rancher (mgmt)  -->  local + ai-prod
```

## Layout

```
.
├── README.md
└── rancher-mcp/
    ├── fleet.yaml          Fleet bundle, namespace ai
    └── rancher-mcp.yaml    Deployment + Service (port 8080, endpoint /mcp)
```

## Requirements

- Rancher with the `ai-prod` cluster labelled `env=ai-prod`
- Parts 1 and 2 completed: namespace `ai`, Open WebUI 0.6.31 or newer and Ollama with `qwen2.5:7b` (from Part 2)
- Pods can reach `https://rancher.gpu.lab` and the npm registry (the package is downloaded at start)

## Setup

### 1. Rancher API token

Rancher > avatar > **Account & API Keys** > **Create API Key**.
Scope: none (to see all clusters). Set an expiry. Copy the **Bearer Token**.

### 2. Secret on ai-prod

The token is not stored in Git. Create it once:

```bash
kubectl -n ai create secret generic rancher-mcp-token --from-literal=token='<BEARER_TOKEN>'
```

### 3. Add the repo to Fleet

Rancher > **Continuous Delivery** > **Git Repos** > **Add Repository** (workspace `fleet-default`):

| Field | Value |
|---|---|
| Name | `rancher-mcp` |
| Repository URL | `https://github.com/<user>/<repo>.git` |
| Branch | `main` |
| Paths | `rancher-mcp` |
| Target | Cluster label `env=ai-prod` |

Check:

```bash
kubectl -n ai get pods -l app=rancher-mcp
kubectl -n ai logs deploy/rancher-mcp
```

### 4. Connect Open WebUI

Admin Settings > **External Tools** > **+**:

| Field | Value |
|---|---|
| Type | **MCP (Streamable HTTP)** (not OpenAPI) |
| URL | `http://rancher-mcp.ai.svc.cluster.local:8080/mcp` |
| Auth | None |

In a chat, select the model `qwen2.5:7b` and enable the `rancher-mcp` tool.
If the model does not call tools, set **Function Calling = Native** in the model's Advanced Params.

## Example questions

- List all clusters in Rancher.
- Show the nodes of ai-prod and their status.
- Which pods in namespace ai are not Running?
- Show the last 50 log lines of deployment vllm in namespace ai.
- Show recent warning events on ai-prod.

## Permissions

The server starts **read-only**. Change the `args` in `rancher-mcp/rancher-mcp.yaml`, commit, and
Fleet updates the pod.

| Flag | Default here | Effect |
|---|---|---|
| `--read-only` | `true` | `false` allows create and patch |
| `--disable-destructive` | `true` | `false` allows delete (needs `--read-only=false`) |
| `--show-sensitive-data` | `false` | `true` shows Secret values to the model |
| `--enable-container-exec` | not set | `true` allows commands in pods (needs `--read-only=false`) |
| `--toolsets` | `kubernetes,rancher` | Tool groups offered to the model |

The Rancher token also limits access: the server can never do more than the Rancher user that
owns the token. For less access, create a separate Rancher user with only the roles it needs and
use its token.

## Change the token

```bash
kubectl -n ai create secret generic rancher-mcp-token --from-literal=token='<NEW_TOKEN>' \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl -n ai rollout restart deploy/rancher-mcp
```

## Notes

- `--rancher-tls-insecure=true` is used because Rancher runs with its own self-signed CA.
- `@latest` downloads the newest version at every pod start. Pin a version (for example
  `@futuretea/rancher-mcp-server@1.0.0`) once you are happy with it.
- The service is ClusterIP only. Do not expose it with an Ingress without authentication.
