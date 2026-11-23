# claude-automate

**Plain English to working Claude automations.**

Built by [Orbitrove Technologies](https://orbitrovetechnologies.com)

---

## What it does

Say what you want automated in plain English. This plugin maps your intent to Claude's native automation primitives and generates working, tested configurations.

```
/automate "Every time I push to main, run the test suite and notify me if anything fails"
```

Claude figures out the right primitive (hook, scheduled task, skill, agent, or MCP server), generates the code, installs it, and tests it.

## Supported primitives

| Primitive | Use case | Example |
|-----------|----------|---------|
| **Hooks** | React to Claude events in real-time | "Block force pushes to main" |
| **Scheduled Tasks** | Run automations on a cron schedule | "Every Monday, scan for security vulnerabilities" |
| **Skills** | Add new slash commands | "Create a /deploy command" |
| **Agents** | Specialized autonomous workers | "An agent that reviews PRs for accessibility" |
| **MCP Servers** | Connect Claude to external tools | "Connect Claude to our internal API" |

## Install

### From Anthropic Directory

Search for **"claude-automate"** in the Claude plugin directory and click Install.

### From Git

```bash
claude plugin marketplace add https://github.com/orbitrovetechnologies/claude-automate
claude plugin install claude-automate
```

## Usage

### Create an automation

```
/automate <describe what you want in plain English>
```

### List your automations

```
/automate list
```

### Test an automation

```
/automate test <automation-name>
```

### Debug a broken automation

```
/automate debug <automation-name>
```

### Remove an automation

```
/automate remove <automation-name>
```

## How it works

```
Your words
    │
    ▼
┌─────────────────────┐
│  Intent Engine       │  Parses what you want
│  (Layer 2)           │  Maps to the right primitive(s)
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Primitive Registry  │  Complete map of every Claude
│  (Layer 1)           │  automation primitive + schemas
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Builder             │  Generates valid code & config
│  (Layer 3)           │  Merges into existing settings
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Lifecycle Manager   │  Install → Test → Monitor → Debug
│  (Layer 4)           │  Tracks everything in manifest
└─────────────────────┘
```

## Architecture

This plugin is built on a **Primitive Registry** — a complete mapping of every automation capability Claude exposes. When Anthropic adds new primitives, we add them to the registry and the entire plugin evolves. Nothing breaks, nothing stagnates.

## Examples

**Guard rails:**
```
/automate "Block any command that deletes production data"
```

**Scheduled pipelines:**
```
/automate "Every Friday at 5pm, generate a weekly summary of all PRs merged this week"
```

**Smart commands:**
```
/automate "Create a /release command that bumps the version, updates the changelog, creates a tag, and pushes"
```

**Session automation:**
```
/automate "When I end a coding session, auto-commit work-in-progress to a WIP branch"
```

## Contributing

PRs welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

MIT — see [LICENSE](LICENSE)

---

Built with purpose by [Orbitrove Technologies](https://orbitrovetechnologies.com)
