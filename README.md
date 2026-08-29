# slopctl-templates

The template catalog for [slopctl](https://github.com/heikopanjas/slopctl), a Rust CLI that
manages coding agent instruction files (AGENTS.md, CLAUDE.md, and friends) across workspaces.

This repo holds no code — only data. `slopctl templates --update` downloads it into a local
cache and installs from there; you never need to clone it directly unless you're editing the
catalog itself.

## Layout

```
v5/
  templates.yml          # catalog definition: which files go where, for which agent/language
  AGENTS.md               # the main AGENTS.md template, with fragment insertion points
  preamble.md              # session-start guard, merged first into AGENTS.md
  *-skills-hint.md, *-format-instructions.*, *-editor-config.ini
                           # per-language config file templates
  claude/ copilot/ cursor/ opencode/
                           # per-agent prompts and instruction stubs
  skills/                 # Agent Skills (agentskills.io) installed by the catalog
```

`v5/` names the V5 template format — see `templates.yml`'s own header comments for the format
history and the insertion-point/placeholder reference. A future format revision would land
alongside as a sibling `v6/`, not replace this directory.

## Making changes

Edit files under `v5/`, following the conventions already in `templates.yml`. Then, from a
slopctl workspace:

```
slopctl templates --update --from /path/to/slopctl-templates/v5
slopctl templates --verify
```

to test against your local copy before publishing.

## License

MIT — see [LICENSE](LICENSE).
