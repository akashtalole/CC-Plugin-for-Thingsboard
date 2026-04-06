---
name: tbce-manage
description: Deploy and manage ThingsBoard CE configurations — import JSON templates to demo.thingsboard.io, export existing configs, list entities (devices, profiles, dashboards, rule chains, assets), and delete entities. Use this skill when the user wants to push configs to ThingsBoard, sync local files with a live instance, or manage entities via API.
argument-hint: "<action> [target] (e.g. 'import smart-building templates', 'list all device profiles', 'export dashboard Fleet Overview', 'deploy all templates')"
allowed-tools: Read, Bash
---

Invoke the `tbce-manage` agent to handle this request: $ARGUMENTS

Before starting, the agent will verify that `TB_URL` and `TB_API_KEY` are set:

```bash
if [ -z "$TB_URL" ] || [ -z "$TB_API_KEY" ]; then
  echo "ERROR: Required environment variables not set."
  echo "  export TB_URL=https://demo.thingsboard.io"
  echo "  export TB_API_KEY=<your-api-key>"
  exit 1
fi
```

**Import order for a complete use case** (to avoid dependency errors):
1. Device profile
2. Rule chain (via `/api/ruleChain/import`)
3. Assets
4. Dashboard
5. Devices

**Template locations:**
- `templates/device-profiles/` — MQTT device profiles with alarm rules
- `templates/rule-chains/` — TBEL rule chains
- `templates/dashboards/` — monitoring dashboards
- `templates/assets/` — asset hierarchy definitions
