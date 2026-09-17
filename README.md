# Kodein Claude Skills

The official [Claude Code](https://claude.com/claude-code) plugin marketplace of
[Kodein Koders](https://kodein.net) — Kotlin, Kotlin Multiplatform and Kodein ecosystem
expertise, packaged as skills Claude can load on demand.

## Using the marketplace

Add the marketplace once:

```sh
claude plugin marketplace add KodeinKoders/kodein-claude-skills
```

Then install the plugins you need:

```sh
claude plugin install <plugin>@kodein
```

Useful commands:

| Command | What it does |
| --- | --- |
| `claude plugin marketplace update kodein` | Pull the latest plugin list |
| `claude plugin list` | Show installed plugins |
| `claude plugin details <plugin>@kodein` | Inspect a plugin's skills and token cost |
| `claude plugin uninstall <plugin>@kodein` | Remove a plugin |

## Available plugins

<!-- Keep this table in sync with .claude-plugin/marketplace.json -->

| Plugin | Description |
| --- | --- |
| [`asciidoc`](plugins/asciidoc) | Kodein's house rules for writing and structuring AsciiDoc documentation |
| [`code-review`](plugins/code-review) | Kodein's code review workflow: findings recorded in an ISSUES.adoc file, then fixed one commit at a time |
| [`gradle`](plugins/gradle) | Kodein's conventions for Kotlin and Kotlin Multiplatform Gradle builds |

See [CONTRIBUTING.md](CONTRIBUTING.md) to add another.

## Repository layout

```
.claude-plugin/
  marketplace.json      # the marketplace manifest — lists every published plugin
plugins/
  <topic>/              # one plugin per topic
    .claude-plugin/
      plugin.json       # the plugin manifest
    skills/
      <skill-name>/
        SKILL.md        # the skill itself
```

One plugin per **topic**, so users install only the expertise they need. A topic plugin
may hold several related skills.

## License

[MIT](LICENSE) © Kodein Koders
