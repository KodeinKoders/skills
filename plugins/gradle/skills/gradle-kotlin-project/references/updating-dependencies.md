# Updating dependencies

Everything versioned in the project: every `[versions]` entry of `gradle/libs.versions.toml`, and
the Gradle wrapper.

The order matters. **Research everything, ask everything, then edit.** Minor and patch bumps all
land in one commit, so that commit cannot be written until every decision — including the ones that
belong to the user — is settled. Editing as you go means rewriting the batch when a later answer
changes what belongs in it.

## 1. Inventory

Two sources:

- `gradle/libs.versions.toml` — the `[versions]` block. One entry can drive several aliases
  (`coroutines` drives `kotlinx-coroutines` and `kotlinx-coroutines-test`); it is still one bump.
- `gradle/wrapper/gradle-wrapper.properties` — the `distributionUrl` version.

## 2. Find the latest of each

**The release page is already in the catalog** — that is what the trailing comment on every
`[versions]` entry is for (§4 of the skill). Fetch it. For the wrapper, the list is
https://services.gradle.org/distributions/.

**Then confirm the artifact actually exists in the repository.** A GitHub release is tagged before
the artifacts are published, sometimes by days, and a version that only exists as a tag will fail
the build:

```sh
curl -s https://repo1.maven.org/maven2/org/jetbrains/kotlinx/kotlinx-coroutines-core/maven-metadata.xml \
  | grep -E '<latest>|<release>'
```

The path is the group with dots replaced by slashes, then the artifact id. Skip this for a Gradle
plugin resolved from the plugin portal rather than Maven Central.

**Stay on the same stability channel.** If the current version is stable, the candidate is the
latest stable — an `-alpha`, `-beta`, `-rc` or `-dev` release is not an upgrade. If the project is
deliberately on a pre-release, the candidate is the latest pre-release of that line.

### Exception: Compose Multiplatform components

**Compose companion libraries have no independent candidate.** Material3, Material3 Adaptive,
Lifecycle, Navigation, Navigation3, Navigation Event, Savedstate and WindowManager Core are pinned
by the Compose Multiplatform release, which lists each one in a **Components** table at the end of
its notes (for example
https://github.com/JetBrains/compose-multiplatform/releases/tag/v1.12.0).

So:

- **Skip them in steps 2–4.** Do not look up their own release pages, do not propose a bump, and do
  not treat a newer version on Maven Central as available. `compose-material3 = "1.12.0-alpha03"`
  being an alpha is not a stability choice to revisit — the table dictates it.
- **When `compose-multiplatform` itself is a candidate**, the bump is the whole set: read the
  Components table of the target release and set every component entry to the version it names, in
  the same edit. A Compose bump that moves the plugin and leaves the components behind is the
  failure this rule exists to prevent.
- That set moves with `compose-multiplatform`, so it follows its classification: in the minor/patch
  batch when the Compose bump is minor, in the major's own commit when it is major.

### Exception: the Android Gradle Plugin

**AGP never moves off its current `{major}.{minor}` line without the user's explicit consent.**
Unlike every other entry, a *minor* bump needs approval here, not just a major one:

- **Patch — always, unasked.** `9.1.0` → `9.1.1` is routine and joins the minor/patch batch like
  anything else.
- **Minor or major — only with consent.** `9.1.x` → `9.2.0` or above is never taken silently, even
  though the rule for every other library would allow the minor.

The reason is that AGP is not only a build dependency: the IDE has to be able to open the project
afterwards. The Android plugin inside IntelliJ pins the AGP versions it accepts, so a bump the
build tolerates can still leave the user unable to load their project — a failure that shows up in
the IDE, not in `./gradlew build`.

**If the user consents, do not take the newest AGP.** Run the `agp-intellij-compatibility` skill to
find the latest version their IDE supports, and take that. Its answer is a ceiling: the newest AGP
on Google's repository is regularly several minor lines above what a current IntelliJ can open.

**If the user declines**, AGP still gets its latest patch on the current line — declining the minor
is not declining the patch.

## 3. Classify each candidate

For each entry, work out the latest **minor/patch** and the latest **major** available above the
current version. They are different questions and both need answering:

| Current | Available | Minor/patch candidate | Major candidate |
| --- | --- | --- | --- |
| `2.3.1` | `2.3.4`, `2.5.0` | `2.5.0` | — |
| `2.3.1` | `2.5.0`, `3.0.1` | `2.5.0` | `3.0.1` |
| `2.3.1` | `3.0.1` | — | `3.0.1` |

## 4. Ask, before touching anything

**Minor and patch bumps need no approval** — they go in. The one exception is AGP, whose minor
bumps are asked about too (below).

**Every major bump needs the user's approval**, because it can require source changes. Present all
of them together, with the alternative spelled out: approving takes the major, declining falls back
to the minor candidate if there is one.

> `ktor` is on 3.5.2. Take 4.0.1 (major, may need source changes), or 3.7.0?

Use `AskUserQuestion` so each is a real choice, and ask about every major in one go — more than one
call if there are many, but all of them before the first edit. The batch commit's contents depend
on these answers: a declined major contributes its minor fallback to the batch, an approved one
does not.

### AGP is asked about even when only a minor is available

Put it in the same round of questions, phrased as whether to look beyond patches at all:

> AGP is on `9.1.0`. Patch `9.1.1` goes in either way — should I also check for a newer AGP line?
> That is bounded by what your IDE can open, not by the newest release.

- **Yes** → run the `agp-intellij-compatibility` skill, and use the version it returns. Announce
  the ceiling with the result (*"IntelliJ 2026.2.3 supports up to AGP 9.1, latest patch 9.1.1"*),
  because it is often the reason the answer is lower than the user expected.
- **No** → take the latest patch of the current line and nothing more.

Ask this whenever the project has an `android-gradlePlugin` entry and a non-patch AGP version
exists — never skip the question on the grounds that the newer line looks routine.

## 5. Apply, in this order

### One commit for all minor and patch bumps

Every minor/patch candidate, plus the fallback of every declined major, in a single edit to
`libs.versions.toml`. Keep each entry's release-page comment.

```sh
./gradlew build
git commit -m "Update minor and patch dependency versions"
```

If the build breaks, fix or drop the offending entry — do not split the batch into per-dependency
commits to isolate it. Note in the commit message which entry was held back and why.

### One commit per approved major, and per AGP line change

Separately, in its own commit, because it is the one that may carry source changes:

```sh
./gradlew build
# fix whatever the new major broke
git commit -m "Update ktor to 4.0.1"
```

The message names the library and the version, and describes any source change the bump forced.

An approved AGP move to a new `{major}.{minor}` line gets its own commit on the same grounds, even
when the version jump is only a minor. Its patch-level bump, by contrast, belongs in the batch.

### The Gradle wrapper

```sh
./gradlew wrapper --gradle-version 9.7.1
./gradlew wrapper
```

Run it twice: the first invocation rewrites `gradle-wrapper.properties`, the second regenerates the
wrapper jar and scripts with the new version. Check that `distributionUrl` still ends in `-bin`
afterwards. A Gradle major goes in its own commit like any other.

## Compatibility, before declaring it done

Three pairs are version-coupled, and a bump on one side can require the other:

- **Kotlin ↔ Compose Multiplatform** — the Compose release notes name the Kotlin versions each
  release supports.
- **Kotlin ↔ AGP** — https://kotlinlang.org/docs/multiplatform/multiplatform-compatibility-guide.html#version-compatibility
- **Kotlin ↔ KSP** — KSP has its own version line now (§4 of the skill), but a given KSP release is
  still built against a particular Kotlin. Its release page says which.

`./gradlew build` catches most of this. What it does not catch is a combination that configures and
compiles but is unsupported, so check the matrix when Kotlin itself moves.
