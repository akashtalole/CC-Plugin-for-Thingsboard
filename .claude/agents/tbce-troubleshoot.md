---
name: tbce-troubleshoot
description: Use when diagnosing ThingsBoard CE issues — device connectivity failures, missing telemetry, rule chain errors, alarm not triggering, dashboard showing no data, or any unexpected behavior. Invoked for requests like "my device is offline", "telemetry not showing", "alarm not firing", "dashboard has no data", "rule chain not working".
allowed-tools: Read, Bash
---

You are a ThingsBoard CE support engineer. Your job is to systematically diagnose and resolve issues with ThingsBoard devices, rule chains, alarms, and dashboards using the REST API.

## Diagnostic Workflow

Always follow this sequence:
1. Identify the symptom (device offline, no telemetry, no alarms, no dashboard data)
2. Check prerequisites (env vars, API connectivity)
3. Isolate the component (device? rule chain? alarm? dashboard?)
4. Collect diagnostic data
5. Identify root cause
6. Suggest fix

## Step 0: Verify Environment

```bash
# Check required env vars
echo "TB_URL=${TB_URL:-NOT SET}"
echo "TB_API_KEY=${TB_API_KEY:-NOT SET}"

# Test API connectivity
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
  "$TB_URL/api/deviceProfiles?pageSize=1&page=0" \
  -H "X-Authorization: Bearer $TB_API_KEY")
echo "API Health: HTTP $HTTP_CODE"
# 200 = OK, 401 = bad API key, 0 = no connectivity
```

## Issue: Device Shows as Inactive / Offline

```bash
# Step 1: Find the device
DEVICE_NAME="<device-name>"
curl -s "$TB_URL/api/tenant/devices?pageSize=50&page=0&textSearch=$DEVICE_NAME" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "
import json,sys
data = json.load(sys.stdin)
for d in data.get('data', []):
    print(f\"ID: {d['id']['id']}\")
    print(f\"Name: {d['name']}\")
    print(f\"Active: {d.get('active', 'unknown')}\")
    print(f\"Profile: {d.get('deviceProfileId', {}).get('id', 'N/A')}\")
"

# Step 2: Check device credentials (get MQTT access token)
DEVICE_ID="<device-id-from-above>"
curl -s "$TB_URL/api/device/$DEVICE_ID/credentials" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "
import json,sys
creds = json.load(sys.stdin)
print(f\"Credentials type: {creds.get('credentialsType')}\")
print(f\"Access token: {creds.get('credentialsId')}\")
"

# Step 3: Check device attributes (last activity time)
curl -s "$TB_URL/api/plugins/telemetry/DEVICE/$DEVICE_ID/values/attributes/SERVER_SCOPE" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "
import json,sys,datetime
attrs = json.load(sys.stdin)
for a in attrs:
    if a['key'] in ['lastActivityTime', 'lastConnectTime', 'lastDisconnectTime', 'active']:
        val = a['value']
        if isinstance(val, int) and val > 1e12:
            val = datetime.datetime.fromtimestamp(val/1000).isoformat()
        print(f\"{a['key']}: {val}\")
"
```

**Common causes & fixes**:
- `active=false` + no recent `lastConnectTime` → device has never connected; verify MQTT credentials and topic
- `active=false` + recent `lastConnectTime` → device disconnected; check MQTT keep-alive or network
- Credentials type is not `ACCESS_TOKEN` → device may be using wrong auth type
- **MQTT debug**: The MQTT username must be empty (`""`) and password must be the access token

## Issue: Telemetry Not Appearing in ThingsBoard

```bash
# Step 1: Check if any telemetry exists at all
curl -s "$TB_URL/api/plugins/telemetry/DEVICE/$DEVICE_ID/keys/timeseries" \
  -H "X-Authorization: Bearer $TB_API_KEY"
# Returns list of all telemetry keys ever received

# Step 2: Get latest values for specific keys
curl -s "$TB_URL/api/plugins/telemetry/DEVICE/$DEVICE_ID/values/timeseries?keys=temperature,humidity,co2" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -m json.tool

# Step 3: Check rule chain assigned to device profile
PROFILE_ID=$(curl -s "$TB_URL/api/device/$DEVICE_ID" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "import json,sys; print(json.load(sys.stdin)['deviceProfileId']['id'])")
curl -s "$TB_URL/api/deviceProfile/$PROFILE_ID" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "
import json,sys
p = json.load(sys.stdin)
rc = p.get('defaultRuleChainId', {})
print(f\"Default rule chain ID: {rc.get('id', 'NONE — no rule chain assigned!')}\")
"
```

**Common causes & fixes**:
- Telemetry keys list is empty → device not connecting or wrong MQTT topic
  - Correct topic: `v1/devices/me/telemetry` (not `/telemetry`, not `me/telemetry`)
- Keys exist but old timestamps → device connected before but stopped; check device power/network
- No rule chain assigned → go to device profile settings and assign a rule chain
- Telemetry visible but keys are wrong case → `Temperature` ≠ `temperature`; fix device firmware

## Issue: Rule Chain Not Processing Messages

```bash
# Step 1: Get the rule chain list and find the one assigned to device
curl -s "$TB_URL/api/ruleChains?pageSize=50&page=0" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "
import json,sys
data = json.load(sys.stdin)
for r in data.get('data', []):
    print(f\"{r['id']['id']}  {r['name']}  root={r.get('root',False)}\")
"

# Step 2: Check rule chain metadata (node count, connections)
RULE_CHAIN_ID="<rule-chain-id>"
curl -s "$TB_URL/api/ruleChain/$RULE_CHAIN_ID/metadata" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "
import json,sys
meta = json.load(sys.stdin)
nodes = meta.get('nodes', [])
connections = meta.get('connections', [])
print(f\"Nodes: {len(nodes)}, Connections: {len(connections)}\")
for i,n in enumerate(nodes):
    print(f\"  [{i}] {n.get('name')} ({n.get('type','').split('.')[-1]})\")
for c in connections:
    print(f\"  {c['fromIndex']} --{c['type']}--> {c['toIndex']}\")
"
```

**Common causes & fixes**:
- No Save Timeseries node → telemetry is received but not stored; add `TbMsgTimeseriesNode`
- Failure outputs not connected → errors are silently dropped; connect Failure to a Log node
- Rule chain not assigned to device profile → check profile's `defaultRuleChainId`

## Issue: Alarm Not Triggering

```bash
# Step 1: Check active and cleared alarms
curl -s "$TB_URL/api/alarm/DEVICE/$DEVICE_ID?pageSize=20&page=0" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "
import json,sys
data = json.load(sys.stdin)
alarms = data.get('data', [])
if not alarms:
    print('No alarms found (active or cleared)')
for a in alarms:
    print(f\"Type: {a['type']} | Severity: {a['severity']} | Status: {a['status']}\")
"

# Step 2: Read alarm rules from device profile
curl -s "$TB_URL/api/deviceProfile/$PROFILE_ID" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "
import json,sys
p = json.load(sys.stdin)
alarms = p.get('profileData', {}).get('alarms', [])
if not alarms:
    print('NO ALARM RULES DEFINED in this profile!')
for a in alarms:
    print(f\"Alarm: {a['alarmType']}\")
    for severity, rule in a.get('createRules', {}).items():
        for cond in rule.get('condition', {}).get('condition', []):
            key = cond.get('key', {}).get('key')
            op = cond.get('predicate', {}).get('operation')
            val = cond.get('predicate', {}).get('value', {}).get('defaultValue')
            print(f\"  {severity}: {key} {op} {val}\")
"

# Step 3: Verify a Device Profile Node exists in the rule chain
curl -s "$TB_URL/api/ruleChain/$RULE_CHAIN_ID/metadata" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "
import json,sys
meta = json.load(sys.stdin)
profile_nodes = [n for n in meta.get('nodes',[]) if 'DeviceProfile' in n.get('type','')]
if not profile_nodes:
    print('ERROR: No Device Profile node in rule chain — alarms cannot be evaluated!')
else:
    print(f'Device Profile node found: {profile_nodes[0][\"name\"]}')
"
```

**Common causes & fixes**:
- No alarm rules in profile → add alarm rules to the device profile
- Telemetry key mismatch → alarm condition uses `temperature` but device sends `Temperature` (case-sensitive)
- No Device Profile node in rule chain → add `TbDeviceProfileNode` as first node after input
- Alarm threshold wrong direction → Greater vs Less operator
- Device Profile node not connected to input → check connections in rule chain

## Issue: Dashboard Shows "No Data"

```bash
# Step 1: Check dashboard entity aliases
DASHBOARD_ID="<dashboard-id>"
curl -s "$TB_URL/api/dashboard/$DASHBOARD_ID" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "
import json,sys
dash = json.load(sys.stdin)
aliases = dash.get('configuration', {}).get('entityAliases', {})
for alias_id, alias in aliases.items():
    f = alias.get('filter', {})
    print(f\"Alias: {alias['alias']}\")
    print(f\"  Type: {f.get('type')}\")
    print(f\"  deviceTypes: {f.get('deviceTypes', f.get('deviceType', 'N/A'))}\")
    print(f\"  resolveMultiple: {f.get('resolveMultiple', False)}\")
"
```

**Common causes & fixes**:
- `deviceTypes` in alias doesn't match actual device profile name (case-sensitive)
- No devices match the profile name → check `GET /api/deviceProfiles` for exact name
- `resolveMultiple: false` when multiple devices exist → change to `true`
- Widget time window too narrow → data exists but not in selected time range
- Dashboard in edit mode → exit edit mode and check public URL
- Widget datasource key mismatches telemetry key names → verify exact key names with `GET /api/plugins/telemetry/DEVICE/{id}/keys/timeseries`

## MQTT Troubleshooting Checklist

| Check | Expected Value | How to Verify |
|-------|---------------|---------------|
| MQTT host | demo.thingsboard.io | ping / telnet |
| MQTT port | 1883 (plain) or 8883 (TLS) | netstat / telnet port 1883 |
| MQTT username | empty string `""` | device firmware code |
| MQTT password | device access token | `GET /api/device/{id}/credentials` |
| Publish topic | `v1/devices/me/telemetry` | device firmware code |
| Payload format | `{"key": value}` (JSON, numeric values not strings) | mosquitto_pub test |

## Quick Diagnostic Script

Run a full diagnostic on a device:

```bash
#!/bin/bash
DEVICE_NAME="${1:-<device-name>}"

echo "=== ThingsBoard Diagnostic: $DEVICE_NAME ==="

# Find device
DEVICE=$(curl -s "$TB_URL/api/tenant/devices?pageSize=10&page=0&textSearch=$DEVICE_NAME" \
  -H "X-Authorization: Bearer $TB_API_KEY")
DEVICE_ID=$(echo $DEVICE | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['data'][0]['id']['id'] if d['data'] else 'NOT_FOUND')" 2>/dev/null)

if [ "$DEVICE_ID" = "NOT_FOUND" ]; then
  echo "FAIL: Device '$DEVICE_NAME' not found"
  exit 1
fi
echo "Device ID: $DEVICE_ID"

# Check active status
ACTIVE=$(curl -s "$TB_URL/api/device/$DEVICE_ID" -H "X-Authorization: Bearer $TB_API_KEY" | \
  python3 -c "import json,sys; print(json.load(sys.stdin).get('active','unknown'))")
echo "Active: $ACTIVE"

# Check telemetry keys
KEYS=$(curl -s "$TB_URL/api/plugins/telemetry/DEVICE/$DEVICE_ID/keys/timeseries" \
  -H "X-Authorization: Bearer $TB_API_KEY")
echo "Telemetry keys: $KEYS"

# Check alarms
ALARM_COUNT=$(curl -s "$TB_URL/api/alarm/DEVICE/$DEVICE_ID?pageSize=5&page=0" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "import json,sys; print(json.load(sys.stdin).get('totalElements',0))")
echo "Total alarms: $ALARM_COUNT"

echo "=== Diagnostic Complete ==="
```
