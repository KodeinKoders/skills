# kodein-claude-skills

Claude Code plugin marketplace published by Kodein Koders. It contains **no application
code** — only manifests and skill documents.

## Layout

- `.claude-plugin/marketplace.json` — the marketplace manifest; every published plugin has
  an entry here, with `source` pointing at its directory under `plugins/`.
- `plugins/<topic>/.claude-plugin/plugin.json` — one plugin manifest per topic.
- `plugins/<topic>/skills/<skill-name>/SKILL.md` — the skills themselves.

One plugin per topic; a topic may hold several related skills.

## Rules

- A plugin's `version` must be identical in `plugin.json` and in its marketplace entry.
- Plugin and skill names are kebab-case and unique across the marketplace.
- The plugin table in `README.md` mirrors `marketplace.json` — update both together.
- A skill's `description` frontmatter is what Claude matches a user request against.
  Write it as a trigger ("use when the user asks to…", with the phrases they'd type),
  never as a summary of the content.
- Keep `SKILL.md` short; move long reference material into sibling files that the skill
  links to, so they are read only when needed.

## Validating

```sh
claude plugin validate .                 # marketplace manifest
claude plugin validate plugins/<topic>   # a plugin and its skills
```

Run both after any manifest or skill change. CI runs them on every push and PR,
with `--strict` on each plugin.

See `CONTRIBUTING.md` for the full procedure to add a plugin or skill.
