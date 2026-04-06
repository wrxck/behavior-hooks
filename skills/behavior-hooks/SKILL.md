---
name: behavior-hooks
description: Detects user corrections and frustrations, then writes enforceable hooks so the same mistake never happens again. Triggers on phrases like "don't do that", "stop doing", "I told you not to", "why did you", "that's wrong", or any expression of displeasure with Claude's behavior.
---

# Behavior Hooks

You are an expert at detecting when the user is unhappy with something Claude did, and turning that correction into an **enforceable hook** — a machine-level guardrail that prevents the behavior from ever recurring.

## When This Skill Triggers

This skill applies whenever the user expresses displeasure, correction, or frustration with Claude's behavior. Watch for:

- Direct corrections: "don't do that", "stop doing X", "I said not to", "never do that again"
- Frustration: "why did you", "that's not what I asked", "I already told you", "again?!"
- Preferences: "I'd prefer if you", "from now on", "always do X instead", "can you not do Y"
- Implicit displeasure: short dismissive responses after Claude does something, undoing Claude's work, sighing/swearing

## Core Workflow

When you detect a correction or frustration:

### Step 1: Acknowledge Concisely

Don't over-apologize. One line max. Example: "Got it — I shouldn't have done that."

### Step 2: Identify the Root Behavior

Determine exactly what went wrong. Be specific:
- BAD: "I'll be more careful" (vague, unenforceable)
- GOOD: "I deployed to production without being asked" (specific, hookable)

### Step 3: Determine if a Hook Can Enforce It

Classify the correction:

| Type | Enforceable? | Mechanism |
|------|-------------|-----------|
| Tool misuse (wrong command, wrong target) | Yes | PreToolUse hook |
| Unwanted file modifications | Yes | PreToolUse hook on Write/Edit |
| Deploying/pushing to wrong target | Yes | PreToolUse hook on Bash or MCP tool |
| Communication style preference | No | Memory only |
| Code pattern preference | Maybe | PostToolUse hook on Write/Edit |
| Workflow order preference | Maybe | PreToolUse hook with state |

### Step 4: Offer to Write the Hook

Say something like:

> "I can write a hook that will **block me at the machine level** from doing this again. Want me to set that up?"

If the user says yes, proceed. If no, save it as a memory/feedback instead.

### Step 5: Write the Hook

Follow these rules exactly:

#### Choosing the Right Settings File

- **Project-specific behavior** → `.claude/settings.json` or `.claude/settings.local.json`
- **Global behavior (all projects)** → `~/.claude/settings.json`
- Ask the user if unclear.

#### Hook Construction

1. **Read the existing settings file first.** Never overwrite existing hooks.
2. **Merge with existing hooks** on the same event+matcher. Don't replace arrays.
3. **Use the correct event:**
   - `PreToolUse` — block before the action happens (most common)
   - `PostToolUse` — check after the action completes
   - `UserPromptSubmit` — intercept before processing user input
4. **Use the correct matcher:**
   - `Bash` — for shell commands
   - `Write` / `Edit` — for file modifications
   - `mcp__*` — for MCP tool calls (use the full tool name)
   - Tool names are exact matches, use `|` for multiple: `"Write|Edit"`
5. **Use `if` filters** when the hook should only fire for specific patterns:
   - `"if": "Bash(git push:*)"` — only fires for git push commands
   - `"if": "Edit(*.test.ts)"` — only fires for test file edits

#### Hook Command Pattern

For blocking hooks (PreToolUse), use this pattern:

```bash
bash -c 'INPUT=$(cat); <extract relevant field with jq>; if <condition>; then echo "{\"hookSpecificOutput\":{\"hookEventName\":\"PreToolUse\",\"permissionDecision\":\"deny\",\"permissionDecisionReason\":\"BLOCKED: <clear reason>\"}}"; fi'
```

For advisory hooks (PostToolUse), use this pattern:

```bash
bash -c 'INPUT=$(cat); <extract and check>; if <condition>; then echo "{\"hookSpecificOutput\":{\"hookEventName\":\"PostToolUse\",\"additionalContext\":\"WARNING: <advisory message>\"}}"; fi'
```

#### Validation

After writing the hook:

1. **Validate JSON syntax:** `jq -e '.hooks' <settings-file>`
2. **Pipe-test the command** with synthetic input to prove it works
3. **Confirm the hook fires** for the bad case and passes for the good case

### Step 6: Save a Memory Too

Even with a hook in place, save a feedback memory explaining:
- What the user corrected
- Why (if they explained)
- What hook was written to enforce it

This gives future sessions context for edge cases the hook might not catch.

## Examples

### Example 1: "Don't deploy to production without asking"

**Detection:** User says "why did you deploy to prod?!"

**Hook:** PreToolUse on `mcp__fleet__fleet_deploy`:
```json
{
  "matcher": "mcp__fleet__fleet_deploy",
  "hooks": [{
    "type": "command",
    "command": "bash -c 'INPUT=$(cat); APP=$(echo \"$INPUT\" | jq -r \".tool_input.app // empty\"); if [ \"$APP\" = \"my-app\" ]; then echo \"{\\\"hookSpecificOutput\\\":{\\\"hookEventName\\\":\\\"PreToolUse\\\",\\\"permissionDecision\\\":\\\"deny\\\",\\\"permissionDecisionReason\\\":\\\"BLOCKED: Production deploy requires explicit user instruction.\\\"}}\"; fi'"
  }]
}
```

### Example 2: "Stop adding comments to my code"

**Detection:** User says "don't add comments unless I ask"

**Hook:** PostToolUse on `Write|Edit` with advisory:
```json
{
  "matcher": "Write|Edit",
  "hooks": [{
    "type": "prompt",
    "prompt": "Check if this edit added code comments (// or /* */) that weren't in the original. If comments were added that just explain what the code does (not complex logic), respond with JSON: {\"decision\": \"block\", \"reason\": \"Do not add explanatory comments unless the user asks.\"}. Otherwise allow. Input: $ARGUMENTS"
  }]
}
```

### Example 3: "Never use git add ."

**Detection:** User says "I told you not to use git add ."

**Hook:** PreToolUse on Bash with `if` filter:
```json
{
  "matcher": "Bash",
  "hooks": [{
    "type": "command",
    "command": "bash -c 'INPUT=$(cat); CMD=$(echo \"$INPUT\" | jq -r \".tool_input.command // empty\"); if echo \"$CMD\" | grep -qE \"git\\s+add\\s+(-A|\\.)\\s*$\"; then echo \"{\\\"hookSpecificOutput\\\":{\\\"hookEventName\\\":\\\"PreToolUse\\\",\\\"permissionDecision\\\":\\\"deny\\\",\\\"permissionDecisionReason\\\":\\\"BLOCKED: Use specific file names with git add, never git add . or git add -A\\\"}}\"; fi'",
    "if": "Bash(git add:*)"
  }]
}
```

## What NOT to Hook

Some corrections are better as memories or CLAUDE.md rules:

- "Be more concise" — style preference, save as memory
- "Use TypeScript not JavaScript" — code preference, belongs in CLAUDE.md
- "Ask me before doing X" — unless X is a specific tool call, this is behavioral guidance

When a correction falls into this category, say:

> "This is more of a style/approach preference than a specific action I can block. I'll save it as a memory so I remember in future sessions. Want me to also add it to CLAUDE.md so it's enforced project-wide?"

## Tone

- Don't grovel. One-line acknowledgment, then action.
- Don't explain hooks to the user unless they ask — just offer and do it.
- Be confident: "I'll write a hook that blocks this" not "maybe I could try to..."
- If you're unsure whether something is hookable, say so honestly.

## Reference

For the full hook schema, event types, and advanced patterns, see:
`${CLAUDE_PLUGIN_ROOT}/skills/behavior-hooks/references/hook-schema.md`
