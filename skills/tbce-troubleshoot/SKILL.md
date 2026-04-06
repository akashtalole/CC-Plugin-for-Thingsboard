---
name: tbce-troubleshoot
description: Diagnose and fix ThingsBoard CE issues — device connectivity failures, missing or incorrect telemetry, alarms not triggering, rule chain errors, dashboard showing no data, or any unexpected ThingsBoard behavior. Use this skill when something is not working as expected in a ThingsBoard deployment.
argument-hint: "<issue description> (e.g. 'device offline', 'telemetry not showing up', 'high temperature alarm not firing', 'dashboard has no data for my devices')"
allowed-tools: Read, Bash
---

Invoke the `tbce-troubleshoot` agent to diagnose this issue: $ARGUMENTS

The agent follows a systematic diagnostic process:

1. **Verify environment** — check `TB_URL` and `TB_API_KEY`, test API connectivity
2. **Locate the entity** — find device/profile/dashboard by name via REST API
3. **Collect diagnostics** — query relevant API endpoints for state information
4. **Identify root cause** — match symptoms to known ThingsBoard issues
5. **Provide fix** — specific steps or commands to resolve the issue

**Issue categories and typical fixes:**

| Symptom | Common Cause |
|---------|-------------|
| Device inactive | Wrong MQTT credentials or topic |
| No telemetry | Rule chain missing Save Timeseries node |
| Alarm not firing | Telemetry key case mismatch; missing Device Profile node |
| Dashboard no data | entityAlias deviceType doesn't match profile name |
| Rule chain errors | Failure outputs unconnected; JS instead of TBEL |
| MQTT connect fails | Using username instead of access token as password |

**MQTT connection checklist:**
- Host: `demo.thingsboard.io`
- Port: 1883 (plain) or 8883 (TLS)
- Username: `""` (empty)
- Password: device access token from `GET /api/device/{id}/credentials`
- Topic: `v1/devices/me/telemetry`
