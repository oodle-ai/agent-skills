---
name: oodle-onboarding
description: Integrate infrastructure with the Oodle observability platform — discover available integrations via setup specs, fetch setup instructions, and execute step-by-step installation for any integration type (Kubernetes, AWS, GCP, etc.).
metadata:
  version: "1.0.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,onboarding,integration,kubernetes,observability,setup
  globs: ""
  alwaysApply: "false"
---

# Oodle Onboarding — Integration Setup

This skill teaches the agent to integrate infrastructure with the Oodle observability platform using the `oodle integrations` commands. The agent should actively discover the user's environment to recommend integrations and adapt setup steps. The setup spec is a blueprint — it defines the approach and parameters, but the agent adapts execution based on what it finds in the codebase and environment.

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

**Note:** `oodle integrations list-setup-specs` and `oodle integrations get-setup-spec` do NOT require authentication. Use them to discover available integrations and fetch setup instructions even before the CLI is fully configured. All other integration commands require valid credentials.

## Command Execution Order

Follow this sequence for every integration onboarding request:

1. **Identify the integration type.** If the user named one (e.g., "kubernetes", "aws-cloudwatch"), use it. Otherwise, run environment discovery (see below) to detect what infrastructure is present, then recommend matching integrations.
2. **Fetch available setup specs** — run `oodle integrations list-setup-specs -o json` (no auth required) to confirm the integration type exists and to cross-reference against environment discovery results.
3. **Fetch integration records** — run `oodle integrations list -o json` to extract: (a) the `type` field (lowercase it for step 4), (b) `apiKey`, (c) `collectorDomain`/`logsCollectorDomain` for helm values, (d) instance ID from the domain pattern. This requires authentication — if auth is not configured, guide the user through setup first (see Prerequisites).
4. **Fetch the setup spec** — run `oodle integrations get-setup-spec <type-lowercase> -o json`. This is the **blueprint** for integration, not a rigid script. It defines the general approach, required tools, and parameters.
5. **Discover the environment in depth** — probe the codebase, running services, and existing configs to understand how the user's infrastructure is set up. Use these findings to resolve parameters and adapt the spec's steps (see "Environment Discovery" below).
6. **Collect remaining parameters** — after environment discovery, ask the user for anything that couldn't be auto-detected. Group questions together rather than asking one at a time.
7. **Execute the setup** — work through the spec's steps, adapting each one based on what you discovered. Confirm with the user before every non-read-only command.
8. **Validate the integration** — run `oodle integrations list -o json` and check the status.
9. **Verify data presence on Oodle side** using the query skills (see "Post-Install Validation with Query Skills" below).

## Environment Discovery

When the user hasn't specified an integration type — or even when they have — probe the environment to understand what's already in place. This serves two purposes: (a) recommending the right integration, and (b) adapting setup steps to the actual environment.

### Detecting infrastructure (to recommend integrations)

Run these checks and match results against available setup specs from `list-setup-specs`:

| Signal | What to check | Suggests integration |
|--------|--------------|---------------------|
| Kubernetes | `kubectl cluster-info`, `kubectl config current-context`, presence of `kubeconfig` | `kubernetes` |
| OpenTelemetry Collector | `otelcol --version`, processes matching `otelcol`, `otel-collector` config files in codebase | `otel` |
| AWS | `aws sts get-caller-identity`, `~/.aws/credentials`, env vars `AWS_ACCESS_KEY_ID`/`AWS_REGION` | `aws-cloudwatch` |
| GCP | `gcloud config get-value project`, `~/.config/gcloud/`, env var `GOOGLE_CLOUD_PROJECT` | `gcp-*` |
| Docker / containers | `docker info`, `docker-compose.yml` / `compose.yaml` in codebase | `kubernetes` (if k8s), or container-based integrations |
| Helm | `helm version`, existing helm releases (`helm list -A`) | Kubernetes-based integrations |
| Terraform | `*.tf` files in codebase, `terraform show` | Cloud provider integrations matching the providers in tf files |
| Prometheus | `prometheus --version`, processes matching `prometheus`, prometheus config files | `otel` or `kubernetes` |
| Existing Oodle components | `kubectl get pods -n oodle-system`, `helm list -n oodle-system` | Already partially integrated — check what's missing |

Present your findings to the user: "I detected X, Y, Z in your environment. Based on available setup specs, I recommend the **foo** integration. Shall I proceed?"

### Adapting setup steps (to execute the blueprint)

The setup spec is a blueprint — it tells you **what** needs to happen, not necessarily the exact commands for the user's specific environment. Before executing each step, probe for existing state:

| Before this step... | Check for... |
|---------------------|-------------|
| Create namespace | `kubectl get namespace <name>` — skip if exists |
| Add helm repo | `helm repo list` — skip if already added, use `--force-update` to refresh |
| Install helm chart | `helm list -n <namespace>` — if release exists, use `helm upgrade` instead |
| Create secrets/ConfigMaps | `kubectl get secret/configmap <name> -n <ns>` — skip or update if exists |
| Apply manifests | Check if resources already exist, diff against desired state |
| Set up cloud credentials | Check if credentials already exist in expected locations |
| Configure collector endpoints | Inspect existing collector config files for current endpoint settings |

The spec's parameter values are defaults — prefer values discovered from the environment:
- **Cluster name**: from `kubectl config current-context` or existing helm release values
- **Namespace**: from existing Oodle deployments or user's conventions
- **Collector endpoints**: from existing config files (e.g., `otel-collector-config.yaml`)
- **API keys**: from `oodle integrations list` output or existing secrets
- **Resource sizing**: adapt to cluster size (e.g., minikube gets `replicaCount: 1`)

## Quick Reference

| Task | Command | Auth required? |
|------|---------|----------------|
| Discover available setup specs | `oodle integrations list-setup-specs -o json` | No |
| Discover available setup specs (table) | `oodle integrations list-setup-specs` | No |
| Get setup spec | `oodle integrations get-setup-spec <type> -o json` | No |
| Get setup spec (YAML) | `oodle integrations get-setup-spec <type> -o yaml` | No |
| List configured integrations | `oodle integrations list -o json` | Yes |
| List configured integrations (table) | `oodle integrations list` | Yes |

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

### Discovering available integration types

```bash
# ✅ CORRECT — discover all types with setup specs (no auth required)
oodle integrations list-setup-specs -o json

# ✅ CORRECT — table output for display (no auth required)
oodle integrations list-setup-specs

# ❌ WRONG — guessing integration types without checking
oodle integrations get-setup-spec some-random-type -o json
```

### Listing configured integrations

```bash
# ✅ CORRECT — check which integrations are already configured (requires auth)
oodle integrations list -o json

# ✅ CORRECT — table output for display
oodle integrations list
```

### Fetching a setup spec

```bash
# ✅ CORRECT — fetch spec for a known type (no auth required)
oodle integrations get-setup-spec kubernetes -o json

# ❌ WRONG — hardcoding setup steps instead of fetching the spec
kubectl apply -f https://some-hardcoded-url/oodle-collector.yaml
```

### Executing setup steps (adapting the blueprint)

The spec tells you what needs to happen. Discover the current state before each step and adapt:

```bash
# ✅ CORRECT — check existing state, then adapt
kubectl get namespace oodle-system 2>/dev/null && echo "Already exists, skipping" || kubectl create namespace oodle-system

# ✅ CORRECT — existing release? upgrade instead of install
helm list -n oodle-system | grep oodle && helm upgrade ... || helm install ...

# ✅ CORRECT — show the command and ask for confirmation before running
echo "I will run: kubectl create namespace oodle-system"
# [wait for user confirmation]
kubectl create namespace oodle-system

# ❌ WRONG — blindly running spec commands without checking existing state
kubectl create namespace oodle-system  # fails if already exists

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

## Post-Install Validation with Query Skills

After the integration status shows `RECEIVING`, verify that data is actually queryable on the Oodle side. Use the companion query skills to confirm each signal type that the integration supports.

**Invoke the appropriate skill for each signal the integration covers:**

| Signal | Skill to use | Validation query |
|--------|-------------|-----------------|
| Metrics | `/oodle-metrics-query` | `oodle metrics query --query 'up{cluster="<clusterName>"}' --time now -o json` |
| Metrics (APM/Beyla) | `/oodle-metrics-query` | `oodle metrics query --query 'http_server_request_duration_seconds_count{k8s_namespace_name="<namespace>"}' --time now -o json` |
| Logs | `/oodle-logs` | `oodle logs query -f <ndjson-file> -o json` (query by cluster or namespace) |
| Traces | `/oodle-traces` | `oodle traces list --start -15m --end now --limit 5 -o json` |
| Service Graph | `/oodle-metrics-query` | `oodle metrics query --query 'traces_service_graph_request_total{client="<service>"}' --time now -o json` |

**How to validate from within the onboarding flow:**

After installation completes, run the CLI commands from the table above directly. These are the same commands the query skills use internally — no need to invoke a separate skill during onboarding. The validation queries are simple enough to run inline.

**Validation is successful when:**
- At least one metric series is returned for the cluster
- Logs return non-zero hits (if logs are in the integration's categories)
- Traces show spans from expected services (if traces are in the integration's categories)

**If validation fails but integration shows RECEIVING:**
- Data may still be propagating — wait 2-3 minutes and retry
- Check that the query targets the correct cluster name / namespace / service name
- For logs: check index pattern with `oodle logs index-patterns -o json` first

## Best Practices

### Always fetch the latest setup spec

Run `oodle integrations get-setup-spec <type> -o json` at the start of every onboarding session. Never cache or reuse a previously fetched spec — requirements and steps change as the platform evolves. Treat the spec as a blueprint: it defines what needs to happen and what parameters are required, but adapt the exact commands and values based on environment discovery.

### Resolve parameters by discovery first, ask second

Collect all required parameters before starting execution. Check these sources in order:
1. **Environment discovery** — probe actively: `kubectl config current-context` for cluster name, `helm list` for existing releases, process lists for running collectors, config files in the codebase for endpoints, env vars for API keys.
2. **User's request** — the user may have specified values in their prompt.
3. **Ask the user** — only for what couldn't be auto-detected. Group remaining questions together.

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
| Integration type not found (404) | Lowercase the type (e.g., `KUBERNETES` → `kubernetes`). If still failing, run `oodle integrations list-setup-specs` to show available types (no auth needed). |
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
