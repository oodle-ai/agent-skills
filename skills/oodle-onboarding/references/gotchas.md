# Critical Gotchas

## Integration type must be lowercase in `get-setup-spec`

The `type` field in `integrations list` is UPPERCASE (e.g., `KUBERNETES`, `OTEL`) but `get-setup-spec` requires **lowercase** (e.g., `kubernetes`). Always lowercase the type before calling `get-setup-spec`.

## Helm values: `oodleMetricsHost` and `oodleLogsHost` MUST include `https://` scheme

The helm chart constructs URLs by appending paths to these values (e.g., `%{OODLE_METRICS_HOST}/v1/prometheus/.../write`). If the scheme is missing, vmagent will crash with `unsupported scheme` error.

```yaml
# CORRECT — include https:// prefix
oodleConfig:
  oodleMetricsHost: "https://inst-example.collector.oodle.ai"
  oodleLogsHost: "https://inst-example-logs.collector.oodle.ai"

# WRONG — missing scheme causes vmagent CrashLoopBackOff
oodleConfig:
  oodleMetricsHost: "inst-example.collector.oodle.ai"
  oodleLogsHost: "inst-example-logs.collector.oodle.ai"
```

## Use `helm search repo` to find chart version — not index.yaml

```bash
# CORRECT — fast and reliable
helm repo add --force-update oodle https://oodle-ai.github.io/helm-charts
helm search repo oodle/oodle-k8s-observability

# WRONG — slow, parsing issues with large YAML
curl -s https://oodle-ai.github.io/helm-charts/index.yaml | grep ...
```

## Resolve `instanceId` from `oodle auth status`

The instance ID is available from the CLI's own config:

```bash
# Use oodle auth status (reads from ~/.oodle/config.yaml)
oodle auth status -o json
# Returns: { "instance": "inst-example-abc123", ... }

# Config file location: ~/.oodle/config.yaml
```

## Kubernetes Helm values: minikube / small clusters

For minikube or single-node clusters, reduce `vmagent.replicaCount` to 1 (default from spec is 2). The default sharding config works but adds unnecessary overhead for small clusters.
