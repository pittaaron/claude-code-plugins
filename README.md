# Claude Code Plugins

Community plugins for [Claude Code](https://claude.com/claude-code).

## Install

```
/plugin marketplace add pittaaron/claude-code-plugins
/plugin install <plugin-name>@aaronpitt
```

## Available plugins

Plugins are grouped by setup cost. **Standard** plugins work out of the box. **Advanced** plugins need external dependencies and one-time configuration.

### Advanced

| Plugin | Description |
|---|---|
| [project-sync](plugins/advanced/project-sync) | Sync claude.ai projects, conversations, and knowledge files into Claude Code for offline context. Requires the [claude-in-chrome](https://claude.ai/chrome) MCP extension and a permission allowlist entry. |

## Developing

Plugins live under `plugins/<category>/<name>/`, each with its own `.claude-plugin/plugin.json` and a `skills/`, `commands/`, `agents/`, or `hooks/` folder as needed. Categories are a convention in this marketplace (`standard`, `advanced`) — not a Claude Code feature.

## License

MIT — see [LICENSE](LICENSE).
