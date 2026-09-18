---
name: agp-intellij-compatibility
description: "Find the latest Android Gradle Plugin version a given IntelliJ IDEA or Android Studio version supports. Use when the user asks which AGP version their IDE supports, which IntelliJ version is needed for an AGP version, why the IDE reports an unsupported or too-new Android Gradle Plugin, or before bumping the android-gradlePlugin entry of a version catalog."
---

# AGP versions supported by IntelliJ

The Android plugin shipped inside IntelliJ IDEA pins the AGP versions the IDE can open. This skill
finds that ceiling for a given IDE version, in four hops — one per step below:

**IntelliJ version → its `JetBrains/android` tag → that tag's `STUDIO_CODENAME` → Google's AGP
compatibility table, giving a `{major}.{minor}` ceiling → Google's Maven repository, giving the
latest stable patch of that line.**

There is no direct IntelliJ-to-AGP table anywhere; the codename is the join between the two
sources, which is why the middle hop exists. The last one exists because the compatibility table
stops at `{major}.{minor}` and never names a patch.

This answers one question only: **the highest AGP the IDE will open a project with.** It says
nothing about AGP against Gradle, or AGP against Kotlin — those are separate matrices, and a
version passing this check can still be rejected by the build.

## Step 1 — Select the IntelliJ version and its android tag

The answer comes from https://github.com/JetBrains/android, whose tags track IntelliJ releases. The
output of this step is one released version, e.g. `2026.2.3` — also the tag `idea/2026.2.3`.

### a. List the released IntelliJ versions

One command. Nothing to install beyond `git`, and no authentication:

```sh
git ls-remote --tags https://github.com/JetBrains/android.git \
  | cut -f2 \
  | grep -v '\^{}' \
  | sed -nE 's/^refs\/tags\/idea\/([0-9]{4}\.[0-9]{1,2}(\.[0-9]{1,2})?)$/\1/p'
```

It returns every release in about a second:

```
2025.2
2025.2.1
…
2026.2.2
2026.2.3
```

The filtering is the whole point of the pipeline:

- `cut -f2` keeps the ref name and drops the object hash.
- `grep -v '\^{}'` drops the peeled entries of annotated tags, which would otherwise duplicate
  every tag.
- The `sed` pattern keeps `{year}.{release}` with an **optional** `.{patch}` and rewrites it to a
  bare version. Being anchored at both ends, it excludes the build-number tags
  (`idea/263.5153.40`) and every pre-release (`idea/2026.3-eap-3`, `-rc`, `-beta`, `-preview`) in
  the same pass.

The optional patch group is what keeps the **initial release of each line** — `2026.2` is a real
IntelliJ version, as much as `2026.2.3`, and a pattern demanding a patch would silently drop it.

The repository carries around 8500 `idea/` refs; this narrows them to the ~24 that are releases.

### b. The newest version is the *last* line

`git ls-remote` sorts by ref name, so the list arrives oldest first:

```sh
… | sort -V | tail -1      # latest release
```

Use `sort -V`, never plain `sort`, for two reasons: the patch is `{1,2}` digits, so a future
`2026.1.10` would rank below `2026.1.5` as a string; and an initial release has to rank below its
own patches, which `sort -V` gets right (`2026.2` then `2026.2.1`). No two-digit patch exists yet,
which is exactly why that half is easy to get wrong.

### c. Confirm the version with the user

**If the user already named a version**, do not ask — they have answered this. Confirm it appears
in the list and carry it forward. Come back to them only if it does not, and then say what was
searched.

Otherwise, never assume which IDE they are running. Use the **`AskUserQuestion` tool** and offer the
three newest versions — the last line of the sorted list is the default, marked `(Recommended)`.
The user picks an older one through "Other"; every release the repository tags — back to `2025.2` —
is already in the list you fetched, so no second query is needed.

## Step 2 — Read the Studio codename from that tag

The Android plugin in IntelliJ is built from an Android Studio release, and that release is
identified by a codename. It is recorded in `studio/version.bzl` at the tag chosen in step 1.

### a. Fetch `version.bzl` for the tag

```sh
curl -sL https://raw.githubusercontent.com/JetBrains/android/refs/tags/idea/<version>/studio/version.bzl
```

`<version>` is the tag without its `idea/` prefix — `2026.2.3` for the tag `idea/2026.2.3`, giving
`.../refs/tags/idea/2026.2.3/studio/version.bzl`.

### b. Extract `STUDIO_CODENAME`

```sh
curl -sL https://raw.githubusercontent.com/JetBrains/android/refs/tags/idea/<version>/studio/version.bzl \
  | sed -n 's/^STUDIO_CODENAME = "\(.*\)"$/\1/p'
```

The file is a handful of assignments:

```python
STUDIO_CODENAME = "Panda 2"
STUDIO_CONFIG = "canary"
STUDIO_VERSION = "Canary"
STUDIO_MICRO_PATCH = "2.1"
STUDIO_RELEASE_NUMBER = 1
```

**`STUDIO_CODENAME` is the only value this skill needs.** `STUDIO_VERSION` and `STUDIO_CONFIG`
describe the Studio release channel that IntelliJ release was cut from, not the AGP ceiling — do
not substitute them.

A codename spans several IntelliJ releases, and the mapping is not one-to-one with the year:

| Tag | `STUDIO_CODENAME` |
| --- | --- |
| `idea/2026.2.3` | `Panda 2` |
| `idea/2026.2` | `Panda 2` |
| `idea/2026.1.5` | `Panda 1` |
| `idea/2026.1.3` | `Panda 1` |

So two different IntelliJ versions can share an answer, and the codename — not the IntelliJ version
— is what the next step looks up.

Carry the codename forward.

## Step 3 — Look the codename up in the compatibility table

Google publishes the AGP range each Android Studio release accepts, at
https://developer.android.com/studio/releases#android_gradle_plugin_and_android_studio_compatibility.
The answer is the top of the range for the codename from step 2.

**Do not use `WebFetch` for this page.** Its extraction drops the table entirely and reports the
page as if no compatibility table existed. Fetch the HTML and parse it:

```sh
curl -sL -A "Mozilla/5.0" https://developer.android.com/studio/releases | python3 -c '
import sys, re, html
s = sys.stdin.read(); want = sys.argv[1]
for row in re.findall(r"<tr.*?</tr>", s, re.S):
    cells = [re.sub(r"\s+", " ", html.unescape(re.sub(r"<[^>]+>", "", c))).strip()
             for c in re.findall(r"<t[hd][^>]*>(.*?)</t[hd]>", row, re.S)]
    if len(cells) == 2 and cells[0].split("|")[0].strip() == want:
        print(cells[1].split("-")[-1].strip()); break
else:
    sys.exit("codename not found: " + want)
' "Panda 2"
```

### How the table is shaped

It has **two** columns, not three, and the first one packs the codename and the Android Studio
version into a single cell separated by a literal `|`:

```html
<tr>
    <td>Panda 2 | 2025.3.2</td>
    <td>7.0-9.1</td>
</tr>
```

So the match is on the text *before* the pipe, and the AGP cell is a range — `7.0-9.1` — whose
**upper bound is the answer**. Splitting the first cell on `|` and the second on `-` is what the
command above does.

| First cell | AGP range | Answer |
| --- | --- | --- |
| `Quail 4 \| 2026.1.4` | `7.1-9.4` | `9.4` |
| `Panda 2 \| 2025.3.2` | `7.0-9.1` | `9.1` |
| `Panda 1 \| 2025.3.1` | `7.0-9.0` | `9.0` |
| `Otter \| 2025.2.1` | `4.0-8.13` | `8.13` |

**The version in that first cell is the Android Studio version, not the IntelliJ version.** `Panda
2` is Studio `2025.3.2` while the IntelliJ release carrying it is `2026.2.3` — never match on that
column, and never report it as the IDE version. The codename is the only reliable key between the
two sources.

Note also that the upper bound is a `{major}.{minor}` with no patch, and that `8.13` sorts above
`8.9` numerically but below it as a string — compare the components as integers if you compare at
all.

### When the codename is absent

A very new IntelliJ release can carry a codename Google has not yet listed. Do not guess it from
the neighbouring row: say the codename is unlisted, report the newest codename the table does
contain with its ceiling, and let the user decide whether to treat that as the bound.

## Step 4 — Find the latest patch of that AGP line

Step 3 gives a `{major}.{minor}` ceiling; the version to actually write is its newest patch.
Google's Maven repository is the authority:

```sh
MINOR=9.1
curl -sL https://dl.google.com/dl/android/maven2/com/android/tools/build/gradle/maven-metadata.xml \
  | grep -oE "<version>${MINOR}\.[0-9]+</version>" \
  | sed 's/<[^>]*>//g' \
  | sort -V | tail -1
```

`com.android.tools.build:gradle` is AGP itself; the `com.android.application` plugin marker carries
the same versions, so either works and this one is canonical.

### Pre-releases must be excluded

The metadata lists every alpha, beta and rc alongside the stable versions:

```
9.1.0-alpha01 … 9.1.0-rc01 9.1.0 9.1.1
9.4.0-alpha01 … 9.4.0-rc02 9.4.0
9.5.0-alpha01 … 9.5.0-alpha06
```

The `</version>` in the pattern above is what excludes them — it forces the string to end right
after the patch digits, so `9.1.0-rc01` cannot match.

**Do not read `<latest>` or `<release>` from that file.** Both currently say `9.5.0-alpha06`: in
this repository `<release>` is not a stable release, and trusting it would hand back an alpha of a
line the IDE does not even support.

`sort -V` rather than `sort`, so `9.1.10` ranks above `9.1.9`.

### Expected results

| Ceiling | Latest patch |
| --- | --- |
| `9.4` | `9.4.0` |
| `9.3` | `9.3.3` |
| `9.1` | `9.1.1` |
| `8.13` | `8.13.2` |

A line can have **no stable patch yet** — `9.5` currently matches nothing, being alpha-only. If that
happens for the ceiling itself, report that the supported line has no stable release yet and give
the newest stable version below it instead of falling back to an alpha.

## Result

Report the whole chain, because the intermediate hops are what make the answer checkable:

- the IntelliJ version chosen in step 1, with its tag;
- the `STUDIO_CODENAME` it maps to;
- the AGP `{major}.{minor}` ceiling for that codename;
- the latest stable patch of that line — the version to use.

> IntelliJ 2026.2.3 (`idea/2026.2.3`) ships Android Studio `Panda 2`, which supports AGP up to
> **9.1**. The latest patch of that line is **9.1.1**.

That is the value for the `android-gradlePlugin` entry of the version catalog. It matters to the
`gradle-kotlin-project` skill, whose dependency-update procedure otherwise takes the newest AGP
from its own release page — which for this IDE would be `9.4.0`, three minor lines above what it
can open.
