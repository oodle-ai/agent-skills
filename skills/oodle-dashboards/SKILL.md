---
name: oodle-dashboards
description: Create, update, and manage Oodle dashboards and folders — safe deletion, panel preservation, and folder organization.
metadata:
  version: "1.1.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,dashboards,visualization
  globs: "**/*dashboard*"
  alwaysApply: "false"
---

# Oodle Dashboards — CRUD and Organization

This skill teaches the agent to manage Oodle dashboards and folders without losing panel state and without orphaning dashboards in the root folder. Setup, auth, and common errors: see the oodle-cli skill.

## Quick Reference

| Task | Command |
|------|---------|
| List dashboards | `oodle dashboards list -o json` (filter with `jq '.[] \| select(.folderUid=="platform") \| {uid, title}'`) |
| Get dashboard | `oodle dashboards get <uid> -o json` → `{"dashboard": {...}, "meta": {"folderUid": ..., "version": ...}}` |
| Create dashboard | `oodle dashboards create -f save.json` |
| Update dashboard | `oodle dashboards create -f save.json` with the existing `dashboard.uid` and `"overwrite": true` |
| Delete dashboard | `oodle dashboards delete <uid> --force` |
| List folders | `oodle folders list -o json` |
| Create folder | `oodle folders create --title "Platform" [--uid <uid>] [--parent-uid <uid>] -o json` |

There is no `dashboards update` command, and no `folders get` or `folders delete` command. `folders create` takes `--title`, not a file.

## Save Request

`oodle dashboards create -f` takes a save request. The dashboard is Grafana-compatible dashboard JSON; a Prometheus panel target keeps its PromQL in `expr`.

```json
{
  "dashboard": {
    "uid": "api-overview",
    "title": "API Overview",
    "description": "Latency, error rate, and throughput for the API service.",
    "tags": ["service:api", "team:platform"],
    "panels": [
      {
        "title": "Request rate",
        "type": "timeseries",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
        "targets": [{"refId": "A", "expr": "sum(rate(http_requests_total{service=\"api\"}[$__rate_interval]))"}]
      }
    ]
  },
  "folderUid": "platform",
  "overwrite": false,
  "message": "Create API overview"
}
```

- **Always set `folderUid`.** Without it the dashboard goes to the root folder, where teams cannot find it. To make a new folder: `FOLDER_UID=$(oodle folders create --title "Platform" -o json | jq -r '.uid')`.
- **Tag dashboards** with `service:`, `team:`, and `env:` so people and tools can find them.

## Changing a Dashboard

A save with `"overwrite": true` replaces the whole dashboard; panels missing from the body are removed. Always start from `get`:

```bash
# ✅ CORRECT
oodle dashboards get api-overview -o json > current.json
jq '{dashboard: (.dashboard | .panels[0].title = "Request rate (per second)"), folderUid: .meta.folderUid, overwrite: true}' current.json > save.json
oodle dashboards create -f save.json

# ❌ WRONG — `dashboards update` does not exist, and a partial dashboard removes panels
oodle dashboards update api-overview -f <(echo '{"title":"new title"}')
```

## Deleting a Shared Dashboard

Runbooks, chat messages, and bookmarks link to dashboards. Delete in two phases: rename it first, then delete after at least 7 days.

```bash
oodle dashboards get api-overview -o json > current.json
jq '{dashboard: (.dashboard | .title = "[MARKED FOR DELETION] " + .title), folderUid: .meta.folderUid, overwrite: true}' current.json > save.json
oodle dashboards create -f save.json
# at least 7 days later:
oodle dashboards delete api-overview --force
```

## Lookback Windows: never `[$__range]`

`$__range` is the whole dashboard range, so every step re-reads almost the same data. A 6h panel with `[$__range]` reads about 180 times more samples than with `[$__dd_interval]`, and the factor grows as the user zooms out. Oodle marks such a panel with a warning triangle. Use a window that scales with the range:

- `$__rate_interval` — for `rate()` and `increase()` on a graph.
- `$__dd_interval` — Oodle's default rollup (20s at 1h, 5m at 24h, 1h at 7d, capped at 4h). Use it for stat totals.
- `$__large_interval` — a coarser rollup (1m at 1h, 20m at 24h, 4h at 7d, capped at 12h). Use it for bar charts and long ranges.

Do not use `increase(...[$__range]) / $__range_s`; use `rate(...[$__rate_interval])`.

For a stat panel that shows one number for the whole range, use a range query with window = min step, and reduce the points:

```json
{
  "type": "stat",
  "targets": [{"refId": "A", "expr": "sum(increase(http_requests_total[$__dd_interval]))", "interval": "$__dd_interval", "instant": false}],
  "transformations": [{"id": "reduce", "options": {"reducers": ["sum"]}}]
}
```

Keep the min step equal to the window, so the windows tile the range (a smaller step counts samples twice, a larger one skips them). Reducer: `sum` for `increase`, `sum_over_time`, `count_over_time`; `max` for `max_over_time`; `min` for `min_over_time`; `mean` for `avg_over_time`. Do not split `quantile_over_time`, `changes`, or `resets` this way.

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| `unknown command "update"` | There is no `dashboards update` | `oodle dashboards create -f save.json` with `"overwrite": true` |
| Version or name conflict on save | The uid or title exists and `overwrite` is not `true` | `get` the dashboard, then save with `"overwrite": true` |
| `folder not found` | `folderUid` does not exist | `oodle folders list -o json`; use an existing uid or create one |
| Panels disappeared after a save | Partial dashboard with `overwrite: true` | Save again from the last `get` output |
| Need to delete a folder | No `folders delete` command | Move or delete its dashboards, then delete the folder in the Oodle UI |

## References

- [Query optimization — lookback windows and `$__range`](https://docs.oodle.ai/metrics/query-optimization)
