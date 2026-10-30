# Failure Handling

| Situation | Action |
|-----------|--------|
| Integration type not found (404) | Lowercase the type (e.g., `KUBERNETES` → `kubernetes`). If still failing, run `oodle integrations list-setup-specs` to show available types (no auth needed). |
| Requirement not met (e.g., no kubectl) | Tell user what's missing, provide install instructions, wait for confirmation |
| Setup step fails | Show the error output, diagnose the issue, suggest a fix. Do not continue to the next step. |
| Validation fails (no data in Oodle) | Follow "If data does not arrive" in `post-install-validation.md`. Pull integrations can take about 15 minutes. |
| Helm `--wait` timeout | Does NOT mean failure. Check `helm list -n <ns>` and `kubectl get pods`. If pods are in CrashLoopBackOff, diagnose logs. If pods are still starting (Pending/ContainerCreating), wait longer. On minikube/resource-constrained clusters, consider `--timeout 10m`. |
| Timeout waiting for pods | Check events with `kubectl describe pod`, present findings to user |
| Permission denied | Tell user which permission is needed and how to grant it |
| The spec has no method for this environment | Tell the customer, and suggest the nearest method in the spec. Do not make a new method. |
| The customer's plan or deploy fails for a cause that is not this setup | Stop. Show the cause. Do not fix other resources or apply around the pipeline. |
| The spec says to use a value that the customer rejects (for example, a key in an environment variable) | Use the customer's choice if their code already supports it (for example, a secret store that the task already reads). Say what changes. |
