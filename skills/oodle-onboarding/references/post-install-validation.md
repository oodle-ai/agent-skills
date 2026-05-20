# Post-Install Validation with Query Skills

After the integration status shows `RECEIVING`, verify that data is actually queryable on the Oodle side. Use the companion query skills to confirm each signal type that the integration supports.

## Validation Queries by Signal Type

| Signal | Skill to use | Validation query |
|--------|-------------|-----------------|
| Metrics | `/oodle-metrics-query` | `oodle metrics query --query 'up{cluster="<clusterName>"}' --time now -o json` |
| Metrics (APM/Beyla) | `/oodle-metrics-query` | `oodle metrics query --query 'http_server_request_duration_seconds_count{k8s_namespace_name="<namespace>"}' --time now -o json` |
| Logs | `/oodle-logs` | `oodle logs query -f <ndjson-file> -o json` (query by cluster or namespace) |
| Traces | `/oodle-traces` | `oodle traces list --start -15m --end now --limit 5 -o json` |
| Service Graph | `/oodle-metrics-query` | `oodle metrics query --query 'traces_service_graph_request_total{client="<service>"}' --time now -o json` |

## Running Validation Inline

After installation completes, run the CLI commands from the table above directly. These are the same commands the query skills use internally — no need to invoke a separate skill during onboarding.

## Success Criteria

Validation is successful when:
- At least one metric series is returned for the cluster
- Logs return non-zero hits (if logs are in the integration's categories)
- Traces show spans from expected services (if traces are in the integration's categories)

## Troubleshooting

If validation fails but integration shows RECEIVING:
- Data may still be propagating — wait 2-3 minutes and retry
- Check that the query targets the correct cluster name / namespace / service name
- For logs: check index pattern with `oodle logs index-patterns -o json` first
