# CAIPE SSO Auth Debug Notes

**Status**: In progress — not yet committed  
**Branch**: `sibu/test`  
**Date started**: 2026-06-18

---

## Problem

When CAIPE is installed via `setup-caipe.sh` with RBAC enabled but **without** a `--domain`/`--ingress` flag (i.e., a local kind cluster accessed via port-forward), the UI does not redirect to Keycloak for login. The chat feature fails silently with:

```
[ChatStore] API call failed:
Object { error: APIClientError, errorMessage: "You are not signed in. Please sign in to continue.", ... }
```

---

## Root Cause

### 1. `setup-caipe.sh` gates SSO on `CAIPE_DOMAIN`

The `deploy_caipe` function (~line 6336) only sets `SSO_ENABLED=true` when `--domain` or `--ingress` is provided:

```bash
if [[ -n "$CAIPE_DOMAIN" ]]; then
  helm_args+=(--set "caipe-ui.config.SSO_ENABLED=true")
else
  helm_args+=(--set "caipe-ui.config.SSO_ENABLED=false")  # <-- even when RBAC is active!
fi
```

Result: RBAC-active local installs have Keycloak deployed but `SSO_ENABLED=false` in the configmap, so `AuthGuard` never redirects to login, and API middleware rejects all requests with 401.

### 2. `auth-config.ts` — `authorization.url` override doesn't work in NextAuth v4

When `OIDC_DISCOVERY_URL` is set to the in-cluster URL (`http://caipe-keycloak:8080/realms/caipe`), the `wellKnown` discovery returns endpoints with `http://caipe-keycloak:8080/...` — URLs the browser cannot reach.

An `authorization.url` override was added to force the browser to use `http://localhost:7080/...`, but **NextAuth v4 ignores `authorization.url` when `wellKnown` is set** — it always uses the `authorization_endpoint` from the discovery document.

Confirmed by testing:
```bash
# POST to trigger NextAuth sign-in with proper CSRF token
REDIRECT_URL=$(curl -sb /tmp/csrf-cookies.txt \
  -d "csrfToken=${CSRF_TOKEN}&callbackUrl=http://localhost:3000&json=true" \
  -s "http://localhost:3001/api/auth/signin/oidc" \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get('url',''))")
# Result: http://caipe-keycloak:8080/... ← in-cluster, browser can't reach
```

### 3. `chat-store.ts` — noisy 401 logging (minor, already fixed)

The catch block used a string match that missed `APIClientError.status === 401`, causing 401s to log as loud errors instead of being silently ignored when SSO is disabled.

---

## Fixes Applied So Far

### ✅ Fix 1: `setup-caipe.sh` — enable SSO when RBAC is active (no domain)

In `deploy_caipe` (~line 6336), changed:

```bash
# Before
if [[ -n "$CAIPE_DOMAIN" ]]; then
  helm_args+=(--set "caipe-ui.config.SSO_ENABLED=true")
else
  helm_args+=(--set "caipe-ui.config.SSO_ENABLED=false")
fi

# After
if [[ -n "$CAIPE_DOMAIN" ]]; then
  helm_args+=(--set "caipe-ui.config.SSO_ENABLED=true")
elif $ENABLE_RBAC_RUNTIME; then
  helm_args+=(--set "caipe-ui.config.SSO_ENABLED=true")
  if [[ -z "$UI_ENV_FILE" ]]; then
    helm_args+=(
      --set "caipe-ui.config.NEXTAUTH_URL=http://localhost:${UI_PORT}"
      --set "caipe-ui.config.OIDC_ISSUER=http://localhost:${KEYCLOAK_PORT}/realms/caipe"
      --set "caipe-ui.config.OIDC_DISCOVERY_URL=http://caipe-keycloak:8080/realms/caipe"
      --set "openfga-authz-bridge.tokenValidation.issuer=http://localhost:${KEYCLOAK_PORT}/realms/caipe"
    )
  fi
else
  helm_args+=(--set "caipe-ui.config.SSO_ENABLED=false")
fi
```

Also updated `provision_ui_secret` (~line 3218) for the `--ui-env-file` path:

```bash
# Before
if [[ -n "$CAIPE_DOMAIN" ]]; then
  HELM_UI_SECRET_ARGS+=(--set "caipe-ui.env.SSO_ENABLED=true")
fi

# After
if [[ -n "$CAIPE_DOMAIN" ]] || $ENABLE_RBAC_RUNTIME; then
  HELM_UI_SECRET_ARGS+=(--set "caipe-ui.env.SSO_ENABLED=true")
fi
```

### ✅ Fix 2: `ui/src/store/chat-store.ts` — correct 401 suppression

```typescript
// Added import
import { APIClientError, apiClient } from "@/lib/api-client";

// In catch block, replaced string-match with:
const is401 =
  (apiError instanceof APIClientError && apiError.status === 401) ||
  errorMessage.includes('Unauthorized') ||
  errorMessage.includes('401') ||
  errorMessage.includes('not signed in');
if (is401) {
  console.log('[ChatStore] User not authenticated - using local storage only');
} else {
  console.error('[ChatStore] API call failed:', { error: apiError, errorMessage, ... });
}
```

### ❌ Fix 3: `ui/src/lib/auth-config.ts` — `authorization.url` override (NOT WORKING)

Added this to the NextAuth provider config:
```typescript
authorization: {
  ...(process.env.OIDC_ISSUER && process.env.OIDC_DISCOVERY_URL
    ? { url: `${process.env.OIDC_ISSUER}/protocol/openid-connect/auth` }
    : {}),
  params: { scope: "openid email profile groups", ... }
}
```

**Result**: Does not work. NextAuth v4 uses `authorization_endpoint` from `wellKnown` discovery doc, ignoring `authorization.url`.

---

## Remaining Problem: `auth-config.ts` — In-cluster Discovery URLs

### What we need

| Caller | Should use |
|--------|-----------|
| Browser (auth redirect) | `http://localhost:7080/realms/caipe/...` |
| Next.js pod (token refresh, userinfo) | `http://caipe-keycloak:8080/realms/caipe/...` |

### Current state

- `wellKnown` points to `OIDC_DISCOVERY_URL` (in-cluster) → all discovered endpoints are in-cluster
- NextAuth uses discovered `authorization_endpoint` for the browser redirect → browser gets `http://caipe-keycloak:8080/...` which it can't reach

### Possible solutions to investigate

#### Option A: Manually specify all endpoints (skip `wellKnown`)

Drop `wellKnown` and hard-code Keycloak's well-known paths:

```typescript
{
  id: "oidc",
  type: "oauth",
  // No wellKnown
  authorization: `${process.env.OIDC_ISSUER}/protocol/openid-connect/auth`, // browser-facing
  token: {
    url: `${process.env.OIDC_DISCOVERY_URL || process.env.OIDC_ISSUER}/protocol/openid-connect/token`, // server-side
  },
  userinfo: {
    url: `${process.env.OIDC_DISCOVERY_URL || process.env.OIDC_ISSUER}/protocol/openid-connect/userinfo`,
  },
  jwks_endpoint: `${process.env.OIDC_DISCOVERY_URL || process.env.OIDC_ISSUER}/protocol/openid-connect/certs`,
  checks: ["pkce", "state"],
  ...
}
```

**Pros**: Full control over each endpoint; no discovery needed  
**Cons**: Assumes Keycloak URL patterns; breaks if paths change; must handle PKCE manually; `refreshAccessToken` in auth-config.ts also uses in-cluster URL and needs auditing  
**Verdict**: Most likely to work; Keycloak paths are stable

#### Option B: Fetch discovery doc server-side and rewrite `authorization_endpoint`

Fetch the discovery doc at startup, replace `authorization_endpoint` with OIDC_ISSUER-based URL, then pass `profile`/endpoints to NextAuth explicitly.

**Pros**: Other endpoints still auto-discovered  
**Cons**: Extra complexity; may interfere with NextAuth internals

#### Option C: Set `KC_HOSTNAME` on Keycloak Helm chart for localhost

Configure Keycloak with `KC_HOSTNAME=localhost` and `KC_HOSTNAME_PORT=7080` so that even when queried via internal service, the discovery doc returns `localhost:7080` URLs.

```yaml
# In Keycloak helm values
env:
  KC_HOSTNAME: localhost
  KC_HOSTNAME_PORT: "7080"
  KC_HOSTNAME_STRICT: "false"
```

**Pros**: No changes to `auth-config.ts`; discovery doc is always browser-correct  
**Cons**: Token endpoint also returns `localhost:7080` → server-side token refresh from the pod fails; Keycloak issuer validation may break; only viable when port-forward is always 7080  
**Verdict**: Not viable for token refresh

#### Option D: Set `OIDC_DISCOVERY_URL=OIDC_ISSUER` (same URL for both)

Use `localhost:7080` for both discovery and redirect, and accept that token refresh fails (users re-login every ~5 min when access token expires).

**Pros**: Simplest change  
**Cons**: Server pod can't reach `localhost:7080` → token refresh fails → short sessions  
**Verdict**: Poor UX; not recommended

---

## Test Environment

- **Cluster**: kind (`caipe-control-plane`)
- **Port-forwards**: UI on `localhost:3001` (pod 3000), Keycloak on `localhost:7080` (pod 8080)
- **Configmap patch applied**: `caipe-caipe-ui-config`
  - `SSO_ENABLED=true`
  - `OIDC_ISSUER=http://localhost:7080/realms/caipe`
  - `OIDC_DISCOVERY_URL=http://caipe-keycloak:8080/realms/caipe`
- **Custom image**: `caipe-ui:local-test` loaded into kind via `kind load docker-image`

### Useful test commands

```bash
# Rebuild and reload image after code changes
docker build -f build/Dockerfile.caipe-ui --target runner -t caipe-ui:local-test ui/
kind load docker-image caipe-ui:local-test --name caipe
kubectl rollout restart deployment/caipe-caipe-ui -n caipe

# Re-apply configmap patch
kubectl patch configmap caipe-caipe-ui-config -n caipe --type merge -p '{
  "data": {
    "SSO_ENABLED": "true",
    "OIDC_ISSUER": "http://localhost:7080/realms/caipe",
    "OIDC_DISCOVERY_URL": "http://caipe-keycloak:8080/realms/caipe"
  }
}'

# Port-forwards
kubectl port-forward -n caipe svc/caipe-caipe-ui 3001:3000 &
kubectl port-forward -n caipe svc/caipe-keycloak 7080:8080 &

# Test sign-in flow (requires CSRF token)
CSRF_RESPONSE=$(curl -sc /tmp/csrf-cookies.txt -s http://localhost:3001/api/auth/csrf)
CSRF_TOKEN=$(echo "$CSRF_RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin)['csrfToken'])")
REDIRECT_URL=$(curl -sb /tmp/csrf-cookies.txt \
  -d "csrfToken=${CSRF_TOKEN}&callbackUrl=http://localhost:3000&json=true" \
  -s "http://localhost:3001/api/auth/signin/oidc" \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get('url',''))")
echo "Redirect URL: $REDIRECT_URL"
# Should be: http://localhost:7080/... (not http://caipe-keycloak:8080/...)
```

---

## Files Modified (not yet committed)

| File | Status |
|------|--------|
| `setup-caipe.sh` | Modified (Fix 1) |
| `ui/src/store/chat-store.ts` | Modified (Fix 2) |
| `ui/src/lib/auth-config.ts` | Modified (Fix 3 — NOT working, needs different approach) |

---

## Recommended Next Step

Implement **Option A** in `ui/src/lib/auth-config.ts`: replace `wellKnown` with manually specified endpoints so `OIDC_ISSUER` drives the browser-facing `authorization` URL and `OIDC_DISCOVERY_URL` drives the server-side `token`/`userinfo`/`jwks` URLs.

After editing, rebuild the image, reload into kind, patch the configmap, restart the deployment, and re-run the CSRF test above. The redirect URL should show `http://localhost:7080/...`.
