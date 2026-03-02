# Grafana Tools Reference

> Loaded by SKILL.md at Step 0.5 (triage) and Step 4 (backend queries).
> Covers: Utility, Configuration Discovery, Alerts, Dashboards, Annotations, Deeplinks, Panel Queries.

---

## Utility

| Tool | Required Params | Behavior | Warnings |
|------|-----------------|----------|----------|
| `datetime_get_current_time` | None | Resolve relative time to absolute UTC | — |
| `get_query_examples` | `DatasourceType` (prometheus/loki/clickhouse/cloudwatch) | Returns ready-to-use query templates for backend | Case-insensitive; unsupported types return error |

---

## Configuration Discovery

| Tool | Required Params | Behavior | Warnings |
|------|-----------------|----------|----------|
| `list_datasources` | `type` (optional filter) | Discover datasources; returns UIDs. Pagination: `limit` max 100 (default 50), `offset`. Returns `total`, `hasMore`. | Type filter is exact match only |
| `get_datasource` | `uid` OR `name` | Resolve datasource details. UID takes priority. Returns config, jsonData, cross-datasource links. | Non-existent datasources return "not found" error |

---

## Alerts & Rules

| Tool | Required Params | Behavior | Warnings |
|------|-----------------|----------|----------|
| `list_alert_rules` | `datasourceUid` (optional), `limit`, `label_selectors` | List Grafana-managed OR datasource-managed rules. Returns UID, title, state, labels. | 🔴 **Default limit=100. ALWAYS use limit=1000+.** `label_selectors` filters by alert labels, NOT state. Filter state client-side. |
| `get_alert_rule_by_uid` | `uid` | Retrieve full rule config (queries, condition, evaluation interval) | — |
| `list_contact_points` | `datasourceUid` (optional), `name` (optional) | Discover notification destinations | — |

---

## Dashboards & Visualization

| Tool | Required Params | Behavior | Warnings |
|------|-----------------|----------|----------|
| `search_dashboards` | `query` (substring, case-insensitive) | Find dashboards. Returns `uid`, `title`, `tags`, `uri`. Pagination: `limit` (default 50), `page`. | Empty array = no matches (expected) |
| `search_folders` | `query` | Find dashboard folders by keyword | — |
| `get_dashboard_summary` | `uid` | Quick overview (title, panel count, types, variables) | — |
| `get_dashboard_property` | `uid`, `jsonPath` | Extract specific data using JSONPath | ❌ Object projection `{field1, field2}` NOT supported. Separate calls per field. |
| `get_dashboard_panel_queries` | `uid` | Retrieve all panel queries. Returns: `title`, `query`, `datasource` (with `uid`, `type`). | Datasource UID may be template variable (e.g., `$datasource`). Check `dashboard.templating.list`. |
| `get_panel_image` | `dashboardUid`, `panelId` (optional), `timeRange` | Render panel/dashboard as PNG (base64). | Requires Grafana Image Renderer service |

---

## Annotations

| Tool | Required Params | Behavior | Warnings |
|------|-----------------|----------|----------|
| `get_annotations` | `From` (epoch ms), `To` (epoch ms) | Query time-correlated events. Filter by dashboard, tag, type. | — |
| `get_annotation_tags` | `tag` (optional filter) | Discover annotation tags for deployment markers, maintenance windows | — |

---

## Deeplinks

| Tool | Required Params | Behavior | Warnings |
|------|-----------------|----------|----------|
| `generate_deeplink` | `resourceType` (dashboard/panel/explore), type-specific UIDs, `timeRange` (optional) | Create shareable Grafana links. **Always include time range.** | dashboard: `dashboardUID`; panel: `dashboardUID` + `panelID`; explore: `datasourceUID`. Use `var-<name>` prefix for Grafana variables. |

---

## Panel Query Execution

| Tool | Required Params | Behavior | Warnings |
|------|-----------------|----------|----------|
| `run_panel_query` | `DashboardUID`, `PanelIDs` (array), `Start`, `End`, `Variables` (optional map) | Execute panel queries directly. Auto-substitutes macros and template variables. Returns per-panel results. | Macro substitution before query. CloudWatch: no query string; structured `RawTarget`. Empty results include debugging hints. |

**Key behaviors:**
- Macro substitution: `$__range`, `$__rate_interval`, `$__interval` substituted BEFORE backend query
- Template variables: `$variable`, `${variable}`, `[[variable]]` all work; skips "All"/"$__all"
- Datasource resolution: Target-level overrides panel-level

---

## Time Format Reference (All Tools)

| Format | Example | Valid | Notes |
|--------|---------|-------|-------|
| **Relative (preferred)** | `"now-1h"`, `"now-30m"`, `"now-6h"` | ✅ | Units: h, m, d |
| **Absolute (RFC3339 UTC)** | `"2024-01-15T10:00:00Z"` | ✅ | **Z suffix REQUIRED** |
| **Empty string** | `""` | ✅ (optional only) | For open-ended ranges |
| Fractional durations | `"now-1.5h"` | ❌ | Unsupported |
| Missing Z suffix | `"2024-01-15T10:00:00"` | ❌ | Timezone ambiguous |
| Natural language | `"yesterday"` | ❌ | Not supported |
| Unix timestamps | `"1609459200"` | ❌ | Not supported |

**Common patterns:**

| Intent | Start | End |
|--------|-------|-----|
| Last 1 hour (live) | `"now-1h"` | `"now"` |
| Last 30 minutes | `"now-30m"` | `"now"` |
| Specific past window | `"2024-01-15T09:00:00Z"` | `"2024-01-15T10:00:00Z"` |
