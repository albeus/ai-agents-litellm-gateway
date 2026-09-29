# ai-agents-litellm-gateway

A local lab that puts a LiteLLM Proxy in front of a local Ollama model and a hand-written MCP server, to test what a gateway can and cannot enforce: one endpoint, separate team identities, an audit trail, and per-team tool access.

> **Part of the [AI Agents Lab](https://github.com/albeus/ai-agents-lab).** This is a learning and governance exercise, not a production deployment. **Per-team MCP tool access was not proven to work in this lab.** Read [Results](#results) and [Known issue](#known-issue-mcp-grants-not-persisted) before relying on any access-control behaviour.

## Safety notice

- Never commit `.env`, generated keys, team or key JSON output, or database volumes. The tutorial below writes key output to a git-ignored `secrets/` directory for this reason.
- Everything is intended to run on one machine. LiteLLM is published on `127.0.0.1` only, but making Ollama and the MCP server reachable from a container may require binding them to `0.0.0.0`. That is **not** loopback-only. Check your host firewall and do not expose these unauthenticated endpoints to an untrusted network.
- Use only lab data. The MCP server behind the gateway runs with the permissions of the local user and any kubeconfig it is given.
- Results here come from one build of each component. Pin versions and retest before drawing conclusions about a later release.

## Goal

Turn a bare local model and a hand-written MCP server into a more operated service by putting a gateway in front of both:

- **One endpoint** for models and MCP tools, instead of clients talking to Ollama and the MCP server directly.
- **Identity separated from capability**: two teams (`itsops`, `servicedesk`) with their own virtual keys, so policy sits on the gateway rather than on trust in the client.
- **A durable record** of who called what, backed by PostgreSQL rather than an in-memory proxy that loses state on restart.
- A concrete, testable difference between "the model works" and "the tool-access policy is enforced".

Non-goals: fine-tuning, RAG design, choosing a portal UI, and container hardening beyond basic network scoping.

## Architecture

```text
curl / client
     │
     ▼
LiteLLM Proxy (Podman, 127.0.0.1:4000)
 ├── model_list          → Ollama (host, qwen3:8b)      via host.docker.internal:11434
 └── mcp_servers.itsops  → mcp-itsops (host, :8000)     via host.docker.internal:8000
                           (Streamable HTTP, stateless, MCP spec 2026-07-28)
PostgreSQL (Podman) ── stores virtual keys, teams, object permissions, spend
```

Two teams, both allowed to use the model `qwen3-local`:

| Team | Purpose | MCP access (intended, not verified) |
| --- | --- | --- |
| `itsops` | ITS Operations tooling | `itsops` MCP server: `search_runbooks`, `host_health`, `k8s_pod_status` |
| `servicedesk` | Service Desk chat | None |

## Repository contents

```text
compose.yaml            # Podman/Docker Compose: litellm + postgres
config.yaml             # model_list, mcp_servers, general_settings
.env.example            # placeholders only; copy to .env and fill in
.gitignore              # excludes .env, secrets/ and database volumes
```

The MCP server lives in the sibling repository [`ai-agents-mcp-itsops`](https://github.com/albeus/ai-agents-mcp-itsops). The tutorial below adds an HTTP mode to it; see [Step 4](#4-expose-the-mcp-server-over-streamable-http).

## Tutorial (what was done, in order)

### 1. Prerequisites

- Podman with Compose (Docker Compose also works).
- Ollama with `qwen3:8b` pulled.
- The `mcp-itsops` project (`uv` and the MCP Python SDK) with its three read-only-style tools.
- `curl` and `jq`.

### 2. Make Ollama reachable from containers

By default Ollama binds to `127.0.0.1`, which containers cannot reach. Adding an override changes the address it listens on:

```bash
sudo systemctl edit ollama.service
```

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart ollama
```

This exposes Ollama on all interfaces. Restrict access with a host firewall, and revert the override when you finish.

### 3. Start LiteLLM and PostgreSQL

Create `.env` from `.env.example` and fill in your own random values. Never commit it:

```text
LITELLM_MASTER_KEY=sk-<random 32+ bytes hex>
POSTGRES_PASSWORD=<random>
LITELLM_SALT_KEY=sk-<random>
```

`config.yaml`:

```yaml
model_list:
  - model_name: qwen3-local
    litellm_params:
      model: ollama_chat/qwen3:8b
      api_base: http://host.docker.internal:11434

mcp_servers:
  itsops:
    url: "http://host.docker.internal:8000/mcp"
    transport: "http"
    description: "Read-only ITS Operations lab tools"
    allow_all_keys: false

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  database_url: os.environ/DATABASE_URL
  store_model_in_db: true
  supported_db_objects: ["mcp"]
```

`compose.yaml` runs a `db` service (`postgres:16-alpine`) and a `litellm` service. The LiteLLM container has `extra_hosts: ["host.docker.internal:host-gateway"]`, and port `4000` is published only on `127.0.0.1`. Pin the LiteLLM image to a specific release tag rather than a moving tag such as `main-stable`, so results are reproducible.

```bash
podman compose up -d
podman compose logs -f db litellm
```

### 4. Expose the MCP server over Streamable HTTP

In `mcp_itsops/server.py` the transport is set in the `run()` call:

```python
mcp = MCPServer("ITS Operations")

if __name__ == "__main__":
    mcp.run(
        transport="streamable-http",
        host="0.0.0.0",   # use 127.0.0.1 while testing from the same host;
        port=8000,        # 0.0.0.0 was needed so the LiteLLM container could reach it
        stateless_http=True,
        json_response=True,
    )
```

```bash
uv run python -m mcp_itsops.server
```

The MCP specification version used here, 2026-07-28, dropped `initialize`: every request carries the protocol version and capabilities in `params._meta`. A minimal `tools/list` request:

```bash
curl -sS http://127.0.0.1:8000/mcp \
  -X POST \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{"_meta":{
        "io.modelcontextprotocol/protocolVersion":"2026-07-28",
        "io.modelcontextprotocol/clientCapabilities":{}}}}'
```

This server has no authentication. Bind it to loopback unless a container has to reach it, and firewall it if you bind wider.

### 5. Create teams and keys

Write the responses to a git-ignored directory, because the key responses contain live bearer tokens:

```bash
mkdir -p secrets && chmod 700 secrets
set -a; source .env; set +a

curl -sS -X POST http://127.0.0.1:4000/team/new \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" -H 'Content-Type: application/json' \
  -d '{"team_alias":"itsops","models":["qwen3-local"]}' | tee secrets/itsops-team.json

curl -sS -X POST http://127.0.0.1:4000/team/new \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" -H 'Content-Type: application/json' \
  -d '{"team_alias":"servicedesk","models":["qwen3-local"]}' | tee secrets/servicedesk-team.json

curl -sS -X POST http://127.0.0.1:4000/key/generate \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" -H 'Content-Type: application/json' \
  -d '{"key_alias":"itsops-lab-service","team_id":"<ITSOPS_TEAM_ID>"}' | tee secrets/itsops-service-key.json

curl -sS -X POST http://127.0.0.1:4000/key/generate \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" -H 'Content-Type: application/json' \
  -d '{"key_alias":"servicedesk-lab-service","team_id":"<SERVICEDESK_TEAM_ID>"}' | tee secrets/servicedesk-service-key.json
```

Replace the team ID placeholders with the `team_id` values from the two team files. Extract each key into an environment variable, for example `ITSOPS_SERVICE_KEY`, rather than pasting it into commands.

### 6. Verify model routing for both teams

```bash
curl -sS http://127.0.0.1:4000/v1/chat/completions \
  -H "Authorization: Bearer $ITSOPS_SERVICE_KEY" -H 'Content-Type: application/json' \
  -d '{"model":"qwen3-local","messages":[{"role":"user","content":"Reply with exactly: ITS team policy working"}],"temperature":0}'
```

Repeat with the `servicedesk` key. Both keys reached `qwen3-local`.

### 7. Register the MCP server and test tool access

```bash
curl -sS http://127.0.0.1:4000/v1/mcp/server \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" | jq

curl -sS http://127.0.0.1:4000/mcp-rest/tools/list \
  -H "x-litellm-api-key: Bearer $LITELLM_MASTER_KEY" \
  -H "x-mcp-servers: itsops" | jq
```

The master key listed all three tools. A team-scoped key could not, and neither could any other non-master key. That is the open problem described below.

## Results

| Area | Observed | Not established |
| --- | --- | --- |
| Model routing | One OpenAI-compatible endpoint in front of a local Ollama model | Behaviour with other providers or later versions |
| Per-team model access | Both team keys reached `qwen3-local` | A denied case (a key blocked from a model) was not recorded here |
| Persistence | PostgreSQL holds keys, teams and spend | Backup, recovery or scale |
| MCP through the gateway | `mcp-itsops` over Streamable HTTP was registered and its tools listed through the gateway using the **master key** | Any non-master key listing or calling the tools |
| Per-team MCP authorisation | **Not established.** Grants were not persisted as expected and non-master keys received 403 | The intended `itsops` versus `servicedesk` tool split is **not** enforced by this lab |

## Known issue: MCP grants not persisted

**Symptom.** Creating a key with an inline `object_permission.mcp_servers`, and updating a team with `object_permission.mcp_access_groups` or `.mcp_servers`, both returned an apparent success. `team/info` even echoed the permission back for the team object. But the database told a different story:

```sql
SELECT * FROM "LiteLLM_ObjectPermissionTable" WHERE object_permission_id = '<id>';
-- mcp_servers, mcp_access_groups, vector_stores and agents were all empty
```

The permission row existed and was linked to the key, but the grant fields inside it were empty, on both the key-level and team-level paths tried. As a result every non-master-key request to `/mcp-rest/tools/list` returned `403 access_denied`, for both teams. That is a fail-closed outcome, so it was safe, but it means the intended access policy could not be demonstrated.

**Ruled out.**

- A network problem: the `x-mcp-debug-*` response headers confirmed the correct upstream URL was reached.
- A header-naming problem: correcting the headers (`x-litellm-api-key`, `x-mcp-servers`) changed nothing once the permission row was seen to be empty.
- An MCP-server-side problem: the master key listed all three tools through the same route.
- A display-only quirk: a known LiteLLM behaviour shows `object_permission: null` in key responses even on success, but here the database itself showed empty grant columns.

**Not yet tried.**

1. Check the installed LiteLLM version against the project's issue tracker for `object_permission` persistence problems.
2. Set the permission through the Admin UI (`http://127.0.0.1:4000/ui`), in case the fault is specific to the REST path.
3. Attach the team to the MCP server through the server's own `teams` field instead of granting the server to the team.
4. Pin a specific release tag and retest.

Treat the result as a finding about this build and this configuration, not as a general verdict on LiteLLM.

## Troubleshooting log

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Cannot connect to host host.docker.internal:11434` | Ollama bound to `127.0.0.1` only | Set `OLLAMA_HOST=0.0.0.0:11434` with a systemd override and restart |
| `DB not connected` on `/key/generate` | Virtual keys and teams need PostgreSQL | Add the `db` service, set `DATABASE_URL` and `store_model_in_db: true` |
| `Method Not Allowed` when calling `/team/update` with `PATCH` | The route accepts `POST` | Use `POST /team/update` |
| `Failed to spawn: mcp` | `uv run mcp run <file>` needs the `mcp` CLI, and the filename was wrong | Use `uv run python -m mcp_itsops.server` with the transport set in code |
| `TypeError` on `stateless_http` in the `MCPServer(...)` constructor | In this SDK version the option belongs to `run()` | Move `stateless_http` and `json_response` into `mcp.run(...)` |
| `400 Bad Request: Missing session ID` on a plain `GET /mcp` | A bare GET is not a valid MCP operation | Use stateless mode and proper POST requests |
| `params._meta must be an object ...` | Spec 2026-07-28 retired `initialize`; each request needs `_meta` | Send the full `_meta` envelope with version and capabilities |
| `406 Not Acceptable` | The endpoint needs both content types accepted | Send `Accept: application/json, text/event-stream` as one header |
| `403 access_denied: key not allowed to access any MCP servers` | The grants were not persisted (see Known issue) | Diagnosed by querying PostgreSQL directly |

## What this lab did show

- LiteLLM as a single OpenAI-compatible gateway in front of a local Ollama model.
- PostgreSQL-backed storage for virtual keys, teams and spend.
- Per-team **model** access, verified with two team identities.
- A custom MCP server moved from stdio to stateless Streamable HTTP under the 2026-07-28 spec, registered as a LiteLLM upstream and queried through the gateway with the master key.

What it did **not** show is per-team MCP tool authorisation. A useful takeaway is the method: verify stored policy in the database, not only in the API response.

## Related

- [`ai-agents-mcp-itsops`](https://github.com/albeus/ai-agents-mcp-itsops): the MCP server behind this gateway.
- [`ai-agents-trace-probe`](https://github.com/albeus/ai-agents-trace-probe): a probe that calls the model through this proxy.
- [Lab index](https://github.com/albeus/ai-agents-lab): reading order and the other projects.

## Licence

Released under the [MIT Licence](LICENSE). This is a learning lab, provided as is, without warranty.
