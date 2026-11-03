# Environment Discovery

Probe the environment to understand what infrastructure is already in place. This serves two purposes: (a) recommending the right integration, and (b) adapting setup steps to the actual environment.

## Detecting Infrastructure

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

Present the findings to the user: "I detected X, Y, Z in the environment. Based on available setup specs, I recommend the **foo** integration." Put this in the question round of step 5, with the other open questions.

## Adapting Setup Steps

The setup spec is a blueprint — it defines **what** needs to happen, not necessarily the exact commands for the specific environment. Before you change the code for a step, read the existing state (read-only), so that the change and the deploy command you hand over fit it:

| For this step... | Read... | So that... |
|---------------------|-------------|-------------|
| A namespace | `kubectl get namespace <name>` | the deploy command has `--create-namespace` only when it is missing |
| A Helm repo | `helm repo list` | the user's commands add it only when it is missing |
| A Helm chart | `helm list -n <namespace>`, `helm get values <release> -n <namespace>` | an existing release gets `helm upgrade` with its current values kept |
| Secrets/ConfigMaps | `kubectl get secret/configmap <name> -n <ns>` | you reuse an existing Secret instead of asking for a new one |
| Manifests | the files in the repo and the live resources | the change edits the existing resource, not a second copy |
| Cloud credentials | the expected locations | you ask only for what is missing |
| Collector endpoints | the existing collector config files | you add the Oodle exporter to the running collector |

## Parameter Resolution

The spec's parameter values are defaults — prefer values discovered from the environment:

- **Cluster name**: from `kubectl config current-context` or existing helm release values
- **Namespace**: from existing Oodle deployments or the user's conventions
- **Collector endpoints**: from existing config files (e.g., `otel-collector-config.yaml`)
- **API keys**: from `oodle api-keys list -o json` output, or create one with `oodle api-keys create --name <name> --scopes <scopes> -o json`
- **Resource sizing**: adapt to cluster size (e.g., minikube gets `replicaCount: 1`)
