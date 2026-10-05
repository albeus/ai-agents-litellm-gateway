# ai-agents-litellm-gateway

A local lab that puts a LiteLLM Proxy in front of a local Ollama model and a hand-written MCP server, to explore model routing, separate team identities and MCP access control.

> Part of the [AI Agents Lab](https://github.com/albeus/ai-agents-lab). This is a learning experiment, not a production reference architecture. Team-scoped MCP discovery and allow/deny execution were demonstrated in follow-up tests on 4–5 October 2026. UI and REST team grants both worked in the tested installation. The original failed permission experiment remains unexplained, not a confirmed LiteLLM bug.

## Safety notice

- Use non-sensitive lab data and lab-only Kubernetes credentials. The MCP process has the permissions of its local user and configured kubeconfigs.
- Never commit `.env`, generated keys, key/team JSON responses, real credentials, database volumes or confidential tool results. Save live responses only in a git-ignored directory.
- The LiteLLM port is published on `127.0.0.1`. Making Ollama or the MCP server reachable from a container may require a wider bind. `0.0.0.0` is not loopback-only: restrict access with the host firewall.
- The upstream MCP server has no authentication. Direct access bypasses gateway policy. Do not expose it to an untrusted network.
- Results apply to the tested installation. Record and pin versions before comparing runs or claiming reproducibility.

## Goal and architecture

Explore the difference between an available tool server and a verified access policy:

- Route local model requests through one OpenAI-compatible gateway endpoint.
- Give ITS Operations and Service Desk separate team identities and service keys.
- Register an MCP server as a gateway upstream.
- Configure team grants and test both allowed and denied tool execution.
- Check permission read-back and effective behaviour, rather than trusting update responses alone.

Non-goals: a production deployment, exhaustive security testing, fine-tuning, RAG design or complete distributed tracing.

```text
Client / curl
     │
     ▼
LiteLLM Proxy (Podman, 127.0.0.1:4000)
 ├── model_list         → Ollama (host, qwen3:8b)
 │                        host.docker.internal:11434
 └── mcp_servers.itsops → mcp-itsops (host, Streamable HTTP)
                          host.docker.internal:8000/mcp
PostgreSQL ── stores virtual keys, teams, permissions and spend
```

The original notes recorded an MCP protocol revision of `2026-07-28`. Treat that as part of the recorded environment, not a universal request format for every SDK release.

| Team | Model | Baseline MCP policy |
| --- | --- | --- |
| `itsops` | `qwen3-local` | Access to the `itsops` server and its four lab tools |
| `servicedesk` | `qwen3-local` | No MCP access initially; access added temporarily for the REST experiment |

The current tools are `search_runbooks`, `host_health`, `k8s_pod_status` and the lab-only `slow_probe`. The original master-key discovery experiment concerned three operations tools. The follow-up execution tests used `slow_probe(seconds=0)` without a model or cluster dependency.

## Repository contents

```text
compose.yaml     # LiteLLM and PostgreSQL services
config.yaml      # model_list, mcp_servers and general_settings
.env.example     # placeholders; copy to .env and fill in
.gitignore       # excludes .env, secrets/ and database volumes
```

The server lives in the sibling [ai-agents-mcp-itsops repository](https://github.com/albeus/ai-agents-mcp-itsops). It defaults to stdio; this gateway lab explicitly selects HTTP through environment variables. Run gateway commands from this repository root and server commands from the MCP repository root.

## Setup

### 1. Prerequisites

- Podman with Compose, or Docker Compose.
- Ollama with `qwen3:8b` pulled.
- The MCP server project with dependencies installed using `uv sync` and its transport environment-variable switch.
- `curl` and `jq`.
- Node.js for the optional Inspector check.

### 2. Make Ollama reachable from the container

The original lab could not reach Ollama's loopback listener from its container. A systemd override widened the listener:

```bash
sudo systemctl edit ollama.service
```

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

This exposes Ollama on all IPv4 interfaces. Restrict network access and revert the override when no longer needed.

### 3. Configure LiteLLM and PostgreSQL

Copy `.env.example` to `.env` and set your own random secrets. Never commit the real values.

```text
LITELLM_MASTER_KEY=sk-<random 32+ bytes hex>
POSTGRES_PASSWORD=<random>
LITELLM_SALT_KEY=sk-<random>
```

The recorded `config.yaml` structure is:

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

The Compose setup uses PostgreSQL and LiteLLM, maps `host.docker.internal` to the host gateway, and publishes port 4000 only on loopback. Pin the LiteLLM image rather than relying on a moving tag. Verify host mapping and network reachability in your container environment.

```bash
podman compose up -d
podman compose logs -f db litellm
```

### 4. Start the MCP HTTP server

In a separate terminal, from the MCP server repository root:

```bash
MCP_TRANSPORT=streamable-http \
MCP_HOST=0.0.0.0 \
MCP_PORT=8000 \
uv run python -m mcp_itsops.server
```

Keep the process running. HTTP mode uses `stateless_http=True` and `json_response=True`. The wider bind supports the documented container setup but must be firewalled. The SDK transport value is `streamable-http`; LiteLLM's corresponding config value is `http`.

For same-host testing without the gateway container, use `MCP_HOST=127.0.0.1` instead. Running the server without `MCP_TRANSPORT=streamable-http` starts stdio, not an HTTP listener.

Optional Inspector check: start Inspector separately with `npx @modelcontextprotocol/inspector`, select Streamable HTTP and enter `http://127.0.0.1:8000/mcp`. Use its proxy connection mode if offered. This is a direct server test, not a gateway authorisation test.

For the separate stdio exercise, let Inspector launch the server:

```bash
MCP_TRANSPORT=stdio \
npx @modelcontextprotocol/inspector \
  uv run python -m mcp_itsops.server
```

### 5. Create teams and keys

Skip creation if reusing existing teams and keys; retrieve their existing IDs instead of creating duplicates.

```bash
mkdir -p secrets
chmod 700 secrets
set -a; source .env; set +a

curl -sS -X POST http://127.0.0.1:4000/team/new \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"team_alias":"itsops","models":["qwen3-local"]}' \
  > secrets/itsops-team.json

curl -sS -X POST http://127.0.0.1:4000/team/new \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"team_alias":"servicedesk","models":["qwen3-local"]}' \
  > secrets/servicedesk-team.json

export ITSOPS_TEAM_ID="$(jq -er '.team_id' secrets/itsops-team.json)"
export SERVICEDESK_TEAM_ID="$(jq -er '.team_id' secrets/servicedesk-team.json)"
```

Create a service key for each team:

```bash
jq -n --arg team_id "$ITSOPS_TEAM_ID" \
  '{team_id: $team_id, key_alias: "itsops-lab-service"}' |
curl -sS -X POST http://127.0.0.1:4000/key/generate \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @- > secrets/itsops-service-key.json

jq -n --arg team_id "$SERVICEDESK_TEAM_ID" \
  '{team_id: $team_id, key_alias: "servicedesk-lab-service"}' |
curl -sS -X POST http://127.0.0.1:4000/key/generate \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @- > secrets/servicedesk-service-key.json

export ITSOPS_SERVICE_KEY="$(jq -er '.key' secrets/itsops-service-key.json)"
export SERVICEDESK_SERVICE_KEY="$(jq -er '.key' secrets/servicedesk-service-key.json)"
```

Check responses for errors before continuing. These files contain live credentials; do not publish them.

### 6. Check model routing

```bash
curl -sS -i http://127.0.0.1:4000/v1/chat/completions \
  -H "Authorization: Bearer $ITSOPS_SERVICE_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen3-local","messages":[{"role":"user","content":"Reply with exactly: model routing works"}],"temperature":0}'
```

Repeat with `SERVICEDESK_SERVICE_KEY`. Both team keys reached the model in the recorded experiment. Successful model routing does not prove MCP access control.

### 7. Check server registration

```bash
curl -sS http://127.0.0.1:4000/v1/mcp/server \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" | jq
```

The server is registered through `mcp_servers` in the gateway configuration. The command above reads the registration; it does not create a team grant.

Set the actual server ID returned for `itsops`, not its name or an ID copied from another installation:

```bash
export MCP_SERVER_ID='your-itsops-mcp-server-id'
```

The original experiment successfully listed tools with the master key using:

```bash
curl -sS http://127.0.0.1:4000/mcp-rest/tools/list \
  -H "x-litellm-api-key: Bearer $LITELLM_MASTER_KEY" \
  -H "x-mcp-servers: itsops" | jq
```

A registration row alone does not prove upstream connectivity. Check server logs and container-to-host reachability if discovery is empty.

## Team grants and validation

### Configure ITS Operations access

On 4 October 2026, the Admin UI was used to grant ITS Operations direct access to the server and all four lab tools. Read-back showed the team grant. Its service key listed tools and called `slow_probe` successfully, while Service Desk was denied.

The following request expresses that grant through REST. The same payload structure was successfully tested with Service Desk on 5 October 2026.

This replaces the supplied server list and tool-permission map. Preserve other grants if adapting it outside these dedicated lab teams.

```bash
jq -n \
  --arg team_id "$ITSOPS_TEAM_ID" \
  --arg server_id "$MCP_SERVER_ID" \
  '{
    team_id: $team_id,
    object_permission: {
      mcp_servers: [$server_id],
      mcp_tool_permissions: {
        ($server_id): [
          "search_runbooks",
          "host_health",
          "k8s_pod_status",
          "slow_probe"
        ]
      }
    }
  }' |
curl -sS -i -X POST http://127.0.0.1:4000/team/update \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @-
```

Read back independently:

```bash
curl -sS --get http://127.0.0.1:4000/team/info \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  --data-urlencode "team_id=$ITSOPS_TEAM_ID" \
  | jq '.team_info | {
      team_alias,
      team_id,
      object_permission_id,
      object_permission
    }'
```

The successful UI configuration used a direct server-ID grant, not an MCP access-group grant. The grant was attached to the team; its service key had no separate `object_permission`. A null key-level permission object alone does not establish that the effective team grant failed.

### Check the allowed and denied baseline

Before granting Service Desk access, check discovery:

```bash
curl -sS -i http://127.0.0.1:4000/mcp-rest/tools/list \
  -H "Authorization: Bearer $ITSOPS_SERVICE_KEY"

curl -sS -i http://127.0.0.1:4000/mcp-rest/tools/list \
  -H "Authorization: Bearer $SERVICEDESK_SERVICE_KEY"
```

On 4 October, ITS Operations listed tools and Service Desk's response body reported `access_denied`. The HTTP status of that discovery response was not captured in the supplied output.

Test execution separately:

```bash
jq -n --arg server_id "$MCP_SERVER_ID" \
  '{server_id: $server_id, name: "slow_probe", arguments: {seconds: 0}}' |
curl -sS -i http://127.0.0.1:4000/mcp-rest/tools/call \
  -H "Authorization: Bearer $ITSOPS_SERVICE_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @-
```

Repeat with `SERVICEDESK_SERVICE_KEY`. Observed on 4 October 2026:

| Caller | HTTP status | Tool-call result |
| --- | --- | --- |
| ITS Operations | 200 OK | `isError: false`; `Completed after 0 seconds.` |
| Service Desk, without a grant | 403 Forbidden | `access_denied`, naming the requested server ID |

This checks execution authorisation, not merely tool-list visibility. It does not establish selective filtering within a granted server because the grant includes all four tools.

### Repeat the team grant through REST

This temporarily grants Service Desk access, changing the denied baseline. Record the baseline first.

```bash
jq -n \
  --arg team_id "$SERVICEDESK_TEAM_ID" \
  --arg server_id "$MCP_SERVER_ID" \
  '{
    team_id: $team_id,
    object_permission: {
      mcp_servers: [$server_id],
      mcp_tool_permissions: {
        ($server_id): [
          "search_runbooks",
          "host_health",
          "k8s_pod_status",
          "slow_probe"
        ]
      }
    }
  }' |
curl -sS -i -X POST http://127.0.0.1:4000/team/update \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @-
```

Verify the team grant:

```bash
curl -sS --get http://127.0.0.1:4000/team/info \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  --data-urlencode "team_id=$SERVICEDESK_TEAM_ID" \
  | jq '.team_info | {
      team_alias,
      team_id,
      object_permission_id,
      object_permission
    }'
```

`GET /team/info` only reads configuration. `POST /team/update` is the operation that adds the permission.

Retry execution with the same Service Desk key:

```bash
jq -n --arg server_id "$MCP_SERVER_ID" \
  '{server_id: $server_id, name: "slow_probe", arguments: {seconds: 0}}' |
curl -sS -i http://127.0.0.1:4000/mcp-rest/tools/call \
  -H "Authorization: Bearer $SERVICEDESK_SERVICE_KEY" \
  -H 'Content-Type: application/json' \
  --data-binary @-
```

On 5 October 2026, the team-grant workflow was reported successful: team-info showed a server grant and four-tool permission map, and this call returned HTTP 200 with `isError: false` and `Completed after 0 seconds.` The original failing REST setup was not reproduced.

Restore Service Desk's original denied permissions after this experiment and repeat the denied call before a baseline demo. Restoration has not yet been recorded. Separately repeat grant read-back and execution after a proxy restart before claiming restart persistence.

## Results

| Area | Observed | Not established |
| --- | --- | --- |
| Model routing | Both team identities reached the configured local model through LiteLLM | A model-denial case or other providers/versions |
| MCP discovery | Original master-key discovery; ITS Operations service-key discovery on 4 October 2026 | Discovery behaviour in every configuration |
| Team execution authorisation | UI-granted ITS Operations received HTTP 200; ungranted Service Desk received HTTP 403 for the same tool on 4 October | All tools, alternative routes or broad security isolation |
| REST team grant | Service Desk grant read-back and successful `slow_probe` execution on 5 October | Other grant paths, including key-level and access-group grants |
| Permission read-back | Successful team grants appeared in `/team/info` | Independent database verification of the new grants and persistence across restart |
| Per-tool filtering and revocation | All four tools were included in the grant | Denial of an omitted tool within an allowed server; denial after removing the grant |

### Follow-up chronology

- **4 October 2026 — UI grant:** ITS Operations received a direct server grant and four-tool allowlist. Its service key listed tools and successfully executed `slow_probe(seconds=0)`. Service Desk was denied the same execution request with HTTP 403.
- **5 October 2026 — REST grant:** The team-update workflow was repeated for Service Desk. Read-back showed the server grant and tool permissions. The same Service Desk key then executed `slow_probe(seconds=0)` with HTTP 200.

These results support team-scoped access control for the tested discovery and execution paths in this installation. They do not establish a general security guarantee.

## Earlier unsuccessful permission experiment

The first experiment recorded apparent success from REST permission updates, empty grant fields in the database checks performed at the time, and denied requests from the tested non-master keys. The master key could list tools through the same gateway route.

The earlier README called this a permission-persistence issue. That interpretation was too strong: the original cause was never established, and the failure has not been reproduced in the successful follow-up. Both the UI team-grant path and the tested REST team-grant path now work.

Possible differences include the original payload, identifiers, permission relationship inspected, configuration or software version. None has been established as the cause. The evidence proves neither a LiteLLM bug nor a user mistake.

The original SQL check was recorded as:

```sql
SELECT * FROM "LiteLLM_ObjectPermissionTable"
WHERE object_permission_id = '<permission-id-under-investigation>';
```

If investigating again, identify whether the effective grant belongs to the team or key before selecting the permission ID. Inspecting a key with no explicit permission is insufficient to determine whether the team's grant exists or applies.

The complete original request payloads and a version comparison are not retained here, so an exact historical diagnosis is not possible from this record alone. Do not present this as a confirmed upstream persistence bug.

## Troubleshooting

| Symptom | Check or response |
| --- | --- |
| Model routing works but MCP discovery is empty | Check the HTTP transport, bind address, container-to-host connectivity and server logs; registration alone does not prove reachability. |
| LiteLLM cannot reach the MCP server | Ensure `MCP_TRANSPORT=streamable-http`, a container-reachable bind, correct host mapping and appropriate firewall rules. |
| Inspector's stdio connection hangs | Use `MCP_TRANSPORT=stdio`, or connect the HTTP UI to an independently started HTTP server. |
| `DB not connected` on key creation | Check PostgreSQL, `DATABASE_URL` and the configured storage settings. |
| `Method Not Allowed` on `PATCH /team/update` | The tested grant workflow uses `POST /team/update`. |
| `TypeError` setting `stateless_http` in the constructor | In the recorded MCP SDK, the option belongs to the HTTP `run()` call. |
| Protocol-envelope or content-negotiation errors | Check client/server versions and the request format expected by the installed SDK. Do not assume the original protocol revision applies universally. |
| `access_denied` for an ungranted team | Expected baseline behaviour; inspect the actual team/key grant if access was intended. |
| Update succeeds but a call is denied | Read back the correct team, verify server IDs and key ownership, and inspect effective grants; do not immediately infer a persistence bug. |

## Next checks

- Record the running LiteLLM image/version and MCP SDK version with the experiment results.
- Repeat grant read-back and execution after restart.
- Remove a grant and verify denial again.
- Permit only a subset of tools and test both allowed and excluded tools within the same server.
- Test the other operations tools only against configured, non-sensitive lab infrastructure.
- If a future REST request fails, compare it with the UI's actual save request, redacting credentials.

The useful operational lesson is to verify configuration read-back and actual allowed/denied calls, rather than treating a successful update response or model-generated text as evidence of policy enforcement.

## Related

- [MCP server lab](https://github.com/albeus/ai-agents-mcp-itsops): tools and stdio/HTTP workflows.
- [Trace-probe lab](https://github.com/albeus/ai-agents-trace-probe): model calls through LiteLLM and direct MCP calls. This does not establish end-to-end tracing through the MCP gateway.
- [Lab index](https://github.com/albeus/ai-agents-lab): reading order and other projects.

## Licence

Released under the MIT Licence; see `LICENSE`. This is a learning lab, provided as is, without warranty.
