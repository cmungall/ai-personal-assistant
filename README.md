# AI Personal Assistant

Personal assistant skills and Claude Code plugins for Google APIs and productivity.
The published marketplace currently includes [gogcli](plugins/gogcli/skills/gogcli/SKILL.md)
for Gmail, Calendar, Tasks, Drive, Docs, Sheets, and other Google Workspace tools.

## Installation

### Install with npx skills

With Node.js and npm installed, use the [Skills CLI](https://skills.sh/docs/cli)
from the project where you want to use the skill:

```bash
# List available skills without installing
npx skills add cmungall/ai-personal-assistant --list

# Install one skill for Claude Code in the current project
npx skills add cmungall/ai-personal-assistant --skill gogcli -a claude-code
```

Use `-a codex` to target Codex instead, or omit `-a` to choose agents.
Installation is project-scoped by default; add `-g` for a user-wide install
available across projects.

This installs skill files. To install a Claude plugin and any bundled hooks,
MCP servers, or plugin commands, use the marketplace instructions below.

### Claude Code marketplace

Run these commands in Claude Code:

```text
/plugin marketplace add cmungall/ai-personal-assistant
/plugin install gogcli@ai-personal-assistant
```

The skill provides instructions for the `gog` CLI. Install and authenticate `gog`
separately as described in the [skill setup instructions](plugins/gogcli/skills/gogcli/SKILL.md).
