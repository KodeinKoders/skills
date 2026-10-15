---
name: kodein-antora-documentation
description: Kodein's conventions for a library's Antora documentation site — the doc/ component layout, antora.yml and version attributes, nav files, the simple-to-complex page progression, one page per context, code samples that compile, and building the site to check it. Use when asked to create, write, restructure or extend the documentation of a Kodein or kosi-libs library ("create the Antora doc", "write the docs in doc/", "document this library", "add a page to the documentation", "add a migration guide"), or when touching antora.yml or a nav.adoc, or when writing or updating the README.adoc of such a library. Not for a README.adoc or spec of a project without an Antora site, which only need kodein-asciidoc-writing.
---

# Writing Kodein documentation

How the documentation of a Kodein library is structured and written.
Every Kodein library documents itself as an Antora component, published either on the shared https://kosi-libs.org site or on its own site.
The conventions below are the ones the MocKMP, Kodein and CuP documentations have in common; where those differ, this skill says which way to go.

This skill covers *what* goes where and how a page reads.
The AsciiDoc syntax itself — one sentence per line, heading spacing, `[#id]`, callouts, constrained markers — comes from the **kodein-asciidoc-writing** skill: load it too, before writing any page.

Two companion files hold the copy-paste material and the checking procedure, so they are read only when needed:

- `references/templates.adoc` — `antora.yml`, the standalone-site playbook, nav files, skeletons for the introduction, getting-started and setup pages, and the `README.adoc`.
- `references/verification.adoc` — compiling the samples against the real library, and building the site with Antora.

**Creating a component, or adding a page to an existing one: read `references/templates.adoc` first. Finishing: follow `references/verification.adoc`.**

## 1. Before writing anything

- **Read the code, not only the README.**
  Public API and KDoc say what exists, tests show real usage and edge cases, the build files give coordinates and versions.
  README samples drift: check every one you reuse against the code.
- **Read the existing component first**, if there is one, and match it where it is consistent.
  For a new component, `MocKMP`, `Kodein` and `CuP` are the references: none is perfect, so follow this skill where they differ.
- **Find the release version** the documentation is for: the `version` that the root `build.gradle.kts` assigns in its `allprojects` block.

## 2. The component

```
doc/
├── antora.yml
└── modules/
    ├── ROOT/          index.adoc, getting-started.adoc
    ├── core/          setup, then the library's concepts
    ├── <context>/     one module per family of pages: guides, framework, extension, plugins, widgets...
    └── migration/     one page per major version, when there is one
```

- The directory is `doc/` at the repository root for a component published on https://kosi-libs.org, or `docs/` for a library that builds its own site (see below).
  The component keeps the name it already has.
- A family module groups pages that a reader may skip entirely:
  `framework` and `extension` (Kodein), `plugins` and `widgets` (CuP), `guides`.
  Each add-on page opens with how to install it, then how to use it.
- `name` is the library name, lowercase (`mockmp`, `kodein`, `inara`); `title` is its display name.
- `nav` lists every module's `nav.adoc`, `ROOT` first, `migration` last.
- **The library's version is written three times in `antora.yml`**, all quoted strings:
  - `version`: the release version, `X.Y.Z`, followed by `-Tag` for a pre-release;
  - `display_version`: `X.Y`, without patch or tag — quoted, or YAML reads `3.10` as the number `3.1`;
  - the `version` attribute of `asciidoc.attributes`: the release version again, which pages write `{version}`.
- **Every other version number a page mentions is an `asciidoc.attributes` entry**, never written in a page: Kotlin, KSP, JDK, Compose, and any other tool.
  Two naming styles exist, and a component keeps the one it uses:
  - an attribute named `XXX-version` must equal the `XXX` version of `gradle/libs.versions.toml`, so `kotlin-version` and `ksp-version` exist only for a catalog alias `kotlin` and `ksp` (MocKMP);
  - plain names — `kotlin`, `jdk`, `compose` — (Kodein, CuP).
  In a new component, use the `-version` form for what is in the catalog and a plain name for what is not, such as `jdk`.
  A pre-release is `version: 'X.Y.Z-Tag'` and `display_version: 'X.Y'`, with the same release version in the attribute.
- These rules are enforced at release time by the `version-check` action of `KodeinKoders/kodein-github-actions`, which also requires the three library versions to match the release and the root build version.
  **Bumping the version for a release means bumping it in `antora.yml` too.**
- The site on https://kosi-libs.org reads one `release/vX.Y` branch per release line, which the `release-branches` action keeps on the latest release of that line.
- **A library that publishes its own site** (CuP) keeps its component in `docs/` and adds an `antora-playbook.yml` and a `package.json` at the repository root (templates in `references/templates.adoc`): the content source with `start_path: docs`, a UI bundle, `start_page`, and any Asciidoctor extension the pages use, such as `@asciidoctor/tabs`.
  It is built with the Antora CLI from `package.json`, and has no `migration` module until it has a previous major version.
  A `branch` attribute naming the branch that the pages link to is common.
- Images live in the `images/` directory of the module that uses them.

## 3. Navigation

- `ROOT/nav.adoc` starts with the introduction and the getting-started page.
  A page that concerns the whole library, such as platform compatibility, may follow.
- Each other module's nav starts with a block title naming the family, followed by a flat list of its pages:
  `.Frameworks` then `* xref:android.adoc[]`.
  `core` may instead open with a top-level entry for its first page.
- **Leave the link text empty** (`xref:page.adoc[]`) so the entry shows the page title, and the two never drift apart.
  Give explicit text only to an anchor entry, or when the navigation label has to differ from the title.
- Pages are listed in reading order — the order of §4.
- **A page with several sections may list them one level below it**, as `xref:page.adoc#section[Section]` entries, so the reader sees what the page covers.
  Leave them out of a short page.
  The anchors are explicit IDs (§6).
  If a page needs many of them to be navigable, it should probably be split (§5).

## 4. From simple to complex

The documentation is read top to bottom by someone who has never used the library.
Each page only relies on the ones before it.

1. **Introduction** (`ROOT:index.adoc`): a `[.lead]` line, one short sample showing the library at its best, then why it exists — its design choices and what it does differently.
   No setup, no API reference.
2. **Getting started** (`ROOT:getting-started.adoc`): the shortest path to something that works.
   One platform — usually the JVM — in numbered steps, each with a code block.
   Every other platform is a link to the setup page.
3. **Setup** (first page of `core`): every supported way to install, Kotlin Multiplatform first, then JVM-only.
   For a code generator, also what gets generated and where.
4. **Concepts**, one per page, ordered by dependency: the basic building block first, then each feature that builds on it.
   The common case before the edge case, the default before the option.
5. **Advanced features**: options most users never touch, restrictions, ignored or rejected inputs.
6. **Guides**: the library used in a real context — testing, scopes and lifecycles, multi-module projects, a framework integration, concurrency, debugging, supported platforms.
7. **Migration**: one page per major version, named after it (`2to3.adoc`, titled `2.x to 3.0`), listed oldest first.

## 5. One context per page

- **A page answers one question**: how do I declare X, how do I test with it, how do I use it on Android.
  A context that a reader may skip entirely — a framework, a platform, a usage — gets its own page.
- Prefer several short pages over one long page with a deep heading tree.
- A page links to the pages it relies on and to the ones that go further, rather than repeating them.
  A short `TIP:` at the end pointing to the related guide works well.

## 6. Writing a page

- **No section directly under the title.**
  A page opens with one or two sentences saying what it covers, then the first `==` section.
- **Every sample is real Kotlin that compiles** against the documented version, with the imports left out.
  Use one small, believable domain per page (`NetworkModule`, `UserRepository`), not `Foo` and `Bar`, and keep the names consistent from one block to the next.
- **Show the call site**, not only the declaration.
  For a code generator, show what is generated too, in a block titled with the generated file name (`.InaraAppModule.kt`).
- **Title a block with a file name** when the file matters (`.build.gradle.kts`), and use `subs="verbatim,attributes"` on any block that writes a version attribute.
- **Alternatives for the same step** (Gradle or Maven, one platform or another) may use `@asciidoctor/tabs` — but only if the component's playbook enables the extension.
  Otherwise write one section per alternative.
- **A callout per non-obvious line**, explaining why rather than paraphrasing the code.
- **Give an explicit `[#id]` to every heading that is a cross-reference or nav target.**
  Existing `[[id]]` anchors may stay in a page that is not being rewritten.
  Antora keeps the Asciidoctor default ID prefix, so an automatic ID is `_my_section`, and `xref:page.adoc#my-section[]` silently misses it.
- **In a numbered step, wrap the content in an open block (`--`) when prose follows a callout list.**
  A `+` after a callout list attaches to its last callout, not to the step.
- **A behaviour that surprises gets a `WARNING`, and only after it is verified** — run it, do not infer it.
- **Document what the user writes and calls.**
  Internal or generated implementation classes are only mentioned to say they must not be used.
- A library built on KSP says, in an `IMPORTANT` admonition on its setup page, that it does not apply KSP itself and that the KSP version must match the Kotlin version.
- Address the reader as "you", in the present tense.

## 7. The README

The `README.adoc` at the repository root is the shop window of the documented library: it pitches the library, shows the shortest install, and sends the reader to the site.
It stays user-facing: module layout and build instructions for contributors belong in `CONTRIBUTING.adoc` or `AGENTS.md`.
The skeleton is in `references/templates.adoc`.

- **Header:** `:toc: macro`, `:toclevels: 2`, `:idprefix:`, `:idseparator: -`, `:icons: font`, `:source-highlighter: highlight.js`, and a `:version:` attribute holding the release version.
- **Opening:** a `[.lead]` tagline, a short description, one sample, then a one-line summary of what the library does not do (no reflection, no runtime graph…), and `toc::[]`.
- **`== Why <Library>`:** the design choices, each a bold lead-in sentence followed by what it means for the user.
- **`== Installation`:** the shortest build file that works, as one block titled with its file name, `subs="verbatim,attributes"` and `{version}`, a callout per line.
  Every other platform or setup is a link to the setup page.
  Do not duplicate the whole setup page.
- **`== Documentation`:** a link to the site, at the library's own path (`https://kosi-libs.org/<name>`) or its standalone URL.
- **No license section**: the `LICENSE` file is enough.
- **The introduction, the sample and "Why" are the same content as `ROOT:index.adoc`.**
  Write one and copy it to the other, and when one changes, change both.
  The README sample uses the same domain as the introduction's.
- **A breaking major gets a `WARNING` at the top of `== Installation`**, saying what breaks and linking the versioned migration page (`https://kosi-libs.org/mockmp/3.0/migration/2to3.html`).
  Remove it once the previous major is no longer something a reader is likely to update from.
- **The version is bumped in the README at each release**, along with `antora.yml`: the `:version:` attribute, and any warning or link that names a version.

## 8. Before calling it done

Follow `references/verification.adoc`.
In short:

- every non-trivial sample compiles against the library, and every behaviour a page claims has been run;
- Asciidoctor parses every page without a warning;
- the README sample and Installation block match the documentation's, and `:version:` is the release version;
- an Antora build of the component succeeds with its failure level at `warn`, which catches every broken xref;
- `git status` shows changes under `doc/` only — any temporary check code is deleted.

Report what was verified and how, and anything found wrong on the way, such as a stale README sample.
