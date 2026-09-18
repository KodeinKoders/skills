---
name: gradle-kotlin-project
description: "Kodein's conventions for Kotlin/Kotlin Multiplatform Gradle builds — settings.gradle.kts, gradle.properties, the gradle/libs.versions.toml version catalog, root and module build scripts, target sets, KSP wiring, Maven Central publishing, .gitignore and CI. Use when creating a new Kotlin or KMP Gradle project, adding a module or subproject to one, adding or upgrading a dependency or plugin version, wiring a KSP processor, setting up publishing, or reviewing an existing build for consistency with these conventions."
---

# Kotlin Gradle projects

How Kodein builds are laid out, for any Kotlin Gradle project — multiplatform, JVM-only, or a
mix of both in one build. Apply these when creating a project, adding a module, or touching an
existing build script — and match the surrounding file first if it already deviates consistently.

Three companion files hold the procedures and the copy-paste material, so they are read only when
needed:

- `references/new-project.md` — the procedure for creating a project, and every root-level file.
- `references/module-templates.md` — the procedure for adding a module, and the build script
  shapes for each kind.
- `references/updating-dependencies.md` — the procedure for upgrading every version in the build.

**Creating a project, adding a module, or updating dependencies: read the matching file first and
follow its checklist.** Each opens with the questions to ask and the order the steps go in. Do not
start editing before reading it.

## 1. Non-negotiables

These hold in every project, and a build script that breaks one is wrong even if it works:

- **Everything is Kotlin DSL** — `settings.gradle.kts`, `build.gradle.kts`. Never Groovy.
- **No coordinate is ever written in a build script.** Every dependency and plugin comes
  from `gradle/libs.versions.toml` — `libs.ktor.client`, `alias(libs.plugins.kotlin.multiplatform)`.
- **No module declares a repository.** Repositories live in `settings.gradle.kts` only.
- **Project dependencies use typed accessors** — `projects.runtime.nomiRuntimeShared`, never
  `project(":runtime:nomi-runtime-shared")`. This is what
  `enableFeaturePreview("TYPESAFE_PROJECT_ACCESSORS")` buys, and it is enabled everywhere.
- **Every module sets its own `jvmToolchain(…)`, from the `jvm.toolchain` property.** It is never
  inherited from the root, and the version is never hardcoded in a module.
- **The root `build.gradle.kts` declares plugins and nothing else** (plus `allprojects` coordinates
  and, when publishing, the shared POM) — no compilation config is applied to subprojects from it.
- **Every plugin in the catalog is declared in the root `build.gradle.kts` as
  `alias(libs.plugins.x) apply false`.** A `[plugins]` entry in `gradle/libs.versions.toml` with no
  matching root declaration is an error — adding one to the catalog means adding both lines. The
  root declaration resolves the plugin once into a single classloader; without it each subproject
  applying that alias re-resolves it into its own.
- **A non-obvious line gets a comment saying why.** See §9.

## 2. Project layout

A project is always multiple modules — never one monolith.

- Flat module directories while there are only a few (`inara-runtime`, `inara-processor`,
  `tests`). Group them into subdirectories once a project grows a second dimension
  (`runtime/`, `processor/`, `demo/`), giving `:group:module` paths.
- **The project-name prefix means published, and nothing else does.** It runs both ways:
  - **A published library module's name starts with the project's short name and a dash** —
    `inara-runtime`, `nomi-runtime-client`, `vie-timemachine-shared`. The name becomes the artifact
    id, so it has to be unique across all of Maven Central, not just across this build.
  - **A module carrying the prefix must be published.** `inara-demo` is an error: the demo is not
    published, so it is named `demo`. Same for `nomi-tests`, `vie-sample` — drop the prefix.
- **Everything else is named for what it is** — `demo-shared`, `demo-android`, `server`, `shared`.
  The prefix would buy nothing there and only lengthen the path.

  Reading a module path therefore answers "is this published?" on its own, with no trip to the
  build script.
- The group directory is *not* prefixed: `runtime/nomi-runtime-client`, not
  `nomi-runtime/nomi-runtime-client`.
- **`rootProject.name` is set explicitly** and is the product name, not the directory name:
  `Kosi-Inara`, `Kosi-Nomi`, `Kosi-VIE`.

## 3. Ordering

**Lexicographic order, with a blank line between domains.** This holds in three places:

- every section of `gradle/libs.versions.toml` — `[versions]`, `[plugins]`, `[libraries]`;
- every `dependencies { }` and `…Main.dependencies { }` block in a build script;
- the `include(…)` call in `settings.gradle.kts`.

A **domain** is the ecosystem a name belongs to — everything `compose-*`, everything `ktor-*`,
everything `kotlinx-*` — or, in settings, a module group. Sort strictly by name inside a domain,
and put one blank line between domains. A short list that is all one domain has nothing to
separate, so it gets no blank lines.

Sorting is on the alias as written, which is what makes the naming of §4 pay off: every entry from
one library lands together without anyone having to group them by hand.

Two things order does *not* follow:

- **Not dependency order.** `include(…)` lists `nomi-runtime-client` before `nomi-runtime-shared`
  even though the first depends on the second. Gradle resolves the graph; a reader resolves names.
- **Not importance.** Nothing is hoisted because it matters more.

In a dependency block, configurations stay grouped — all `implementation` together, then
`testImplementation` — and project dependencies (`projects.*`) form the first domain, ahead of
catalog libraries:

```kotlin
dependencies {
    implementation(projects.processor.nomiProcessorCommon)

    implementation(libs.kotlinPoet.ksp)
    implementation(libs.ksp.symbolProcessingApi)

    testImplementation(libs.kctfork.ksp)
    testImplementation(libs.kotlin.test.junit)
}
```

## 4. The version catalog

`gradle/libs.versions.toml`, in `[versions]`, `[plugins]`, `[libraries]` order, each block sorted
per §3.

**Every `[versions]` entry carries a trailing comment linking its release page**, aligned into a
column. This is how a version gets checked for an upgrade, so an entry without one is incomplete:

```toml
[versions]
coroutines = "1.11.0"               # https://github.com/Kotlin/kotlinx.coroutines/releases
kotlin = "2.4.10"                   # https://kotlinlang.org/docs/releases.html#release-history
ksp = "2.3.11"                      # https://github.com/google/ksp/releases
mavenPublish = "0.37.0"             # https://github.com/vanniktech/gradle-maven-publish-plugin/releases
```

- **One `kotlin` version drives every Kotlin plugin** — `kotlin-jvm`, `kotlin-multiplatform`,
  `kotlin-plugin-serialization`, `kotlin-plugin-compose` all `version.ref = "kotlin"`.
- **KSP has its own version line**, independent of Kotlin's — the old
  `<kotlin version>-<ksp version>` pairing is gone, so `ksp = "2.3.11"` alongside
  `kotlin = "2.4.10"` is not a mismatch. Upgrade it from its own release page.
- **Compose Multiplatform components are pinned by the Compose release, not chosen.** Every
  release's notes end with a **Components** table naming the exact version of each companion
  library — Material3, Lifecycle, Navigation, Navigation3, Savedstate, WindowManager Core — and
  those are the versions the catalog carries. They are never picked independently and never bumped
  on their own; they move only when `compose-multiplatform` moves, and then they all move together.
  This is why a stable `compose-multiplatform = "1.12.0"` sits beside
  `compose-material3 = "1.12.0-alpha03"`: the table says so. Their release-page comment points at
  https://github.com/JetBrains/compose-multiplatform/releases, not at the component's own repo.
- **Alias naming**: segments separated by `-`, camelCase *inside* a segment —
  `kotlinx-coroutines-test`, `ktor-client-contentNegotiation`, `ksp-symbolProcessingApi`,
  `kotlinPoet-ksp`. The first segment is the ecosystem, so everything from one library sorts together.
- **Android SDK levels do not go in the catalog** — they are `gradle.properties` entries (§5).

## 5. gradle.properties

```properties
org.gradle.jvmargs = -Xmx4g -XX:MaxMetaspaceSize=2g
org.gradle.parallel = true
org.gradle.caching = true
org.gradle.configuration-cache = true
```

All four performance flags are set on every project. If the configuration cache breaks on a
plugin, note why in a comment next to the disabled line rather than dropping the line.

**This file is where every build-wide version level is centralised**, so no module hardcodes one.
The JVM toolchain, on every project:

```properties
# Toolchain
jvm.toolchain = 17
```

and the Android SDK levels, on projects with an Android target:

```properties
# Android
android.compileSdk = 37
android.targetSdk = 37
android.minSdk = 26
```

Publishing projects add:

```properties
# Maven Publish
SONATYPE_HOST=CENTRAL_PORTAL
```

## 6. Modules

A module is multiplatform (`kotlin.multiplatform`) or JVM-only (`kotlin.jvm`). Pick JVM-only when
the code can never run anywhere else — a KSP processor, a server, a Gradle plugin — and
multiplatform otherwise. The two differ in how they declare targets, source sets and test
dependencies; everything else in this skill applies to both.

Full templates for each are in `references/module-templates.md`. The rules they encode:

**Every module**

- **`jvmToolchain(findProperty("jvm.toolchain")!!.toString().toInt())` is the first statement in
  `kotlin { }`** — the project default, `17`, comes from `gradle.properties` (§5). A module that
  genuinely needs another toolchain writes the literal instead (an Android app on 25, say) and says
  why; that is a choice, not a defect.
- **`explicitApi()` on every library module**; omit it on demos, samples and applications.
- **`api` only for a dependency that appears in the module's public signatures**, `implementation`
  for everything else — and record why when it isn't obvious.
- **`Experimental…` opt-ins are module-wide; `Delicate…` opt-ins never are.** The prefix says which:
  - An **`Experimental…`** marker (`ExperimentalAtomicApi`, `ExperimentalContracts`,
    `ExperimentalCoroutinesApi`, `ExperimentalUuidApi`, …) means *this API may still change*. The
    module has already accepted that by using it, so it goes in `compilerOptions { optIn.add(…) }`
    once, not on every file and declaration that touches it.
  - A **`Delicate…`** marker (`DelicateCoroutinesApi`, …) means *this API is stable but easy to
    misuse*. **A module-wide `Delicate…` opt-in is an error**: it silences the warning at every
    future call site, including the ones nobody has thought about yet. Opt in at the usage instead,
    with `@OptIn(DelicateCoroutinesApi::class)` on the smallest scope that covers it — the
    annotation is the record that this particular call was deliberate.

**Multiplatform modules**

- Declare **the full target set** (`references/module-templates.md` has it verbatim, including the
  `// No macosX64` note). Narrow it only when a dependency genuinely does not support a target — a
  Compose UI module, for instance, keeps jvm/android/ios/js/wasmJs.
- `js { browser(); nodejs() }` and `wasmJs { … }` under `@OptIn(ExperimentalWasmDsl::class)`; commit
  the resulting `kotlin-js-store/` (yarn.lock and `wasm/`).
- **Source sets use the accessor shorthand** — `commonMain.dependencies { }`,
  `commonTest.dependencies { }`. Fall back to `val desktopMain by getting` only where a custom
  target name (`jvm("desktop")`) leaves no accessor.
- **`kotlin-test` and `kotlinx-coroutines-test` in `commonTest`** as the baseline.

**JVM-only modules**

- No `sourceSets` block: dependencies go in a top-level `dependencies { }`, and `kotlin { }` holds
  just the toolchain and `explicitApi()`.
- **`kotlin-test-junit` in `testImplementation`**, where a multiplatform module uses `kotlin-test`.

**Android modules** read their SDK levels from the same properties as the toolchain:
`compileSdk = findProperty("android.compileSdk")!!.toString().toInt()`.

### KSP

**A JVM-only module needs nothing special**: plain `ksp(projects.someProcessor)` in its
`dependencies { }` block.

**A multiplatform module** running a processor over `commonMain` needs three things together —
the `kspCommonMainMetadata` configuration, the generated-source directory, and a task-dependency
workaround. They are copied verbatim, comment included, from `references/module-templates.md`.

## 7. License

**MIT, unless the project explicitly says otherwise.** It is the default, not a decision to
surface: a new project gets a `LICENSE` file with the MIT text, and the POM's `licenses` block
names MIT. Never ask which license to use when creating a project — write MIT and move on; the user
changes it if they need to.

Changing it later means changing both places, plus the `license` field of any plugin manifest the
repository carries.

## 8. Publishing

Vanniktech's `mavenPublish` plugin, with **the shared POM configured once in the root**, applied
to whichever subprojects carry the plugin:

```kotlin
val mavenPublishPluginId = libs.plugins.mavenPublish.get().pluginId
subprojects {
    pluginManager.withPlugin(mavenPublishPluginId) {
        extensions.configure<MavenPublishBaseExtension> {
            pom { /* url, licenses, issueManagement, scm, developers */ }
        }
    }
}
```

A publishable module then declares only what is specific to it — never a license block, never an
scm block:

```kotlin
mavenPublishing {
    pom {
        name = "nomi-runtime-client"
        description = "Nomi runtime for client applications"
    }
}
```

The full root block is in `references/new-project.md`.

## 9. Comments

Build scripts explain *why*. A line whose reason isn't obvious from reading it carries a comment,
and **the same explanation is repeated verbatim wherever the line is copied** — the reader of
module #4 needs it as much as the reader of module #1. The recurring ones (foojay pinning, the
dropped `macosX64` target, the KSP task dependency) are in the reference files; keep them when you
copy the surrounding code.

Write them for someone hitting the failure the line prevents: what breaks without it.

## 10. Git hygiene

`.gitignore` covers `.gradle`, `build/`, `.kotlin`, IDE noise and `.DS_Store`, and explicitly
un-ignores the wrapper jar (`!gradle/wrapper/gradle-wrapper.jar`). It also ignores **`ISSUES.adoc`**,
the local review scratch file — see the `issues-adoc` skill. The baseline file is in
`references/new-project.md`.

**The whole `.idea/` directory is ignored**, not a list of files inside it. IDEA writes new files
there on its own schedule, so an allow-list of specific entries (`.idea/modules.xml`,
`.idea/compiler.xml`, …) goes stale and lets machine-specific state into the repository. Nothing
under `.idea/` is shared — run configurations included; those belong in the README or a script.

Committed: the Gradle wrapper (jar included), `kotlin-js-store/`, `README.adoc`, and the agent
instructions (`.claude/`, `CLAUDE.md`).

## 11. CI

Two GitHub Actions workflows, both in `references/new-project.md`:

- `.github/workflows/test.yml` — `check` on push to `main`, on pull request, and
  `workflow_dispatch`; skips doc-only changes via `paths-ignore`.
- `.github/workflows/release.yml` — builds and publishes to Maven Central on a published release.

Both run on `macOS-latest` (the only runner that can build the Apple targets), set up Temurin JDK
17, use `gradle/actions/setup-gradle@v6`, and invoke `./gradlew --stacktrace --scan build`.

## 12. Verifying a change

After editing any build script:

```sh
./gradlew build            # what CI runs
```

Narrower checks while iterating:

- `./gradlew projects` — after touching `include(…)` in settings.
- `./gradlew :some-module:dependencies --configuration commonMainCompileClasspath` — after a
  dependency change.
- `./gradlew help` — cheapest way to confirm the catalog and settings still configure.

Run the full `build` before declaring the change done; a script that configures can still fail to
compile a target.
