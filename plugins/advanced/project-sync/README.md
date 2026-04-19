# project-sync

Sync claude.ai projects, conversations, knowledge files, and project memory into Claude Code — so your CLI sessions can reference the context you've built up in claude.ai without re-explaining anything.

## Prerequisites

1. **Chrome** (the sync runs against claude.ai via a browser extension)
2. **[Claude in Chrome](https://claude.ai/chrome) MCP extension** — installed and connected to Claude Code
3. **Active claude.ai session** in that Chrome profile (log in once, cookie auth does the rest)

## Install

```
/plugin marketplace add pittaaron/claude-code-plugins
/plugin install project-sync@aaronpitt
```

## First-run setup (one-time)

The skill's `refresh` command writes to `~/.claude/claude-ai-projects/`. To let background subagents write there too (which makes `refresh` parallelize across projects), add this to `~/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Read(~/.claude/claude-ai-projects/**)",
      "Write(~/.claude/claude-ai-projects/**)"
    ]
  }
}
```

Or ask Claude: *"add those permissions to my settings"* — the `update-config` skill will merge them for you.

## Commands

| Command | What it does |
|---|---|
| `/project-sync` | List synced projects with conversation counts |
| `/project-sync <name>` | Show full context for one project |
| `/project-sync search <keywords>` | Search the conversation index by title |
| `/project-sync read <chat_id>` | Open a conversation in Chrome and read the transcript live |
| `/project-sync refresh` | Re-sync everything from claude.ai (run periodically) |
| `/project-sync schedule` | Schedule `refresh` to run unattended (uses the `schedule` skill) |
| `/project-sync push <project>` | Push a note back to a claude.ai project's memory via the UI |

## How it works

1. Uses claude.ai's internal REST API via same-origin `fetch()` from the extension tab (no scraping for conversation/doc data — only for memory, which isn't API-exposed)
2. Downloads assembled JSON via browser Blob downloads into `~/Downloads/`, then moves into `~/.claude/claude-ai-projects/`
3. For project memory (not exposed via API), launches parallel subagents — one tab per project — to scrape the memory section from the project page

## Data layout

```
~/.claude/claude-ai-projects/
├── conversation-index.json    # all conversations with project tags
└── <project-name>.json        # one file per claude.ai project
                               # contains: metadata, prompt_template, memory, files[]
```

Project names are normalized: lowercase, spaces to hyphens, punctuation stripped.

## Scheduling

Run `/project-sync schedule` once to set up a daily refresh. This wraps the [`schedule`](https://github.com/anthropics/claude-code) skill and runs `/project-sync refresh` at 3 AM local time by default. Edit the resulting cron entry to change the cadence.

## Limitations

- **claude.ai/design** is a separate product — not synced
- **Artifact content** is not separately extracted (available via `/project-sync read`)
- **UI scraping for memory** — if Anthropic restructures the project page, memory extraction may break until the parser is updated
- **Chrome must be open and logged in** for refresh to run

## License

MIT
