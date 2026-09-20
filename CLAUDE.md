# skills

Claude Code plugin marketplace published by Kodein Koders (`KodeinKoders/skills`). It
contains **no application code** — only manifests and skill documents. The same skills
also install into other agents (OpenCode, Codex, Zed, …) via `npx skills`; see
README.adoc.

## Layout

- `.claude-plugin/marketplace.json` — the marketplace manifest; every published plugin has
  an entry here, with `source` pointing at its directory under `plugins/`.
- `plugins/<topic>/.claude-plugin/plugin.json` — one plugin manifest per topic.
- `plugins/<topic>/skills/<skill-name>/SKILL.md` — the skills themselves.

One plugin per topic; a topic may hold several related skills.

## Rules

- A plugin's `version` must be identical in `plugin.json`, in its marketplace entry, and in
  the `README.adoc` plugin table.
- Plugin and skill names are kebab-case and unique across the marketplace. Skill
  directories and their `name:` frontmatter are prefixed `kodein-`; plugin names are not
  (a plugin already namespaces its skills in Claude Code, but the `npx skills` channel to
  other agents does not, so the prefix lives on the skill itself).
- The plugin table in `README.adoc` mirrors `marketplace.json` — update both together.
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

See `CONTRIBUTING.adoc` for the full procedure to add a plugin or skill.
