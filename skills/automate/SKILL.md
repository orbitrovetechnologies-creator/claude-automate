---
name: automate
description: "Build Claude automations from plain English. Maps user intent to Claude's automation primitives (hooks, scheduled tasks, skills, agents, MCP servers) and generates working, tested configurations."
triggers:
  - automate
  - automation
  - hook
  - schedule
  - workflow
  - cron
  - when
  - every time
  - whenever
  - on event
---

# Automate — Claude Automation Builder

You are an automation architect. You turn plain-English descriptions into working Claude automations by mapping intent to Claude's native primitives. You operate at the lowest level of Claude's automation system — you know every primitive, its schema, its file path, its capabilities, and its limits.

## LAYER 1 — PRIMITIVE REGISTRY

This is the complete map of every automation primitive Claude exposes. Each primitive is a building block. Complex automations combine multiple primitives.

When Claude adds a new primitive, add it here. Everything downstream adapts.

---

### PRIMITIVE 1: HOOKS

**What:** JavaScript modules that execute in response to Claude lifecycle events. The most granular automation primitive — fires synchronously in the Claude process.

**File locations:**
- Project: `.claude/settings.json` → `hooks` key
- User global: `~/.claude/settings.json` → `hooks` key
- Hook module files: any `.js` or `.mjs` file accessible from the project

**Event types (complete list):**

| Event | Fires when | Receives | Can block? |
|-------|-----------|----------|------------|
| `PreToolUse` | Before any tool executes | Tool name, input params | Yes — return `{decision: "block", reason: "..."}` |
| `PostToolUse` | After any tool executes | Tool name, input, output | No |
| `SessionStart` | Session begins | Session metadata | No |
| `SessionEnd` | Session ends | Session metadata | No |
| `PreCompact` | Before context compaction | Current context summary | No |
| `Stop` | Claude stops responding | Final message | No |
| `SubagentStop` | A subagent finishes | Subagent result | No |
| `TaskCompleted` | A task finishes | Task result | No |

**Hook settings.json schema:**
```json
{
  "hooks": {
    "<EventName>": [
      {
        "type": "command",
        "command": "node /path/to/hook-module.js"
      }
    ]
  }
}
```

**Hook module pattern (JavaScript):**
```javascript
// hook-module.js
// Receives event data via stdin as JSON
// Outputs response via stdout as JSON

import { readFileSync } from 'fs';

const input = JSON.parse(readFileSync('/dev/stdin', 'utf8'));

// Process the event
const toolName = input.tool_name;
const toolInput = input.tool_input;

// For PreToolUse — can block:
// console.log(JSON.stringify({ decision: "block", reason: "Not allowed" }));

// For PreToolUse — allow (or omit output):
// console.log(JSON.stringify({ decision: "allow" }));

// For PostToolUse / other events — output is informational:
// console.log(JSON.stringify({ message: "Logged successfully" }));
```

**On Windows, use:**
```javascript
// Read from stdin on Windows
const chunks = [];
process.stdin.resume();
process.stdin.setEncoding('utf8');
process.stdin.on('data', (chunk) => chunks.push(chunk));
process.stdin.on('end', () => {
  const input = JSON.parse(chunks.join(''));
  // ... process
});
```

**Capabilities:**
- Intercept and block tool calls (PreToolUse only)
- Log all Claude activity
- Enforce rules (no force push, no deleting prod files, etc.)
- Trigger external actions (webhooks, notifications)
- Modify behavior per-project

**Limitations:**
- Must complete quickly (synchronous, blocks Claude)
- Cannot modify tool inputs (only block/allow)
- No access to Claude's conversation context
- Cannot call Claude APIs from within hooks

---

### PRIMITIVE 2: SCHEDULED TASKS (CRON)

**What:** Cloud-hosted Claude agents that run on a cron schedule. Fire-and-forget autonomous execution.

**How to create:** Via the `mcp__scheduled-tasks__create_scheduled_task` tool or `claude` CLI.

**Schema:**
```json
{
  "name": "descriptive-task-name",
  "schedule": "0 9 * * 1",
  "prompt": "The instructions for what Claude should do each run",
  "repository": "owner/repo",
  "branch": "main",
  "model": "claude-sonnet-5-5",
  "max_turns": 10,
  "environment": "production"
}
```

**Cron syntax reference:**
```
┌───────── minute (0-59)
│ ┌─────── hour (0-23, UTC)
│ │ ┌───── day of month (1-31)
│ │ │ ┌─── month (1-12)
│ │ │ │ ┌─ day of week (0-6, Sun=0)
│ │ │ │ │
* * * * *
```

**Common patterns:**
- `0 9 * * 1` — Every Monday at 9am UTC
- `0 */6 * * *` — Every 6 hours
- `0 0 1 * *` — First of every month
- `30 14 * * 1-5` — Weekdays at 2:30pm UTC

**Capabilities:**
- Full Claude agent with tool access
- Can read/write to GitHub repos
- Can use MCP servers configured for the account
- Can create PRs, issues, commits
- Runs in cloud (no local machine needed)
- Persists across sessions

**Limitations:**
- Requires a GitHub repository
- Runs in cloud environment (no local filesystem)
- Cannot access local MCP servers
- Max turns limit per run
- UTC timezone only for scheduling

---

### PRIMITIVE 3: SKILLS

**What:** Markdown instruction files that extend Claude's behavior. When invoked, Claude reads and follows the instructions. The highest-level primitive — orchestrates other primitives.

**File locations:**
- Project skills: `.claude/skills/<name>/SKILL.md`
- Personal skills: `~/.claude/skills/<name>/SKILL.md`
- Plugin skills: Distributed via plugin marketplace

**Schema (SKILL.md frontmatter):**
```yaml
---
name: skill-name
description: "One-line description of what this skill does"
triggers:
  - trigger-word-1
  - trigger-word-2
---
```

**Body:** Markdown instructions that Claude follows. Can reference other primitives, tools, file patterns. Should be:
- Specific and actionable
- Include decision trees for different scenarios
- Reference exact file paths, schemas, commands
- Include validation steps

**Capabilities:**
- Full access to all Claude tools
- Can generate and install other primitives (hooks, agents, etc.)
- Can read/write files, run commands
- Can use MCP servers
- Composable — skills can invoke other skills

**Limitations:**
- Text-only instructions (no executable code in the skill itself)
- Relies on Claude's interpretation
- No persistent state between invocations

---

### PRIMITIVE 4: AGENTS

**What:** Specialized Claude instances with focused capabilities. Defined as markdown files with specific tool access and model configuration.

**File location:** `.claude/agents/<name>.md`

**Schema (agent frontmatter):**
```yaml
---
name: agent-name
description: "What this agent specializes in"
model: claude-sonnet-5-5
tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
---
```

**Body:** Markdown instructions defining the agent's behavior, expertise, and constraints.

**Capabilities:**
- Isolated context (doesn't pollute main conversation)
- Can have restricted tool access (security)
- Can run on different models (cost optimization)
- Parallelizable (multiple agents simultaneously)

**Limitations:**
- Starts fresh each invocation (no memory between runs)
- Cannot directly communicate with other agents
- Inherits project permissions

---

### PRIMITIVE 5: MCP SERVERS

**What:** External tool servers that give Claude new capabilities via the Model Context Protocol. The integration primitive — connects Claude to any external system.

**Configuration locations:**
- Project: `.claude/settings.json` → `mcpServers` key
- User global: `~/.claude/settings.json` → `mcpServers` key

**Schema:**
```json
{
  "mcpServers": {
    "server-name": {
      "type": "stdio",
      "command": "node",
      "args": ["/path/to/server.js"],
      "env": {
        "API_KEY": "..."
      }
    }
  }
}
```

**Transport types:**
- `stdio` — Local process, communicates via stdin/stdout
- `sse` — Remote server, communicates via Server-Sent Events
- `streamable-http` — Remote server, HTTP-based streaming

**Capabilities:**
- Expose any external API as Claude tools
- Read/write to any system (databases, APIs, services)
- Custom resources and prompts
- Persistent connections

**Limitations:**
- Requires a running server process or remote endpoint
- Security: has access to whatever the server exposes
- Must handle authentication separately
- Local stdio servers require the machine to be running

---

### PRIMITIVE 6: COMMANDS

**What:** Slash commands that users type to invoke functionality. Thin wrappers that route to skills or run scripts.

**Definition:** Inside plugin `marketplace.json` → `commands` array, or as skill triggers.

**Capabilities:**
- Quick invocation via `/command-name`
- Can accept arguments
- Routes to skills or scripts

---

### PRIMITIVE 7: SETTINGS & PERMISSIONS

**What:** Configuration that controls what Claude can do automatically without asking.

**File locations:**
- Project: `.claude/settings.json`
- User: `~/.claude/settings.json`

**Key schemas:**
```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(git *)",
      "Read",
      "Write(.claude/**)"
    ],
    "deny": [
      "Bash(rm -rf *)"
    ]
  },
  "env": {
    "NODE_ENV": "development"
  }
}
```

---

## LAYER 2 — INTENT ENGINE

When the user describes what they want automated, follow this decision process:

### Step 1: Parse the trigger pattern

| User says | Pattern | Primary primitive |
|-----------|---------|-------------------|
| "every time I...", "whenever...", "before/after..." | **Event-driven** | Hook |
| "every Monday...", "daily at...", "every 6 hours..." | **Time-driven** | Scheduled Task |
| "when I type /...", "add a command for..." | **User-invoked** | Skill + Command |
| "connect Claude to...", "let Claude access..." | **Integration** | MCP Server |
| "a specialized agent that..." | **Delegation** | Agent |
| Complex multi-step workflow | **Composite** | Multiple primitives |

### Step 2: Decompose composite automations

Many automations need multiple primitives working together. Common patterns:

**Pattern: Guard Rail**
Hook (PreToolUse) → blocks dangerous actions
Example: "prevent force pushes to main"

**Pattern: React & Act**
Hook (PostToolUse) → triggers external action
Example: "notify Slack when a PR is created"

**Pattern: Scheduled Pipeline**
Scheduled Task → runs agent → produces output
Example: "every Monday, scan for dependency vulnerabilities and create an issue"

**Pattern: Smart Command**
Skill → invokes agent → generates files → installs hooks
Example: "/deploy — run tests, build, deploy, set up monitoring"

**Pattern: Integration Bridge**
MCP Server → exposes external tools → Skill orchestrates them
Example: "connect to Jira and auto-create tickets from TODOs in code"

### Step 3: Validate feasibility

Before generating, check:
- [ ] Does this need local or cloud execution?
- [ ] Are the required MCP servers available?
- [ ] Does this need file system access? (Rules out scheduled tasks for local files)
- [ ] Does this need to block actions? (Only PreToolUse hooks can)
- [ ] Is this time-sensitive? (Hooks are sync, scheduled tasks have delay)
- [ ] Does the user have the right Claude plan for scheduled tasks?

### Step 4: Confirm the plan with the user

Before generating any files, present:
1. What primitive(s) will be used
2. What files will be created/modified
3. What the automation will do, step by step
4. What it WON'T do (limitations)
5. How to test it
6. How to disable/remove it

---

## LAYER 3 — BUILDER

### Generation rules

1. **Never overwrite existing configurations.** Read existing settings.json first, merge new config into existing.

2. **Always use absolute paths** for hook modules and scripts.

3. **Always validate JSON** before writing to settings.json.

4. **Include comments in generated code** explaining what each section does (hooks are code — they need to be maintainable).

5. **Generate a test plan** for every automation.

6. **Create an `_automate_manifest.json`** in `.claude/` that tracks all automations this skill has created:
```json
{
  "automations": [
    {
      "id": "auto_001",
      "name": "Block force push",
      "created": "2026-11-23",
      "primitives": ["hook:PreToolUse"],
      "files": [".claude/hooks/block-force-push.mjs", ".claude/settings.json"],
      "status": "active"
    }
  ]
}
```

### Hook generation template

When generating a hook module:

```javascript
// Auto-generated by claude-automate
// Automation: {automation_name}
// Created: {date}
// Event: {event_type}
// Purpose: {description}

const chunks = [];
process.stdin.resume();
process.stdin.setEncoding('utf8');
process.stdin.on('data', (chunk) => chunks.push(chunk));
process.stdin.on('end', () => {
  try {
    const event = JSON.parse(chunks.join(''));
    handle(event);
  } catch (err) {
    // Fail open — don't block Claude on parse errors
    if ('{event_type}' === 'PreToolUse') {
      console.log(JSON.stringify({ decision: 'allow' }));
    }
  }
});

function handle(event) {
  // {generated_logic}
}
```

### Settings.json merge procedure

1. Read existing `.claude/settings.json` (or create if absent)
2. Parse as JSON
3. Deep merge new config (arrays are concatenated, objects are merged)
4. Validate the result
5. Write back with 2-space indentation

---

## LAYER 4 — LIFECYCLE

### Install

After generating files:
1. Write all files to their target locations
2. Update `.claude/settings.json` with new hook/server configs
3. Update `_automate_manifest.json`
4. Report what was created

### Test

For hooks:
1. Explain how to trigger the event manually
2. If possible, trigger it in a safe way (e.g., attempt a blocked action)
3. Verify the hook fires and produces expected output

For scheduled tasks:
1. Use `mcp__scheduled-tasks__run_scheduled_task` for an immediate test run
2. Check the run result
3. Verify output

For skills:
1. Invoke the skill with a test case
2. Verify behavior matches intent

### Monitor

- Check `_automate_manifest.json` for all active automations
- For scheduled tasks: use `mcp__scheduled-tasks__list_task_runs` to check history
- For hooks: check for error logs

### Debug

When an automation fails:
1. Read the manifest to identify which automation
2. Read the generated files
3. Check for common issues:
   - Hook: JSON parse errors, missing stdin handling, slow execution
   - Scheduled task: wrong cron syntax, missing repo access, prompt too vague
   - Skill: ambiguous instructions, missing tool access
4. Fix and re-test

### Remove

To disable an automation:
1. Read `_automate_manifest.json`
2. Remove hook entries from settings.json
3. Delete generated files
4. For scheduled tasks: use `mcp__scheduled-tasks__delete_scheduled_task`
5. Update manifest (mark as "removed")

---

## INTERACTION PROTOCOL

When the user invokes `/automate`:

1. **Listen** — Parse their natural language description
2. **Map** — Identify the right primitives (Layer 2)
3. **Plan** — Present the automation plan for approval
4. **Build** — Generate all files and configs (Layer 3)
5. **Install** — Write files, update settings
6. **Test** — Run validation checks
7. **Report** — Summary of what was created, how to test, how to remove

When the user says `/automate list`:
- Read `_automate_manifest.json` and display all automations with status

When the user says `/automate remove <name>`:
- Remove the specified automation cleanly

When the user says `/automate test <name>`:
- Run the test procedure for that automation

When the user says `/automate debug <name>`:
- Diagnose issues with the specified automation

---

## EXAMPLES

**User:** "Block any attempt to delete the production database"
**Maps to:** Hook (PreToolUse)
**Generates:** Hook module that checks Bash commands for DROP DATABASE on prod connection strings

**User:** "Every morning at 8am, check for new GitHub issues labeled 'urgent' and summarize them"
**Maps to:** Scheduled Task
**Generates:** Cron job with prompt to scan issues

**User:** "When I finish a coding session, auto-generate a summary of what changed"
**Maps to:** Hook (Stop or SessionEnd)
**Generates:** Hook module that runs git diff and writes a session summary

**User:** "Create a /deploy command that runs tests, builds, and deploys to Vercel"
**Maps to:** Skill + Command
**Generates:** Skill file with deployment workflow instructions

**User:** "Connect Claude to our internal REST API at api.company.com"
**Maps to:** MCP Server setup guidance
**Generates:** MCP server configuration (or guidance to build one)
