# Creating a new project

The procedure, then every file at the root of a fresh Kodein Gradle project. Fill in the project
name, group and module list; keep the comments.

## Before you start

Ask these six, in one go. Nothing is written until they are answered — each one decides the content
of a file, and guessing means rewriting.

| Question | What it decides |
| --- | --- |
| **Project name**, and `rootProject.name` if it differs | The directory, the module name prefix (§2 of the skill), `rootProject.name` |
| **Maven group** | `allprojects { group = … }` |
| **JVM-only, multiplatform, or both** | Which plugins the root declares, which module template applies |
| **Android target?** | The `google()` repositories, the `android.*` properties, the AGP plugin aliases |
| **Compose Multiplatform?** | The Compose plugin aliases — **and the `google()` repositories, even with no Android target**, since the Compose plugin resolves AndroidX artifacts that are not on Maven Central |
| **Published to Maven Central?** | The `mavenPublish` plugin, `SONATYPE_HOST`, the root POM block, the release workflow |

A published project needs its GitHub org and repository too — the POM's `url`, `scm` and
`issueManagement` all name it. Ask for it with the publishing question rather than after.

## Order of operations

### 1. Write the root build files

`settings.gradle.kts` and `build.gradle.kts` (root), from the templates below.

Write `gradle.properties` and `gradle/libs.versions.toml` in the same pass: the root build script
resolves `libs.plugins.*` aliases, so the catalog has to exist before step 3 can configure at all.

Nothing else yet — the project files come in step 4, once the build is known to work.

### 2. Install the Gradle wrapper

There is no wrapper yet, so this needs a real Gradle. **Prefer a distribution the user already
has**, under `~/.gradle/wrapper/dists/` — any directory matching `*-bin` or `*-all` (the suffix
matters: a stray `gradle-9.5.1.bin` with a dot is not one), and take the highest version:

```sh
GRADLE=$(ls ~/.gradle/wrapper/dists/gradle-*-bin/*/gradle-*/bin/gradle \
             ~/.gradle/wrapper/dists/gradle-*-all/*/gradle-*/bin/gradle 2>/dev/null \
         | sort -V | tail -1)
"$GRADLE" wrapper
```

Each distribution sits under a hash directory, which is why the glob has a `*` in the middle:
`~/.gradle/wrapper/dists/gradle-9.7.1-bin/1w1c7tv4s851m17nbqdsro2tv/gradle-9.7.1/bin/gradle`.

**Only if there is none**, download the latest stable `-bin` from
https://services.gradle.org/distributions/, use it, and delete it again:

```sh
mkdir .tmp-gradle
cd .tmp-gradle
curl -L https://services.gradle.org/distributions/gradle-9.7.1-bin.zip -o gradle.zip
unzip gradle.zip
cd ..
.tmp-gradle/gradle-9.7.1/bin/gradle wrapper
rm -rf .tmp-gradle
```

The temporary directory goes inside the project and is removed in the same step — never leave it
behind, and never add it to `.gitignore` instead of deleting it.

### 3. Validate the scripts

```sh
./gradlew
```

One bare run of the wrapper: it downloads the distribution, configures the build, and fails loudly
on anything wrong in settings, the catalog or the root script. Do this before writing a single
module — a mistake found here is one file to fix, not ten.

### 4. Write the project files

Now that the build works, add the files around it — `README.adoc`, `.gitignore`, and the GitHub
Actions workflows. Templates for all three are below.

Keep them basic: a `README.adoc` with the standard header, the project name and a one-line
description is enough at this point, and it is the right time to create it — not because there is
much to say yet, but because the file exists to be filled in as the project grows. The
`asciidoc-writing` skill has the house rules for the prose.

Write `.github/workflows/release.yml` only if the project publishes.

### 5. Offer to create modules

The project root is complete and configures. Offer the module set that matches the answers from
*Before you start* — and let the user pick; do not create modules unasked. `module-templates.md`
has the shape for each.

## `settings.gradle.kts`

```kotlin
dependencyResolutionManagement {
    @Suppress("UnstableApiUsage")
    repositories {
        mavenCentral()
    }
}

plugins {
    // Downloads the JDKs the subprojects' toolchains ask for. Pinned here rather than in
    // gradle/libs.versions.toml because a settings plugin can't resolve a version-catalog alias -
    // the catalog isn't built yet at this point.
    // https://github.com/gradle/foojay-toolchains/releases
    id("org.gradle.toolchains.foojay-resolver-convention") version "1.0.0"
}

rootProject.name = "Kosi-Nomi"

enableFeaturePreview("TYPESAFE_PROJECT_ACCESSORS")

include(
    ":demo:demo-server",
    ":demo:demo-shared",

    ":processor:nomi-processor-common",
    ":processor:nomi-processor-shared",

    ":runtime:nomi-runtime-client",
    ":runtime:nomi-runtime-server",
    ":runtime:nomi-runtime-shared",
)
```

A single `include(…)` call, one module per line, sorted lexicographically with a blank line
between groups (§3 of the skill) — not in dependency order.

An Android build needs `google()` in both repository blocks, and a `pluginManagement` block,
since AGP is not on Maven Central. **A Compose Multiplatform build needs the same, even with no
Android target** — the Compose plugin pulls AndroidX artifacts that Maven Central does not carry:

```kotlin
pluginManagement {
    repositories {
        gradlePluginPortal()
        google()
        mavenCentral()
    }
}

dependencyResolutionManagement {
    @Suppress("UnstableApiUsage")
    repositories {
        google()
        mavenCentral()
    }
}
```

Add `mavenLocal()` only while consuming a locally published snapshot of a sibling project, and
take it out again.

## `gradle.properties`

```properties
org.gradle.jvmargs = -Xmx4g -XX:MaxMetaspaceSize=2g
org.gradle.parallel = true
org.gradle.caching = true
org.gradle.configuration-cache = true

# Toolchain
jvm.toolchain = 17

# Android
android.compileSdk = 37
android.targetSdk = 37
android.minSdk = 26

# Maven Publish
SONATYPE_HOST=CENTRAL_PORTAL
```

Drop the `Android` block on a project with no Android target, and the `Maven Publish` block on one
that publishes nothing. `jvm.toolchain` stays on every project — every module reads it with
`jvmToolchain(findProperty("jvm.toolchain")!!.toString().toInt())`, so the JDK moves for the whole
build in one edit.

## `gradle/libs.versions.toml`

The starting point for a library with a KSP processor — `kotlin-multiplatform` and `kotlin-jvm` are
both declared, since a project that publishes a multiplatform runtime almost always has a JVM-only
processor beside it:

```toml
[versions]
coroutines = "1.11.0"               # https://github.com/Kotlin/kotlinx.coroutines/releases
kctfork = "0.13.0"                  # https://github.com/ZacSweers/kotlin-compile-testing/releases
kotlin = "2.4.10"                   # https://kotlinlang.org/docs/releases.html#release-history
kotlinPoet = "2.4.0"                # https://github.com/square/kotlinpoet/releases
ksp = "2.3.11"                      # https://github.com/google/ksp/releases
mavenPublish = "0.37.0"             # https://github.com/vanniktech/gradle-maven-publish-plugin/releases

[plugins]
kotlin-jvm = { id = "org.jetbrains.kotlin.jvm", version.ref = "kotlin" }
kotlin-multiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
kotlin-plugin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
mavenPublish = { id = "com.vanniktech.maven.publish", version.ref = "mavenPublish" }

[libraries]
kotlin-test = { module = "org.jetbrains.kotlin:kotlin-test", version.ref = "kotlin" }
kotlin-test-junit = { module = "org.jetbrains.kotlin:kotlin-test-junit", version.ref = "kotlin" }
kotlinPoet-ksp = { module = "com.squareup:kotlinpoet-ksp", version.ref = "kotlinPoet" }
kotlinx-coroutines = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "coroutines" }
kotlinx-coroutines-test = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-test", version.ref = "coroutines" }
kctfork-ksp = { module = "dev.zacsweers.kctfork:ksp", version.ref = "kctfork" }
ksp-symbolProcessingApi = { module = "com.google.devtools.ksp:symbol-processing-api", version.ref = "ksp" }
```

A processor module that tests with kctfork also needs the implementation pin — the comment is
part of the entry:

```kotlin
testImplementation(libs.kctfork.ksp)
// Pins the KSP2 implementation to match ksp-symbolProcessingApi's version - without this,
// kctfork:ksp's own POM drags in an older impl (2.3.9) than the API we compile against (2.3.11),
// an API-newer-than-impl skew that risks AbstractMethodError.
testImplementation(libs.ksp.aaEmbeddable)
```

```toml
ksp-aaEmbeddable = { module = "com.google.devtools.ksp:symbol-processing-aa-embeddable", version.ref = "ksp" }
```

## `build.gradle.kts` (root)

Plugins, coordinates, and — on a publishing project — the shared POM. Nothing else.

```kotlin
import com.vanniktech.maven.publish.MavenPublishBaseExtension
import org.gradle.kotlin.dsl.configure

plugins {
    alias(libs.plugins.kotlin.multiplatform) apply false
    alias(libs.plugins.kotlin.jvm) apply false
    alias(libs.plugins.kotlin.plugin.serialization) apply false
    alias(libs.plugins.mavenPublish) apply false
    alias(libs.plugins.ksp) apply false
}

allprojects {
    group = "org.kodein.rest.nomi"
    version = "0.1.0"
}

val mavenPublishPluginId = libs.plugins.mavenPublish.get().pluginId
subprojects {
    pluginManager.withPlugin(mavenPublishPluginId) {
        extensions.configure<MavenPublishBaseExtension> {
            pom {
                url = "https://github.com/kosi-libs/Nomi"
                licenses {
                    license {
                        name = "MIT License"
                        url = "https://opensource.org/licenses/MIT"
                        distribution = "repo"
                    }
                }
                issueManagement {
                    system.set("Github")
                    url.set("https://github.com/kosi-libs/Nomi/issues")
                }
                scm {
                    connection.set("https://github.com/kosi-libs/Nomi.git")
                    url.set("https://github.com/kosi-libs/Nomi")
                }
                developers {
                    developer {
                        name = "Kodein Koders"
                        email = "contact@kodein.net"
                        url = "https://kodein-koders.com"
                    }
                }
            }
        }
    }
}
```

`apply false` on every plugin: the root declares them so each is loaded into one classloader
instead of being re-resolved per subproject, but applies none of them. The list mirrors the
catalog's `[plugins]` block exactly — a plugin added to the catalog and not declared here is a
defect, so the two are edited together.

`pluginManager.withPlugin` is what keeps a module's own `mavenPublishing { }` block down to a name
and a description — the POM boilerplate is configured once, and only for modules that actually
publish.

## `README.adoc`

```adoc
= Nomi
:toc: macro
:toclevels: 2
:idprefix:
:idseparator: -
:source-highlighter: highlight.js

*Compile-time REST for Kotlin Multiplatform — declare your API once, get both sides.*

Nomi turns plain Kotlin interfaces into a working REST API.
```

The title is the product name alone. The bold line under it is a single-sentence tagline: what it
is, for whom, in the user's terms — not a feature list. Then a short paragraph of what it actually
does.

Everything else (usage, a `[source,kotlin]` sample, installation) arrives as the project grows. Add
`:icons: font` when the document gains its first admonition, not before. See the `asciidoc-writing`
skill for the rest.

## `.gitignore`

```gitignore
.gradle
build/
!gradle/wrapper/gradle-wrapper.jar
!**/src/main/**/build/
!**/src/test/**/build/

### IntelliJ IDEA ###
.idea/modules.xml
.idea/jarRepositories.xml
.idea/compiler.xml
.idea/libraries/
*.iws
*.iml
*.ipr
out/
!**/src/main/**/out/
!**/src/test/**/out/

### Kotlin ###
.kotlin

### VS Code ###
.vscode/

### Mac OS ###
.DS_Store

### Local review scratch file
ISSUES.adoc
```

Android projects add `local.properties`; projects with an Xcode workspace add `xcuserdata/`,
`*.xcuserstate` and `DerivedData/`.

## `.github/workflows/test.yml`

```yaml
name: check

on:
  push:
    branches:
      - main
    paths-ignore:
      - '**.md'
      - '**.adoc'
      - '**/.gitignore'
  pull_request:
    paths-ignore:
      - '**.md'
      - '**.adoc'
      - '**/.gitignore'
  workflow_dispatch:

jobs:

  check:
    runs-on: macOS-latest
    steps:
      - name: Check out
        uses: actions/checkout@v7
      - name: Set up JDK Temurin 17
        uses: actions/setup-java@v5
        with:
          distribution: temurin
          java-version: 17
      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v6
      - name: Build
        run: ./gradlew --stacktrace --scan build
        shell: bash
```

## `.github/workflows/release.yml`

```yaml
name: build and publish a release

on:
  release:
    types: [published]

jobs:
  build-upload:
    runs-on: macOS-latest
    steps:
      - name: Check out
        uses: actions/checkout@v7
      - name: Set up JDK Temurin 17
        uses: actions/setup-java@v5
        with:
          distribution: temurin
          java-version: 17
      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v6
      - name: Build
        run: ./gradlew --stacktrace --scan build
        shell: bash
      - name: Upload to Maven Central
        env:
          ORG_GRADLE_PROJECT_mavenCentralUsername: ${{ secrets.CENTRAL_PORTAL_TOKEN_USERNAME }}
          ORG_GRADLE_PROJECT_mavenCentralPassword: ${{ secrets.CENTRAL_PORTAL_TOKEN_PASSWORD }}
          ORG_GRADLE_PROJECT_signingInMemoryKey: ${{ secrets.PGP_SIGNING_KEY }}
          ORG_GRADLE_PROJECT_signingInMemoryKeyPassword: ${{ secrets.PGP_SIGNING_PASSWORD }}
        run: ./gradlew --stacktrace --scan -PRELEASE_SIGNING_ENABLED=true publishAndReleaseToMavenCentral
        shell: bash
```

`macOS-latest` on both: it is the only runner that can compile the Apple targets, so a Linux
runner would silently check less than the target set declares. Credentials are repository secrets,
never properties in the repo.
