# xuxi-plugins

Claude Code plugin marketplace for Dev team.

| Plugin                               | Description |
|--------------------------------------| ----------- |
| [`compare-pr`](./plugins/compare-pr) | PR descriptions in format, Magento 2 aware |

## Install

Push this directory to an internal git repository, then each developer runs:

```
/plugin marketplace add <git-url-or-owner/repo>
/plugin install compare-pr@xuxi-plugins
/reload-plugins
```

Then, on any feature branch:

```
/compare-pr:pr
```

## Test locally before pushing

```bash
claude plugin validate .
```

Then, from the parent directory of `xuxi-plugins`:

```
/plugin marketplace add ./xuxi-plugins
/plugin install compare-pr@xuxi-plugins
```

## Roll out to a project automatically

Commit this to a project's `.claude/settings.json` so anyone who trusts the folder is prompted to
install the marketplace, with the plugin enabled by default:

```json
{
  "extraKnownMarketplaces": {
    "xuxi-plugins": {
      "source": {
        "source": "url",
        "url": "https://<git-host>/<owner>/xuxi-plugins.git"
      }
    }
  },
  "enabledPlugins": {
    "compare-pr@xuxi-plugins": true
  }
}
```

## Versioning

Neither `marketplace.json` nor `plugin.json` declares a `version`, so each commit to this
repository is treated as a new version and developers receive updates automatically. Do not add a
`version` field unless you intend to pin releases — once set, pushing commits without bumping that
string ships nothing to existing installs.

## Adding a plugin

```
plugins/<name>/
├── .claude-plugin/plugin.json    # required
├── commands/*.md                 # slash commands
├── agents/*.md                   # subagents
├── skills/<name>/SKILL.md        # skills
├── hooks/hooks.json              # event handlers
└── .mcp.json                     # MCP servers
```

Then add an entry to `.claude-plugin/marketplace.json`. Two constraints worth knowing up front:

- Plugins are copied to a cache directory on install, so a plugin cannot reference files outside
  its own directory with `../`. Use `${CLAUDE_PLUGIN_ROOT}` for paths inside the plugin, and
  symlinks if two plugins must share a file.
- Plugin names must be kebab-case.
