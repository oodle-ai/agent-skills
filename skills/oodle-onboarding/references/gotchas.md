# Critical Gotchas

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

## Kubernetes Helm values: minikube / small clusters

For minikube or single-node clusters, reduce `vmagent.replicaCount` to 1 (default from spec is 2). The default sharding config works but adds unnecessary overhead for small clusters.

## REST registration steps need an Admin key

Some specs register the integration with a REST call to `/v1/api/instance/<id>/integrations` (GCP, Confluent Cloud, CockroachDB Cloud, MCP servers). A data ingestion key gets HTTP 403 there. Use a key with the Admin role, and only for that call.

## Pull integrations take time to show data

Cloud integrations that Oodle pulls (CloudWatch metrics, GCP, Azure, Confluent Cloud) can take about 15 minutes to show `RECEIVING`. Do not change the setup while you wait.

## Keys in browser code

RUM code runs in the browser, so its key is public. Use only the RUM key there, never an API key that can write other data. Public build variables (`NEXT_PUBLIC_*`, `VITE_*`) go into the bundle at build time, so set them in the CI build too, not only on the laptop.

## Changing the agent host stops the old destination

Methods that point an existing agent at Oodle as its only host (Datadog single write, New Relic `NEW_RELIC_HOST`) stop the data to the old vendor. Say so in the plan. Prefer the dual write method when the customer still uses the old vendor.

## An alert destination needs a route

A notifier gets no alerts until a notification policy or a monitor names it. After you make a notifier, add the policy that the spec gives, and send a test notification.

## MCP specs configure two different things

`mcp_claude`, `mcp_cursor`, `mcp_codex` and `mcp_generic` connect the user's own coding agent to Oodle. `mcp_github` and `external_mcp` give tools to Oodle's AI Assistant. Do not mix them.

## Merge coding agent config files

Claude Code `settings.json`, Codex `config.toml`, OpenCode `opencode.json` and MCP config files hold the user's other settings. Keep a `.bak` copy, merge the Oodle keys in, and never write a second `[otel]` table or a second `mcpServers` object.

## Agent Observability: one tracer provider

If the app already sets up OpenTelemetry, add the Oodle exporter to its tracer provider. A second tracer provider sends spans twice or drops them. Pick the instrumentation from the table in the `llm_observability` spec. Do not install a provider SDK (for example, Langfuse) into an app that does not use it.

