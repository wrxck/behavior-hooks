# Hook Schema Reference

Quick reference for writing Claude Code hooks. Use this when constructing hooks from user corrections.

## Hook Events

| Event | When | Use For |
|-------|------|---------|
| `PreToolUse` | Before tool executes | Blocking unwanted actions |
| `PostToolUse` | After successful tool | Checking output, advisory warnings |
| `PostToolUseFailure` | After tool fails | Cleanup, alternative suggestions |
| `UserPromptSubmit` | User sends message | Input validation |
| `Stop` | Claude stops responding | Cleanup, summaries |
| `PreCompact` | Before context compaction | Preserving important info |
| `PostCompact` | After compaction | Injecting reminders |
| `SessionStart` | New session begins | Setup, loading context |

## Hook Types

### Command Hook (most common)
```json
{
  "type": "command",
  "command": "bash -c '...'",
  "timeout": 30,
  "if": "Bash(git:*)"
}
```

### Prompt Hook (LLM-evaluated)
```json
{
  "type": "prompt",
  "prompt": "Check if $ARGUMENTS violates rule X. Respond with JSON.",
  "model": "claude-haiku-4-5-20251001"
}
```

### Agent Hook (full agent with tools)
```json
{
  "type": "agent",
  "prompt": "Verify that the edit at $ARGUMENTS follows the coding standards."
}
```

## stdin JSON Structure

Hooks receive JSON on stdin:

```json
{
  "session_id": "abc123",
  "tool_name": "Bash",
  "tool_input": {
    "command": "git push origin main"
  },
  "tool_response": { ... }  // PostToolUse only
}
```

## Output JSON

### Blocking (PreToolUse)
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "BLOCKED: reason here"
  }
}
```

### Advisory (PostToolUse)
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "WARNING: advisory message"
  }
}
```

### Force permission prompt (PreToolUse)
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "ask",
    "permissionDecisionReason": "This action needs confirmation: reason"
  }
}
```

## Common jq Extractions

```bash
# Get bash command
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

# Get file path from Edit/Write
FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

# Get MCP tool input field
APP=$(echo "$INPUT" | jq -r '.tool_input.app // empty')

# Get tool name
TOOL=$(echo "$INPUT" | jq -r '.tool_name // empty')
```

## `if` Filter Syntax

The `if` field uses permission rule syntax to avoid spawning the hook unnecessarily:

```
"if": "Bash(git push:*)"     # Only git push commands
"if": "Bash(npm:*)"          # Only npm commands
"if": "Edit(*.test.ts)"      # Only test files
"if": "Write(/etc/*)"        # Only /etc files
```

## Settings File Locations

| File | Scope | Committed? |
|------|-------|-----------|
| `~/.claude/settings.json` | All projects | N/A |
| `.claude/settings.json` | This project, all users | Yes |
| `.claude/settings.local.json` | This project, this user | No (.gitignore) |

Later files override earlier ones. Hooks merge across all files.

## Merging Rules

**Always merge, never replace.** Read existing file first.

If a hook already exists on the same event + matcher:
1. Show the existing hook to the user
2. Ask: keep, replace, or add alongside?
3. Never silently overwrite

## Testing Pattern

Always pipe-test before committing:

```bash
# Test a PreToolUse Bash hook
echo '{"tool_name":"Bash","tool_input":{"command":"git push origin main"}}' | <your-command>

# Test a PreToolUse MCP hook  
echo '{"tool_name":"mcp__fleet__fleet_deploy","tool_input":{"app":"my-app"}}' | <your-command>

# Test a PreToolUse Write hook
echo '{"tool_name":"Write","tool_input":{"file_path":"/path/to/file.ts"}}' | <your-command>
```

Verify: blocked case outputs deny JSON, allowed case outputs nothing.
