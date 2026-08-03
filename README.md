# ap-plugins

Personal Claude Code plugin marketplace.

## Install

```bash
claude plugin marketplace add antoinepalazzolo/ap-plugins
claude plugin install agent-pipeline@ap-plugins
```

Or from inside Claude Code: `/plugin marketplace add <path-or-repo>` then `/plugin install`.

## Plugins

| Plugin | Description |
|--------|-------------|
| [agent-pipeline](plugins/agent-pipeline) | Autonomous plan → implement → review → fix pipeline driven by an orchestrator agent |

## Layout

```
.claude-plugin/marketplace.json   marketplace manifest
plugins/<name>/                   one directory per plugin
```

To add a plugin: create `plugins/<name>/` with a `.claude-plugin/plugin.json`, then add an entry to the `plugins` array in `.claude-plugin/marketplace.json`.
