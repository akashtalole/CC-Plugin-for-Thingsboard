# CC-Plugin-for-ThingsBoard

A Claude Code and GitHub Copilot plugin that accelerates IoT solution development on **ThingsBoard Community Edition**. Inspired by [skills-for-copilot-studio](https://github.com/microsoft/skills-for-copilot-studio), this plugin provides four specialized AI agents and ready-to-import JSON templates for the most common IoT use cases.

## Features

- **4 Specialized Agents** — author, manage, test, and troubleshoot ThingsBoard solutions using natural language
- **4 Use Case Templates** — Smart Building, Fleet Tracking, Industrial Monitoring, Smart Agriculture
- **Production-ready JSON** — all templates follow ThingsBoard CE best practices and import directly to [demo.thingsboard.io](https://demo.thingsboard.io)
- **MQTT-first** — all device profiles use MQTT transport with proper topic configuration
- **TBEL rule chains** — rule nodes use ThingsBoard Expression Language (not deprecated JavaScript)

## Quick Start

### 1. Install plugin

```bash
# Clone into your Claude Code plugins directory
git clone https://github.com/akashtalole/CC-Plugin-for-Thingsboard ~/.claude/plugins/thingsboard
```

### 2. Set environment variables

```bash
export TB_URL=https://demo.thingsboard.io
export TB_API_KEY=<your-api-key>          # For REST API calls
export TB_USERNAME=<your-email>            # For MCP server
export TB_PASSWORD=<your-password>         # For MCP server
```

### 3. Use an agent

In Claude Code or GitHub Copilot chat:

```
/tbce-author create a smart building device profile with temperature, humidity and CO2 sensors
```

```
/tbce-manage import smart-building templates to demo.thingsboard.io
```

```
/tbce-test validate templates/device-profiles/smart-building-profile.json
```

```
/tbce-troubleshoot my device is not sending telemetry
```

## Agents

### tbce-author
Creates ThingsBoard components as valid JSON files ready for import.

**Examples:**
- `create a fleet tracking device profile with GPS and speed monitoring`
- `design a dashboard for industrial machine monitoring with OEE metrics`
- `write a rule chain for smart agriculture with irrigation automation`

### tbce-manage
Deploys and manages configurations on demo.thingsboard.io via REST API.

**Examples:**
- `import all smart building templates`
- `list all device profiles on my ThingsBoard instance`
- `export the dashboard named "Fleet Overview" to file`

### tbce-test
Validates configurations and simulates IoT device behavior.

**Examples:**
- `validate all templates in templates/device-profiles/`
- `simulate MQTT telemetry for a smart building sensor`
- `test that the high temperature alarm triggers at 31°C`

### tbce-troubleshoot
Diagnoses issues with ThingsBoard devices and configurations.

**Examples:**
- `my device shows as inactive even though it's sending data`
- `the high temperature alarm is not triggering`
- `dashboard widgets show "No data" for my devices`

## Templates

All templates are importable into demo.thingsboard.io.

| Template Set | Components |
|-------------|------------|
| **Smart Building** | Sensor profile (temp/humidity/CO2/energy), processing rule chain, monitoring dashboard, Building→Floor→Room asset hierarchy |
| **Fleet Tracking** | Vehicle profile (GPS/speed/fuel/engine), geofencing rule chain, map dashboard, Fleet→Vehicle asset hierarchy |
| **Industrial Monitoring** | Machine profile (vibration/temp/voltage/pressure), predictive maintenance rule chain, OEE dashboard, Factory→Line→Machine assets |
| **Smart Agriculture** | Field sensor profile (soil/weather/irrigation), automation rule chain, field dashboard, Farm→Field→Zone assets |

### Import a complete use case

```bash
# 1. Import device profile
curl -s -X POST $TB_URL/api/deviceProfile \
  -H "X-Authorization: Bearer $TB_API_KEY" \
  -H "Content-Type: application/json" \
  -d @templates/device-profiles/smart-building-profile.json

# 2. Import rule chain
curl -s -X POST $TB_URL/api/ruleChain/import \
  -H "X-Authorization: Bearer $TB_API_KEY" \
  -H "Content-Type: application/json" \
  -d @templates/rule-chains/smart-building-rules.json

# 3. Import dashboard
curl -s -X POST $TB_URL/api/dashboard \
  -H "X-Authorization: Bearer $TB_API_KEY" \
  -H "Content-Type: application/json" \
  -d @templates/dashboards/smart-building-dashboard.json
```

## MCP Server

This plugin references the official ThingsBoard MCP server. See `.mcp.json` for configuration.

```bash
# Install ThingsBoard MCP server (when available via npm)
npx thingsboard-mcp
```

## Project Structure

```
CC-Plugin-for-Thingsboard/
├── CLAUDE.md                    # Plugin instructions for Claude
├── .mcp.json                    # ThingsBoard MCP server config
├── .claude/
│   └── agents/
│       ├── tbce-author.md
│       ├── tbce-manage.md
│       ├── tbce-test.md
│       └── tbce-troubleshoot.md
├── skills/
│   ├── tbce-author/SKILL.md
│   ├── tbce-manage/SKILL.md
│   ├── tbce-test/SKILL.md
│   └── tbce-troubleshoot/SKILL.md
└── templates/
    ├── device-profiles/         # 4 MQTT device profiles with alarm rules
    ├── rule-chains/             # 4 TBEL rule chains
    ├── dashboards/              # 4 monitoring dashboards
    └── assets/                  # 4 asset hierarchy definitions
```

## License

MIT — see [LICENSE](LICENSE)
