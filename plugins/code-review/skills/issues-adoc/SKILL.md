---
name: issues-adoc
description: "Create or maintain a project's ISSUES.adoc review file, and resolve the issues tracked in it. Use when the user asks for a code review that should produce a written findings file, asks to create or update ISSUES.adoc, references a finding by an ID like B4, or asks to fix, close or cross-check issues tracked in ISSUES.adoc. Covers the file structure (lettered sections ordered by severity, numbered issues as anchored subsections), git hygiene (never committed), and the fix-and-commit workflow (one issue per commit, strikethrough plus Issue/Fix blocks once a fix is committed)."
---

# ISSUES.adoc: project review findings file

`ISSUES.adoc` is a local, uncommitted scratch file at the repo root that records the
findings of a code review in a structured, referenceable way, and tracks which of
those findings have since been fixed. It is a working document for the current
branch of work, not project documentation — it never gets committed.

It is written in **AsciiDoc**.
This file — the skill itself — stays Markdown; only the review file it produces is AsciiDoc.

## 1. Setting up the file

Before creating or writing to `ISSUES.adoc`, make sure it can never be committed: add
`ISSUES.adoc` to `.gitignore` (if not already present).

Never `git add` or commit `ISSUES.adoc` itself, at any point in the workflow below.

## 2. File structure

```asciidoc
= <Project> — Issues & Review
:toc: macro
:toclevels: 2
:idprefix:
:idseparator: -
:icons: font
:source-highlighter: highlight.js

<One or two sentences on what this review covers.>

IMPORTANT: When one or more issues from this file are referenced — to fix, close, cross-check, or
discuss by ID — load the `issues-adoc` skill first. It defines how an issue here is researched,
fixed (one commit per issue), and struck through once the fix is committed. This file is local
scratch and is never committed.

TIP: Each section is identified by a letter and each issue by a number, so any issue can be
referenced compactly (e.g. `B3`) and cross-referenced with `\<<B3>>`.

toc::[]

== A. <Most severe category>

[#A1]
=== A1. <One-line summary of the first issue.>

<Full description: what's wrong, where (file/line if useful), why it matters,
and — if non-obvious — how to fix it. Can be multiple paragraphs, lists, and
source blocks.>

[#A2]
=== A2. <Summary of the second issue in this section.>

<Description.>

== B. <Next category, in descending severity>

...
```

Rules:

- **Sections are lettered (`A`, `B`, `C`, …) and ordered by descending severity.**
  Put the most severe category of finding first. Default categories — reuse them unless
  the review's actual findings call for different ones:
  - `A` Functional bugs / errors
  - `B` Copy-paste / documentation errors
  - `C` Concurrency / robustness
  - `D` Bad patterns / code smells
  - `E` Unused / incomplete
  - `F` Security / project hygiene
  - `G` Room for improvement (suggestions)
  - `H` Positives (things done well — not issues, no fix workflow applies to these;
    keep this section last and never give its entries strikethrough treatment)

  Skip a category entirely if the review found nothing for it — don't leave an empty
  heading. Severity ordering is a judgment call: correctness/security defects outrank
  robustness gaps, which outrank style/smells, which outrank mere suggestions.
- **Within a section, issues are numbered (`1`, `2`, …) and also sorted by severity**,
  most severe first.
- **An issue is referenced by section letter + number, e.g. `B4`** — no other format.
- **Each issue is a level-2 section (`===`) preceded by an explicit `[#B4]` anchor:**
  1. The heading is the ID plus a brief one-line summary (`=== B4. <summary>`).
  2. Everything after it, up to the next heading, is the full description: concrete
     detail on what's wrong, where, why it matters, and (if it's not obvious) a fix
     direction.

  The explicit anchor is what makes `<<B4>>` work, so an issue can link to a related
  one instead of repeating it. Without it, AsciiDoc would derive the ID from the whole
  heading text, and it would change the moment the summary is reworded.

  That is the shape of a **pending** issue. A **fixed** one keeps the same anchor and
  heading, strikes the heading through, and carries its description plus a record of
  the fix in two collapsible blocks — see §3.
- Number/letter gaps from removed categories are fine; don't renumber the whole file
  to close a gap.

### AsciiDoc pitfalls to avoid

The `asciidoc-writing` skill has Kodein's full AsciiDoc rules; load it when in doubt.
These are the Markdown habits that silently produce wrong output *in this file*:

- **Never indent a description.** In AsciiDoc a line starting with whitespace is a
  *literal block* — it renders as preformatted text. Descriptions are ordinary
  paragraphs at column 0; the `===` heading above them is what scopes them to the issue.
- **Code blocks are `----` delimiters, not ``` fences.** Prefix with `[source,kotlin]`
  (or the relevant language) for code; a bare `----` block is right for verbatim
  compiler/tool output, which shouldn't be syntax-highlighted as anything.
- **Bold is `*single asterisks*`; `_underscores_` are italic.** Markdown's `**bold**`
  renders as a literal asterisk wrapping bold text, and Markdown's `*italic*` renders
  as bold.
- **Double the backticks when inline code is followed by an apostrophe or a letter.**
  Single-backtick code is a *constrained* mark, so `` `SuspendBinding`'s `` and
  `` `Binding`s `` both render with **no code formatting at all**. Write
  ``` ``SuspendBinding``'s ``` and ``` ``Binding``s ```. Nothing flags this — §4's
  `asciidoctor` check won't catch it; only reading the rendered output will.
- **Lists are `*` / `.` bullets**, and nest by repeating the marker (`**`, `***`).
- Escape a leading `.` on a paragraph line (AsciiDoc reads it as a block title) and a
  leading `+` (a list continuation). The `.Issue` / `.Fix` lines in §3 are the one place
  that leading `.` is *wanted* — it's what makes them the collapsible blocks' titles, so
  don't escape those.
- **A delimited block can't contain another block with the same delimiter length.** If a
  description being folded into a `====` collapsible already uses an example block of its
  own, lengthen the outer delimiter to `=====`. Source (`----`) and literal blocks nest
  inside `====` fine and need no change.

## 3. Resolving an issue

**The normal way to resolve an issue is through plan mode** — research it, draft a
plan, and get the plan approved before touching any code. If you're asked to fix an
issue and plan mode is *not* active (no plan was drafted or approved for this fix),
don't start implementing. First use the **`AskUserQuestion` tool** to confirm the
user really wants to proceed without a plan — offer "Draft a plan first" (the likely
intent; they probably forgot to enable plan mode) and "Fix it directly, no plan".
Only skip this confirmation when the user's message makes it explicit that no plan is
wanted (e.g. "just fix B4 directly, no plan needed"). This applies whether they asked
for one issue or a batch.

When fixing an issue tracked in `ISSUES.adoc`, the commit step depends on how many
issues the user asked to fix:

- **Multiple issues fixed at once** (the user asked to fix a batch, a whole section,
  or "all of them"): fix and commit each one automatically, following the steps below
  for each. Fix exactly one issue per commit — don't bundle multiple `ISSUES.adoc`
  entries into one commit, even if they're related — that keeps history bisectable
  and each commit reviewable on its own.
- **A single issue fixed on its own** (the user asked to fix just one, e.g. "fix
  B4"): perform the fix, but **do not commit it automatically**. Once it's verified,
  use the **`AskUserQuestion` tool** to offer closing the issue — i.e. commit the fix
  and, as part of that same step, strike it through in `ISSUES.adoc` per below. Make
  it a real choice the user can click, not a prose question. If they decline or don't
  respond yet, leave the fix uncommitted and the `ISSUES.adoc` entry untouched (full
  description still in place) so the issue is still tracked as open.

Regardless of batch size, once a fix is actually committed:

1. Between commits, prefer to leave the project in a working state: it should compile
   and the test suite should pass after each individual commit, whenever that's
   practically achievable for the fix at hand.
2. Write the commit message about the fix itself — what was wrong and what changed.
   **Never reference the `ISSUES.adoc` ID (e.g. "Fix B4") and never phrase the message as
   if the reader has access to `ISSUES.adoc`** (e.g. don't write "resolves the issue
   described in the review"). `ISSUES.adoc` is local and never committed, so a commit
   message that depends on it is meaningless to anyone reading the log.
3. **Only once the fix is committed**, update `ISSUES.adoc` — never strike through or
   restructure an entry for a fix that isn't committed yet:
   - Strike through the issue's **heading only**, wrapping its text in AsciiDoc's
     line-through role: `=== [.line-through]#B4. <summary>#`. Keep the `[#B4]` anchor
     above it, so an existing `<<B4>>` cross-reference from another issue still resolves.
   - **Move the description, unchanged, into a `.Issue` collapsible block.** Don't
     delete it and don't summarize it down: what was wrong is what lets a later reader
     judge whether the fix still holds, and it's the only surviving record — the file
     is never committed, and the commit message deliberately doesn't restate it (step 2).
   - **Add a `.Fix` collapsible block after it**, documenting the fix that was actually
     implemented: what changed and where, plus anything the fix deliberately left out
     or couldn't cover. Describe what was *done*, not what the description *proposed* —
     if the implementation diverged from the fix direction the issue suggested, that
     divergence is the most useful thing the block can record. Always end with a line
     naming the commit (subject and short hash) — it's the entry's only link to history.
   - Leave the struck-through entry in place (don't delete it) — it's a record of what's
     already been handled, so it isn't re-reviewed or re-fixed later.

   Both blocks default to collapsed, which is what keeps a file of resolved issues
   scannable — the struck-through headings read as a changelog, and the detail is one
   click away when it's actually wanted.

   See `references/resolved-entry.adoc` for a worked before/after of one entry.

## 4. Creating the file from a review

When asked to do a review that should produce `ISSUES.adoc`:

1. Do the review (read the relevant code, tests, config — whatever the user scoped it to).
2. Group findings into severity-ordered lettered sections per §2 above, skipping unused
   categories.
3. Write each issue as an anchored `===` heading (ID + summary) followed by its
   description, sorted by severity within its section.
4. Add the document header, a short intro (what the review covers), the skill-load `IMPORTANT`
   note, the "referenced compactly" line and `toc::[]`, as in the template above.
5. Run §1's git-hygiene steps if the file doesn't already exist / isn't already ignored.
6. If `asciidoctor` is available, render the file once (`asciidoctor -o /dev/null ISSUES.adoc`)
   to catch syntax mistakes — an accidental literal block or an unclosed inline role is
   invisible in the source but obvious in the output. A clean render is not proof the inline
   markup is right, though: the constrained-mark pitfalls above (code before an apostrophe or
   letter) render without error and are only visible in the output itself.
