# Failure Handling

| Situation | Action |
|-----------|--------|
| `oodle` CLI not installed | Install via `brew install oodle-ai/oodle/oodle`. If brew unavailable, use `go install github.com/oodle-ai/oodle-cli/cmd/oodle@latest`. |
| CLI not authenticated | Run `oodle auth login` — this opens a browser for OAuth. Prompt the user to complete the browser flow. Verify with `oodle auth status`. For `get-setup-spec` and `list-setup-specs`, proceed without auth. |
| Integration type not found (404) | Lowercase the type (e.g., `KUBERNETES` → `kubernetes`). If still failing, run `oodle integrations list-setup-specs` to show available types (no auth needed). |
| Requirement not met (e.g., no kubectl) | Tell user what's missing, provide install instructions, wait for confirmation |
| Setup step fails | Show the error output, diagnose the issue, suggest a fix. Do not continue to the next step. |
| The user's deploy fails | Read the error the user gives you, fix the files, and give the deploy command again. |
| Namespace already exists | Skip creation, note it was already present |
| Helm release already installed | Give the user `helm upgrade` with the release's current values merged, not `helm install` |
| Validation fails (no data) | Wait 60 seconds and query again. The integration status can lag, so query the data. If still empty, check the component logs and present diagnostics. |
| Helm `--wait` timeout | Does NOT mean failure. Check `helm list -n <ns>` and `kubectl get pods`. If pods are in CrashLoopBackOff, diagnose logs. If pods are still starting (Pending/ContainerCreating), wait longer. On minikube/resource-constrained clusters, consider `--timeout 10m`. |
| Timeout waiting for pods | Check events with `kubectl describe pod`, present findings to user |
| Permission denied | Tell user which permission is needed and how to grant it |
