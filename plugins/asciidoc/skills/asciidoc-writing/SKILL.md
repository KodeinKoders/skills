---
name: asciidoc-writing
description: Kodein's house rules for writing and editing AsciiDoc — document header and attributes, one-sentence-per-line prose, heading levels and anchors, source blocks with callouts, admonitions, lists, tables and cross-references. Use whenever creating or modifying a .adoc file (README.adoc, Antora page, spec document), converting Markdown to AsciiDoc, or when asked about AsciiDoc/Asciidoctor formatting, style or conventions.
---

# Writing AsciiDoc

Rules for authoring `.adoc` files in Kodein projects. Follow them when writing new
documents and when editing existing ones — match the file you are in first if it
already deviates consistently.

## Document header

Start the file with the document title as a level-0 heading, immediately followed by
attribute entries, then a blank line.

```adoc
= Kodein-DI: Dependency Injection
:toc: left
:toclevels: 3
:icons: font
:source-highlighter: rouge
```

- One `=` title per file, and it is the first non-empty line.
- Set `:icons: font` whenever the document uses admonitions.
- Set `:toc:` / `:toclevels:` on standalone documents (READMEs, specs). Omit them on
  Antora pages, where the site supplies navigation.
- Do not put a blank line between the title and the attributes.

## Prose

**Write one sentence per line.** Never wrap a sentence across lines, and never put two
sentences on one line. This keeps diffs readable and reviewable.

```adoc
A binding always starts with `bind<TYPE> { }`.
The provided function will be called *each time* you need an instance of the bound type.
```

Line length is a consequence of the sentence, not a target — do not reflow to a column.

Use a trailing ` +` only to force a rendered line break inside a paragraph:

```adoc
This binds a type to a provider function. +
The function takes no arguments.
```

Mark an introductory paragraph with `[.lead]` when the document needs a summary line
under its title.

## Inline formatting: single vs doubled markers

A single marker (`` ` ``, `*`, `_`, `#`) is **constrained**: Asciidoctor only recognises
it at a word boundary. Doubling it (```` `` ````, `**`, `__`, `##`) makes it
**unconstrained**, which works anywhere. Double the marker whenever the formatted span
touches a word character or a quote.

```adoc
`Header`'s option          // BROKEN — renders as `Header's, backtick left visible
``Header``'s option        // correct

`Bean`s are cheap          // BROKEN — renders literally
``Bean``s are cheap        // correct

un`mid`believable          // BROKEN
un``mid``believable        // correct
```

When to double — the marker must **not** sit directly against:

| Side | Forbidden neighbour |
| --- | --- |
| After the closing marker | a letter, a digit, or `_` |
| Before the opening marker | a letter, a digit, `_`, `:`, `;`, `}`, `&`, `<`, `>` |

The backtick has one extra trap on **both** sides: `'` and `"`. `` `' `` and `` `" `` are
Asciidoctor's curly-quote syntax, so `` `Header`'s `` is parsed as a smart apostrophe and
the backtick survives into the output. This is why a possessive after inline code always
needs `` ``Header``'s ``, while `*Bold*'s` and `_Ital_'s` are fine as single markers.

Two more consequences:

- Whitespace directly inside a single-marker pair is never recognised (`` ` x` ``). The
  doubled form accepts it, but trim the span instead.
- A preceding `\` escapes the marker; doubling does not override that.

Default to the single form. Reach for the doubled one for possessives, plurals and
mid-word fragments — that is the whole of it in practice.


## Headings

- `=` document title, `==` section, `===` subsection, `====` sub-subsection.
- Never use Markdown `#` headings.
- Leave **two blank lines** before a section heading, one after it.
- Sentence case, no trailing punctuation.

Give a heading an anchor when it is a cross-reference target. The anchor goes on the
line directly above the heading, in kebab-case, with no blank line between them:

```adoc
[[tagged-bindings]]
== Tagged bindings
```

## Source blocks

Use a `[source,<lang>]` attribute line, an optional block title, and `----` delimiters:

```adoc
[source,kotlin]
.Example: different Dice bindings
----
val di = DI {
    bind<Dice> { ... } // <1>
    bind<Dice>(tag = "DnD10") { ... } // <2>
}
----
<1> Default binding (with no tag)
<2> Binding with a tag
```

- Always name the language (`kotlin`, `groovy`, `properties`, `xml`, `bash`, …).
- Indent code with **4 spaces, never tabs**.
- Block titles start with `.` and describe the sample: `.Example: …`.
- Attach callouts with a trailing `// <n>` comment and explain each one on its own line
  directly under the block, in order.
- Add `subs="verbatim,attributes"` when the block interpolates an attribute such as
  `{version}`.

## Admonitions

Single-line form for one sentence, block form for more:

```adoc
TIP: The tag is of type `Any`, it does not have to be a `String`.

[IMPORTANT]
====
Tag objects must support equality & hashcode comparison.
It is therefore recommended to either use primitives or data classes.
====
```

Use `NOTE` for context, `TIP` for optional advice, `IMPORTANT` for things that break
expectations, `WARNING` for breaking changes and data loss, `CAUTION` for risky steps.
Do not stack admonitions back to back to emphasise ordinary prose.

## Lists

- Unordered lists use `*`, nested with `**` — not `-`.
- Ordered lists use explicit `1.`, `2.`, … when the numbers are referred to in the text;
  otherwise `.` lets Asciidoctor number them.
- Attach a paragraph, block or code sample to a list item with a `+` continuation line.

```adoc
1. Apply the Gradle plugin:
+
[source,kotlin]
----
plugins {
    id("org.kodein.mock.mockmp") version "{version}"
}
----
2. Create a test class.
```

## Cross-references and links

- `<<anchor>>` or `<<anchor,link text>>` within a document.
- `xref:page.adoc[text]` between Antora pages; `xref:module:page.adoc[text]` across modules.
- Bare URLs get an explicit label: `https://kodein.net[Kodein Koders]`.
- Images: `image::path.png[Alt text, 700]` as a block, `image:path.png[Alt]` inline.

## Tables

```adoc
[cols="1,3"]
|===
| Option | Description

| `tag`
| Distinguishes bindings of the same type.
|===
```

Declare `[cols=…]` when the default equal widths read badly, keep the header row
separated by a blank line, and put each cell on its own line.

See `references/snippets.adoc` for copy-paste templates of these constructs.
