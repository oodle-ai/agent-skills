# Failure Handling

| Situation | Action |
|-----------|--------|
| `oodle` CLI not installed | Install via `brew install oodle-ai/oodle/oodle` or `go install github.com/oodle-ai/oodle-cli/cmd/oodle@latest` |
| CLI not configured (auth error) | Guide user through `oodle configure` or env vars. For `get-setup-spec` and `list-setup-specs`, proceed without auth. |
| Integration type not found (404) | Lowercase the type (e.g., `KUBERNETES` → `kubernetes`). If still failing, run `oodle integrations list-setup-specs` to show available types (no auth needed). |
| Requirement not met (e.g., no kubectl) | Tell user what's missing, provide install instructions, wait for confirmation |
| Setup step fails | Show the error output, diagnose the issue, suggest a fix. Do not continue to the next step. |
| Namespace already exists | Skip creation, note it was already present |
| Helm release already installed | Suggest `helm upgrade` instead of `helm install` |
| Validation fails (integration not active) | Wait 60 seconds and retry. If still failing, check component logs and present diagnostics. |
| Helm `--wait` timeout | Does NOT mean failure. Check `helm list -n <ns>` and `kubectl get pods`. If pods are in CrashLoopBackOff, diagnose logs. If pods are still starting (Pending/ContainerCreating), wait longer. On minikube/resource-constrained clusters, consider `--timeout 10m`. |
| Timeout waiting for pods | Check events with `kubectl describe pod`, present findings to user |
| Permission denied | Tell user which permission is needed and how to grant it |
