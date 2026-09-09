# Contributing a skill

## 1. Create the plugin (once per topic)

Pick a topic that groups related skills — for example `kodein-di`, `kmp`, `gradle`.
Plugin names are kebab-case and must be unique within the marketplace.

```sh
mkdir -p plugins/<topic>/.claude-plugin plugins/<topic>/skills
```

`plugins/<topic>/.claude-plugin/plugin.json`:

```json
{
  "$schema": "https://anthropic.com/claude-code/plugin.schema.json",
  "name": "<topic>",
  "version": "0.1.0",
  "description": "What this plugin gives Claude, in one sentence",
  "author": {
    "name": "Kodein Koders",
    "url": "https://kodein.net"
  },
  "skills": ["./skills"]
}
```

## 2. Add a skill

`plugins/<topic>/skills/<skill-name>/SKILL.md`:

```markdown
---
name: <skill-name>
description: Describe WHEN Claude should load this, with the trigger phrases a user would actually type. This string is what Claude matches the request against — be specific.
---

# <Skill title>

What the skill does, and the steps Claude should take.
```

Guidelines:

- The `description` is a router, not a summary. Name the technologies, tasks and phrases
  that should trigger it, and say when *not* to use it if that is ambiguous.
- Keep `SKILL.md` focused. Put long references, templates and scripts in sibling files
  (`references/`, `templates/`, `scripts/`) and link to them so they load only when needed.
- Write instructions for Claude, not documentation for humans.

## 3. Register the plugin in the marketplace

Add an entry to the `plugins` array in `.claude-plugin/marketplace.json`:

```json
{
  "name": "<topic>",
  "description": "Same one-sentence description as plugin.json",
  "source": "./plugins/<topic>",
  "version": "0.1.0",
  "category": "development",
  "tags": ["kotlin", "..."]
}
```

The `version` here must match `plugin.json`. Update the plugin table in
[README.md](README.md) too.

## 4. Validate

```sh
claude plugin validate .                    # the marketplace manifest
claude plugin validate plugins/<topic>      # the plugin and its skills
```

Both must pass before opening a pull request. CI runs the same checks, with `--strict`
on each plugin.

## 5. Try it locally

```sh
claude plugin marketplace add ./
claude plugin install <topic>@kodein
```

## Releasing

Bump `version` in both `plugin.json` and the marketplace entry, then tag:

```sh
claude plugin tag plugins/<topic>
```

This creates a `<topic>--v<version>` tag and verifies the two manifests agree.
