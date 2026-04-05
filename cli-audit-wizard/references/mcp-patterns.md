# MCP Patterns -- Exposing Wizards as MCP Tools

> **When to read:** When auditing multi-surface consistency (Step 4) or when the project exposes config via MCP.
>
> **Staleness:** Verified 2026-04-05. MCP spec is evolving rapidly (Apps spec Jan 2026, Registry Q4 2026). Tool naming conventions (`wizard_*`) are skill-specific patterns, not MCP standards. Verify current MCP capabilities before recommending specific features.

---

## Core principle

The MCP surface calls the same wizard engine as CLI and web. It writes the same TOML config file. JSON is transport only -- never on disk, never user-facing.

```
CLI (Huh/Gum)           -+
MCP tool calls           -+---> wizard engine ---> config.toml ---> app reload
Web UI (Caddy-served)    -+
```

---

## Standard MCP tool set for config wizards

### Read tools

| Tool name | Input | Output | Idempotent |
|-----------|-------|--------|------------|
| `wizard_get_config` | (none) | Full config as JSON struct | Yes |
| `wizard_get_section` | `section: string` | One config section as JSON | Yes |
| `wizard_get_schema` | (none) | Config schema with types, defaults, descriptions | Yes |

### Write tools

| Tool name | Input | Output | Idempotent |
|-----------|-------|--------|------------|
| `wizard_set_{section}` | Section-specific fields | `{ ok: bool, diff: string }` | Yes |
| `wizard_remove_{key}` | `key: string` | `{ ok: bool, diff: string }` | Yes |

### Credential tools

| Tool name | Input | Output | Idempotent |
|-----------|-------|--------|------------|
| `wizard_check_credentials` | (none) | `{ providers: [{name, status, expires_at, source}] }` | Yes |
| `wizard_discover_credential` | `service: string` | `{ found: bool, source: string, valid: bool }` | Yes |
| `wizard_set_credential` | `service: string, ref: string` | `{ ok: bool, valid: bool, diff: string }` | Yes |
| `wizard_start_oauth` | `service: string` | `{ auth_url: string, device_code?: string }` | No |

### Lifecycle tools

| Tool name | Input | Output | Idempotent |
|-----------|-------|--------|------------|
| `wizard_run_doctor` | (none) | `{ checks: [{name, status, message, fix}] }` | Yes |
| `wizard_diff` | (none) | Pending changes as unified diff | Yes |
| `wizard_apply` | (none) | `{ ok: bool, reloaded: bool }` | Yes |
| `wizard_migrate` | `from_version: string` | `{ ok: bool, diff: string, breaking: [string] }` | No |

---

## Design rules for MCP wizard tools

### 1. Every tool is idempotent (except migrate)

Calling `wizard_set_backend(name="mistral", url="...")` twice produces the same config. The agent can retry safely.

### 2. Write tools return diffs

Every mutation tool returns a diff showing what changed. The agent (or human reviewing agent actions) can see exactly what happened.

```json
{
  "ok": true,
  "diff": "+[backends.mistral]\n+url = \"https://api.mistral.ai\"\n+model = \"mistral-large\""
}
```

### 3. Doctor before apply

The recommended agent flow is:
1. `wizard_get_config()` -- understand current state
2. `wizard_set_*()` -- make changes (returns diff each time)
3. `wizard_run_doctor()` -- verify health
4. `wizard_apply()` -- reload only if doctor passes

### 4. Schema tool enables autonomous config

`wizard_get_schema()` returns enough metadata for an agent to configure the tool without human guidance:

```json
{
  "backends": {
    "type": "map",
    "value_schema": {
      "url": { "type": "string", "required": true, "description": "Backend API endpoint" },
      "model": { "type": "string", "required": true, "description": "Model name" },
      "api_key_env": { "type": "string", "default": "API_KEY", "description": "Env var containing API key" }
    }
  }
}
```

### 5. No wizard_setup tool

There is no `wizard_setup` tool. Setup is just a sequence of `wizard_set_*` calls. The agent decides what to ask the user, not the MCP tool.

---

## MCP Apps (2026 spec)

MCP Apps allow tools to return interactive UI components rendered as sandboxed iframes in the conversation:

```json
{
  "name": "wizard_configure",
  "inputSchema": {},
  "_meta": {
    "ui": {
      "resourceUri": "ui://wizard/config-form"
    }
  }
}
```

This means a config wizard can render an interactive form directly in the chat, collect input, validate, and write config -- all without leaving the conversation.

**Client support (as of 2026):** Claude (web + desktop), Goose, VS Code Insiders, ChatGPT (rolling out).

---

## Audit checklist for MCP surface

```
[ ] All MCP tools call the same engine as CLI
[ ] Write tools return diffs
[ ] Config written as TOML (not JSON on disk)
[ ] Doctor tool exists and returns structured checks
[ ] Schema tool exists for autonomous agent config
[ ] Tools are idempotent (safe to retry)
[ ] Apply tool validates before reloading
[ ] Tool names follow wizard_{action} convention
```

---

## Reference implementations

| Project | Pattern | Notes |
|---------|---------|-------|
| mcp-manager | Web GUI for MCP server management | Auto-scanning, interactive setup wizard |
| continuedev-hub-config-wizard | Reusable AI assistant configs | Models, rules, prompts published via Continue Dev Hub |
| mssql-mcp-config-builder | Step-by-step config UI | Visual builder for mcp_config.json |
