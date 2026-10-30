# Post-Install Validation

A signal is set up only when its data is in Oodle. Run the spec's `validation.checks` first: they name the metrics, log filters and queries for that integration. Then use the checks below for each signal that the integration gives.

Some integrations show a status in `oodle integrations list -o json` (`RECEIVING`). Many do not, and logs or traces can arrive with no integration record at all. Thus, a query that returns data is the check. A status is not. The metrics status also trails the data: it can show `NOT_RECEIVING` for a minute or two after the first data arrives. When a query returns data, report the signal as done and say that the status can take a few minutes to change.

## Checks by signal

| Signal | Command | Success |
|---|---|---|
| Metrics | `oodle metrics query --query 'count(up{cluster="<clusterName>"})' --time now -o json`, or a metric from the spec's checks | At least one series |
| Metrics (APM from traces) | `oodle metrics query --query 'traces_service_graph_request_total{client="<service>"}' --time now -o json` | At least one series |
| Logs | `oodle logs index-patterns -o json`, then `oodle logs query -f <ndjson-file> --start -15m -o json`. Set `index` to a `title` from that list (a guessed name such as `logs-*` returns no hits and no error). Filter on the field the spec's check names (for example `source_type`). Put a value with a hyphen in quotes in `query_string`. | Hits more than zero |
| Traces | `oodle traces list --start -15m --end now --limit 5 -o json` | Spans from the expected services |
| Agent traces (LLM apps, coding agents) | `oodle traces list --start -15m --end now --limit 5 -o json`, filtered on the service name the spec sets | Spans with `gen_ai.*` attributes |
| Profiles, RUM | The spec's `validation.checks`. Ask the customer to load a page (RUM) or to let the app run for some minutes (profiles). | The data shows in the views the spec names |
| Alerting, MCP, AI tools | The spec's `validation.checks` (for example, a test notification, or one MCP tool call) | The check result |

Run the commands directly. No other skill is necessary for these checks.

## Make some traffic

A new pipeline with no traffic looks the same as a broken one. Before you report a failure, make sure that the app does work: call an endpoint, load a page, run one LLM request, or run the coding agent for one prompt.

## If data does not arrive

- Wait 2 to 3 minutes and try again. A new pipeline can take some minutes.
- Make sure that the query uses the correct cluster, service, namespace or log group.
- For logs, check the index pattern with `oodle logs index-patterns -o json` first.
- Read the collector or sidecar logs (`kubectl logs`, `aws logs tail`, `docker logs`). Look for HTTP 401 or 403 (a bad key), DNS or TLS errors (a bad host), and 413 (batches that are too large).
- For a pipeline deploy, make sure that the deploy is complete and that the new revision runs (`kubectl rollout status`, `aws ecs describe-services`).
