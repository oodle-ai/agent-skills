---
name: oodle-onboarding
description: This skill should be used when the user asks to "set up oodle integration", "onboard to oodle", "integrate kubernetes with oodle", "connect AWS to oodle", "send logs/metrics/traces to oodle", "add RUM", "instrument my LLM app / agent", "install oodle collector", or mentions setting up observability with Oodle. Discovers the environment, picks the matching setup specs for each signal, and executes step-by-step installation in the customer's own deployment path. Not for querying existing metrics, logs, or traces (use /oodle-metrics-query, /oodle-logs, /oodle-traces instead).
version: 1.2.0
---

# Oodle Onboarding: Integration Setup

Connect a customer's apps and infrastructure to Oodle with `oodle integrations` commands and the setup specs they serve.

**The setup spec decides the method. The customer decides the scope. The customer deploys.** Oodle knows which collector, image, config and key handling work for each platform, and the spec holds those decisions. The customer knows which services, environments and deploy path to use. Ask the customer only about their part. Every extra question, and every path that is not in the spec, makes the setup slower and more fragile. You change code and config; you never apply or deploy them (see "Change code, do not deploy").

## Prerequisites

1. **CLI installed and current:** `oodle integrations get-setup-spec --help` must show `--summary` and `--method`. If `oodle` is not found, or the flags are not there, install or update it: `brew upgrade oodle-ai/oodle/oodle` (or `brew install`), or without brew `curl -fsSL https://www.oodle.ai/skills.sh | bash -s -- --no-onboarding`.
2. **CLI logged in:** `oodle auth status -o json`. If it is not logged in, run `oodle auth login`. An expired OAuth login does not matter when an API key is set (`OODLE_API_KEY` or the config): log in again only when a command fails with 401. This opens a browser. Tell the user to complete the login, and wait for their confirmation.

With a login, `get-setup-spec` fills in the values of the instance (instance ID, API URL, collector domains). Without one, those stay as `{{placeholders}}`.

## The loop

1. **Discover.** Read the repo and run read-only commands to make an inventory: where the apps run, the languages and frameworks, any frontend, LLM SDKs and agent frameworks, the telemetry that is already there, the cloud accounts, and the infrastructure as code with its deploy path. `references/environment-discovery.md` has the checks. If the user named one integration, discover only what that integration needs.
2. **Pick a spec for each signal.** Run `oodle integrations list-setup-specs -o json`. Each entry has `type`, `description`, `signals`, `detect` and `related`. For each signal the environment can give (metrics, logs, traces, profiles, RUM, agent traces), pick the spec whose `detect` entries match:
   - **A platform spec before a general one.** If the user names a tool (FireLens, Fluent Bit, OTel), first look for the spec of their platform (`ecs`, `kubernetes`, `aws_lambda`). It has the hosted configs, images and task changes. The general spec does not.
   - **Follow `related`.** A spec that names a better spec for this environment sends you there.
   - **Keep what is there.** If an agent runs already (for example, the Datadog Agent), use the method that sends its data to Oodle as well. Do not replace it unless the user asks.
   - **Cover every component.** Each item of the inventory (a database, a frontend, a queue, an LLM call) gets a spec, or a line in the summary that no spec covers it. Say that only after you compared it with the `detect` entries of every spec, not from the description alone.
3. **Pick the method.** Run `oodle integrations get-setup-spec <type> --summary -o json`. If the spec has a `decisionTree`, read its rules in order and use the method of the first rule whose `if` is true for this environment. A rule with `also: true` adds its method, and you continue to the next rule. The last rule has no `if`: it is the default. Check each `if` against the inventory; ask the user only when the repo cannot show it. A spec with one method has no tree. Do not make a method that is not in the spec. In the plan, give the rule that picked the method.
4. **Present one plan, and ask one round of questions.** Show a table: signal, spec, method, and the file or resource that changes. Fill in every default from the spec. Then ask, in one message, only what the user must decide (see "Questions"). Example: "I will add FireLens with the Oodle-hosted Fluent Bit config to the `api` and `worker` task definitions in `infra/ecs.tf`, with the API key as an environment variable. I will only change the files. You apply them with your pipeline, and then I check that the logs arrive. Change anything?"
5. **Get the steps.** Run `oodle integrations get-setup-spec <type> --method <method> -o json`. Fetch it in each session, because specs change. Then fill the placeholders that are left:
   - `{{apiKey}}`: the `token` of a key in `oodle api-keys list -o json`. The CLI does not fill it, because it is a secret. Put it where the step says, which is usually an environment variable. Never print it in full.
   - Every other `{{name}}`: a value of the user's system. Its source is in `requirements.templateVariables`. Find it in the inventory, or ask in step 4.
   - Placeholders in other syntaxes (`${{ secrets.X }}`, `{{.Comm}}`, `${VAR}`) belong to the user's tools. Keep them.
6. **Change the code, not the system.** Edit the files that define the system: app code, IaC, Helm values, task definitions, config files. Read `references/infrastructure-as-code.md` before you change Terraform, CloudFormation, CDK, Pulumi, Helm or Kustomize, and `references/gotchas.md` before you run anything. Run only the local checks that need no apply (`terraform validate`, `helm template`, `cdk synth`, a build, a test).
7. **Hand over the deploy.** Show the changed files and the exact commands that apply them (`terraform apply`, `helm upgrade --install`, a merge that starts the pipeline, a restart of the app). The spec steps that deploy something go in this list: do not run them. Tell the user to deploy and to tell you when it is done. Only for a local, throwaway target can you run the deploy yourself, when the user asks (see "Change code, do not deploy").
8. **Verify each signal** after the user says that the deploy is done. Run the `validation.checks` of the method, then `references/post-install-validation.md`. A signal is done only when its data is in Oodle.
9. **Summarize.** For each signal: the status, what changed (files), what the user must still deploy, what is not done and why, and how to check later (`oodle integrations list`).

## Questions

Ask at most three questions, and only about what the spec and the environment cannot tell you:

- which services, clusters, environments or accounts to cover
- how they deploy changes (a pipeline or a local apply), if the repo does not show it
- approval to change shared resources, such as a production task definition or an IAM role

Everything else is an **assumption** in the plan, not a question: "I assume that the tasks have outbound HTTPS. Say if not." Take the spec's value for what it decides (the collector, the image, the config source, how to pass the key, ports and endpoints), and say which value you took. Detect what you can (the architecture, the region, the runtime) from the files. A value that only the user has (a vendor console key, an account ID with no CLI access) is a question only when a step cannot start without it.

Ask all questions in one message. If you must ask again later, say why the first answers were not sufficient.

## Rules

### Stay on the spec

- Use the values in the spec: hosted config URLs, images, endpoints, and the key as an environment variable. Do not add a secret store, a new bucket, a new image or a different agent unless the customer asks for it.
- If the spec cannot do what the customer needs, tell the customer. Then suggest the nearest method in the spec. Do not invent a new method.
- If a spec step says the work happens in the Oodle UI or in a vendor console, tell the customer the exact place and what to enter. Then continue with the steps you can do.

### Change code, do not deploy

The customer owns their deploys: their pipeline, their approvals, their maintenance windows, their rollback. A deploy from the agent goes around all of them, and it is the step that can break production.

- Change files only. Do not run a command that changes a running system or a cloud account: `terraform apply`, `pulumi up`, `cdk deploy`, `aws cloudformation deploy`, `helm install` / `upgrade`, `kubectl apply` / `patch` / `rollout restart`, `aws` / `gcloud` / `az` commands that create, update or delete, `docker push`, `docker compose up` on a server, a service restart.
- This is also true when a spec step gives such a command, and when the customer has no IaC. Give the command to the customer to run.
- Do not commit, push or open a PR unless the customer asks. Tell them which files changed.
- You can run read-only commands (list, get, describe, `terraform plan`, `helm template`) and local checks (build, test, lint, `validate`).
- Changes in the Oodle account with the `oodle` CLI (an API key, `import-assets`, `s3-logs add`) are not a deploy of the customer's system. Show each one and run it after the customer agrees.
- One exception: a local, throwaway target. When the customer asks you to start it, you can run `docker compose up` on their own machine, or `helm` / `kubectl` on a local kind, minikube, k3d or Docker Desktop cluster. Before you run a cluster command, run `kubectl config current-context` and make sure that it names the local cluster. Nobody else uses these targets, and the customer can delete them, so a fast first check is worth more than a handover. Never do this for a shared or production target, for `terraform apply` or other IaC applies, or for cloud CLI changes, also when the customer asks: give those commands to the customer.

### Change the system of record

If infrastructure as code manages a resource, change the code. Do not give a cloud CLI command for it: a direct change makes drift, and the next `terraform apply` removes it. Give a cloud CLI command only for a resource that no code manages, and say that it is not in code.

### Confirm before changing anything

Show each file change and each `oodle` command that changes the Oodle account, and wait for confirmation. Read-only commands do not need confirmation.

### Keep existing state

If a resource already exists (a namespace, a Helm release, a sidecar, a collector config), update it or skip it. Do not delete it and make it again. Merge into existing config files. Do not overwrite them.

### Secrets

- Show only the first and last 4 characters of a key (for example, `oodl...k9x2`).
- Put a key where the spec says, which is usually an environment variable. If the customer's code already reads secrets from a store, you can use that store. Ask first.
- Do not write a key into a file that goes into git. Use the customer's existing method for secret values (a `.env` file in `.gitignore`, a CI variable, a Terraform variable).

### One step at a time

Do the steps in sequence. Make sure that each step is successful before you start the next one. If a step fails, find the cause and suggest a fix. Do not continue past a failed step. When the customer reports that a deploy failed, read the error and fix the files; the customer deploys again.

## Quick Reference

| Task | Command | Login? |
|------|---------|--------|
| List specs, with discovery fields | `oodle integrations list-setup-specs -o json` | No |
| Methods of a spec | `oodle integrations get-setup-spec <type> --summary -o json` | No |
| Steps of one method, values filled in | `oodle integrations get-setup-spec <type> --method <method> -o json` | No (values need it) |
| List configured integrations | `oodle integrations list -o json` | Yes |
| List API keys | `oodle api-keys list -o json` | Yes |
| Make an API key | `oodle api-keys create --name <name> --scopes <scopes> -o json` | Yes |
| Import the Oodle dashboards and monitors of an integration (safe to run again) | `oodle integrations import-assets <name-or-type> [--dashboards] [--monitors] -o json` | Yes (Editor or Admin) |
| Make the S3 logs record and its CloudFormation values | `oodle integrations s3-logs add --region <region> --bucket-patterns '<patterns>' [--sns-topic-arn <arn>] -o json` | Yes (Editor or Admin) |

Aliases: `oodle integration`, `oodle integ`

```bash
# CORRECT: choose from the list, with its discovery fields
oodle integrations list-setup-specs -o json

# WRONG: guess a type
oodle integrations get-setup-spec some-random-type -o json

# WRONG: write a manifest from memory instead of the spec
kubectl apply -f <hardcoded-collector-manifest>
```

## References

- `references/environment-discovery.md`: environment checks and the spec for each environment
- `references/infrastructure-as-code.md`: changes to Terraform, CloudFormation, CDK, Pulumi, Helm, and deploy pipelines
- `references/gotchas.md`: gotchas to read before you run setup steps
- `references/post-install-validation.md`: how to make sure that data arrives, for each signal
- `references/failure-handling.md`: what to do when a step fails
- [Oodle CLI repo](https://github.com/oodle-ai/oodle-cli)
- [Oodle docs](https://docs.oodle.ai)
- [oodle-cli skill](../oodle-cli/SKILL.md): CLI use, login, output formats
