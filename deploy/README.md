# Local Integration Environment

This directory is the S-05 local/demo integration baseline. It starts shared
infrastructure plus the backend service loop through gateway. It is not a
production deployment baseline.

Production or staging uses a separate baseline:
[`production-baseline.md`](./production-baseline.md),
`docker-compose.production.yml`, `.env.production.example`, and
`nginx/production.conf`. Do not promote the local `.env.example`, local seed
data, or local demo credentials into shared or long-lived environments. The
production baseline exposes only the `ingress` service publicly; it routes
browser `/api/v1/**` calls to gateway and keeps frontend/gateway service ports
inside the Compose network.

## Entry Points

- Browser/frontend entrypoint: `http://localhost:8080` through gateway only.
- Do not point frontend code at `auth`, `file`, `knowledge`, `qa`,
  `document`, `ai-gateway`, PostgreSQL, Redis, Qdrant, or MinIO directly.
- Internal service ports are exposed for local debugging only.

## Start

```powershell
cd deploy
Copy-Item .env.example .env
docker compose up -d --build
```

Mainland China recommended overlay:

```powershell
cd deploy
Copy-Item .env.example .env
Get-Content .env.china.example | Add-Content .env
$env:DOCKER_BUILDKIT = "1"
docker compose up -d --build
```

For Bash:

```bash
cd deploy
cp .env.example .env
cat .env.china.example >> .env
DOCKER_BUILDKIT=1 docker compose up -d --build
```

This overlay uses explicit registry rewrites and package mirrors. It is the
preferred path for users with no Docker mirror/proxy configured, and it avoids
depending on daemon-level mirror behavior.

AI/模型功能必需：AI Gateway profile

The default stack starts the core services. Enable the `ai` profile before using
admin model profiles, QA real model calls, Document AI generation, real
embedding/rerank paths, or AI Gateway provider smoke.

```powershell
cd deploy
docker compose --profile ai up -d --build
```

To start only the AI Gateway profile service on top of an already running core
stack:

```powershell
docker compose --env-file .env --profile ai up -d --build ai-gateway
Invoke-RestMethod http://localhost:8086/readyz
```

`gateway /readyz` does not prove AI Gateway or provider readiness. Seeded local
AI profiles are placeholders until replaced with real provider credentials.

Default seeded login:

```text
username: admin
password: LocalDemoAdmin#12345
```

These credentials and all secrets in `.env.example` are local placeholders.
Replace them for any shared or long-lived environment.

## Docker Images Required

If Docker has no local images, install them with:

```powershell
docker pull postgres:16-alpine
docker pull redis:7-alpine
docker pull minio/minio:RELEASE.2025-09-07T16-13-09Z
docker pull minio/mc:RELEASE.2025-08-13T08-35-41Z
docker pull golang:1.25-alpine
docker pull alpine:3.22
```

Then build service images:

```powershell
cd deploy
docker compose build
docker compose --profile ai build
```

The local MinIO server, MinIO `mc`, Redis, PostgreSQL, and Alpine runtime
images are pinned to explicit tags in this repository.

## Ports

| Component | Host port | Container port | Purpose |
| --- | ---: | ---: | --- |
| gateway | 8080 | 8080 | Browser/backend entrypoint |
| auth | 8001 | 8001 | Internal auth service |
| file | 8082 | 8082 | Internal file service |
| knowledge | 8083 | 8083 | Vendor contract adapter |
| qa | 8084 | 8084 | Internal QA service |
| document | 8085 | 8085 | Internal document service |
| ai-gateway | 8086 | 8086 | Model/profile service for AI features |
| postgres | 5432 | 5432 | Local relational databases |
| redis | 6379 | 6379 | Sessions, queues, coordination |
| minio | 9000/9001 | 9000/9001 | Object storage and console |

Override host ports in `deploy/.env`.

## Environment Variables

| Variable | Service | Required | Description |
| --- | --- | --- | --- |
| `INTERNAL_SERVICE_TOKEN` | gateway/auth/file/parser/knowledge/qa/document/ai-gateway | yes | Local service-to-service token placeholder. |
| `TOKEN_HASH_SECRET` | gateway/auth | yes | Local HMAC secret for opaque token hashes. |
| `GATEWAY_AUTH_BASE_URL` | gateway | set in Compose | Internal auth base URL. |
| `GATEWAY_KNOWLEDGE_BASE_URL` | gateway | set in Compose | Internal knowledge base URL. |
| `GATEWAY_QA_BASE_URL` | gateway | set in Compose | Internal QA base URL. |
| `GATEWAY_DOCUMENT_BASE_URL` | gateway | set in Compose | Internal document base URL. |
| `GATEWAY_AI_GATEWAY_BASE_URL` | gateway | set in Compose | Internal AI Gateway base URL; AI/model routes require the `ai` profile to run. |
| `AUTH_DATABASE_URL` | auth | yes | Auth PostgreSQL DSN. |
| `FILE_DATABASE_URL` | file | yes | File metadata PostgreSQL DSN. |
| `FILE_STORAGE_BACKEND` | file | no | `local` in Compose for durable local smoke tests. |
| `DATABASE_URL` | knowledge | no | Optional PostgreSQL for parser-config admin routes. |
| `VENDOR_RUNTIME_URL` | knowledge | yes | RAGFlow vendor HTTP base URL (default in Compose: `http://knowledge-runtime-api:9380`). |
| `KNOWLEDGE_AUTO_START_INGESTION` | knowledge | no | Auto-call vendor `/documents/parse` after upload (default `true`). |
| `QA_DATABASE_URL` | qa | yes | QA PostgreSQL DSN. |
| `KNOWLEDGE_SERVICE_URL` | qa | yes | Internal Knowledge Service URL. |
| `AI_GATEWAY_URL` | qa | yes | Internal chat completions URL; QA real model calls require `--profile ai`. |
| `AI_GATEWAY_PROFILE_ID` / `MODEL_ID` | qa | no | Optional default QA AI Gateway chat profile/model. QA settings versions can override these; model must exactly match the selected AI Gateway profile. |
| `QA_SETTINGS_OPEN` / `QA_ADMIN_USER_IDS` | qa | no | Local QA settings-write allowance. Keep closed by default; enable only for trusted local smoke or configure explicit admin user ids. |
| `MCP_TRANSPORT` / `MCP_SERVER_ALIAS` / `MCP_SERVER_URL` | qa | no | Local Compose defaults to `streamable_http` / `document` / `http://document:8085/mcp` so QA can discover Document report tools. |
| `MCP_SERVER_TOKEN` / `MCP_SERVER_TOKEN_HEADER` | qa | no | Document MCP credential. Defaults to the same `INTERNAL_SERVICE_TOKEN` placeholder and `Authorization` header used by Document MCP. |
| `MCP_TOOL_TIMEOUT` | qa | no | Per-tool timeout for remote MCP calls; defaults to `30s`. |
| `DOCUMENT_DATABASE_URL` | document | yes | Document PostgreSQL DSN. |
| `DOCUMENT_REDIS_ADDR` | document | yes | Redis/asynq endpoint. |
| `DOCUMENT_FILE_SERVICE_URL` | document | yes | Internal File Service URL. |
| `DOCUMENT_FILE_SERVICE_TOKEN` | document | yes | Local service token for File Service calls without gateway request context. |
| `DOCUMENT_AI_GATEWAY_URL` | document | yes | Internal AI Gateway base URL. |
| `DOCUMENT_AI_GATEWAY_PROFILE_ID` | document | yes | Seeded placeholder profile id, `default-chat`. |
| `DOCUMENT_AI_GATEWAY_SERVICE_TOKEN` | document | yes | Local service token for AI Gateway internal profile APIs. |
| `DOCUMENT_MCP_SERVICE_TOKEN` / `DOCUMENT_MCP_TOKEN_HEADER` | document | yes | Streamable HTTP MCP credential; defaults to `INTERNAL_SERVICE_TOKEN` and `Authorization` in local Compose. |
| `AI_GATEWAY_DATABASE_URL` | ai-gateway | yes | AI Gateway PostgreSQL DSN. |
| `AI_GATEWAY_SERVICE_TOKEN_HASHES` | ai-gateway | yes | SHA-256 hashes for allowed service tokens. |
| `AI_GATEWAY_CREDENTIAL_ENCRYPTION_KEY_REF` | ai-gateway | yes | Local encryption key reference placeholder. |
| `AI_GATEWAY_CREDENTIAL_ENCRYPTION_KEY` | ai-gateway | yes | Local encryption key placeholder. |

## Health And Readiness

Use gateway for the top-level signal:

```powershell
Invoke-RestMethod http://localhost:8080/healthz
Invoke-RestMethod http://localhost:8080/readyz
```

Service-level readiness endpoints:

```powershell
Invoke-RestMethod http://localhost:8001/readyz
Invoke-RestMethod http://localhost:8082/readyz
Invoke-RestMethod http://localhost:8083/readyz
Invoke-RestMethod http://localhost:8084/readyz
Invoke-RestMethod http://localhost:8085/readyz
Invoke-RestMethod http://localhost:8086/readyz
Invoke-RestMethod http://localhost:8087/readyz
```

`gateway /readyz` checks Redis and auth, and verifies owner service URLs are
configured. It does not call Knowledge, QA, Document, or AI Gateway readiness
endpoints and does not prove business workflows such as upload, retrieval, QA
answers, report generation, model profile bootstrap, or real provider calls.
Auth, document, and ai-gateway readiness identify PostgreSQL problems. Compose
health checks identify container-level dependency failures.

Use the targeted smoke checks in
[`docs/runbooks/local-integration.md`](../docs/runbooks/local-integration.md)
for complete cross-service availability. #352 owns the repeatable
Auth/Gateway/Redis smoke, and #125 owns the broader Gateway -> owner service
and MCP/cross-service smoke coverage.

## Seed Data

`seed-local` applies `deploy/seeds/001-local-demo-seed.sql` after migrations:

CI-safe checks validate static seed contracts without starting containers:

```powershell
python scripts/verify_local_seed_contract.py
docker compose --env-file deploy/.env.example -f deploy/docker-compose.yml config --quiet
docker compose --env-file deploy/.env.example -f deploy/docker-compose.yml --profile ai config --quiet
```

The local/manual seed path is the Compose run itself:

```powershell
cd deploy
docker compose --env-file .env.example up -d --build gateway
docker compose --env-file .env.example --profile ai up -d --build ai-gateway
```

Seeded local resources:

| Area | Deterministic resource |
| --- | --- |
| Auth | user `usr_local_admin`, username `admin`, password `LocalDemoAdmin#12345`, role `admin` |
| Auth permissions | `admin:model-profile:write`, `admin:parser-config:write`, `qa:settings:read`, and `qa:settings:write`; `system:admin` is not required for this local admin |
| Knowledge | knowledge base `kb_local_demo`, document `doc_local_demo_seed`, chunk `chunk_local_demo_seed_001` |
| Document | material `22222222-2222-4222-8222-222222222201`, report `22222222-2222-4222-8222-222222222301`, outline `22222222-2222-4222-8222-222222222401` |
| QA | conversation `33333333-3333-4333-8333-333333333301`, user message `33333333-3333-4333-8333-333333333401`, assistant message `33333333-3333-4333-8333-333333333402` |
| AI Gateway | optional placeholder profiles `default-chat`, `default-embedding`, and `default-rerank` |

The local admin password hash in `001-local-demo-seed.sql` is an `argon2id`
PHC string for `LOCAL_ADMIN_PASSWORD=LocalDemoAdmin#12345` using the documented
`argon2id-v1` parameters: `m=65536`, `t=3`, `p=2`, 16-byte salt, and 32-byte
key. For rotation, generate a new local-only `argon2id` hash, update
`deploy/.env.example`, `001-local-demo-seed.sql`, and this README together,
then rerun `seed-local`. Never reuse the demo password or hash in a shared or
long-lived environment.

After the stack is up, verify the seeded admin through Gateway:

```powershell
$body = @{ username = "admin"; password = "LocalDemoAdmin#12345" } | ConvertTo-Json
$session = Invoke-RestMethod -Method Post -Uri http://localhost:8080/api/v1/sessions -ContentType "application/json" -Body $body
$session.data.user.roles
$session.data.user.permissions
$token = $session.data.session.accessToken
$headers = @{ Authorization = "Bearer $token" }
Invoke-RestMethod -Uri http://localhost:8080/api/v1/admin/parser-configs -Headers $headers
```

The response should include role `admin` and admin runtime config permissions
such as `admin:model-profile:write`, `admin:parser-config:write`, or
`qa:settings:read`. The
`GET /api/v1/admin/parser-configs` call proves the seeded admin token passes a
Gateway admin route preflight; use `/api/v1/admin/model-profiles` when the
optional AI profile is running.

To remove only the deterministic local demo rows after migrations are present:

```powershell
cd deploy
docker compose --env-file .env.example run --rm seed-local sh -c "psql -v ON_ERROR_STOP=1 -h postgres -U postgres -d postgres -f /seeds/099-local-demo-cleanup.sql"
```

For a full reset, remove volumes and rerun the stack:

```powershell
cd deploy
docker compose --env-file .env.example --profile ai down -v
docker compose --env-file .env.example up -d --build gateway
```

The AI profiles are enabled local placeholders for readiness checks and include
fake encrypted provider credentials. They are not real API keys, so model
invocation still requires operators to configure a real provider key.
Their default provider URL is `http://host.docker.internal:11434/v1`; Compose
maps that hostname to the Docker host for Linux engines with
`host.docker.internal:host-gateway`.

Seed contract notes:

- The deterministic demo rows include `usr_local_admin`, `doc_local_demo_seed`, `22222222-2222-4222-8222-222222222301`, and `33333333-3333-4333-8333-333333333301`.
- The seeded admin password is hashed with `argon2id`, and rotation / refresh flows stay CI-safe for local/manual demo maintenance.
- Admin runtime config coverage includes `/api/v1/admin/parser-configs`, `admin:model-profile:write`, `admin:parser-config:write`, and `system:admin` or admin runtime config permissions.
- Use `cleanup` rows from `deploy/seeds/099-local-demo-cleanup.sql` for targeted teardown; use `docker compose down -v` for a full manual reset.

## Request Id Troubleshooting

Every service returns or propagates `X-Request-Id`.

```powershell
$rid = "req_local_debug_001"
Invoke-RestMethod http://localhost:8080/readyz -Headers @{ "X-Request-Id" = $rid }
docker compose logs gateway auth knowledge qa document | Select-String $rid
```

For frontend issues, capture the response `requestId` or `X-Request-Id`, then
search gateway logs first. If gateway reports a dependency error, search the
same id in the owner service logs.

## Knowledge Integration Notes

Knowledge active operations are exposed through gateway:

```powershell
# after logging in and setting $token to the returned access token
$headers = @{ Authorization = "Bearer $token"; "X-Request-Id" = "req_knowledge_local_001" }
Invoke-RestMethod "http://localhost:8080/api/v1/knowledge-bases" -Headers $headers
Invoke-RestMethod "http://localhost:8080/api/v1/knowledge-bases/kb_local_demo/documents" -Headers $headers
Invoke-RestMethod "http://localhost:8080/api/v1/documents/<documentId>/chunks" -Headers $headers
Invoke-WebRequest "http://localhost:8080/api/v1/documents/<documentId>/content" -Headers $headers -OutFile .\knowledge-content.bin
Invoke-RestMethod "http://localhost:8080/api/v1/knowledge-queries" -Method Post -Headers $headers -ContentType "application/json" -Body '{"query":"local demo","topK":3}'
```

Knowledge routes require the RAGFlow runtime at `VENDOR_RUNTIME_URL`
(default `http://knowledge-runtime-api:9380` inside Compose). With the
`knowledge-v2` profile you can run `knowledge-runtime-api` and
`knowledge-runtime-worker` in compose:

```powershell
cd deploy
docker compose --profile knowledge-v2 up -d elasticsearch knowledge-minio-init knowledge-runtime-api knowledge-runtime-worker knowledge
```

Use an explicit `VENDOR_RUNTIME_URL` override only when the runtime is started
outside Compose for local Python development.

See `services/knowledge/runtime/README.md` and `services/knowledge-runtime/README.md`.

## Common Dependency Failures

| Symptom | Likely cause | Check |
| --- | --- | --- |
| `gateway /readyz` returns `503 dependency_error` | Redis, auth, or required owner service base URL configuration is not ready | `docker compose ps`, `docker compose logs redis auth gateway` |
| `auth /readyz` returns `postgres unavailable` | Auth migration or PostgreSQL failed | `docker compose logs postgres migrate-auth auth` |
| Knowledge upload/query returns `502 dependency_error` | Vendor runtime unreachable or ES/MinIO not ready | `docker compose logs knowledge`, verify vendor :9380 and `knowledge-v2` profile |
| Document readyz returns dependency error | Document DB migration failed or DB is unreachable | `docker compose logs migrate-document document postgres` |
| QA message call fails on model invocation | AI Gateway profile is not running, fake local credential is still in use, or host provider is not listening on `host.docker.internal:11434` | `docker compose --profile ai ps`, `docker compose logs ai-gateway qa` |
| MinIO bucket missing | `minio-init` did not complete | `docker compose logs minio minio-init` |
| Host port conflict | Another local process uses a default port | Change the matching `*_PORT` in `deploy/.env` |

## Reset

```powershell
cd deploy
docker compose down -v
docker compose --profile ai down -v
```
