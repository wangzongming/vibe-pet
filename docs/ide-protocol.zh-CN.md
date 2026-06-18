# IDE / Agent 协议

这份文档说明 IDE、CLI Agent、hook 或插件如何把活动状态上报到 Vibe Pet 本地 bridge。

协议是本地优先的。集成端只需要把规范化后的状态 POST 到 `127.0.0.1`，bridge 会把这些会话转换为主窗口桌宠、桌面浮动桌宠和硬件状态包。

## 传输方式

| 项目 | 值 |
| --- | --- |
| 默认 bridge 地址 | `http://127.0.0.1:17384` |
| 运行时文件 | `~/.code-pet/runtime.json` |
| 端口覆盖 | `CODE_PET_PORT` |
| 事件写入接口 | `POST /api/hook` |
| 兼容写入接口 | `POST /state` |
| 快照接口 | `GET /api/snapshot` |
| 事件流 | `GET /api/events` |
| Content-Type | `application/json; charset=utf-8` |

Vibe Pet 启动后会把当前 bridge 端口写入 `~/.code-pet/runtime.json`。hook 查找端口时建议按这个顺序：`CODE_PET_PORT`、运行时文件、默认端口范围 `17384` 到 `17388`。

## 事件负载

向 `POST /api/hook` 发送 JSON 对象：

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

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `agentId` | string | 是 | 稳定的集成 id，例如 `codex`、`cursor`、`windsurf` 或自定义 id。 |
| `agentName` | string | 否 | 在桌面应用中显示的名称。 |
| `sessionId` | string | 否 | 稳定的会话 id，默认是 `default`。 |
| `state` | string | 是 | 规范化后的桌宠状态，未知值会变成 `idle`。 |
| `event` | string | 否 | 来源事件名，用于调试和展示。 |
| `cwd` | string | 否 | 工作区或项目路径。 |
| `cwdBasename` | string | 否 | 项目目录名；不传时 bridge 会从 `cwd` 推导。 |
| `title` | string | 否 | 会话标题、prompt 摘要、工作区标题或任务标题。 |
| `sessionTitle` | string | 否 | `title` 的别名。 |
| `output` | string | 否 | 最近的 Agent 输出，会显示在桌宠卡片中，最长保留 1200 个字符。 |
| `message` | string | 否 | 当 `output` 缺失时作为输出内容使用。 |
| `text` | string | 否 | 当 `output` 和 `message` 缺失时作为输出内容使用。 |
| `source` | string | 否 | 集成来源，例如 `codex-official`、`codex-log`、`cursor-composer`。 |
| `activityUpdatedAt` | number | 否 | 来源活动时间戳，单位毫秒，用于排序。 |
| `sourceUpdatedAt` | number | 否 | `activityUpdatedAt` 的别名。 |

bridge 的会话 key 是 `agentId:sessionId`。如果一个 IDE 同时有多个真实会话，需要为每个会话提供不同的 `sessionId`，这样 Vibe Pet 才能做到一个会话一个桌宠。

## 状态

| 状态 | 含义 |
| --- | --- |
| `idle` | Agent 已打开，但当前没有执行任务。 |
| `thinking` | Agent 正在读取上下文、规划或准备回复。 |
| `working` | Agent 正在调用工具、编辑文件、读取文件或运行命令。 |
| `typing` | Agent 正在输出文本。 |
| `building` | 多个工作型任务同时活跃，或正在执行构建类任务。 |
| `juggling` | Agent 正在处理多个子任务或 subagent。 |
| `attention` | 当前轮次完成，或需要用户轻微关注。 |
| `notification` | 需要审批、授权或重要用户操作。 |
| `permission` | 兼容状态；给硬件时会规范化为 `notification`。 |
| `error` | Agent 或 hook 出错。 |
| `sweeping` | Agent 正在压缩、整理或清理上下文。 |
| `sleeping` | 会话结束，或应用存在但不活跃。 |

`codex-permission` 也会被接受，并在发送到硬件时规范化为 `notification`。

## 内置集成

`npm install`、`npm start` 和 `npm run dev` 默认会安装或同步 hooks。也可以手动执行：

```bash
npm run install:hooks
```

| Agent | 集成位置 | 主要来源 |
| --- | --- | --- |
| Codex | `~/.codex/hooks.json`、`~/.codex/config.toml`，以及 JSONL 会话监听 | `codex-official`、`codex-log` |
| Cursor | `~/.cursor/hooks.json`、Cursor composer 数据库、transcript 监听 | `cursor-hook`、`cursor-composer`、`cursor-transcript` |
| Windsurf | `~/.codeium/windsurf/hooks.json` | `windsurf-hook` |
| Claude CLI / Claude Code | `~/.claude/settings.json` | `claude-code-hook` |
| Gemini CLI | `~/.gemini/settings.json` | `gemini-cli-hook` |
| Copilot CLI | `$COPILOT_HOME/hooks/hooks.json` 或 `~/.copilot/hooks/hooks.json` | `copilot-cli-hook` |
| CodeBuddy | `~/.codebuddy/settings.json` | `codebuddy-hook` |
| Kimi Code CLI | `~/.kimi/config.toml` | `kimi-cli-hook` |
| Qwen Code | `~/.qwen/settings.json` | `qwen-code-hook` |
| Qoder | `~/.qoder/settings.json` | `qoder-hook` |
| Reasonix CLI | `~/.reasonix/settings.json` | `reasonix-hook` |
| OpenClaw | OpenClaw 配置中的插件路径 | `openclaw-plugin` |
| opencode | `~/.config/opencode/opencode.json` 中的插件路径 | `opencode-plugin` |
| Hermes Agent | 插件复制到 `~/.hermes/plugins/code-pet` | `hermes-plugin` |

## 常见事件映射

自定义集成不必使用完全相同的事件名，但内置 hooks 大致按下面的方式映射：

| 来源事件示例 | 规范化状态 |
| --- | --- |
| `SessionStart` | `idle` |
| `SessionEnd` | `sleeping` |
| `UserPromptSubmit`、`beforeSubmitPrompt`、`BeforeAgent`、`pre_user_prompt` | `thinking` |
| `PreToolUse`、`BeforeTool`、`pre_write_code`、`pre_run_command` | `working` |
| `PostToolUse`、`AfterTool`、`post_write_code`、`post_run_command` | 依来源映射为 `working` 或 `thinking` |
| `SubagentStart` | `juggling` |
| `Stop`、`AfterAgent`、`post_cascade_response` | 依来源映射为 `attention` 或 `idle` |
| `PermissionRequest`、`Notification`、`Elicitation` | `notification` |
| `PreCompact`、`PreCompress`、`event_msg:context_compacted` | `sweeping` |
| `PostToolUseFailure`、`StopFailure`、`errorOccurred` | `error` |

## 读取状态

使用 `GET /api/snapshot` 获取当前规范化状态：

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

使用 `GET /api/events` 可以订阅 SSE。第一条消息是完整快照，后续消息包含 `{ type, at, item, aggregate, sessions }`。

## 隐私边界

IDE 集成可以为桌面 UI 上报本地标题和简短输出摘要。硬件协议只会收到适合显示的小字段，不会收到 prompt、工具参数、审批动作或完整 transcript。

## 最小自定义集成

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
