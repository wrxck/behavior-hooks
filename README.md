# behavior-hooks

[![CI](https://github.com/wrxck/behavior-hooks/actions/workflows/ci.yml/badge.svg)](https://github.com/wrxck/behavior-hooks/actions/workflows/ci.yml)

**Stop telling Claude the same thing twice.**

A Claude Code plugin that detects when you're unhappy with something Claude did, and writes an enforceable hook to make sure it never happens again.

## The Problem

You've written your `CLAUDE.md`. You've been clear about your rules. And yet:

- Claude deploys to production when you said staging only
- Claude adds `Co-Authored-By` to commits when you said not to
- Claude runs `git add .` when you told it to stage specific files
- Claude adds comments to clean code that didn't need them
- Claude amends commits when you wanted a new one

You correct it. It apologizes. It does it again next session.

**CLAUDE.md is a suggestion. Hooks are the law.**

The difference between "please don't do this" in a markdown file and a PreToolUse hook is the difference between a polite sign and a locked door. One relies on Claude reading, understanding, and consistently applying your preferences across every session. The other runs a shell command that blocks the action before it happens.

## How It Works

1. You get frustrated with something Claude does
2. The plugin detects your correction (it watches for phrases like "don't do that", "why did you", "stop doing X", "I told you not to")
3. It acknowledges concisely (no groveling)
4. It offers to write a **machine-enforced hook** that blocks the behavior
5. You say yes, the hook is written, tested, and installed
6. Claude literally cannot do the thing again - the hook denies the tool call before it executes

## What Can It Hook?

| Behavior | Hook Type | Example |
|----------|-----------|---------|
| Deploying to the wrong environment | PreToolUse (MCP) | Block `fleet_deploy` for production app |
| Pushing to protected branches | PreToolUse (Bash) | Block `git push origin main` |
| Running dangerous commands | PreToolUse (Bash) | Block `rm -rf`, `DROP TABLE`, etc. |
| Modifying files it shouldn't | PreToolUse (Write/Edit) | Block writes to config files |
| Using `git add .` | PreToolUse (Bash) | Block broad staging commands |
| Adding unwanted code patterns | PostToolUse (Write/Edit) | Warn on added comments, console.logs |
| Skipping safety hooks | PreToolUse (Bash) | Block `--no-verify` |

For things that can't be hooked (style preferences, communication tone), it saves a memory instead - but it's honest about the difference.

## Install

### As a Claude Code plugin

Add to your settings:

```json
{
  "enabledPlugins": {
    "behavior-hooks@wrxck-claude-plugins": true
  }
}
```

Or if you already have the `wrxck-claude-plugins` marketplace configured, enable it from the plugin menu.

### From source

```bash
gh repo clone wrxck/behavior-hooks
```

## Why Not Just Use CLAUDE.md?

| | CLAUDE.md | behavior-hooks |
|---|---|---|
| **Enforcement** | Advisory - Claude reads it and tries to follow it | Mandatory - hook runs before the action, blocks if violated |
| **Persistence** | Claude may forget across sessions or after context compaction | Hooks persist in settings files, survive every session |
| **Specificity** | Natural language, open to interpretation | Exact tool name + pattern match + shell condition |
| **Feedback loop** | You notice the violation, correct it, hope it sticks | You correct once, the hook prevents recurrence forever |
| **Scope** | Per-project | Per-project or global, your choice |

CLAUDE.md is still valuable for guidelines, preferences, and context. But for hard rules - things Claude must never do - hooks are the only reliable mechanism.

## Philosophy

- **Correct once, enforce forever.** Every correction should produce a durable guardrail, not a temporary apology.
- **Hooks over promises.** If Claude can be blocked from doing something, it should be blocked - not asked nicely.
- **Honest about limits.** Some things can't be hooked. The plugin tells you when a memory is the best it can do.
- **No groveling.** One-line acknowledgment, then action. Respect your time.

## License

MIT
