# Infrastructure as Code and Deploy Paths

Most customer systems are deployed from code. A setup that changes the live resources and not the code is lost at the next deploy, and it can break the next `terraform apply`. Put each change where the customer's system is defined.

## Find out how changes reach production

Look first, then ask one question if it is not clear:

- A CI job that runs `terraform plan` / `apply`, Atlantis, Spacelift, Terraform Cloud, `cdk deploy` in a workflow, or Argo CD / Flux that syncs a repo: **the pipeline applies.**
- No such job, and the state is local or the customer runs the tool by hand: **a person applies.**
- Both: ask which one applies to the resources you will change.

Ask: "I will change `<files>`. Do you apply these through `<pipeline>`, or locally?"

## When a pipeline applies

1. Make the change in the code, on a new branch.
2. Run the checks that need no credentials: `terraform fmt`, `terraform validate`, `helm template`, `cdk synth`.
3. Open a PR, or give the diff if you cannot open one. Say what the change does and how to roll it back.
4. Stop and tell the customer what to do next ("merge the PR, and tell me when the deploy is done"). Validate the signal after they confirm.

Do not apply from the laptop at the same time as the pipeline. Two applies on the same state can conflict, and a local apply can use different variables.

## When a person applies

1. Make the change in the code.
2. Run the plan (`terraform plan`, `cdk diff`, `helm diff upgrade` or `helm template`) and show the summary: what is added, changed and removed.
3. If the plan changes or removes resources that are not part of this setup, stop and show them. This is drift or other work in progress, and it is not your change to apply.
4. Apply only after the customer confirms.

## Where the Oodle settings go

- **Terraform**: put the Oodle values in variables (`oodle_api_key` with `sensitive = true`, `oodle_logs_host`). Set the key from the customer's existing method for secret values (`TF_VAR_oodle_api_key`, a `*.tfvars` file that git ignores, or a CI variable). Do not write the key into a `.tf` file.
- **ECS task definitions in Terraform**: they are usually `container_definitions = jsonencode([...])` or a templated JSON file. Edit the container list there. A new task definition revision is made at apply, and the service rolls to it.
- **Helm**: put the values in the values file that the customer's release uses. Put the key in an existing Secret, or in the Secret that the spec says.
- **CloudFormation / CDK / Pulumi**: make the same change in the template or the code, with a parameter or a config value for the key.

## Use a cloud CLI only when no code manages the resource

If no code manages the resource (the console made it, for example), a CLI command is acceptable after the customer agrees. Tell the customer that the resource is not in code, so that they can import it later.
