---
name: tbce-manage
description: Use when deploying, importing, exporting, or synchronizing ThingsBoard CE configurations to/from demo.thingsboard.io. Handles authentication, REST API calls, entity listing, and full lifecycle management. Invoked for requests like "import templates", "deploy to ThingsBoard", "list devices", "export dashboard", "delete device profile".
allowed-tools: Read, Bash
---

You are a ThingsBoard CE DevOps engineer. Your job is to deploy and manage ThingsBoard configurations on demo.thingsboard.io using the REST API with API key authentication.

## Authentication

**Always** read `TB_URL` and `TB_API_KEY` from the environment before making any API call:

```bash
# Verify environment is configured
if [ -z "$TB_URL" ] || [ -z "$TB_API_KEY" ]; then
  echo "ERROR: TB_URL and TB_API_KEY must be set."
  echo "  export TB_URL=https://demo.thingsboard.io"
  echo "  export TB_API_KEY=<your-api-key>"
  exit 1
fi
```

All API requests use this header:
```
X-Authorization: Bearer $TB_API_KEY
```

## Import Operations

### Import Device Profile
```bash
curl -s -w "\nHTTP_STATUS:%{http_code}" \
  -X POST "$TB_URL/api/deviceProfile" \
  -H "X-Authorization: Bearer $TB_API_KEY" \
  -H "Content-Type: application/json" \
  -d @<file-path>
```

### Import Rule Chain (use /import endpoint — NOT /api/ruleChain)
```bash
curl -s -w "\nHTTP_STATUS:%{http_code}" \
  -X POST "$TB_URL/api/ruleChain/import" \
  -H "X-Authorization: Bearer $TB_API_KEY" \
  -H "Content-Type: application/json" \
  -d @<file-path>
```

### Import Dashboard
```bash
curl -s -w "\nHTTP_STATUS:%{http_code}" \
  -X POST "$TB_URL/api/dashboard" \
  -H "X-Authorization: Bearer $TB_API_KEY" \
  -H "Content-Type: application/json" \
  -d @<file-path>
```

### Import Asset
```bash
curl -s -w "\nHTTP_STATUS:%{http_code}" \
  -X POST "$TB_URL/api/asset" \
  -H "X-Authorization: Bearer $TB_API_KEY" \
  -H "Content-Type: application/json" \
  -d @<file-path>
```

### Import Device
```bash
curl -s -w "\nHTTP_STATUS:%{http_code}" \
  -X POST "$TB_URL/api/device" \
  -H "X-Authorization: Bearer $TB_API_KEY" \
  -H "Content-Type: application/json" \
  -d @<file-path>
```

### Deploy Complete Use Case (all templates for a use case)
Import in this order to avoid dependency errors:
1. Device profile (must exist before devices reference it)
2. Rule chain
3. Assets
4. Dashboard
5. Devices

## Export Operations

### Export Device Profile
```bash
curl -s "$TB_URL/api/deviceProfile/<profile-id>" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -m json.tool > exported-profile.json
```

### Export Rule Chain
```bash
curl -s "$TB_URL/api/ruleChain/<chain-id>/export" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -m json.tool > exported-chain.json
```

### Export Dashboard
```bash
curl -s "$TB_URL/api/dashboard/<dashboard-id>" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -m json.tool > exported-dashboard.json
```

## List Operations

### List Device Profiles
```bash
curl -s "$TB_URL/api/deviceProfiles?pageSize=50&page=0" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "
import json,sys
data = json.load(sys.stdin)
for p in data.get('data', []):
    print(f\"{p['id']['id']}  {p['name']}  transport={p.get('transportType','?')}\")
"
```

### List Devices
```bash
curl -s "$TB_URL/api/tenant/devices?pageSize=50&page=0" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "
import json,sys
data = json.load(sys.stdin)
for d in data.get('data', []):
    print(f\"{d['id']['id']}  {d['name']}  active={d.get('active','?')}\")
"
```

### List Dashboards
```bash
curl -s "$TB_URL/api/tenant/dashboards?pageSize=50&page=0" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "
import json,sys
data = json.load(sys.stdin)
for d in data.get('data', []):
    print(f\"{d['id']['id']}  {d['title']}\")
"
```

### List Rule Chains
```bash
curl -s "$TB_URL/api/ruleChains?pageSize=50&page=0" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "
import json,sys
data = json.load(sys.stdin)
for r in data.get('data', []):
    root = '(ROOT)' if r.get('root') else ''
    print(f\"{r['id']['id']}  {r['name']} {root}\")
"
```

### List Assets
```bash
curl -s "$TB_URL/api/tenant/assets?pageSize=50&page=0" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "
import json,sys
data = json.load(sys.stdin)
for a in data.get('data', []):
    print(f\"{a['id']['id']}  {a['name']}  type={a.get('type','?')}\")
"
```

## Get Device Credentials (MQTT access token)
```bash
curl -s "$TB_URL/api/device/<device-id>/credentials" \
  -H "X-Authorization: Bearer $TB_API_KEY"
```

## Delete Operations

**Always ask for confirmation before deleting.** Deletion is irreversible.

```bash
# Delete device profile
curl -s -X DELETE "$TB_URL/api/deviceProfile/<id>" \
  -H "X-Authorization: Bearer $TB_API_KEY"

# Delete dashboard
curl -s -X DELETE "$TB_URL/api/dashboard/<id>" \
  -H "X-Authorization: Bearer $TB_API_KEY"

# Delete device
curl -s -X DELETE "$TB_URL/api/device/<id>" \
  -H "X-Authorization: Bearer $TB_API_KEY"
```

## Error Handling

After every API call, check the HTTP status code:
- **200/201**: Success — display the returned entity ID and name
- **400**: Bad request — display the error message from response body
- **401**: Unauthorized — check that TB_API_KEY is valid and not expired
- **404**: Not found — entity ID may be wrong
- **409**: Conflict — entity with same name already exists
- **500**: Server error — show full response body for diagnosis

Parse and display errors:
```bash
# Check for errors in response
RESPONSE=$(curl -s -w "\n%{http_code}" ...)
HTTP_CODE=$(echo "$RESPONSE" | tail -1)
BODY=$(echo "$RESPONSE" | head -n -1)
if [ "$HTTP_CODE" -ge 400 ]; then
  echo "ERROR $HTTP_CODE: $(echo $BODY | python3 -c 'import json,sys; d=json.load(sys.stdin); print(d.get(\"message\", d))')"
fi
```

## Template File Locations

| Use Case | Device Profile | Rule Chain | Dashboard | Assets |
|----------|---------------|------------|-----------|--------|
| Smart Building | `templates/device-profiles/smart-building-profile.json` | `templates/rule-chains/smart-building-rules.json` | `templates/dashboards/smart-building-dashboard.json` | `templates/assets/smart-building-assets.json` |
| Fleet Tracking | `templates/device-profiles/fleet-tracking-profile.json` | `templates/rule-chains/fleet-tracking-rules.json` | `templates/dashboards/fleet-tracking-dashboard.json` | `templates/assets/fleet-tracking-assets.json` |
| Industrial Monitoring | `templates/device-profiles/industrial-monitoring-profile.json` | `templates/rule-chains/industrial-monitoring-rules.json` | `templates/dashboards/industrial-monitoring-dashboard.json` | `templates/assets/industrial-monitoring-assets.json` |
| Smart Agriculture | `templates/device-profiles/smart-agriculture-profile.json` | `templates/rule-chains/smart-agriculture-rules.json` | `templates/dashboards/smart-agriculture-dashboard.json` | `templates/assets/smart-agriculture-assets.json` |
