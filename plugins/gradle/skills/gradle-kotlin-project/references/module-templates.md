# Adding a module

The procedure, then one build script shape per kind of module. Copy a template whole, including
the comments.

Multiplatform and JVM-only modules sit side by side in the same build; pick the shape that matches
what the module is, not what the rest of the project happens to be. Jump to
[JVM-only module](#jvm-only-module) if the code can never leave the JVM — the target-set section
does not apply to it.

## Before you start

Ask these five, in one go.

**1. Is it a library, an application, or a demo?**

This decides `explicitApi()` (libraries only), whether a `mavenPublishing` block exists at all, and
where the module lives.

**2. If it is a library — will it be published?**

Publishing decides the module's *name*, in both directions:

- **Published → the name starts with the project's short name and a dash** —
  `nomi-runtime-client`, `vie-timemachine-shared` — because that name becomes the artifact id on
  Maven Central.
- **Not published → no prefix.** It is named for what it is: `demo-shared`, `server`. Prefixing an
  unpublished module is an error — `inara-demo` should be `demo`.

Getting this wrong is expensive to correct once published, so settle it before creating the
directory.

A published module also needs a one-line description for its POM.

**3. Multiplatform or JVM-only — and if multiplatform, which platforms?**

JVM-only when the code can never run anywhere else: a KSP processor, a server, a Gradle plugin.
Multiplatform otherwise, and then: the full target set (the default for a library), or a narrowed
set? Ask which platforms rather than assuming — a Compose UI module and a serialization-only
utility do not carry the same list.

**4. What will it use?**

Offer these, since each pulls in both plugins and dependencies:

| Offer | Plugins | Libraries |
| --- | --- | --- |
| **KotlinX Serialization** | `kotlin-plugin-serialization` | `kotlinx-serialization-json` |
| **Compose** | `kotlin-plugin-compose`, `compose-multiplatform` | `compose-runtime`, `compose-foundation`, `compose-material3`, `compose-ui` |
| **Ktor Client** (multiplatform) | — | `ktor-client`, `ktor-client-contentNegotiation`, `ktor-serialization-json`, plus one engine per target |
| **Ktor Server** (JVM-only) | — | `ktor-server`, `ktor-server-cio`, `ktor-server-contentNegotiation`, `ktor-serialization-json` |

Ktor Client engines are per-target, not common — `ktor-client-okhttp` in `jvmMain`,
`ktor-client-darwin` in `iosMain`, `ktor-client-js` in `webMain`.

Compose needs `google()` in the settings repositories even with no Android target; add it if the
project does not have it yet.

**A new Compose component takes its version from the Components table** at the end of the notes for
the `compose-multiplatform` release the project is already on — not from its own latest release.
Adding Navigation3 to a project on Compose 1.12.0 means
https://github.com/JetBrains/compose-multiplatform/releases/tag/v1.12.0 and the version that table
gives, even when a newer one exists. Point the catalog entry's comment at the Compose releases page.

**5. Which directory?**

Propose one, don't ask open-ended. Per §2 of the skill: a flat directory while the project has only
a few modules, a group directory once a second dimension appears — a new `demo-client` joins the
existing `demo/`, a third runtime variant turns `nomi-runtime-*` into `runtime/`. Say which you
propose and why, and let the user override.

## Order of operations

### 1. Catalog first

Add any new `[versions]`, `[plugins]` and `[libraries]` entries to `gradle/libs.versions.toml` —
each version entry with its release-page comment.

**Every new plugin also gets an `alias(libs.plugins.x) apply false` line in the root
`build.gradle.kts`.** A catalog plugin entry without one is an error (§1 of the skill).

### 2. Register the module

Add it to the `include(…)` call in `settings.gradle.kts`, in its group, at its lexicographic
position (§3 of the skill) — not after the modules it depends on.

### 3. Write the build script

From the matching template below. A published module ends with its `mavenPublishing { pom { … } }`
block carrying only `name` and `description`.

### 4. Create the source tree

`src/commonMain/kotlin/<package>/` for a multiplatform module, `src/main/kotlin/<package>/` for a
JVM-only one, plus the matching test source set. An empty source set is fine; a missing one makes
the module look broken in the IDE.

### 5. Validate

```sh
./gradlew :path:to:module:build
```

Then `./gradlew build` before calling it done — a new module can break the root build through a
plugin classpath conflict without failing on its own.

## The full target set

Every multiplatform library declares this block, verbatim, right after the `jvmToolchain(…)` line:

```kotlin
    jvm()

    iosArm64()
    iosSimulatorArm64()
    iosX64()

    watchosDeviceArm64()
    watchosSimulatorArm64()
    watchosArm32()
    watchosArm64()

    tvosArm64()
    tvosSimulatorArm64()

    androidNativeArm32()
    androidNativeArm64()
    androidNativeX64()
    androidNativeX86()

    linuxX64()
    linuxArm64()
    macosArm64()
    // No macosX64: Kotlin deprecated it in 2.3.20 and will remove it. iosX64 above is the Intel
    // iOS simulator (still supported), not Intel macOS - the two are unrelated.
    mingwX64()

    js {
        browser()
        nodejs()
    }
    @OptIn(org.jetbrains.kotlin.gradle.ExperimentalWasmDsl::class)
    wasmJs {
        browser()
        nodejs()
    }
```

Narrow it only when a dependency does not support a target. A Compose UI module, for example,
carries just `jvm()`, the Android target, `iosArm64()`, `iosSimulatorArm64()`, `js { browser() }`
and `wasmJs { browser() }` — and an application module replaces `browser()` with
`browser(); binaries.executable()`.

## Multiplatform library

```kotlin
plugins {
    alias(libs.plugins.kotlin.multiplatform)
    alias(libs.plugins.mavenPublish)
}

kotlin {
    jvmToolchain(findProperty("jvm.toolchain")!!.toString().toInt())

    // ... the full target set ...

    explicitApi()

    sourceSets {
        commonMain.dependencies {
            api(projects.runtime.nomiRuntimeShared)
            implementation(libs.kotlinx.coroutines)
        }
        commonTest.dependencies {
            implementation(libs.kotlin.test)
            implementation(libs.kotlinx.coroutines.test)
        }
    }
}

mavenPublishing {
    pom {
        name = "nomi-runtime-client"
        description = "Nomi runtime for client applications"
    }
}
```

`api` for a dependency that appears in this module's public signatures, `implementation` for
everything else. The POM carries only `name` and `description` — the rest comes from the root
(see `new-project.md`).

Per-target dependencies go in the matching accessor:

```kotlin
        jvmMain.dependencies {
            implementation(libs.ktor.client.okhttp)
        }
        iosMain.dependencies {
            implementation(libs.ktor.client.darwin)
        }
        webMain.dependencies {
            implementation(libs.ktor.client.js)
        }
```

## JVM-only module

A processor, a server runtime, anything that never leaves the JVM:

```kotlin
plugins {
    alias(libs.plugins.kotlin.jvm)
    alias(libs.plugins.mavenPublish)
}

kotlin {
    jvmToolchain(findProperty("jvm.toolchain")!!.toString().toInt())
    explicitApi()
}

dependencies {
    implementation(projects.processor.nomiProcessorCommon)

    implementation(libs.kotlinPoet.ksp)
    implementation(libs.ksp.symbolProcessingApi)

    testImplementation(projects.runtime.nomiRuntimeShared)

    testImplementation(libs.kotlin.test.junit)
}

mavenPublishing {
    pom {
        name = "nomi-processor-shared"
        description = "Nomi processor for generating shared code"
    }
}
```

Note `kotlin-test-junit` here, where a multiplatform module uses `kotlin-test`.

A module whose own public signatures expose a dependency's types promotes it to `api`, with the
reason recorded:

```kotlin
    // KSP types appear in this module's own public signatures (ResourceAnalyzer, Reporter,
    // AnalyzedResource, ...), so they belong in its published POM's compile scope.
    api(libs.ksp.symbolProcessingApi)
```

## Multiplatform module that runs a KSP processor

Three pieces, and all three are required — the generated-source directory, the
`kspCommonMainMetadata` configuration, and the task-dependency workaround:

```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompilationTask

plugins {
    alias(libs.plugins.kotlin.multiplatform)
    alias(libs.plugins.kotlin.plugin.serialization)
    alias(libs.plugins.ksp)
}

kotlin {
    jvmToolchain(findProperty("jvm.toolchain")!!.toString().toInt())

    // ... the full target set ...

    sourceSets {
        commonMain {
            kotlin.srcDir(layout.buildDirectory.dir("generated/ksp/metadata/commonMain/kotlin"))
        }
        commonMain.dependencies {
            implementation(projects.runtime.nomiRuntimeClient)
        }
        commonTest.dependencies {
            implementation(libs.kotlin.test)
            implementation(libs.kotlinx.coroutines.test)
        }
    }
}

dependencies {
    add("kspCommonMainMetadata", projects.processor.nomiProcessorClient)
}

// Every compilation - and every per-target KSP task, which also reads commonMain as part of its
// merged source set even though it has no processor registered - consumes the KSP-generated
// commonMain sources, so all of them must wait for KSP to run.
tasks.matching { it.name != "kspCommonMainKotlinMetadata" }.configureEach {
    if (name.startsWith("ksp") || this is KotlinCompilationTask<*>) {
        dependsOn("kspCommonMainKotlinMetadata")
    }
}
```

The processor can equally come from the catalog when it is an external library:
`add("kspCommonMainMetadata", libs.inara.processor)`.

A **JVM-only** consumer needs none of this — the `ksp(…)` configuration is enough:

```kotlin
dependencies {
    implementation(projects.runtime.nomiRuntimeServer)

    ksp(projects.processor.nomiProcessorServer)
}
```

## Android library in a multiplatform module

Applied alongside `kotlin.multiplatform`, with the SDK levels read from `gradle.properties`:

```kotlin
plugins {
    alias(libs.plugins.kotlin.multiplatform)
    alias(libs.plugins.android.kotlin.multiplatform.library)
    alias(libs.plugins.kotlin.plugin.compose)
    alias(libs.plugins.compose.multiplatform)
}

kotlin {
    jvmToolchain(findProperty("jvm.toolchain")!!.toString().toInt())

    jvm()

    android {
        namespace = "org.kodein.arch.vie.demo"
        compileSdk = findProperty("android.compileSdk")!!.toString().toInt()
        minSdk = findProperty("android.minSdk")!!.toString().toInt()
    }

    iosArm64()
    iosSimulatorArm64()

    js {
        browser()
        binaries.executable()
    }

    @OptIn(org.jetbrains.kotlin.gradle.ExperimentalWasmDsl::class)
    wasmJs {
        browser()
        binaries.executable()
    }
}
```

On AGP below 9.1 that block is named `androidLibrary { }`; it is `android { }` from 9.1 on.
Android resources need `androidResources { enable = true }` inside it.

## Android application

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.compose.multiplatform)
    alias(libs.plugins.kotlin.plugin.compose)
}

android {
    namespace = "org.kodein.arch.vie.demo.android"
    compileSdk = findProperty("android.compileSdk")!!.toString().toInt()

    defaultConfig {
        applicationId = "org.kodein.arch.vie.demo.android"
        minSdk = findProperty("android.minSdk")!!.toString().toInt()
        targetSdk = findProperty("android.targetSdk")!!.toString().toInt()
        versionCode = 1
        versionName = "1.0"
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}

kotlin {
    jvmToolchain(25)
}

dependencies {
    implementation(projects.demo.demoShared)

    implementation(libs.androidx.activity.compose)

    implementation(libs.compose.ui)
    implementation(libs.compose.ui.tooling)
}
```

An Android application is the usual reason to leave the default toolchain: AGP's own tooling wants
a newer JDK than 17, while `compileOptions` still targets 17 bytecode.

## Compose desktop / web application

```kotlin
plugins {
    alias(libs.plugins.kotlin.multiplatform)
    alias(libs.plugins.kotlin.plugin.compose)
    alias(libs.plugins.compose.multiplatform)
}

kotlin {
    jvmToolchain(findProperty("jvm.toolchain")!!.toString().toInt())

    jvm()

    js {
        browser()
        binaries.executable()
    }

    @OptIn(org.jetbrains.kotlin.gradle.ExperimentalWasmDsl::class)
    wasmJs {
        browser()
        binaries.executable()
    }

    sourceSets {
        commonMain.dependencies {
            implementation(projects.demo.demoShared)

            implementation(libs.compose.foundation)
            implementation(libs.compose.runtime)
        }
        jvmMain.dependencies {
            implementation(compose.desktop.currentOs)
            implementation(libs.kotlinx.coroutines.swing)
        }
    }
}

compose {
    desktop.application.mainClass = "org.kodein.arch.vie.demo.MainKt"
}
```

`compose.desktop.currentOs` is one of the few dependencies that is *not* a catalog alias — it is
an accessor the Compose plugin provides.

Naming the JVM target (`jvm("desktop")`) is a deliberate choice for a project that ships a desktop
app beside a server; it costs the source-set accessors, so those modules use
`val desktopMain by getting` instead of `jvmMain`.

## iOS framework

```kotlin
kotlin {
    jvmToolchain(findProperty("jvm.toolchain")!!.toString().toInt())

    listOf(
        iosArm64(),
        iosSimulatorArm64(),
    ).forEach { target ->
        target.binaries.framework {
            baseName = "DemoIos"
            isStatic = true
        }
    }

    sourceSets {
        iosMain.dependencies {
            implementation(projects.demo.demoShared)
            implementation(libs.compose.runtime)
            implementation(libs.compose.ui)
        }
    }
}
```

`export(projects.someModule)` inside the framework block when the Swift side must see that
module's API too.

## Compiler options

Opt-ins and flags go in a `compilerOptions { }` block at the end of `kotlin { }`, never as a
source-file annotation repeated across the module:

```kotlin
    compilerOptions {
        optIn.addAll("kotlin.uuid.ExperimentalUuidApi", "kotlin.time.ExperimentalTime")

        progressiveMode = true
    }
```

A flag that works around a specific problem carries the reason:

```kotlin
    compilerOptions {
        freeCompilerArgs.add("-Xwarning-level=ERROR_SUPPRESSION:disabled")
    }
```
