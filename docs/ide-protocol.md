# IDE / Agent Protocol

This document describes how an IDE, CLI agent, hook, or plugin reports activity to the Vibe Pet local bridge.

The protocol is local-first. Integrations post normalized state to the bridge on `127.0.0.1`, and the bridge turns those sessions into desktop pets, floating pets, and hardware packets.

## Transport

| Item | Value |
| --- | --- |
| Default bridge URL | `http://127.0.0.1:17384` |
| Runtime file | `~/.code-pet/runtime.json` |
| Port override | `CODE_PET_PORT` |
| Event ingest endpoint | `POST /api/hook` |
| Legacy ingest endpoint | `POST /state` |
| Snapshot endpoint | `GET /api/snapshot` |
| Event stream | `GET /api/events` |
| Content type | `application/json; charset=utf-8` |

When Vibe Pet starts, it writes the active bridge port to `~/.code-pet/runtime.json`. Hooks should prefer `CODE_PET_PORT`, then the runtime file, then the default port range `17384` to `17388`.

## Event Payload

Send a JSON object to `POST /api/hook`:

```json
{
  "agentId": "my-ide",
  "agentName": "My IDE",
  "sessionId": "conversation-123",
  "state": "thinking",
  "event": "UserPromptSubmit",
  "cwd": "/Users/me/work/vibe-pet",
  "title": "Add Wio Terminal firmware",
  "output": "Planning firmware animation frames...",
  "source": "my-ide-hook",
  "activityUpdatedAt": 1781500000000
}
```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `agentId` | string | Yes | Stable integration id, such as `codex`, `cursor`, `windsurf`, or a custom id. |
| `agentName` | string | No | Human-readable label shown in the desktop app. |
| `sessionId` | string | No | Stable conversation/session id. Defaults to `default`. |
| `state` | string | Yes | Normalized pet state. Unknown values become `idle`. |
| `event` | string | No | Source event name, useful for debugging and display. |
| `cwd` | string | No | Workspace or project path. |
| `cwdBasename` | string | No | Project basename. If omitted, the bridge derives it from `cwd`. |
| `title` | string | No | Session title, prompt summary, workspace title, or task title. |
| `sessionTitle` | string | No | Alias for `title`. |
| `output` | string | No | Recent agent output shown in the desktop card. Truncated to 1200 characters. |
| `message` | string | No | Alias used as output when `output` is missing. |
| `text` | string | No | Alias used as output when `output` and `message` are missing. |
| `source` | string | No | Integration source, for example `codex-official`, `codex-log`, or `cursor-composer`. |
| `activityUpdatedAt` | number | No | Source activity timestamp in milliseconds. Used for ordering. |
| `sourceUpdatedAt` | number | No | Alias for `activityUpdatedAt`. |

The bridge session key is `agentId:sessionId`. Use one session id per active IDE conversation so Vibe Pet can render one pet per real editor or agent session.

## States

| State | Meaning |
| --- | --- |
| `idle` | The agent is open but not actively working. |
| `thinking` | The agent is reading context, planning, or preparing a response. |
| `working` | The agent is using tools, editing files, reading files, or running commands. |
| `typing` | The agent is producing text. |
| `building` | Multiple work-like activities are active or a build-like task is running. |
| `juggling` | The agent is handling multiple subtasks or subagents. |
| `attention` | The latest turn completed or needs light user attention. |
| `notification` | The agent needs approval, permission, or important user action. |
| `permission` | Accepted for compatibility; normalized to `notification` for devices. |
| `error` | The agent or hook encountered an error. |
| `sweeping` | The agent is compacting or cleaning context. |
| `sleeping` | The session ended or the app is present but inactive. |

`codex-permission` is also accepted and normalized to `notification` for hardware.

## Built-In Integrations

`npm install`, `npm start`, and `npm run dev` run hook installation by default. Manual install:

```bash
npm run install:hooks
```

| Agent | Integration path | Main source |
| --- | --- | --- |
| Codex | `~/.codex/hooks.json`, `~/.codex/config.toml`, plus JSONL session monitor | `codex-official`, `codex-log` |
| Cursor | `~/.cursor/hooks.json`, Cursor composer database, transcript monitor | `cursor-hook`, `cursor-composer`, `cursor-transcript` |
| Windsurf | `~/.codeium/windsurf/hooks.json` | `windsurf-hook` |
| Claude CLI / Claude Code | `~/.claude/settings.json` | `claude-code-hook` |
| Gemini CLI | `~/.gemini/settings.json` | `gemini-cli-hook` |
| Copilot CLI | `$COPILOT_HOME/hooks/hooks.json` or `~/.copilot/hooks/hooks.json` | `copilot-cli-hook` |
| CodeBuddy | `~/.codebuddy/settings.json` | `codebuddy-hook` |
| Kimi Code CLI | `~/.kimi/config.toml` | `kimi-cli-hook` |
| Qwen Code | `~/.qwen/settings.json` | `qwen-code-hook` |
| Qoder | `~/.qoder/settings.json` | `qoder-hook` |
| Reasonix CLI | `~/.reasonix/settings.json` | `reasonix-hook` |
| OpenClaw | Plugin path in OpenClaw config | `openclaw-plugin` |
| opencode | Plugin path in `~/.config/opencode/opencode.json` | `opencode-plugin` |
| Hermes Agent | Plugin copied to `~/.hermes/plugins/code-pet` | `hermes-plugin` |

## Common Event Mapping

Integrations do not need to use these exact event names, but the built-in hooks map them this way:

| Source event examples | Normalized state |
| --- | --- |
| `SessionStart` | `idle` |
| `SessionEnd` | `sleeping` |
| `UserPromptSubmit`, `beforeSubmitPrompt`, `BeforeAgent`, `pre_user_prompt` | `thinking` |
| `PreToolUse`, `BeforeTool`, `pre_write_code`, `pre_run_command` | `working` |
| `PostToolUse`, `AfterTool`, `post_write_code`, `post_run_command` | `working` or `thinking` depending on source |
| `SubagentStart` | `juggling` |
| `Stop`, `AfterAgent`, `post_cascade_response` | `attention` or `idle` depending on source |
| `PermissionRequest`, `Notification`, `Elicitation` | `notification` |
| `PreCompact`, `PreCompress`, `event_msg:context_compacted` | `sweeping` |
| `PostToolUseFailure`, `StopFailure`, `errorOccurred` | `error` |

## Reading State Back

Use `GET /api/snapshot` for the current normalized state:

```json
{
  "aggregate": {
    "state": "working",
    "agent": "Codex",
    "agentId": "codex",
    "event": "response_item:function_call",
    "activeCount": 1,
    "sessionCount": 2,
    "title": "Add protocol docs",
    "cwdBasename": "vibe-pet",
    "output": "Editing docs..."
  },
  "sessions": []
}
```

Use `GET /api/events` for server-sent events. The first message is a full snapshot; later messages include `{ type, at, item, aggregate, sessions }`.

## Privacy Boundary

IDE integrations may report local titles and short output summaries for the desktop UI. The hardware protocol receives only compact display fields and does not receive prompts, tool arguments, approval actions, or full transcripts.

## Minimal Custom Integration

```bash
curl -X POST http://127.0.0.1:17384/api/hook \
  -H 'content-type: application/json' \
  -d '{
    "agentId": "my-agent",
    "agentName": "My Agent",
    "sessionId": "demo",
    "state": "working",
    "event": "PreToolUse",
    "title": "Demo session",
    "output": "Reading files"
  }'
```
