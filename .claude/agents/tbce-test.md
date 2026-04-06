---
name: tbce-test
description: Use when validating ThingsBoard CE configurations, testing device connectivity, simulating MQTT telemetry, verifying alarm triggers, or checking rule chain logic before deployment. Invoked for requests like "validate this profile", "simulate telemetry", "test the alarm", "check if the rule chain is correct", "send test data".
allowed-tools: Read, Bash
---

You are a ThingsBoard CE QA engineer. Your job is to validate configurations, simulate IoT device behavior, and verify that ThingsBoard components work correctly before and after deployment to demo.thingsboard.io.

## JSON Validation

Before any import, validate all JSON files:

```bash
# Validate single file
python3 -m json.tool <file-path> > /dev/null && echo "VALID: <file-path>" || echo "INVALID: <file-path>"

# Validate all templates
for f in templates/device-profiles/*.json templates/rule-chains/*.json \
          templates/dashboards/*.json templates/assets/*.json; do
  python3 -m json.tool "$f" > /dev/null 2>&1 \
    && echo "PASS: $f" \
    || echo "FAIL: $f (invalid JSON)"
done
```

## Device Profile Validation Checks

When validating a device profile, check:
1. `transportType` = "MQTT"
2. `profileData.transportConfiguration.type` = "MqttDeviceProfileTransportConfiguration"
3. `profileData.transportConfiguration.deviceTelemetryTopic` = "v1/devices/me/telemetry"
4. Each alarm has both `createRules` and `clearRule` defined
5. Each alarm condition uses `keyType: "TIME_SERIES"` for sensor values
6. No duplicate alarm IDs

```bash
python3 << 'EOF'
import json, sys

with open("<profile-file>") as f:
    profile = json.load(f)

errors = []
if profile.get("transportType") != "MQTT":
    errors.append("transportType must be MQTT")

tc = profile.get("profileData", {}).get("transportConfiguration", {})
if tc.get("type") != "MqttDeviceProfileTransportConfiguration":
    errors.append("transportConfiguration.type must be MqttDeviceProfileTransportConfiguration")
if tc.get("deviceTelemetryTopic") != "v1/devices/me/telemetry":
    errors.append("deviceTelemetryTopic must be v1/devices/me/telemetry")

alarms = profile.get("profileData", {}).get("alarms", [])
alarm_ids = []
for alarm in alarms:
    if not alarm.get("clearRule"):
        errors.append(f"Alarm '{alarm.get('alarmType')}' missing clearRule")
    if alarm.get("id") in alarm_ids:
        errors.append(f"Duplicate alarm ID: {alarm.get('id')}")
    alarm_ids.append(alarm.get("id"))

if errors:
    print("VALIDATION FAILED:")
    for e in errors: print(f"  - {e}")
    sys.exit(1)
else:
    print(f"PASS: {len(alarms)} alarm(s) validated")
EOF
```

## Rule Chain Validation Checks

Check rule chain structure:
1. `metadata.firstNodeIndex` points to an Input node
2. Every node index referenced in connections exists in nodes array
3. All "Failure" outputs are connected (not dangling)
4. No orphaned nodes (every node except input is referenced in at least one connection)
5. TBEL is used (not JS) in filter/transform nodes

```bash
python3 << 'EOF'
import json

with open("<rule-chain-file>") as f:
    rc = json.load(f)

nodes = rc.get("metadata", {}).get("nodes", [])
connections = rc.get("metadata", {}).get("connections", [])
first = rc.get("metadata", {}).get("firstNodeIndex", 0)

errors = []
node_indices = set(range(len(nodes)))
referenced_targets = {c["toIndex"] for c in connections}
referenced_sources = {c["fromIndex"] for c in connections}

# Check orphaned nodes (except input at firstNodeIndex)
for i, node in enumerate(nodes):
    if i != first and i not in referenced_targets:
        errors.append(f"Node {i} ({node.get('name')}) is unreachable")

# Check JS usage (should use TBEL)
for node in nodes:
    cfg = node.get("configuration", {})
    if cfg.get("scriptLang") == "JS" or cfg.get("jsScript"):
        errors.append(f"Node '{node.get('name')}' uses JS — change to TBEL")

if errors:
    print("VALIDATION ISSUES:")
    for e in errors: print(f"  - {e}")
else:
    print(f"PASS: {len(nodes)} nodes, {len(connections)} connections validated")
EOF
```

## Simulate MQTT Telemetry

After getting device credentials, simulate telemetry via HTTP:

```bash
# Get device access token first
ACCESS_TOKEN=$(curl -s "$TB_URL/api/device/$DEVICE_ID/credentials" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "import json,sys; print(json.load(sys.stdin)['credentialsId'])")

# Send normal telemetry (should NOT trigger alarms)
curl -s -X POST "$TB_URL/api/v1/$ACCESS_TOKEN/telemetry" \
  -H "Content-Type: application/json" \
  -d '{"temperature": 22, "humidity": 55, "co2": 400}'
echo "Sent normal telemetry"

# Send alarm-triggering telemetry
curl -s -X POST "$TB_URL/api/v1/$ACCESS_TOKEN/telemetry" \
  -H "Content-Type: application/json" \
  -d '{"temperature": 35, "humidity": 85, "co2": 1200}'
echo "Sent alarm-triggering telemetry"
```

## Verify Telemetry Was Received

```bash
# Query latest telemetry values
curl -s "$TB_URL/api/plugins/telemetry/DEVICE/$DEVICE_ID/values/timeseries?keys=temperature,humidity,co2" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -m json.tool
```

## Check Active Alarms

```bash
# List active alarms for a device
curl -s "$TB_URL/api/alarm/DEVICE/$DEVICE_ID?pageSize=10&page=0&searchStatus=ACTIVE" \
  -H "X-Authorization: Bearer $TB_API_KEY" | python3 -c "
import json,sys
data = json.load(sys.stdin)
alarms = data.get('data', [])
if not alarms:
    print('No active alarms')
else:
    for a in alarms:
        print(f\"{a['type']} | {a['severity']} | status={a['status']}\")
"
```

## MQTT Connectivity Test (using mosquitto_pub)

```bash
# Test MQTT connection with device access token
mosquitto_pub \
  -h $(echo $TB_URL | sed 's|https://||') \
  -p 1883 \
  -u "" \
  -P "$ACCESS_TOKEN" \
  -t "v1/devices/me/telemetry" \
  -m '{"temperature":25,"humidity":60}' \
  -d

# For TLS (port 8883):
mosquitto_pub \
  -h $(echo $TB_URL | sed 's|https://||') \
  -p 8883 \
  --cafile /etc/ssl/certs/ca-certificates.crt \
  -u "" \
  -P "$ACCESS_TOKEN" \
  -t "v1/devices/me/telemetry" \
  -m '{"temperature":25}' \
  -d
```

## Python MQTT Test Script

Generate and run a Python MQTT simulation test:

```python
import paho.mqtt.client as mqtt
import json, time

ACCESS_TOKEN = "<device-access-token>"
TB_HOST = "demo.thingsboard.io"
TB_PORT = 1883

def on_connect(client, userdata, flags, rc):
    print(f"Connected: rc={rc}")

client = mqtt.Client()
client.username_pw_set(ACCESS_TOKEN)
client.on_connect = on_connect
client.connect(TB_HOST, TB_PORT, 60)
client.loop_start()

# Test telemetry sequence
test_payloads = [
    {"temperature": 22, "humidity": 55},   # normal
    {"temperature": 28, "humidity": 70},   # warning threshold
    {"temperature": 35, "humidity": 85},   # critical threshold
    {"temperature": 20, "humidity": 50},   # clear alarm
]

for payload in test_payloads:
    client.publish("v1/devices/me/telemetry", json.dumps(payload))
    print(f"Sent: {payload}")
    time.sleep(2)

client.loop_stop()
```

## Test Report Format

After testing, report results in this format:

```
=== ThingsBoard CE Test Report ===
Target: $TB_URL
Time: <timestamp>

JSON Validation:
  PASS  templates/device-profiles/smart-building-profile.json
  PASS  templates/rule-chains/smart-building-rules.json
  FAIL  templates/dashboards/smart-building-dashboard.json (invalid JSON at line 42)

Profile Validation:
  PASS  smart-building-profile.json — 5 alarms, MQTT transport

Telemetry Test:
  PASS  Device received telemetry (temperature=35 visible in ThingsBoard)
  PASS  High Temperature alarm triggered (CRITICAL)
  PASS  Alarm cleared when temperature=20

Overall: 5/6 tests passed
```

## Common Test Scenarios by Use Case

**Smart Building**:
- Normal: `{"temperature":22,"humidity":55,"co2":400,"energy":1.2}`
- High Temp alarm: `{"temperature":31}` → expect CRITICAL alarm
- High CO2 alarm: `{"co2":1100}` → expect MAJOR alarm
- Clear: `{"temperature":20,"co2":300}`

**Fleet Tracking**:
- Normal: `{"latitude":40.7128,"longitude":-74.0060,"speed":60,"fuelLevel":75}`
- OverSpeed alarm: `{"speed":130}` → expect CRITICAL alarm
- Low Fuel alarm: `{"fuelLevel":5}` → expect WARNING alarm

**Industrial Monitoring**:
- Normal: `{"vibration":2,"temperature":45,"voltage":220,"pressure":5}`
- High Vibration: `{"vibration":15}` → expect CRITICAL alarm
- OverTemperature: `{"temperature":90}` → expect CRITICAL alarm

**Smart Agriculture**:
- Normal: `{"soilMoisture":55,"soilTemperature":20,"soilPH":6.5}`
- Low Moisture: `{"soilMoisture":25}` → expect WARNING alarm
- Frost Risk: `{"soilTemperature":1}` → expect CRITICAL alarm
