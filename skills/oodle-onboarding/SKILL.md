---
name: oodle-onboarding
description: Integrate infrastructure with the Oodle observability platform — list available integrations, fetch setup specs, and execute step-by-step installation for any integration type (Kubernetes, AWS, GCP, etc.).
metadata:
  version: "1.0.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,onboarding,integration,kubernetes,observability,setup
  globs: ""
  alwaysApply: "false"
---

# Oodle Onboarding — Integration Setup

This skill teaches the agent to integrate infrastructure with the Oodle observability platform using the `oodle integrations` commands. The setup spec fetched from the API is the source of truth — never hardcode integration steps.

## Prerequisites

```bash
# Install + configure (see oodle-cli skill)
brew install oodle-ai/oodle/oodle
oodle configure
# or
export OODLE_API_KEY=<key>
export OODLE_INSTANCE=<instance>
export OODLE_DEPLOYMENT=<url>
```

Verify the CLI works:

```bash
oodle --version
```

**Note:** `oodle integrations get-setup-spec` does NOT require authentication. Use it to fetch setup instructions even before the CLI is fully configured. All other integration commands require valid credentials.

## Command Execution Order

Follow this sequence for every integration onboarding request:

1. **Identify the integration type** from the user's request (e.g., "kubernetes", "aws-cloudwatch").
2. Run `oodle integrations list -o json` to get the full integration records. From this output, extract: (a) the `type` field (lowercase it for step 3), (b) `apiKey`, (c) `collectorDomain`/`logsCollectorDomain` for helm values, (d) instance ID from the domain pattern. If auth is not configured, guide the user through setup first (see Prerequisites), or skip to step 3 if the user already named a type.
3. Run `oodle integrations get-setup-spec <type-lowercase> -o json` to fetch the authoritative setup specification. This does NOT require auth.
4. Check every requirement listed in the spec (tools, access, permissions).
5. Collect every required parameter (from environment, user's request, or by asking).
6. Execute the setup steps from the spec **one at a time**, confirming with the user before every non-read-only command (any command that creates, modifies, or deletes resources).
7. Validate the integration by running `oodle integrations list -o json` and checking the status.

Do not skip steps. Do not invent steps that are not in the setup spec.

## Quick Reference

| Task | Command |
|------|---------|
| List integrations | `oodle integrations list -o json` |
| List integrations (table) | `oodle integrations list` |
| Get setup spec | `oodle integrations get-setup-spec <type> -o json` |
| Get setup spec (YAML) | `oodle integrations get-setup-spec <type> -o yaml` |

Aliases: `oodle integration`, `oodle integ`

## Critical Gotchas

### Integration type must be lowercase in `get-setup-spec`

The `type` field in `integrations list` is UPPERCASE (e.g., `KUBERNETES`, `OTEL`) but `get-setup-spec` requires **lowercase** (e.g., `kubernetes`). Always lowercase the type before calling `get-setup-spec`.

### Helm values: `oodleMetricsHost` and `oodleLogsHost` MUST include `https://` scheme

The helm chart constructs URLs by appending paths to these values (e.g., `%{OODLE_METRICS_HOST}/v1/prometheus/.../write`). If the scheme is missing, vmagent will crash with `unsupported scheme` error.

```yaml
# ✅ CORRECT — include https:// prefix
oodleConfig:
  oodleMetricsHost: "https://inst-example.collector.oodle.ai"
  oodleLogsHost: "https://inst-example-logs.collector.oodle.ai"

# ❌ WRONG — missing scheme causes vmagent CrashLoopBackOff
oodleConfig:
  oodleMetricsHost: "inst-example.collector.oodle.ai"
  oodleLogsHost: "inst-example-logs.collector.oodle.ai"
```

### Use `helm search repo` to find chart version — not index.yaml

```bash
# ✅ CORRECT — fast and reliable
helm repo add --force-update oodle https://oodle-ai.github.io/helm-charts
helm search repo oodle/oodle-k8s-observability

# ❌ WRONG — slow, parsing issues with large YAML
curl -s https://oodle-ai.github.io/helm-charts/index.yaml | grep ...
```

### Resolve `instanceId` from `oodle auth status`

The instance ID is available from the CLI's own config:

```bash
# ✅ CORRECT — use oodle auth status (reads from ~/.oodle/config.yaml)
oodle auth status -o json
# Returns: { "instance": "inst-example-abc123", ... }

# Config file location: ~/.oodle/config.yaml
```

## Common Operations

### Listing available integrations

```bash
# ✅ CORRECT — JSON output for parsing
oodle integrations list -o json

# ✅ CORRECT — table output for display
oodle integrations list

# ❌ WRONG — guessing integration types without checking
oodle integrations get-setup-spec some-random-type -o json
```

### Fetching a setup spec

```bash
# ✅ CORRECT — fetch spec for a known type (no auth required)
oodle integrations get-setup-spec kubernetes -o json

# ❌ WRONG — hardcoding setup steps instead of fetching the spec
kubectl apply -f https://some-hardcoded-url/oodle-collector.yaml
```

### Executing setup steps from the spec

```bash
# ✅ CORRECT — show the command and ask for confirmation before running
echo "I will run: kubectl create namespace oodle-system"
# [wait for user confirmation]
kubectl create namespace oodle-system

# ✅ CORRECT — check if resource already exists before creating
kubectl get namespace oodle-system 2>/dev/null && echo "Already exists, skipping" || kubectl create namespace oodle-system

# ❌ WRONG — running destructive commands without confirmation
kubectl delete namespace oodle-system

# ❌ WRONG — running all steps at once without verification
kubectl create namespace oodle-system && helm install ... && kubectl apply ...
```

### Validating the integration

```bash
# ✅ CORRECT — check integration status after setup
oodle integrations list -o json

# ✅ CORRECT — check pods are running (for Kubernetes integrations)
kubectl get pods -n oodle-system

# ❌ WRONG — assuming success without verification
echo "Integration complete!"
```

## Best Practices

### Always fetch the latest setup spec

Run `oodle integrations get-setup-spec <type> -o json` at the start of every onboarding session. Never cache or reuse a previously fetched spec — requirements and steps change as the platform evolves.

### Resolve parameters before executing

Collect all required parameters before starting execution. Check these sources in order:
1. **Environment** — e.g., `kubectl config current-context` for cluster name, env vars for API keys.
2. **User's request** — the user may have specified values in their prompt.
3. **Ask the user** — group remaining questions together rather than asking one at a time.

### Confirm before creating or modifying resources

Show the command you will run and wait for user confirmation before executing any command that creates, modifies, or deletes resources. Read-only commands (list, get, describe) do not need confirmation. **Redact secrets in the displayed command** — show the redacted version for confirmation, then run the unredacted version internally. For commands that require secret values (e.g., `helm install --set apiKey=...`), prefer passing secrets via temp files, stdin, or environment variables rather than command-line arguments to avoid shell history exposure.

### Handle existing resources gracefully

If a resource already exists (namespace, Helm release, ConfigMap), skip the creation step and note it was already present. Do not fail or attempt to delete and recreate.

### Redact secrets in output

Never display full API keys, tokens, or passwords. Show only the first and last 4 characters (e.g., `oodl...k9x2`). When rendering config templates from the spec, redact secret values before showing them to the user.

### One step at a time

Execute setup steps sequentially. Verify each step succeeded before moving to the next. If a step fails, diagnose the issue and suggest a fix — do not continue to the next step.

### Kubernetes Helm values: minikube / small clusters

For minikube or single-node clusters, consider reducing `vmagent.replicaCount` to 1 (default from spec is 2). The default sharding config works but is unnecessary overhead for small clusters.

### Present a summary when done

After all steps complete and validation passes, present a summary:
- Integration type and name
- What was installed/configured
- Current status
- How to check status later (`oodle integrations list`)

## Failure Handling

| Situation | Action |
|-----------|--------|
| `oodle` CLI not installed | Install via `brew install oodle-ai/oodle/oodle` or `go install github.com/oodle-ai/oodle-cli/cmd/oodle@latest` |
| CLI not configured (auth error) | Guide user through `oodle configure` or env vars. For `get-setup-spec`, proceed without auth. |
| Integration type not found (404) | Lowercase the type (e.g., `KUBERNETES` → `kubernetes`). If still failing, run `oodle integrations list` to show available types. |
| Requirement not met (e.g., no kubectl) | Tell user what's missing, provide install instructions, wait for confirmation |
| Setup step fails | Show the error output, diagnose the issue, suggest a fix. Do not continue to the next step. |
| Namespace already exists | Skip creation, note it was already present |
| Helm release already installed | Suggest `helm upgrade` instead of `helm install` |
| Validation fails (integration not active) | Wait 60 seconds and retry. If still failing, check component logs and present diagnostics. |
| Helm `--wait` timeout | Does NOT mean failure. Check `helm list -n <ns>` and `kubectl get pods`. If pods are in CrashLoopBackOff, diagnose logs. If pods are still starting (Pending/ContainerCreating), wait longer. On minikube/resource-constrained clusters, consider `--timeout 10m`. |
| Timeout waiting for pods | Check events with `kubectl describe pod`, present findings to user |
| Permission denied | Tell user which permission is needed and how to grant it |

## References

- [Oodle CLI repo](https://github.com/oodle-ai/oodle-cli) — source of the `oodle` binary
- [Oodle docs](https://docs.oodle.ai) — product documentation
- [oodle-cli skill](../oodle-cli/SKILL.md) — core CLI usage, auth, output formats
