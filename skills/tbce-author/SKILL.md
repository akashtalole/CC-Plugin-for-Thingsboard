---
name: tbce-author
description: Author ThingsBoard CE components — device profiles, dashboards, rule chains, assets. Creates valid JSON templates ready to import into demo.thingsboard.io. Use this skill when the user asks to create, design, or generate any ThingsBoard configuration component.
argument-hint: "<component-type> for <use-case> (e.g. 'device profile for smart building', 'dashboard for fleet tracking', 'rule chain with temperature alarm')"
allowed-tools: Read, Write, Bash
---

Invoke the `tbce-author` agent to handle this request: $ARGUMENTS

The agent will:
1. Determine the component type (device profile / dashboard / rule chain / asset / device)
2. Read any existing templates from `templates/` that are relevant
3. Generate valid ThingsBoard CE JSON following best practices:
   - MQTT transport for device profiles
   - Alarm rules with both createRules and clearRule
   - TBEL (not JavaScript) in rule chain nodes
   - entityAliases using deviceSearchQuery in dashboards
4. Save the output to the appropriate `templates/` subdirectory
5. Show the curl command to import it to demo.thingsboard.io

**Template naming convention:**
- Device profiles: `templates/device-profiles/<use-case>-profile.json`
- Rule chains: `templates/rule-chains/<use-case>-rules.json`
- Dashboards: `templates/dashboards/<use-case>-dashboard.json`
- Assets: `templates/assets/<use-case>-assets.json`
