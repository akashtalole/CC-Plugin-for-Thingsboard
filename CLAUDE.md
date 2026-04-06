# CC-Plugin-for-ThingsBoard

A Claude Code and GitHub Copilot plugin for accelerating IoT solution development on ThingsBoard Community Edition (TBCE). Provides four specialized agents and domain skills for authoring, managing, testing, and troubleshooting ThingsBoard components.

## Agents

| Agent | When to invoke |
|-------|---------------|
| **tbce-author** | Creating/editing devices, device profiles, dashboards, rule chains, assets |
| **tbce-manage** | Importing/exporting configs, deploying to demo.thingsboard.io, listing entities |
| **tbce-test** | Validating JSON, simulating telemetry, testing alarm triggers, rule chain checks |
| **tbce-troubleshoot** | Diagnosing connectivity failures, missing telemetry, alarm/rule chain issues |

## Environment Setup

```bash
export TB_URL=https://demo.thingsboard.io
export TB_API_KEY=<your-thingsboard-api-key>
```

**Getting an API Key on demo.thingsboard.io:**
1. Log in → Profile → API Keys → Generate Key
2. Copy the key and export as `TB_API_KEY`

**Note:** The ThingsBoard MCP server (`.mcp.json`) uses username/password auth (`TB_USERNAME` / `TB_PASSWORD`). The REST API curl commands in agents use `TB_API_KEY` directly.

## ThingsBoard REST API Quick Reference

All requests require the header:
```
X-Authorization: Bearer $TB_API_KEY
```

### Key Endpoints

| Entity | Create | List | Get by ID |
|--------|--------|------|-----------|
| Device | `POST /api/device` | `GET /api/tenant/devices?pageSize=50&page=0` | `GET /api/device/{id}` |
| Device Profile | `POST /api/deviceProfile` | `GET /api/deviceProfiles?pageSize=50&page=0` | `GET /api/deviceProfile/{id}` |
| Dashboard | `POST /api/dashboard` | `GET /api/tenant/dashboards?pageSize=50&page=0` | `GET /api/dashboard/{id}` |
| Rule Chain | `POST /api/ruleChain` | `GET /api/ruleChains?pageSize=50&page=0` | `GET /api/ruleChain/{id}` |
| Asset | `POST /api/asset` | `GET /api/tenant/assets?pageSize=50&page=0` | `GET /api/asset/{id}` |

### Special Import Endpoints

```bash
# Import rule chain (use this instead of POST /api/ruleChain for full metadata)
POST /api/ruleChain/import

# Get device credentials (access token for MQTT)
GET /api/device/{id}/credentials

# Query timeseries data
GET /api/plugins/telemetry/DEVICE/{id}/values/timeseries?keys=temperature,humidity

# Send test telemetry via HTTP
POST /api/v1/{ACCESS_TOKEN}/telemetry
```

## Template Locations

| Use Case | Device Profile | Rule Chain | Dashboard | Assets |
|----------|---------------|------------|-----------|--------|
| Smart Building | `templates/device-profiles/smart-building-profile.json` | `templates/rule-chains/smart-building-rules.json` | `templates/dashboards/smart-building-dashboard.json` | `templates/assets/smart-building-assets.json` |
| Fleet Tracking | `templates/device-profiles/fleet-tracking-profile.json` | `templates/rule-chains/fleet-tracking-rules.json` | `templates/dashboards/fleet-tracking-dashboard.json` | `templates/assets/fleet-tracking-assets.json` |
| Industrial Monitoring | `templates/device-profiles/industrial-monitoring-profile.json` | `templates/rule-chains/industrial-monitoring-rules.json` | `templates/dashboards/industrial-monitoring-dashboard.json` | `templates/assets/industrial-monitoring-assets.json` |
| Smart Agriculture | `templates/device-profiles/smart-agriculture-profile.json` | `templates/rule-chains/smart-agriculture-rules.json` | `templates/dashboards/smart-agriculture-dashboard.json` | `templates/assets/smart-agriculture-assets.json` |

## Import Order for a New Use Case

Import in this order to avoid dependency errors:

1. **Device Profile** — must exist before devices reference it
2. **Rule Chain** — import via `/api/ruleChain/import`
3. **Assets** — create asset hierarchy (Building → Floor → Room, etc.)
4. **Dashboard** — references entityAliases by profile name
5. **Devices** — assign to the device profile

```bash
# Example: Deploy complete Smart Building solution
curl -s -X POST $TB_URL/api/deviceProfile \
  -H "X-Authorization: Bearer $TB_API_KEY" \
  -H "Content-Type: application/json" \
  -d @templates/device-profiles/smart-building-profile.json | python3 -m json.tool

curl -s -X POST $TB_URL/api/ruleChain/import \
  -H "X-Authorization: Bearer $TB_API_KEY" \
  -H "Content-Type: application/json" \
  -d @templates/rule-chains/smart-building-rules.json | python3 -m json.tool

curl -s -X POST $TB_URL/api/dashboard \
  -H "X-Authorization: Bearer $TB_API_KEY" \
  -H "Content-Type: application/json" \
  -d @templates/dashboards/smart-building-dashboard.json | python3 -m json.tool
```

## ThingsBoard CE Best Practices

### Device Profiles
- Always configure a **default rule chain** for the profile
- Use **MQTT transport** for most IoT devices (lowest overhead)
- Define **alarm rules in the profile**, not in custom rule chains — this centralizes alarm logic
- Use alarm **severity levels**: CRITICAL > MAJOR > MINOR > WARNING > INDETERMINATE
- Always define **clearRule** for every alarm to prevent alarm flooding

### Rule Engine
- Prefer **TBEL** (ThingsBoard Expression Language) over JavaScript — use `"scriptLang": "TBEL"` in node config
- TBEL script access pattern: `msg.temperature` (direct field access, no `JSON.parse` needed)
- Always connect **Failure** outputs of every node to a log/debug node in production
- Use the **Device Profile node** as the first processing node — it applies profile-level alarms
- Chain: `Input → Save Timeseries → Device Profile Node → [alarm logic] → notifications`

### Dashboards
- Use **entityAliases** with `"type": "deviceSearchQuery"` filtered by device profile name — do not hardcode device IDs
- Set `"resolveMultiple": true` on aliases to support multi-device dashboards
- Use **time window** settings: default `"realtime": {"realtimeType": 1, "timewindowMs": 60000}` for live monitoring
- Widget datasource: `"type": "entity"` referencing an entityAlias

### Telemetry
- Telemetry keys are case-sensitive — `temperature` ≠ `Temperature`
- Numeric values: use double/long (not strings) for time-series data
- MQTT publish topic: `v1/devices/me/telemetry`
- MQTT credentials: use **device access token** as the MQTT password (username can be empty or `""`)

### Assets & Relations
- Create a **root asset** per site/customer (Building, Fleet, Farm)
- Link devices to assets via `POST /api/relation` with `relationType: "Contains"`
- Use asset hierarchy for dashboard drill-down navigation

## TBEL Quick Reference

```javascript
// Filter: pass messages where temperature > 25
return msg.temperature > 25;

// Transform: add metadata field
msg.location = metadata.deviceName;
return {msg: msg, metadata: metadata, msgType: msgType};

// Check message type
return msgType == 'POST_TELEMETRY_REQUEST';

// Access metadata
var deviceName = metadata.deviceName;

// Numeric comparison with null safety
return msg.containsKey('temperature') && msg.temperature > 30;
```
