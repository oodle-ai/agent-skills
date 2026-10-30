---
name: oodle-onboarding
description: This skill should be used when the user asks to "set up oodle integration", "onboard to oodle", "integrate kubernetes with oodle", "connect AWS to oodle", "install oodle collector", or mentions setting up observability with Oodle. Discovers the environment, recommends matching integrations from available setup specs, and executes step-by-step installation. Not for querying existing metrics, logs, or traces (use /oodle-metrics-query, /oodle-logs, /oodle-traces instead).
version: 1.0.0
---

# Oodle Onboarding — Integration Setup

Integrate infrastructure with the Oodle observability platform using `oodle integrations` commands. Actively discover the environment to recommend integrations and adapt setup steps. The setup spec is a blueprint — it defines the approach and parameters, but execution adapts based on what the codebase and environment reveal.

## Prerequisites

Before starting any onboarding work, verify the Oodle CLI is installed and authenticated. Run these checks at the beginning of every session:

### Step 1: Check if the CLI is installed

```bash
oodle version
```

If the command is not found, install it:

```bash
curl -fsSL https://www.oodle.ai/skills.sh | bash -s -- --no-onboarding
```

Or with Homebrew:

```bash
brew install oodle-ai/oodle/oodle
```

If `brew` is not available, fall back to:

```bash
go install github.com/oodle-ai/oodle-cli/cmd/oodle@latest
```

### Step 2: Check if the CLI is authenticated

```bash
oodle auth status
```

If not authenticated or the status shows an error, run interactive login:

```bash
oodle auth login
```

This opens a browser for OAuth login. The user must complete the browser flow — prompt them to do so and wait for confirmation. After login succeeds, verify with `oodle auth status`.

**Note:** `oodle integrations list-setup-specs` and `oodle integrations get-setup-spec` do NOT require authentication. Use these to discover available integrations and fetch setup instructions even before auth is configured. All other integration commands require valid credentials — complete the auth steps above before proceeding past step 2 in the Command Execution Order.

## Command Execution Order

Follow this sequence for every integration onboarding request. You change the customer's code and config; the customer deploys it.

1. **Identify the integration type.** If the user named one (e.g., "kubernetes", "aws-cloudwatch"), use it. Otherwise, run environment discovery to detect present infrastructure and recommend matching integrations (see `references/environment-discovery.md`).
2. **Fetch available setup specs** — run `oodle integrations list-setup-specs -o json` (no auth required) to confirm the type exists and cross-reference against discovery results.
3. **Fetch the setup spec** — run `oodle integrations get-setup-spec <type-lowercase> -o json`. When the CLI is logged in, it fills the Oodle values of the instance (instance ID, collector domains); the `filledValues` field names them. Only `{{apiKey}}` and the values of the customer's own system stay placeholders. For the API key, run `oodle api-keys list -o json`, or `oodle api-keys create --name <name> --scopes <scopes>` when none fits. The spec is the **blueprint**, not a rigid script.
4. **Discover the environment in depth** — probe the codebase, running services, existing configs, and how changes reach production (IaC, pipeline, Helm release). See `references/environment-discovery.md` and `references/infrastructure-as-code.md`.
5. **Ask every open question before you change anything.** List each decision that the repo does not settle and each value you could not find. Ask them all in one message, with your recommended answer for each. If an answer opens new questions, ask those in the next round. Stop when nothing is open. Then show the plan (what changes, in which files) and wait for the user to agree. See "Questions" below.
6. **Change the code, not the system** — edit the files that define the system: app code, IaC, Helm values, task definitions, compose files. Follow the spec's steps and adapt them to what discovery found. Run only checks that change nothing (`terraform validate`, `helm template`, `cdk synth`, a build, a test). Review `references/gotchas.md` first.
7. **Hand over the deploy** — show the changed files and the exact commands that apply them (`terraform apply`, `helm upgrade --install ...`, a merge that starts the pipeline). Tell the user to deploy and to tell you when it is done.
8. **Verify each signal** after the user says the deploy is done: run a query that returns the new data (see `references/post-install-validation.md`). A signal is done only when its data is in Oodle.

For failure scenarios, consult `references/failure-handling.md`.

## Questions

Ask once, early and completely, so the user (or the person beside them in a meeting) can answer everything in one go:

- **A choice between tools the customer already runs is theirs.** For example: keep Datadog and also send to Oodle, or move off it; keep Loki or Elasticsearch next to Oodle, or replace it; OpenTelemetry or the existing agent. Never pick one in silence.
- **Scope**: which services, clusters, environments (production, staging) and accounts.
- **Values** that discovery did not find, such as a cluster name or a log group.
- **Deploy path**: whether a pipeline or a person applies the change, if the repo does not show it.
- Do not ask what the spec, the CLI or the repo already answers.

## Change code, do not deploy

The customer owns their deploys: their pipeline, approvals and rollback. A deploy from the agent goes around them, and it is the step that can break production.

- Do not run a command that changes a running system or a cloud account: `terraform apply`, `pulumi up`, `cdk deploy`, `helm install` / `upgrade`, `kubectl apply`, cloud CLI commands that create, update or delete, a service restart. This is also true when a spec step gives such a command: give it to the user.
- If infrastructure as code manages a resource, change the code. A cloud CLI change next to IaC is drift, and the next apply removes it.
- Do not commit, push or open a PR unless the user asks.
- Read-only commands are fine. A change in the Oodle account (creating an API key, importing dashboards) needs the user's agreement first.
- Never create, change or delete anything to find out how a command behaves, not even to delete it again. Read `--help` (for example `oodle api-keys create --help` for the roles and scopes) or the spec instead.
- For a new API key, propose the least role that the spec names (an ingest key for an app or a collector, a RUM key for browser code) and create it only after the user agrees.
- One exception: when the user asks, you can start a local, throwaway target yourself (`docker compose up` on their machine, or a local kind or minikube cluster after `kubectl config current-context` names it).

## Quick Reference

| Task | Command | Auth required? |
|------|---------|----------------|
| Discover available setup specs | `oodle integrations list-setup-specs -o json` | No |
| Discover available setup specs (table) | `oodle integrations list-setup-specs` | No |
| Get setup spec | `oodle integrations get-setup-spec <type> -o json` | No |
| Get setup spec (YAML) | `oodle integrations get-setup-spec <type> -o yaml` | No |
| List configured integrations | `oodle integrations list -o json` | Yes |
| List configured integrations (table) | `oodle integrations list` | Yes |
| List API keys | `oodle api-keys list -o json` | Yes |
| Create API key | `oodle api-keys create --name <name> --scopes <scopes> -o json` | Yes |

Aliases: `oodle integration`, `oodle integ`

## Common Operations

### Discovering available integration types

```bash
# CORRECT — discover all types with setup specs (no auth required)
oodle integrations list-setup-specs -o json

# CORRECT — table output for display (no auth required)
oodle integrations list-setup-specs

# WRONG — guessing integration types without checking
oodle integrations get-setup-spec some-random-type -o json
```

### Listing configured integrations

```bash
# CORRECT — check which integrations are already configured (requires auth)
oodle integrations list -o json

# CORRECT — table output for display
oodle integrations list
```

### Fetching a setup spec

```bash
# CORRECT — fetch spec for a known type (no auth required)
oodle integrations get-setup-spec kubernetes -o json

# WRONG — hardcoding setup steps instead of fetching the spec
kubectl apply -f <hardcoded-collector-manifest>   # never hardcode a manifest source; fetch the spec instead
```

### Changing the code (adapting the blueprint)

The spec defines what needs to happen. Read the current state, then change the files that define it:

```bash
# CORRECT — read the current state (read-only)
helm list -A | grep oodle
helm get values oodle-observability -n oodle-monitoring   # keep the user's values when you write the new file

# CORRECT — change the values file in the repo, check it, and give the user the command
helm template oodle-observability oodle/oodle-k8s-observability --values deploy/oodle-values.yaml > /dev/null
echo "Run: helm upgrade --install oodle-observability oodle/oodle-k8s-observability -n oodle-monitoring --values deploy/oodle-values.yaml"

# WRONG — applying the change yourself
helm upgrade --install oodle-observability ...

# WRONG — a cloud CLI change for a resource that Terraform manages
aws logs put-subscription-filter ...   # change the .tf file instead
```

### Validating the integration

```bash
# CORRECT — a query that returns the new data
oodle metrics query --query 'count(up{cluster="<clusterName>"})' --time now -o json

# NOT ENOUGH — the status can show NOT_RECEIVING for a minute or two after data arrives
oodle integrations list -o json

# WRONG — assuming success without verification
echo "Integration complete!"
```

## Best Practices

### Always fetch the latest setup spec

Run `oodle integrations get-setup-spec <type> -o json` at the start of every onboarding session. Never cache or reuse a previously fetched spec — requirements and steps change as the platform evolves. Treat the spec as a blueprint: it defines what needs to happen and what parameters are required, but adapt the exact commands and values based on environment discovery.

### Resolve parameters by discovery first, ask second

Collect all required parameters before starting execution. Check these sources in order:
1. **Environment discovery** — probe actively: `kubectl config current-context` for cluster name, `helm list` for existing releases, process lists for running collectors, config files in the codebase for endpoints, env vars for API keys.
2. **The user's request** — the user may have specified values in the prompt.
3. **Ask the user** — only for what could not be auto-detected, in the question round of step 5.

### Keep secrets out of files and commands

Do not write the API key into a file that git tracks. Use the customer's existing way to pass secrets (a Kubernetes Secret, `TF_VAR_*`, a CI variable, a `.env` file that git ignores). In a command you give the user, use an environment variable for the key, not the value.

### Handle existing resources gracefully

If a resource already exists (a Helm release, a log router, a collector), change it instead of adding a second one, and keep the user's settings. Do not delete and recreate.

### Redact secrets in output

Never display full API keys, tokens, or passwords. Show only the first and last 4 characters (e.g., `oodl...k9x2`). When rendering config templates from the spec, redact secret values before showing to the user.

### When the user's deploy fails

Read the error the user gives you, fix the files, and give the deploy command again. Do not continue to the next signal until this one works.

### Present a summary when done

After all steps complete and validation passes, present a summary:
- Integration type and name
- The files that changed, and what the user must still deploy
- Each signal: verified by a query, or not yet
- How to check status later (`oodle integrations list`)

## References

- `references/environment-discovery.md` — infrastructure detection and setup step adaptation
- `references/infrastructure-as-code.md` — where a change goes when Terraform, CDK, Helm or a pipeline manages the system
- `references/gotchas.md` — critical gotchas to review before executing setup
- `references/post-install-validation.md` — verifying data presence with query skills
- `references/failure-handling.md` — troubleshooting common failure scenarios
- [Oodle CLI repo](https://github.com/oodle-ai/oodle-cli) — source of the `oodle` binary
- [Oodle docs](https://docs.oodle.ai) — product documentation
- [oodle-cli skill](../oodle-cli/SKILL.md) — core CLI usage, auth, output formats
