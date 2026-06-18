# Pending Chore Issues

Trivial bugs and cleanups identified but not yet filed as GitHub issues. Review when batching chore PRs.

---

## [setup-caipe] Duplicate `HELM_AGENT_ARGS` in `deploy_caipe()`

**File:** `setup-caipe.sh`
**Function:** `deploy_caipe()`

### Problem

`HELM_AGENT_ARGS` is appended to the `helm_args` array **twice** within the same function:

| Occurrence | Lines | Log message |
|---|---|---|
| First | ~6386–6389 | `"Wiring ... agent Helm flags from interactive selection"` |
| Second | ~6830–6832 | `"Agent helm args added"` |

Every `--set tags.agent-X`, `--set agent-X.agentSecrets.secretName`, and `--set supervisor-agent.singleNode.enabledSubAgents.X` flag is passed twice to `helm upgrade --install caipe`.

### Impact

Low — Helm silently uses the last occurrence of a duplicate `--set` key, so deployments are not broken. However, the duplication clutters debug output and could cause subtle bugs if the two occurrences ever diverge in value.

### Fix

Delete the second append block (~lines 6829–6833, including its leading comment):

```bash
# REMOVE:
  # Agent and UI secrets provisioned from --env-file / --ui-env-file
  if [[ ${#HELM_AGENT_ARGS[@]} -gt 0 ]]; then
    helm_args+=("${HELM_AGENT_ARGS[@]}")
    log "Agent helm args added (${#HELM_AGENT_ARGS[@]} flags)"
  fi
```

The first append at ~line 6386 covers all code paths (interactive, env-file, and unattended-upgrade re-population).

### Notes

- `HELM_UI_SECRET_ARGS` is **not** affected — it is only appended once.
- Consider clubbing with GitHub [#1843](https://github.com/cnoe-io/ai-platform-engineering/issues/1843) (duplicate keys in `values.yaml`) as part of a Helm duplication cleanup chore.
