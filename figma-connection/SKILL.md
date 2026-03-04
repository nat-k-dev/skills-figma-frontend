---
name: figma-connection
description: Use this skill before executing ANY Figma-related task. Applies when the user asks anything about Figma design data, colors, tokens, components, selections, variables, or any task that requires accessing Figma. Also applies when the user says "check figma connection", "is figma connected", "set up figma", or similar.
version: 1.0.0
---

# Figma Desktop Bridge Connection

The Figma MCP server requires the **Figma Desktop Bridge plugin** to be running and connected before any design data can be accessed. Without it, no Figma tools work.

## CRITICAL: Never use Figma REST API

**Never use REST API-based tools** (`figma_get_file_data`, `figma_get_component`, `figma_get_styles`, `figma_get_variables` with REST fallback, etc.) as a workaround when the Desktop Bridge is not connected.

The REST API:
- Cannot resolve variable alias chains (token resolution is broken)
- Cannot detect per-fill or per-stroke visibility
- Returns stale cloud data, not live plugin state
- Cannot access selection, real-time changes, or plugin data

If the Desktop Bridge is not connected, **ask the user to connect it**. Do not silently fall back to the REST API.

## How to check connection

Always call `figma_get_status` first. Check:
- `setup.valid === true` → connected, proceed
- `setup.valid === false` → not connected, stop and help the user

## How to set up the Desktop Bridge (guide for the user)

If not connected, walk the user through these steps:

### First-time setup
1. Open **Figma Desktop** (not the browser version)
2. Enable **Dev Mode** (toggle in the top-right toolbar)
3. Enable the developer VM: go to **Plugins → Development → Use developer VM** and turn it on
4. Go to **Plugins → Development → Import plugin from manifest**
5. Select the manifest file at:
   `<project-root>/figma-desktop-bridge/manifest.json`
6. Go to **Plugins → Development → Figma Desktop Bridge**
7. Click to run it — a panel appears showing **"MCP ready"**
8. Keep the panel open while using the MCP server

### Subsequent sessions
1. Open Figma Desktop
2. Go to **Plugins → Development → Figma Desktop Bridge**
3. Run it — wait for **"MCP ready"**

### If "MCP ready" shows but connection still fails
The MCP server may be running on a fallback port. Re-import the plugin from the manifest to get the latest multi-port scanning version, then run it again.

## Port behavior

The MCP server prefers port `9223`. If that port is taken by another Claude Code instance, it falls back to `9224`, `9225`, etc. (up to `9232`). The Desktop Bridge plugin scans all ports in this range automatically — but only if it was imported recently enough to have multi-port support.

If `portFallbackUsed: true` appears in status and the plugin won't connect, re-importing the plugin manifest fixes it.

## After confirming connection

Once `setup.valid === true`, report:
- Connected file name
- Current page
- Active transport (websocket)
- Port in use

Then proceed with the user's actual Figma task using `figma_execute` for all data access.
