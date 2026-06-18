# Plan: Issue #1853 — SPIFFE/SPIRE Workload Identity for AgentGateway Config Bridge

## Context

PR #1852 fixed the config-bridge crash by introducing a shared bearer token (`AGENTGATEWAY_TARGETS_TOKEN`). That token lives in a Kubernetes Secret and is validated with `timingSafeEqual` on the BFF side. It works but is a long-lived static secret — unsuitable for production because it never rotates, is readable by anyone with secret access in the `caipe` namespace, and relies on a human remembering to rotate it.

This issue tracks replacing that token with **SPIFFE/SPIRE workload identity**: cryptographic, short-lived, automatically rotated identities issued to pods based on their Kubernetes service account / pod attributes.

Docker Compose development must keep working without SPIRE — so the token fallback must remain.

---

## Current Auth Code (what we'd be modifying)

**config-bridge (`deploy/agentgateway/config_bridge.py` lines 666–682, 744–747):**
- Reads `AGENTGATEWAY_TARGETS_TOKEN` from env at startup (raises `RuntimeError` if absent)
- Sends `Authorization: Bearer <token>` on every poll to `AGENTGATEWAY_TARGETS_URL`

**BFF (`ui/src/app/api/internal/agentgateway/mcp-targets/route.ts`):**
- Reads `AGENTGATEWAY_TARGETS_TOKEN` from `process.env` (returns 503 if unset)
- Validates incoming `Authorization: Bearer` with `timingSafeEqual`

Both sides already use the `Authorization: Bearer` pattern — JWT-SVIDs fit into it naturally.

---

## Authentication Strategy Decision

Three viable approaches; **JWT-SVID (Option B) is recommended**.

### Option A: mTLS with X.509-SVIDs
Each workload presents a TLS client certificate (SPIFFE X.509-SVID) to the other.
- Config-bridge connects over HTTPS presenting its cert; BFF validates peer cert SPIFFE ID.
- **Why rejected**: Next.js BFF is an HTTP-only service in-cluster. Making it a TLS server for internal traffic adds significant complexity (cert mounting, `https.createServer`, TLS termination). High blast radius for what is a config-polling path.

### Option B: JWT-SVIDs (recommended)
- Config-bridge requests a short-lived JWT from the SPIRE Workload API socket and sends it as `Authorization: Bearer <jwt-svid>`.
- BFF validates the JWT against the SPIRE bundle (JWKS), checks the SPIFFE ID in the `sub` claim.
- **Why chosen**: Fits the existing `Authorization: Bearer` pattern exactly. No TLS server changes to the BFF. JWT validation is standard. Token fallback is trivially maintained (if no socket → use static token).

### Option C: Service mesh (Istio/Linkerd)
SPIFFE mTLS injected at the sidecar layer — zero application code change.
- **Why rejected for now**: Requires Istio/Linkerd in every target cluster. Makes CAIPE depend on a cluster-level prerequisite most users won't have. Good future option once a mesh is a stated dependency.

---

## SPIFFE ID Conventions

Standard Kubernetes SPIFFE ID format: `spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>`

Proposed IDs for this project:
- **config-bridge**: `spiffe://caipe/ns/caipe/sa/caipe-agentgateway`
- **BFF (caipe-ui)**: `spiffe://caipe/ns/caipe/sa/caipe-caipe-ui`

Trust domain (`caipe`) is configurable via SPIRE Server `trust_domain` setting. Both IDs should be wired into the Helm chart as configurable values so operators can override for their org's trust domain.

---

## High-Level Implementation Plan

### 1. config-bridge (Python) — `deploy/agentgateway/config_bridge.py`

**New behavior:**
- At startup, check if `SPIFFE_ENDPOINT_SOCKET` is set.
  - If yes → SPIFFE mode: fetch a JWT-SVID from the Workload API
  - If no → token mode: use `AGENTGATEWAY_TARGETS_TOKEN` (current behavior, unchanged)
- Startup validation: if neither `SPIFFE_ENDPOINT_SOCKET` nor `AGENTGATEWAY_TARGETS_TOKEN` is set → raise `RuntimeError`
- JWT-SVIDs have short TTLs (~5 min); cache the SVID and refresh before expiry (check expiry on each poll iteration)

**New env vars consumed:**
- `SPIFFE_ENDPOINT_SOCKET` — path to SPIRE agent Workload API socket (standard SPIFFE env var)
- `AGENTGATEWAY_BRIDGE_AUDIENCE` — audience claim to request in the JWT-SVID (defaults to BFF SPIFFE ID)

**Library choice for Workload API:** `pyspiffe` (PyPI) provides a high-level `WorkloadApiClient`. Alternatively, use `grpc` directly against the socket — `pyspiffe` is preferred to avoid re-implementing the gRPC protocol.

**New dependency:** `pyspiffe` (add to `deploy/agentgateway/requirements.txt` and container image).

---

### 2. BFF (TypeScript) — `ui/src/app/api/internal/agentgateway/mcp-targets/route.ts`

**New behavior:**
- `authorizeBridgeRequest()` becomes auth-mode aware:
  - If `SPIFFE_JWKS_URL` (or `SPIRE_BUNDLE_ENDPOINT`) is set → SPIFFE mode: validate JWT-SVID
  - Otherwise → token mode: existing `timingSafeEqual` check (unchanged)
- JWT-SVID validation steps:
  1. Fetch JWKS from `SPIFFE_JWKS_URL` (SPIRE bundle endpoint, e.g. `https://spire-server:8081/bundle`)
  2. Verify signature, `exp`, `aud` claims
  3. Check `sub` claim equals expected config-bridge SPIFFE ID (`AGENTGATEWAY_BRIDGE_EXPECTED_SPIFFE_ID`)
- Cache the JWKS (short TTL, ~60s) to avoid a bundle fetch on every poll

**New env vars consumed:**
- `SPIFFE_JWKS_URL` — SPIRE bundle endpoint for JWKS (if set → SPIFFE mode active)
- `AGENTGATEWAY_BRIDGE_EXPECTED_SPIFFE_ID` — expected `sub` in the JWT-SVID (e.g. `spiffe://caipe/ns/caipe/sa/caipe-agentgateway`)

**Library:** `jose` (already likely present in Next.js ecosystem) for JWT verification.

---

### 3. Helm Chart — `charts/ai-platform-engineering/`

**New values under `global.agentgateway.static.configBridge.bff`:**
```yaml
spiffe:
  enabled: false
  # Path to SPIRE agent Workload API socket (mounted via CSI driver or hostPath)
  endpointSocket: "/run/spire/sockets/agent.sock"
  # SPIFFE ID the config-bridge will assert (used as JWT-SVID audience + BFF validation)
  bridgeSpiffeId: "spiffe://caipe/ns/caipe/sa/caipe-agentgateway"
  # SPIRE bundle endpoint URL (BFF uses this to fetch JWKS)
  bundleEndpointUrl: ""
```

**Changes to agentgateway `deployment.yaml`:**
- When `spiffe.enabled=true`:
  - Mount the SPIRE agent socket into the config-bridge container via a `hostPath` volume (or SPIRE CSI driver volume)
  - Inject `SPIFFE_ENDPOINT_SOCKET` and `AGENTGATEWAY_BRIDGE_AUDIENCE` env vars
  - The `existingSecret` block for `AGENTGATEWAY_TARGETS_TOKEN` becomes optional (can be omitted in SPIFFE mode)

**Changes to caipe-ui chart (or umbrella values):**
- When SPIFFE enabled, inject `SPIFFE_JWKS_URL` and `AGENTGATEWAY_BRIDGE_EXPECTED_SPIFFE_ID` into the caipe-ui deployment

**SPIRE server/agent deployment:**
- Out of scope for the chart — operators are expected to have SPIRE running
- Add a `prerequisites` note in values.yaml: SPIRE Server + Agent DaemonSet must be deployed, and the trust domain must match

---

### 4. `setup-caipe.sh`

- The token creation block we added in the previous fix (`AGENTGATEWAY_TARGETS_TOKEN` in `caipe-ui-secret`) should be gated on `!ENABLE_SPIFFE` — when SPIFFE is active, the secret is not needed
- Add a new flag `ENABLE_SPIFFE="${ENABLE_SPIFFE:-false}"` and `--spiffe` CLI arg
- When `ENABLE_SPIFFE=true`:
  - Skip the `AGENTGATEWAY_TARGETS_TOKEN` patch
  - Pass `--set "global.agentgateway.static.configBridge.bff.spiffe.enabled=true"`
  - Pass `--set "global.agentgateway.static.configBridge.bff.spiffe.bundleEndpointUrl=<url>"`
  - Accept `SPIRE_BUNDLE_ENDPOINT_URL` from env / CLI

---

### 5. Docker Compose (`docker-compose.yaml` / `docker-compose.dev.yaml`)

No changes needed. SPIFFE_ENDPOINT_SOCKET is not set in Compose → both sides stay in token mode. The existing `agentgateway-config-bridge-dev-token` fallback continues to work.

---

### 6. Tests

**config-bridge (Python):**
- `test_config_bridge.py`: add cases for SPIFFE mode
  - Mock `WorkloadApiClient.fetch_jwt_svid()` returning a fake SVID
  - Assert the SVID is sent as `Authorization: Bearer`
  - Assert fallback to token when socket env var is absent

**BFF (TypeScript):**
- Add test cases for JWT-SVID validation path in `route.ts` tests
  - Valid JWT-SVID with correct SPIFFE ID → 200
  - Valid JWT-SVID with wrong SPIFFE ID → 401
  - Expired JWT-SVID → 401
  - Fallback: no `SPIFFE_JWKS_URL` set → token validation (existing tests still pass)

---

## Open Questions to Resolve Before Coding

1. **pyspiffe vs raw gRPC**: `pyspiffe` adds a new dependency to the container image. Is that acceptable, or do we prefer a lightweight raw gRPC implementation (more code, no new dep)?

2. **JWKS caching strategy on the BFF**: Should JWKS be cached in memory (lost on pod restart) or in a shared Redis/MongoDB store? For a low-frequency polling path, in-memory with a 60s TTL is probably fine.

3. **Bundle endpoint exposure**: The SPIRE bundle endpoint is typically internal. Do we need a Kubernetes Service for it, or will operators configure external SPIRE federation?

4. **Trust domain**: Hardcoded to `caipe` in examples above, but this must be configurable. Decide on the Helm value path and default.

5. **Audience claim**: SPIRE JWT-SVIDs require an audience. What is the canonical audience string? Options: the BFF SPIFFE ID, the BFF service URL, or a fixed string like `agentgateway-config-bridge`. The BFF SPIFFE ID is the most semantically correct.

6. **Grace period on token deprecation**: When SPIFFE mode lands, should `AGENTGATEWAY_TARGETS_TOKEN` support become a deprecation warning or hard-remove? Recommend: keep as fallback indefinitely for Docker Compose compat; log a warning in production if token mode is active.

---

## Files to Modify (summary)

| File | Change |
|------|--------|
| `deploy/agentgateway/config_bridge.py` | JWT-SVID fetch + fallback logic |
| `deploy/agentgateway/requirements.txt` | Add `pyspiffe` |
| `deploy/agentgateway/Dockerfile` (if any) | No change if requirements.txt is used |
| `deploy/agentgateway/tests/test_config_bridge.py` | New SPIFFE test cases |
| `ui/src/app/api/internal/agentgateway/mcp-targets/route.ts` | JWT-SVID validation + fallback |
| `ui/src/app/api/internal/agentgateway/mcp-targets/route.test.ts` | New SPIFFE test cases |
| `charts/ai-platform-engineering/values.yaml` | New `spiffe` block under configBridge.bff |
| `charts/ai-platform-engineering/charts/agentgateway/templates/deployment.yaml` | Conditional SPIRE socket mount + SPIFFE env vars |
| `setup-caipe.sh` | ENABLE_SPIFFE flag, skip token patch when SPIFFE active |

---

## Non-Goals (from issue)

- Per-tool MCP authorization in the BFF request path
- Changes to the target DTO contract from PR #1852
- Deploying SPIRE Server/Agent (operator responsibility)
