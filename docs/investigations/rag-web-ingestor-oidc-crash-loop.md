# Investigation: RAG web-ingestor OIDC — single container + UI 401 after PR #1803

**Date:** 2026-06-11
**Status:** Partially fixed on branch `sibu/bug/rag` — fresh installs still require two `setup-caipe.sh` runs; structural fix (Option C) deferred
**Affected version:** `0.5.11` (post-merge of `a16b3538`)
**Install path affected:** RBAC enabled, no DNS domain (most common interactive install)

---

## Symptoms

After deploying with the PR #1803 Helm chart:

1. **`rag-server` pod starts with only one container.** The `web-ingestor` sidecar is absent — pod shows `1/1 Running` instead of `2/2`.
2. **`caipe-caipe-ui` health check fails with HTTP 401.** The rag-server has no `OIDC_ISSUER` configured so it rejects all browser bearer tokens.

---

## Background: the original crash loop

Before PR #1803, the `web-ingestor` sidecar crash-looped on startup. Root causes:

- **`f653f0af`** (`fix(rbac): complete RAG OpenFGA access model`, 2026-05-27) removed the trusted-network fallback in `ingestor.py`, making OIDC credentials mandatory. `_get_access_token()` now raises `RuntimeError` immediately when credentials are absent instead of returning `None`.
- **`setup-caipe.sh`** never provisioned `INGESTOR_OIDC_CLIENT_SECRET` into the deployment, and the OIDC propagation block was gated on `OIDC_ISSUER` in `caipe-ui-secret` — which is never set in the default no-domain install.

> RBAC (Keycloak + OpenFGA) is required in 0.5.10+ and is always enabled. Commit `347e753d` hardcoded `ENABLE_RBAC_RUNTIME=true` in `choose_features()`. RAG without RBAC is not a supported path.

A temporary cluster fix on 2026-06-09 used the in-cluster URL and `caipe-platform` service account:

```bash
kubectl set env deployment/rag-server -n caipe \
  INGESTOR_OIDC_ISSUER="http://caipe-keycloak:8080/realms/caipe" \
  INGESTOR_OIDC_CLIENT_ID="caipe-platform" \
  INGESTOR_OIDC_CLIENT_SECRET="caipe-platform-dev-secret"
```

Ephemeral — overwritten on the next `helm upgrade`.

---

## What PR #1803 changed

**PR:** `fix(setup-caipe): Ollama FQDN, RAG ingestor client, dynamic-agents bearer auth`
**Merge commit:** `a16b3538` | **Key commits:** `6d799cb7`, `1f9a11f7`

| Commit | Change | Scope |
|---|---|---|
| `6d799cb7` | Added `provision_rag_ingestor_client()` — creates `caipe-web-ingestor` Keycloak client, stores credentials in `rag-ingestor-secret` | Domain path only |
| `1f9a11f7` | Fixed `deployment.yaml` missing `{{- with .Values.envFrom }}` block so `webIngestor.envFrom` renders | All paths |
| `1f9a11f7` | Fixed `post_deploy_patches()` to read `INGESTOR_OIDC_*` from `rag-ingestor-secret` instead of `caipe-ui-secret` | Domain path only |
| `1f9a11f7` | Added `OIDC_ISSUER/CLIENT_ID/GROUP_CLAIM` to rag-server Helm args | Domain path only |
| `1f9a11f7` | Wired `webIngestor.envFrom` → `rag-ingestor-secret` in Helm args | Only when `RAG_INGESTOR_SECRET_READY=true` |

**Working only when `CAIPE_DOMAIN` is a DNS hostname. All other paths broken.**

---

## Root causes

Three bugs combine in the no-domain path.

### Bug 1 — `provision_rag_ingestor_client()` was domain-gated

The function bailed immediately without a domain:

```bash
provision_rag_ingestor_client() {
  $ENABLE_RAG || return 0
  [[ -n "${CAIPE_DOMAIN:-}" ]] || return 0   # ← exits; rag-ingestor-secret never created
```

`rag-ingestor-secret` never existed in the cluster. `RAG_INGESTOR_SECRET_READY` stayed `false`. The Helm args block then hit its `else`:

```bash
if [[ "${RAG_INGESTOR_SECRET_READY:-false}" == "true" ]]; then
  helm_args+=(--set 'rag-stack.rag-server.webIngestor.enabled=true' ...)
else
  helm_args+=(--set 'rag-stack.rag-server.webIngestor.enabled=false')   # ← always
fi
```

The web-ingestor sidecar is **explicitly disabled at deploy time**. The original crash loop is gone only because the container was removed.

### Bug 2 — call site was inside the `CAIPE_DOMAIN` block in `post_deploy_patches()`

Even if the function had a no-domain fallback, it was unreachable. In `post_deploy_patches()` the call was nested inside `if [[ -n "${CAIPE_DOMAIN:-}" ]]; then ... fi`:

```bash
if [[ -n "${CAIPE_DOMAIN:-}" ]]; then
  ...
  provision_rag_ingestor_client   # ← never reached without a domain
  provision_caipe_ui_audience_mapper
fi
```

### Bug 3 — `provision_rag_ingestor_client()` runs after helm

`post_deploy_patches()` is called after `deploy_caipe()` in the main execution:

```bash
deploy_caipe        # helm install/upgrade — RAG_INGESTOR_SECRET_READY evaluated here
post_deploy_patches # provision_rag_ingestor_client called here — too late
```

Even if Bugs 1 and 2 were fixed, `RAG_INGESTOR_SECRET_READY` would always be `false` at helm time on a fresh install. The secret can only be created after Keycloak is running, and Keycloak is deployed by helm.

### Bug 4 — `OIDC_ISSUER` for rag-server not set in no-domain path (→ HTTP 401)

The rag-server `OIDC_ISSUER` Helm arg was domain-gated:

```bash
if [[ -n "${CAIPE_DOMAIN:-}" ]]; then   # ← skipped with no domain
  helm_args+=(
    --set "rag-stack.rag-server.env.OIDC_ISSUER=https://${CAIPE_DOMAIN}/realms/caipe"
    ...
  )
fi
```

Without `OIDC_ISSUER`, the rag-server auth manager has no OIDC provider → rejects all browser bearer tokens → HTTP 401.

---

## Install path state (post-PR #1803, pre-fix)

| Path | `CAIPE_DOMAIN` | `provision_rag_ingestor_client` called? | `webIngestor.enabled` | `OIDC_ISSUER` on rag-server? | Result |
|---|---|---|---|---|---|
| 1 — **most common** | not set | ✗ (domain guard in caller) | false | ✗ | single container; UI 401 |
| 2 | IP address | ✗ (same domain guard) | false | ✗ | single container; UI 401 |
| 3 | DNS hostname | ✓ | true | ✓ | working |

---

## Fix applied (branch `sibu/bug/rag`)

Three changes. Together they resolve all four bugs.

### Fix 1 — Refactor `provision_rag_ingestor_client()` to work without a domain

**File:** `setup-caipe.sh:4382`

The domain only affects the issuer URL. The Keycloak client creation logic is identical for both paths. Removed the domain guard and the incorrect `caipe-platform-secret` fallback. The function now always creates a dedicated `caipe-web-ingestor` Keycloak client, using the in-cluster URL when no public domain is available.

```bash
provision_rag_ingestor_client() {
  $ENABLE_RAG || return 0

  # Pick issuer: public HTTPS with a domain, in-cluster URL without.
  local issuer
  if [[ -n "${CAIPE_DOMAIN:-}" ]]; then
    issuer="https://${CAIPE_DOMAIN}/realms/caipe"
  else
    issuer="http://caipe-keycloak:8080/realms/caipe"
  fi

  # Get Keycloak admin credentials and port-forward — same for both paths.
  ...

  # Create caipe-web-ingestor client (idempotent) and store in rag-ingestor-secret.
  kubectl create secret generic rag-ingestor-secret -n caipe \
    --from-literal=INGESTOR_OIDC_ISSUER="${issuer}" \
    --from-literal=INGESTOR_OIDC_CLIENT_ID="caipe-web-ingestor" \
    --from-literal=INGESTOR_OIDC_CLIENT_SECRET="${client_secret}" \
    --dry-run=client -o yaml | kubectl apply -f - &>/dev/null
  RAG_INGESTOR_SECRET_READY=true
  RAG_INGESTOR_OIDC_ISSUER="${issuer}"
  RAG_INGESTOR_OIDC_CLIENT_ID="caipe-web-ingestor"
}
```

### Fix 2 — Move call outside `CAIPE_DOMAIN` block in `post_deploy_patches()`

**File:** `setup-caipe.sh:4270` (call site)

Removed the call from inside `if [[ -n "${CAIPE_DOMAIN:-}" ]]; then`. Added it as a standalone block after the domain-gated section:

```bash
  fi  # end of CAIPE_DOMAIN block

  # Runs regardless of domain — creates rag-ingestor-secret after Keycloak is ready.
  if $ENABLE_RAG; then
    provision_rag_ingestor_client
  fi
}
```

### Fix 3 — Detect existing `rag-ingestor-secret` before helm args in `deploy_caipe()`

**File:** `setup-caipe.sh:6268` (inside the `if $ENABLE_RAG; then` block)

On upgrades, `rag-ingestor-secret` already exists from the prior run. Reading it before building helm args means `webIngestor.enabled=true` is passed to helm without waiting for `post_deploy_patches` again:

```bash
if [[ "${RAG_INGESTOR_SECRET_READY:-false}" != "true" ]]; then
  local _ri_issuer _ri_client
  _ri_issuer=$(kubectl get secret rag-ingestor-secret -n caipe \
    -o jsonpath='{.data.INGESTOR_OIDC_ISSUER}' 2>/dev/null | base64 -d || true)
  _ri_client=$(kubectl get secret rag-ingestor-secret -n caipe \
    -o jsonpath='{.data.INGESTOR_OIDC_CLIENT_ID}' 2>/dev/null | base64 -d || true)
  if [[ -n "$_ri_issuer" && -n "$_ri_client" ]]; then
    RAG_INGESTOR_SECRET_READY=true
    RAG_INGESTOR_OIDC_ISSUER="$_ri_issuer"
    RAG_INGESTOR_OIDC_CLIENT_ID="$_ri_client"
    log "RAG ingestor: existing rag-ingestor-secret detected (issuer=${_ri_issuer})"
  fi
fi
```

The `OIDC_ISSUER` Helm arg also gained a no-domain branch (already applied earlier):

```bash
if [[ -n "${CAIPE_DOMAIN:-}" ]]; then
  helm_args+=(--set "rag-stack.rag-server.env.OIDC_ISSUER=https://${CAIPE_DOMAIN}/realms/caipe" ...)
elif $ENABLE_RBAC_RUNTIME; then
  helm_args+=(--set 'rag-stack.rag-server.env.OIDC_ISSUER=http://caipe-keycloak:8080/realms/caipe' ...)
fi
```

### Fix 4 — Escape comma in `OIDC_GROUP_CLAIM` Helm arg

**File:** `setup-caipe.sh:6296, 6303`

Helm `--set` treats commas as list separators. `members,groups` was parsed as two items, causing:

```
Error: failed parsing --set data: key "groups" has no value
```

Fixed by escaping:

```bash
--set 'rag-stack.rag-server.env.OIDC_GROUP_CLAIM=members\,groups'
```

---

## Behaviour after fix

| Scenario | Run 1 | Run 2 |
|---|---|---|
| **Fresh install (no domain)** | helm with `webIngestor.enabled=false`; `post_deploy_patches` creates `rag-ingestor-secret` via Keycloak | secret detected before helm → `webIngestor.enabled=true` → 2/2 |
| **Upgrade (secret exists)** | secret detected before helm → `webIngestor.enabled=true` → 2/2 | — |
| **DNS domain** | unchanged — was already working | — |

Fresh installs require two `setup-caipe.sh` runs because `provision_rag_ingestor_client()` needs Keycloak running to create the client via the Admin API, but Keycloak is deployed by the same helm release. This is a known limitation of the current fix and is tracked for resolution in the next step.

---

## Known limitation — fresh install requires two runs

The two-run requirement is a structural problem. Three approaches were discussed.

### Option A — Targeted `helm upgrade --reuse-values` after provisioning

After `provision_rag_ingestor_client()` creates the secret in `post_deploy_patches()`, check whether the web-ingestor container is missing from the deployment and immediately run a focused upgrade:

```bash
helm upgrade caipe "$CAIPE_OCI_REPO" \
  --version "$CAIPE_CHART_VERSION" --namespace caipe --reuse-values \
  --set 'rag-stack.rag-server.webIngestor.enabled=true' \
  --set 'rag-stack.rag-server.webIngestor.envFrom[0].secretRef.name=rag-ingestor-secret' \
  --set "rag-stack.rag-server.env.INGESTOR_OIDC_ISSUER=${RAG_INGESTOR_OIDC_ISSUER}" \
  --set "rag-stack.rag-server.env.INGESTOR_OIDC_CLIENT_ID=${RAG_INGESTOR_OIDC_CLIENT_ID}"
```

- **Pros:** Minimal change (~20 lines). Single run.
- **Cons:** `--reuse-values` is fragile — it picks up the stored release values, which include `webIngestor.enabled=false`. The `--set` override wins but this is a subtle dependency. Adds ~15s helm pull time to every fresh install. Perpetuates the `SKIP_INIT_TESTS` architectural debt.

### Option B — Two-pass helm inside `deploy_caipe()`

Split the single helm call into two: first without rag-stack (to get Keycloak running), then with rag-stack after provisioning the ingestor client.

- **Pros:** Single run. No `--reuse-values` fragility.
- **Cons:** Helm invoked twice on every install/upgrade (~30–60s overhead). Requires splitting the monolithic `helm_args` array carefully.

### Option C — Two-phase deploy with proper dependency ordering ✅ chosen

The chart already has `tags.rag-stack` to enable/disable the entire RAG stack. No chart changes required — all work is in `setup-caipe.sh`.

**Phase 1** — Deploy everything except rag-stack. Wait for Keycloak. Provision `caipe-web-ingestor` client.

**Phase 2** — Deploy with rag-stack enabled. By this point: `rag-ingestor-secret` exists, `RAG_INGESTOR_SECRET_READY=true`, `webIngestor.enabled=true` is passed to helm. Wait for Milvus, then finalize rag-server.

This also resolves the Milvus ordering issue — rag-server currently starts before Milvus via the `SKIP_INIT_TESTS=true` workaround. With Phase 2 deploying rag-stack after Phase 1 waits for the cluster to settle, Milvus starts before rag-server.

**Scope:** ~100–150 lines, `setup-caipe.sh` only.

| Change | Details |
|---|---|
| `deploy_caipe()` | Split `helm_args` into `common_args` + `rag_args`; two `helm upgrade` calls |
| `_wait_for_keycloak()` | New ~15-line function mirroring `_wait_for_milvus()` |
| `provision_rag_ingestor_client()` call | Moves from `post_deploy_patches()` into `deploy_caipe()` between the two helm calls |
| `post_deploy_patches()` | Remove the now-redundant `provision_rag_ingestor_client` call |
| `SKIP_INIT_TESTS` | Can be retired — Milvus is running before rag-server starts in Phase 2 |

**Comparison:**

| | Option A | Option B | Option C |
|---|---|---|---|
| Fresh install | Single run | Single run | Single run |
| Upgrade | Single run | Single run | Single run |
| Milvus ordering | No change (still racy) | No change | Guaranteed |
| `SKIP_INIT_TESTS` workaround | Stays | Stays | Retired |
| Script changes | ~20 lines | ~60 lines | ~100–150 lines |
| Risk | `--reuse-values` edge cases | Low | Medium — two helm calls, higher blast radius |
| Correctness | Workaround on a workaround | Better | Structurally correct |

---

## Next steps

1. **Implement Option C** in `setup-caipe.sh`:
   - Add `_wait_for_keycloak()` (mirrors `_wait_for_milvus()`)
   - Refactor `deploy_caipe()`: build `common_args` (all non-RAG flags) and `rag_args` (rag-stack flags) separately; Phase 1 helm with `tags.rag-stack=false`; wait for Keycloak; call `provision_rag_ingestor_client()`; Phase 2 helm with full args
   - Remove `provision_rag_ingestor_client()` call from `post_deploy_patches()`
   - Remove `SKIP_INIT_TESTS=true` from helm args; simplify `_finalize_rag_startup()`

2. **Remove the existing-secret detection block** added in the current fix (Fix 3 above) — it becomes unnecessary once Option C is in place, since the secret is always created before the rag-stack helm call.

3. **Test paths:** fresh install (no domain), fresh install (DNS domain), upgrade from a cluster that already has `rag-ingestor-secret`.

---

## Key file locations

| File | Relevance |
|---|---|
| `setup-caipe.sh:4382` | `provision_rag_ingestor_client()` — refactored; no domain guard; always creates `caipe-web-ingestor` client |
| `setup-caipe.sh:4279` | Call site in `post_deploy_patches()` — now outside `CAIPE_DOMAIN` block |
| `setup-caipe.sh:6268` | Existing-secret detection before helm args |
| `setup-caipe.sh:6292` | `OIDC_ISSUER` Helm arg — now has `elif $ENABLE_RBAC_RUNTIME` branch for no-domain |
| `setup-caipe.sh:6308` | `webIngestor.enabled` Helm arg — gated on `RAG_INGESTOR_SECRET_READY` |
| `setup-caipe.sh:2691` | `ENABLE_RBAC_RUNTIME=true` hardcoded — RBAC always on in 0.5.10+ |
| `charts/rag-stack/charts/rag-server/templates/deployment.yaml:58–60` | `{{- with .Values.envFrom }}` block — added in PR #1803; renders correctly |
| `ai_platform_engineering/knowledge_bases/rag/common/src/common/ingestor.py` | `_get_access_token()` — raises `RuntimeError` when credentials absent; acceptable since credentials are now always wired after first run |

---

## Commit history

| Commit | Date | Description |
|---|---|---|
| `01515003` | 2026-01-29 | `feat(ingestor)`: OIDC with trusted-network fallback (the correct original design) |
| `f653f0af` | 2026-05-27 | `fix(rbac)`: removed trusted-network fallback — made OIDC mandatory |
| `6d799cb7` | 2026-06-10 | PR #1803: `provision_rag_ingestor_client()` — domain-gated |
| `1f9a11f7` | 2026-06-10 | PR #1803: fixed `deployment.yaml` envFrom; OIDC Helm args — domain-gated |
| `sibu/bug/rag` | 2026-06-11 | This fix: refactored `provision_rag_ingestor_client`, moved call outside domain guard, added existing-secret detection, escaped `OIDC_GROUP_CLAIM` comma |

---

## What NOT to do

- **Do not use `caipe-platform-secret` for the ingestor.** It belongs to the platform, not the RAG service. The ingestor must have its own `caipe-web-ingestor` Keycloak client.
- **Do not gate `provision_rag_ingestor_client()` on `CAIPE_DOMAIN`.** The issuer URL differs but the Keycloak client creation is identical. The domain guard was the direct cause of both failures.
- **Do not read `OIDC_ISSUER` from `caipe-ui-secret` for the ingestor.** It is only written there in the domain path; in the default install it is absent.
- **Do not set `CAIPE_UNSAFE_RBAC_BYPASS=true` on deployed clusters.** RBAC is always on in 0.5.10+; this bypasses all authorization checks.
