# Environment Discovery

Discovery has two purposes: to pick the spec for each signal, and to adapt the spec's steps to what is already there. Do the read-only checks first. They need no confirmation.

## Use the discovery fields of the specs

`oodle integrations list-setup-specs -o json` gives `signals`, `detect` and `related` for each spec. The `detect` entries name files, packages, commands and processes. Search the repo for them, and run the read-only commands. The table below gives the usual checks. The `detect` fields are the most current list.

## Where the apps run

| Signal in the environment | What to check | Spec |
|---|---|---|
| Kubernetes (EKS, GKE, AKS, other) | `kubectl config current-context`, `kubectl get nodes`, Helm charts or manifests in the repo | `kubernetes` |
| Amazon ECS / Fargate | `aws_ecs_task_definition` or `aws_ecs_service` in `*.tf`, `taskdef*.json`, `AWS::ECS::TaskDefinition` in CloudFormation, `ecs.` in CDK code, `aws ecs list-clusters` | `ecs` |
| Convox on ECS | `convox.yml` | `convox_ecs` |
| AWS Lambda | `serverless.yml`, `template.yaml` (SAM), `aws_lambda_function` in `*.tf`, `aws lambda list-functions` | `aws_lambda` |
| VMs, bare metal, Docker Compose | `docker-compose.yml`, `compose.yaml`, systemd units, no orchestrator | `otel` (OTel Collector on the host) |
| GPU nodes | `nvidia-smi`, DCGM exporter, GPU node pools | `gpu_monitoring` |
| Temporal | Temporal SDK in the dependencies, a Temporal server | `temporal` |
| CockroachDB (self-hosted / Cloud) | `cockroach` binary or image / a CockroachDB Cloud cluster | `cockroachdb` / `crdb_cloud` |
| Databases (Postgres, MySQL, others) | database images, connection strings, RDS in `*.tf` | `db_monitoring` |

## Cloud accounts

| Signal | What to check | Spec |
|---|---|---|
| AWS | `aws sts get-caller-identity`, `~/.aws/config`, `provider "aws"` in `*.tf` | `cloudwatch` (metrics, CloudWatch logs) |
| GCP | `gcloud config get-value project`, `provider "google"` | `gcp` |
| Azure | `az account show`, `provider "azurerm"` | `azure_metrics` |
| Confluent Cloud | Confluent provider or cluster ids | `confluent_cloud` |
| Cloudflare | Cloudflare provider, `wrangler.toml` | `logs_cloudflare_logpush` |

## Telemetry that is already there

Prefer the method that sends what is already collected to Oodle as well. Do not replace a customer's agent unless they ask.

| Signal | What to check | Spec |
|---|---|---|
| Datadog Agent or `dd-trace` | `DD_API_KEY`, `datadog-agent` image or chart, `dd-trace` / `ddtrace` packages | `datadog`, `apm_datadog` |
| New Relic agent | `newrelic` packages, `NEW_RELIC_LICENSE_KEY` | `apm_newrelic` |
| OpenTelemetry SDK or Collector | `@opentelemetry/*`, `opentelemetry-*` packages, `otelcol` config files, `OTEL_EXPORTER_OTLP_ENDPOINT` | `otel` |
| Prometheus | `prometheus.yml`, `ServiceMonitor` resources, `/metrics` endpoints | `prometheus` |
| Grafana Alloy / Agent | `config.alloy`, the `grafana/alloy` image | `grafana_alloy` |
| Log shippers | Fluent Bit, Fluentd, Vector, Logstash, Promtail configs | `logs_fluent_bit`, `logs_fluentd`, `logs_vector`, `logs_logstash`, `logs_promtail` (read their `related` first) |
| Jaeger or Zipkin clients | `jaeger-client`, `zipkin` packages, ports 14268, 9411 | `traces_jaeger`, `traces_zipkin` |
| Pyroscope | `pyroscope` packages or agent | `pyroscope` |
| Alertmanager | `alertmanager.yml` | `alertmanager` |

## Application code

| Signal | What to check | Spec |
|---|---|---|
| Browser frontend (React, Next.js, Vue, Angular, plain HTML) | `package.json` with a frontend framework, `index.html` | `rum` |
| LLM calls or agent frameworks | OpenAI, Anthropic, Bedrock, Vertex AI SDKs; LangChain, LangGraph, LlamaIndex, Vercel AI SDK, Pydantic AI, CrewAI, OpenAI Agents SDK, LiteLLM | `llm_observability` (read its `related` for the provider specs) |
| App logs from a logging library | `pino`, `winston`, Python `logging`, Go `slog` and others | `logs_pino`, `otel`, `logs_http` |
| Coding agents used by the team | `~/.claude`, `~/.codex`, `~/.factory`, OpenCode config | `claude_code_observability`, `codex_observability`, `factory_droid_observability`, `opencode_observability` |

## Infrastructure as code and the deploy path

| Signal | What to check |
|---|---|
| Terraform / OpenTofu | `*.tf`, `.terraform.lock.hcl`, the backend block (remote state means a shared apply) |
| CloudFormation / SAM / CDK | `template.yaml`, `cdk.json` |
| Pulumi | `Pulumi.yaml` |
| Helm / Kustomize / Argo CD / Flux | `Chart.yaml`, `kustomization.yaml`, `Application` resources, Flux `Kustomization` resources |
| CI pipeline | `.github/workflows`, `.gitlab-ci.yml`, `Jenkinsfile`, Atlantis or Spacelift config: look for a `plan` / `apply` job |

These decide where each change goes. See `infrastructure-as-code.md`.

## Existing Oodle components

`oodle integrations list -o json` shows what is configured already. Also check `helm list -A | grep -i oodle` and Oodle hosts (`*.collector.oodle.ai`) in the config files. Add only what is missing.

## Adapt each step to the existing state

Before each step, check what is there:

| Before this step | Check |
|---|---|
| Make a namespace | `kubectl get namespace <name>`. Skip it if it exists. |
| Add a Helm repo | `helm repo list`. Use `--force-update` to refresh it. |
| Install a chart | `helm list -n <namespace>`. If the release exists, the command you give the user is `helm upgrade`, not `helm install`. |
| Make a Secret or ConfigMap | `kubectl get secret/configmap <name> -n <ns>`. Update it or skip it. |
| Add a sidecar or log router | Look for one in the task definition or pod spec. Update it. Do not add a second one. |
| Change a collector config | Read the current config. Add an exporter and a pipeline entry. Keep the current ones. |

## Resolve parameters

Prefer values from the environment to the spec's defaults:

- **Cluster name**: `kubectl config current-context`, or the values of an existing release
- **Service names and environments**: the names in the manifests, task definitions or `OTEL_SERVICE_NAME`
- **Region and account**: `aws configure get region`, the provider block
- **Sizing**: the cluster size (for example, minikube gets `replicaCount: 1`)
