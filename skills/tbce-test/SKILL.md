---
name: tbce-test
description: Test and validate ThingsBoard CE configurations — check JSON file validity, simulate MQTT telemetry, verify that alarms trigger correctly, test rule chain logic, and confirm dashboard data sources resolve. Use this skill before importing templates or after deployment to verify everything works.
argument-hint: "<what to test> (e.g. 'validate all templates', 'simulate smart building telemetry', 'test high temperature alarm', 'check fleet tracking rule chain')"
allowed-tools: Read, Bash
---

Invoke the `tbce-test` agent to handle this request: $ARGUMENTS

The agent will run the appropriate tests:

**JSON Validation** (no ThingsBoard connection needed):
- Syntax check all JSON files in `templates/`
- Validate device profile alarm rule structure (createRules + clearRule)
- Validate rule chain node/connection integrity
- Check TBEL is used instead of JavaScript

**Live Tests** (requires `TB_URL` + `TB_API_KEY`):
- Send test telemetry via HTTP API (`/api/v1/{ACCESS_TOKEN}/telemetry`)
- Query timeseries to verify data was received
- Check alarm creation after threshold-crossing telemetry
- Check alarm clearance after normal telemetry
- Generate MQTT test commands (`mosquitto_pub`)

**Test report format:**
```
PASS/FAIL  <test-name>  [details]
```

Each test scenario per use case:
- **Smart Building**: temperature, humidity, CO2, energy
- **Fleet Tracking**: GPS coordinates, speed, fuel level
- **Industrial**: vibration, voltage, pressure, run hours
- **Agriculture**: soil moisture, soil temperature, soil pH
