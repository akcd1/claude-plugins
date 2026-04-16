# Whitehead Claude Code Plugins

A shared plugin marketplace for [Claude Code](https://code.claude.com/) at Whitehead Institute. Plugins provide reusable skills, agents, hooks, and MCP server configurations that extend what Claude Code can do.

## Available Plugins

| Plugin | Skill | What it does |
|--------|-------|--------------|
| [`fry-python-tool`](plugins/fry-python-tool/) | `fry-python-tool` | Find existing GPU-powered computational biology tools on the Whitehead fry cluster, or scaffold a new one. For single-purpose Python tools that produce outputs (embeddings, segmentations, predictions) for downstream analysis. |
| [`session-tools`](plugins/session-tools/) | `self-assess` | Audit the current Claude Code session for errors, inefficiencies, and repeated workarounds, then produce actionable recommendations to improve skills, scripts, CLAUDE.md, and tooling. |

After installing (see below), invoke a skill with `/whitehead:<skill-name>` — e.g., `/whitehead:self-assess`.

## Using This Marketplace

### Add the marketplace

In any Claude Code session, run:

```
/plugin marketplace add whitehead/claude-plugins
```

This registers the marketplace. You only need to do this once — it persists across sessions.

### Browse and install plugins

List available plugins:

```
/plugin search @whitehead
```

Install a plugin:

```
/plugin install plugin-name@whitehead
```

Installed plugins are available immediately. Plugin skills show up as `/whitehead:skill-name` commands.

### Update plugins

Claude Code checks for updates automatically at startup. To update manually:

```
/plugin marketplace update whitehead
```

### Remove the marketplace

```
/plugin marketplace remove whitehead
```

This removes the marketplace and uninstalls its plugins.

## Contributing

We welcome contributions from anyone at Whitehead. This marketplace is for **shared tools that are broadly useful across the institute.** If a plugin is only useful to your personal workflow, keep it in your personal `~/.claude/skills/` directory instead. A good test: "Would someone outside my lab find this useful?"

### Naming

Plugin names describe **what they do**, not where they came from. No lab prefixes, team prefixes, or usernames in plugin names.

| Good | Bad |
|------|-----|
| `rnaseq-qc` | `young-lab-rnaseq-qc` |
| `demux-pipeline` | `core-demux` |
| `fastq-quality-check` | `jdoe-qc-helper` |

Author attribution belongs in `plugin.json` using the `author` field, not in the name.

### Plugin structure

Each plugin lives in its own directory under `plugins/`:

```
plugins/my-plugin/
├── .claude-plugin/
│   └── plugin.json       # Plugin manifest (name, description, version)
└── skills/
    └── my-skill/
        └── SKILL.md       # Skill instructions with YAML frontmatter
```

A minimal `plugin.json`:

```json
{
  "name": "my-plugin",
  "description": "What this plugin does",
  "version": "1.0.0"
}
```

After creating the plugin directory, add an entry to `.claude-plugin/marketplace.json`:

```json
{
  "name": "my-plugin",
  "source": "my-plugin",
  "description": "What this plugin does"
}
```

For full plugin authoring docs, see [Create plugins](https://code.claude.com/docs/en/plugins).

### PR review checklist

**This is a public repository on the public internet.** Anyone can read it. Before submitting or approving a PR, verify the following:

- [ ] **No usernames or account names.** Do not include MIT Kerberos principals, LDAP usernames, email addresses, GitHub handles, or service account names.
- [ ] **No real names.** Use generic placeholders like `your-username` or `the project maintainer` instead of identifying individuals.
- [ ] **No internal hostnames or paths.** Do not reference internal servers, network paths, NFS mounts, or home directory paths (e.g., `/home/jdoe/`, `//fileserver/share`).
- [ ] **No credentials or tokens.** No API keys, passwords, certificates, or secrets of any kind — even examples that look fake could be real.
- [ ] **No internal URLs.** Do not include links to intranet sites, internal wikis, Jira/Confluence instances, or other systems not accessible to the public.
- [ ] **No institutional operational details.** Avoid references to specific lab names, internal project codenames, or infrastructure details that are not already public.

If a skill needs to reference user-specific values (paths, usernames, endpoints), use environment variables or clearly marked placeholders that each user fills in locally.

### Development workflow

1. Fork or branch from `master`.
2. Create your plugin under `plugins/`.
3. Add the marketplace entry in `.claude-plugin/marketplace.json`.
4. Test locally:
   ```
   claude --plugin-dir ./plugins/my-plugin
   ```
5. Open a PR. Another team member reviews using the checklist above.
