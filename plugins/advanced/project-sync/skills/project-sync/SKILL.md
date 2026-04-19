---
name: project-sync
description: Sync claude.ai projects, conversations, and knowledge files into Claude Code. Uses claude.ai's internal API via the claude-in-chrome MCP extension for fast, reliable data extraction.
---

# Project Sync

Bridges context between claude.ai and Claude Code. Extracts projects, conversations, knowledge file contents, and project memory from claude.ai and stores them locally so Claude Code sessions can reference accumulated project context without re-explaining anything.

## Requirements

- User must be logged into claude.ai in Chrome
- [claude-in-chrome](https://claude.ai/chrome) MCP extension installed and connected
- `~/.claude/claude-ai-projects/` directory (created on first run if missing)
- **Recommended**: grant subagents read/write access to the sync directory (see "First-run setup" below) — enables parallel memory extraction

## First-run setup

If `~/.claude/claude-ai-projects/` doesn't exist, create it on first command:

```
Bash: mkdir -p ~/.claude/claude-ai-projects/
```

For parallel memory extraction in `refresh`, the user should add this to `~/.claude/settings.json`:

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

If the permission is missing, the skill falls back to main-agent memory extraction (slower — see Step 7 fallback).

## Commands

### `/project-sync`

List all synced projects with conversation counts.

1. Read all `*.json` files in `~/.claude/claude-ai-projects/` (skip `conversation-index.json`)
2. Read `conversation-index.json`, count conversations per project
3. Print a summary table: project name, description (truncated), conversation count, last synced date

### `/project-sync <name>`

Show full context for a specific project.

1. Read `~/.claude/claude-ai-projects/{name}.json` — print description, custom instructions, file names with token counts, memory summary
2. Read `conversation-index.json`, filter where `project` matches (case-insensitive)
3. Print the project's conversation list sorted by date, with chat_ids

### `/project-sync search <keywords>`

Search the conversation index by keyword.

1. Read `conversation-index.json`
2. Case-insensitive match `keywords` against each conversation's `title`
3. Print matches grouped by project, including chat_id
4. Remind user they can run `/project-sync read <chat_id>` for full transcript

### `/project-sync read <chat_id>`

Open a specific conversation in Chrome and read the full transcript live.

1. Load browser tools: `ToolSearch("select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__get_page_text")`
2. `tabs_context_mcp(createIfEmpty=true)`, then `tabs_create_mcp()`
3. `navigate(url="https://claude.ai/chat/{chat_id}", tabId=<new_tab>)`
4. `get_page_text(tabId=<new_tab>)` — returns full conversation transcript
5. Present the relevant content to the user

### `/project-sync refresh`

Full re-sync from claude.ai. This is the core operation.

**Overview:** One Chrome tab for API calls; parallel subagents for memory extraction. All data comes from claude.ai's REST API (cookie-authenticated) except project memory, which requires DOM scraping on the project page.

**Step 1 — Load browser tools and verify connection**

```
ToolSearch("select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__javascript_tool")
```

Call `tabs_context_mcp(createIfEmpty=true)`. If no claude.ai tab exists, create one and navigate to `https://claude.ai`. All subsequent API calls run via `javascript_tool` on this tab.

Ensure `~/.claude/claude-ai-projects/` exists (`mkdir -p` if not).

**Step 2 — Discover the organization**

```javascript
fetch('/api/organizations')
  .then(r => r.json())
  .then(orgs => {
    window.__orgs = orgs;
    JSON.stringify(orgs.map(o => ({uuid: o.uuid, name: o.name})));
  })
```

Users may have multiple orgs. Pick the one that contains projects (check project count per org with a Promise.all over `/projects` endpoints). Store as `window.__orgId`.

**Step 3 — Fetch all conversations**

```javascript
(async () => {
  const orgId = window.__orgId;
  let all = [], offset = 0, limit = 100, batch;
  do {
    const r = await fetch(`/api/organizations/${orgId}/chat_conversations?limit=${limit}&offset=${offset}`);
    batch = await r.json();
    all = all.concat(batch);
    offset += limit;
  } while (batch.length === limit);
  window.__allConvos = all;
  `fetched ${all.length} conversations`
})()
```

**Step 4 — Fetch projects and all their docs/details**

```javascript
fetch(`/api/organizations/${window.__orgId}/projects`)
  .then(r => r.json())
  .then(ps => { window.__projects = ps; ps.length + ' projects' })
```

Then for all projects at once:

```javascript
(async () => {
  const orgId = window.__orgId;
  const out = {};
  for (const p of window.__projects) {
    const [detail, docs] = await Promise.all([
      fetch(`/api/organizations/${orgId}/projects/${p.uuid}`).then(r => r.json()),
      fetch(`/api/organizations/${orgId}/projects/${p.uuid}/docs`).then(r => r.json())
    ]);
    out[p.uuid] = { detail, docs };
  }
  window.__projectData = out;
  Object.keys(out).length + ' projects loaded'
})()
```

**Step 5 — Build JSON bundles in the browser, download in one shot**

`javascript_tool` truncates at ~1KB. Don't try to pull full JSON through the return value. Instead, assemble the final JSON in-browser and trigger a Blob download — `~/Downloads/` is the landing pad, and the main agent moves files into place.

Build the conversation index:

```javascript
(() => {
  const uuidToName = {};
  for (const p of window.__projects) uuidToName[p.uuid] = p.name;
  const index = window.__allConvos.map(c => ({
    title: c.name,
    project: c.project_uuid ? (uuidToName[c.project_uuid] || null) : null,
    last_active: (c.updated_at || '').split('T')[0],
    chat_id: c.uuid
  }));
  index.sort((a, b) => (b.last_active || '').localeCompare(a.last_active || ''));
  const json = JSON.stringify({
    synced_at: new Date().toISOString().split('T')[0],
    total_conversations: index.length,
    projects_found: [...new Set(window.__projects.map(p => p.name))],
    conversations: index
  }, null, 2);
  const blob = new Blob([json], {type: 'application/json'});
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = 'conversation-index.json';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  'downloaded ' + json.length + ' bytes'
})()
```

Build and download each project snapshot (in a single loop):

```javascript
(async () => {
  function normalize(name) {
    return name.toLowerCase().replace(/[^a-z0-9\s-]/g, '').trim().replace(/\s+/g, '-');
  }
  const syncedAt = new Date().toISOString();
  const results = [];
  for (const p of window.__projects) {
    const { detail, docs } = window.__projectData[p.uuid];
    const snap = {
      name: detail.name,
      id: p.uuid,
      url: `https://claude.ai/project/${p.uuid}`,
      description: detail.description || '',
      prompt_template: detail.prompt_template || '',
      memory: null,
      files: (docs || []).map(d => ({
        file_name: d.file_name,
        estimated_token_count: d.estimated_token_count,
        content: d.content || ''
      })),
      created_at: p.created_at,
      updated_at: p.updated_at,
      synced_at: syncedAt
    };
    const json = JSON.stringify(snap, null, 2);
    const blob = new Blob([json], {type: 'application/json'});
    const a = document.createElement('a');
    a.href = URL.createObjectURL(blob);
    a.download = normalize(p.name) + '.json';
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    results.push(`${normalize(p.name)}: ${json.length}`);
    await new Promise(r => setTimeout(r, 250));
  }
  results.join('\n')
})()
```

**Step 6 — Move downloads into place, preserving existing memory**

Each project snapshot is set with `memory: null`. Before overwriting, preserve any existing memory value in case Step 7 fails for a project.

```bash
python3 - <<'EOF'
import json, os, re
from pathlib import Path

downloads = Path.home() / "Downloads"
target = Path.home() / ".claude/claude-ai-projects"
target.mkdir(parents=True, exist_ok=True)

UUID_RE = re.compile(r"^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$", re.IGNORECASE)
FILENAME_RE = re.compile(r"^[a-z0-9]+(-[a-z0-9]+)*\.json$")  # normalized names only

def is_valid_snapshot(data):
    """Strict schema check for project snapshot JSON."""
    if not isinstance(data, dict):
        return False
    uid = data.get("id", "")
    url = data.get("url", "")
    return (
        isinstance(uid, str) and UUID_RE.match(uid)
        and isinstance(url, str) and url.startswith(f"https://claude.ai/project/{uid}")
        and isinstance(data.get("name"), str)
        and "files" in data
    )

def is_valid_index(data):
    return (
        isinstance(data, dict)
        and isinstance(data.get("conversations"), list)
        and "total_conversations" in data
        and "synced_at" in data
    )

for src in downloads.glob("*.json"):
    if not FILENAME_RE.match(src.name):
        continue
    try:
        data = json.load(open(src))
    except Exception:
        continue
    if src.name == "conversation-index.json":
        if not is_valid_index(data):
            continue
    else:
        if not is_valid_snapshot(data):
            continue
    dst = target / src.name
    # Preserve existing memory if the new snapshot's memory is null
    if dst.exists() and isinstance(data, dict) and data.get("memory") is None:
        try:
            old = json.load(open(dst))
            if old.get("memory"):
                data["memory"] = old["memory"]
        except Exception:
            pass
    json.dump(data, open(dst, "w"), indent=2)
    os.remove(src)
    print(f"{src.name}: wrote {dst.stat().st_size} bytes")
EOF
```

**Step 7 — Extract project memory (API doesn't expose it)**

Project memory must be scraped from the project page. It appears between "Purpose & context" and "Instructions"/"Files" headings in the page text, optionally divided into six sections.

**Primary path: parallel subagents.** If the user has allowlisted `Read`/`Write` on `~/.claude/claude-ai-projects/**`, launch N subagents in a single message (one per project, `run_in_background: true`). Each subagent:

1. Loads chrome tools via `ToolSearch`
2. Creates its own tab, navigates to `https://claude.ai/project/{uuid}`
3. Calls `get_page_text(tabId)`
4. Parses the memory block
5. Reads the existing project JSON, updates only the `memory` field, writes it back

**Fallback path: main agent, sequential.** If the permission isn't granted, subagents will fail on `Write(~/.claude/...)`. In that case the main agent does the extraction itself:

1. For each project, `navigate` the existing tab to `https://claude.ai/project/{uuid}`, then `get_page_text`
2. Parse in the main agent
3. `Read` + `Write` the target JSON

The fallback is ~5x slower (N round trips vs. one parallel batch) but has no setup cost.

**Parsing the memory block.** The text starts with "Purpose & context" and ends before "Last updated N ..." or "Instructions". Section headers appear in order, separated by `---`:

1. Purpose & context
2. Current state
3. On the horizon
4. Key learnings & principles
5. Approach & patterns
6. Tools & resources

Parse into an object with keys:

```
purpose_and_context
current_state
on_the_horizon
key_learnings_and_principles
approach_and_patterns
tools_and_resources
```

Some projects will have fewer sections. Missing ones: set to null or omit.

**Projects without memory.** New projects show "Project memory will show here after a few chats" — or simply have no "Purpose & context" heading in the scraped text. In either case, set `memory: null`.

**Existing memory as fallback.** If extraction fails mid-way for a project, leave the existing `memory` value alone (already preserved in Step 6). Never overwrite a non-null memory with null unless the page clearly confirms the project has no memory yet.

**Step 8 — Report results**

Print: conversations indexed, projects synced, total knowledge file tokens, memory extraction status per project (success / no memory / failed).

### `/project-sync schedule [cron]`

Set up an unattended daily refresh using the `schedule` skill.

1. Default cadence: `0 3 * * *` (3 AM local). Accepts an optional cron override.
2. Invoke `schedule` to register a job that runs `/project-sync refresh` on schedule.
3. Confirm to the user, print the cron expression, and note that Chrome must be logged in for the run to succeed.

### `/project-sync push <project>`

Push a learning from the current session back to a claude.ai project's memory.

1. Load browser tools, create a tab
2. Navigate to `https://claude.ai/project/{id}` (id from the local project JSON)
3. `read_page` → find the "Edit memory" button, click with `computer(action="left_click")`
4. Find the textbox with placeholder "Tell Claude what to remember or forget"
5. Click, type the summary, press Enter
6. Confirm back to the user

## API reference

All endpoints are cookie-authenticated (requires an active logged-in claude.ai tab). Called via same-origin `fetch()` from any claude.ai page — no CORS, no navigation.

| Endpoint | Returns |
|---|---|
| `GET /api/organizations` | `[{uuid, name}]` |
| `GET /api/organizations/{org}/chat_conversations?limit=N&offset=N` | Paginated: `{uuid, name, project_uuid, updated_at, created_at, model}` |
| `GET /api/organizations/{org}/projects` | `[{uuid, name, description, created_at, updated_at}]` |
| `GET /api/organizations/{org}/projects/{id}` | Single project incl. `prompt_template` |
| `GET /api/organizations/{org}/projects/{id}/docs` | Knowledge files with full `content`: `{uuid, file_name, content, estimated_token_count}` |
| `GET /api/organizations/{org}/projects/{id}/memory` | **404** — not exposed. Memory must be scraped from the project page. |

## What this does NOT sync

- **Project memory via API** — not exposed; scraped from the project page DOM instead
- **Artifact content** — linked to conversations but no dedicated endpoint found. Use `/project-sync read` to view artifacts inside their source conversation.
- **claude.ai/design** — separate product, different namespace. Not covered.

## Practical notes

- **`javascript_tool` output truncation:** ~1KB cap. Don't pull large JSON through return values — assemble in-browser and use Blob downloads.
- **Blob downloads go to `~/Downloads/`.** The skill cleans up after itself in Step 6.
- **One tab for API calls.** Same-origin fetch works from any claude.ai page. Only memory extraction (Step 7) and `/project-sync read`/`push` need additional tabs.
- **Stale data.** Snapshots are point-in-time. Schedule `refresh` or re-run manually.
- **File naming.** Project names normalize to lowercase, hyphen-separated, punctuation stripped.

## File layout

```
~/.claude/claude-ai-projects/
├── conversation-index.json       # all conversations with project tags
└── <project-name>.json           # one file per project
                                  # fields: name, id, url, description,
                                  #         prompt_template, memory, files[],
                                  #         created_at, updated_at, synced_at
```
