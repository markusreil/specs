# SPEC — New Gradle Project Layout & Plugin Usage

Short spec for starting a project that consumes the `com.mreil.easy` plugin.

## Requirements

* Java 21+ — run Gradle with a 21 daemon (e.g. `JAVA_HOME=/usr/lib/jvm/java-21-openjdk`).
* Gradle 9.x via the wrapper.
* Configuration cache / parallel / build cache stay enabled; builds must stay CC-compatible.
* Kotlin Gradle Plugin 2.4.x. The implementation language is Kotlin.
* The Gradle project is always multi-module with an empty root.

## Layout

```
settings.gradle.kts          # plugin resolution + project includes
gradle.properties            # group, version, java.toolchainVersion, CC/parallel/caching
gradle/libs.versions.toml    # all plugins/dependencies, referenced by alias
build.gradle.kts             # root: easy.project + shared easy { } config
<module>/build.gradle.kts    # per-module + repo-specific deps only
<module>/src/main/{kotlin}/
<module>/src/test/{kotlin}          # unit suite (built-in `test`)
<module>/src/<name>Test/{kotlin}    # functional suite, auto-registered, optional
```

## settings.gradle.kts

```kotlin
pluginManagement {
    repositories { gradlePluginPortal(); mavenCentral() }
}
plugins { id("com.mreil.easy.settings") version "<version>" }

rootProject.name = "my-project"
include(":module1", ":module2", ...)
```

The settings plugin discovers contributors via SPI and fans them out to projects. It is the only
plugin that must be applied in settings.

## gradle.properties

```properties
group=com.example
version=1.0.0
java.toolchainVersion=21
org.gradle.configuration-cache=true
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.warning.mode=all
```

`group` and `version` are required — the `projectDefaults` contributor fails fast when either is
missing. Prefer declaring them here (the `release` contributor rewrites the `version=` line).

## build.gradle.kts

```kotlin
plugins {
    `java-library`                    // or `java-gradle-plugin`
    id("com.mreil.easy.project")
}
```

`jvm-defaults` picks up the `java` plugin automatically: it applies `withSourcesJar()` /
`withJavadocJar()`, the Java toolchain, JaCoCo, test suites, and root report aggregation.

## easy { } Contributor Usage

All contributors are **enabled by default**; opt out with `enabled.set(false)`.

```kotlin
easy {
    publish {
        toMavenLocal()
        mavenRepo("myReleases", url = "https://repo.example.com/releases", withPasswordCredentials = true)
    }
    semver { }                 // validates group/version as SEMVER; EasySemver.of(project)
    codemeta { filename.set("codemeta.json") }
    release { }                // version bump/commit/tag on release branches
    vcs { }                    // git-backed VcsService for release/commit
    projectDefaults { }        // enforces group/version, applies `base`
    jvmDefaults { }            // configureTestSuites / aggregateReports / jacocoEnabled
}
```

| Contributor | Extension | Purpose |
|---|---|---|
| `publish` | `easy.publish` | Wraps `maven-publish`, default publication, `mavenRepo {}`, snapshot/release routing |
| `jvm-defaults` | `easy.jvmDefaults` | JVM toolchain, sources/javadoc jars, test suites, JaCoCo, report aggregation |
| `project-defaults` | `easy.projectDefaults` | Applies `base`; requires `group`/`version` |
| `semver` | `easy.semver` | SEMVER validation + typed version access |
| `codemeta` | `easy.codemeta` | Generates/maintains `codemeta.json` |
| `vcs` | `easy.vcs` | Git operations for release workflows |
| `release` | `easy.release` | Pre-release checks, version commit/tag, rollback on failure |

## Test Suites

* `src/test/**` → built-in `test` suite: JUnit Jupiter pinned from the catalog, catalog test deps
  added, covered by `jacocoTestReport`.
* `src/<name>Test/**` → registered automatically, wired into `check` (after `test`), JaCoCo, and a
  root `<name>AggregateTestReport` / `<name>CodeCoverageReport`.
* Plugin projects get `gradleTestKit()` + plugin-under-test metadata on functional suites.

Declare the catalog aliases the suite wiring probes in `gradle/libs.versions.toml`:
`junit-jupiter`, `assertj-core` (or `assertj`), `junit-pioneer`, `junit-jupiter-params`,
`mockito-core` (unit only; functional suites deliberately exclude Mockito).

Use assertJ soft assertions.

## Do Not Declare

`jvm-defaults` owns these; adding them is redundant:

* `useJUnitJupiter(...)` and explicit AssertJ/JUnit/Mockito test dependencies
* `jacoco` plugin and coverage report wiring
* `testSourceSets` / `check.dependsOn(functionalTest)`
* `withSourcesJar()` / `withJavadocJar()`

Module build files declare only repo-specific test-helper dependencies and `jvmArgs`.

## Multi-Project

Configure `easy { }` once in the root; subprojects inherit via `ExtensionCopier` and
`@ApplyToSubprojects`. Apply `com.mreil.easy.project` (and the language plugin) in each module.

## AGENTS.md

Instructions for AI agents working on this Gradle project:

1. **Adhere strictly to plugin defaults**: Do not re-declare extensions or tasks owned by `jvm-defaults` or `project-defaults` (see "Do Not Declare").
2. **Preserve Configuration Cache compatibility**: Avoid accessing `project` during task execution or making builds CC-incompatible.
3. **Use version catalogs**: Always reference dependencies and plugins via aliases in `gradle/libs.versions.toml`.
4. **Multi-module structure**: Keep the root project empty and configure modules consistently via the root `easy { }` block and module plugins.
5. **Testing**: Place unit tests in `src/test/kotlin` and functional suites in `src/<name>Test/kotlin`. Use AssertJ soft assertions.

