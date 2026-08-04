# Version lock — deriving one coordinated set

Every version number in a Compose Multiplatform project is a consequence of
one decision, not six independent ones. Pick the Compose Multiplatform
release first. Every other number — Kotlin, the JetBrains-mirrored
libraries, Material3 Adaptive — is read off that release, never chosen on
its own merits.

## The anchor

**The anchor** is the Compose Multiplatform version a project is pinned to
— for example `1.11.1`. It is the first version decision made, and every
other version decision in the project follows from it, not the other way
around.

This works because a Compose Multiplatform release is cut from a specific
Jetpack Compose release commit and stabilized against a specific set of
mirrored library versions — Lifecycle, Navigation, Navigation3, SavedState,
Material3, Material3 Adaptive, WindowManager — and that exact set is what
JetBrains built and tested together. A newer Lifecycle exists on Maven
Central almost every week; it was not tested against this anchor. Choosing
library versions independently of the anchor recreates, one dependency at a
time, exactly the compatibility matrix the anchor exists to remove.

## Reading a release

Read on 2026-08-04, from
<https://github.com/JetBrains/compose-multiplatform/releases/tag/v1.11.1>
(published 2026-06-02):

| Library group | Coordinates | Version | Based on Jetpack |
|---|---|---|---|
| Gradle plugin | `org.jetbrains.compose` | `1.11.1` | — |
| Runtime | `org.jetbrains.compose.runtime:runtime*` | `1.11.1` | Runtime 1.11.2 |
| UI | `org.jetbrains.compose.ui:ui*` | `1.11.1` | UI 1.11.2 |
| Foundation | `org.jetbrains.compose.foundation:foundation*` | `1.11.1` | Foundation 1.11.2 |
| Material | `org.jetbrains.compose.material:material*` | `1.11.1` | Material 1.11.2 |
| Material3 | `org.jetbrains.compose.material3:material3*` | `1.11.0-alpha07` | Material3 1.5.0-alpha17 |
| Material3 Adaptive | `org.jetbrains.compose.material3.adaptive:adaptive*` | `1.3.0-alpha07` | Material3 Adaptive 1.3.0-alpha10 |
| Lifecycle | `org.jetbrains.androidx.lifecycle:lifecycle-*` | `2.11.0-beta01` | Lifecycle 2.11.0-beta01 |
| Navigation | `org.jetbrains.androidx.navigation:navigation-*` | `2.9.2` | Navigation 2.9.7 |
| Navigation3 | `org.jetbrains.androidx.navigation3:navigation3-*` | `1.1.1` | Navigation3 1.1.1 |
| Navigation Event | `org.jetbrains.androidx.navigationevent:navigationevent-compose` | `1.1.0` | Navigation Event 1.1.1 |
| SavedState | `org.jetbrains.androidx.savedstate:savedstate*` | `1.4.0` | Savedstate 1.4.0 |
| WindowManager Core | `org.jetbrains.androidx.window:window-core` | `1.5.1` | WindowManager 1.5.1 |

This is the complete Dependencies section for 1.11.1 — every row the release
page lists, not a subset picked for a code sample.

Two rules follow directly from this table:

- **Pre-release versions in the set are not a mistake.** Lifecycle is
  pinned to `2.11.0-beta01` and Material3 to `1.11.0-alpha07` because those
  are the exact builds 1.11.1 was stabilized against. A stable `2.11.0` now
  exists on Maven Central (confirmed 2026-08-04 — see
  [`## Verifying a pin`](#verifying-a-pin) below), and a newer
  `1.3.0-beta02` of Material3 Adaptive exists too. Replacing the pin with
  either one because it looks tidier is a way to introduce a conflict, not
  remove one — the anchor was never tested against them.
- **The set moves together.** A single mirrored library bumped on its own
  leaves the anchor: the project is now running a combination JetBrains
  never built, and the failure mode is whatever incompatibility existed
  between the two versions, discovered at build or runtime instead of in a
  release note.

A separate, faster-moving channel exists for testing an unreleased anchor
before it is tagged: JetBrains publishes dev builds to
`https://redirector.kotlinlang.org/maven/compose-dev`, with version strings
like `1.12.10-alpha01+dev4593` (fetched from that repository's own
`maven-metadata.xml` on 2026-08-04 — the exact suffix changes on every
build). A dev build is not an anchor a project should ship on; treat it as a
preview of what a future `## Reading a release` table will eventually say,
never as a substitute for reading the tagged release page once it exists.

## Kotlin floors

From
<https://kotlinlang.org/docs/multiplatform/compose-compatibility-and-versioning.html>
(read 2026-08-04):

> The latest Compose Multiplatform is always compatible with the latest
> version of Kotlin. There is no need to manually align their versions.

> Compose Multiplatform requires the Compose Compiler Gradle plugin applied
> with the same version as the Kotlin Multiplatform plugin.

The Compose Compiler Gradle plugin id is `org.jetbrains.kotlin.plugin.compose`
(same page area, cross-referenced from the migration guide). It ships out of
the Kotlin repository itself, one release per Kotlin version, which is why
its version must equal the Kotlin Multiplatform plugin's version — they are
two plugin ids pointing at the same numbered release, not two independent
version choices.

Same page, on the floor introduced by the 1.8.0 K2 transition:

> Starting with Compose Multiplatform 1.8.0, the UI framework fully
> transitioned to the K2 compiler. To use the latest Compose Multiplatform
> release: use at least Kotlin 2.1.0 for your projects, depend on libraries
> based on Compose Multiplatform only if they are compiled against at least
> Kotlin 2.1.0, upgrade to Kotlin 2.2.20 for projects targeting platforms
> with rapidly evolving support, such as iOS and web.

A second, later floor is stated on the "What's new in Compose Multiplatform
1.11.1" page
(<https://kotlinlang.org/docs/multiplatform/whats-new-compose-111.html>,
read 2026-08-04), under "Minimum Kotlin version increased":

> If your project includes native or web targets, the latest features
> require an upgrade to Kotlin 2.3.10.

| Scope | Kotlin floor | Source, read 2026-08-04 |
|---|---|---|
| Any project, on Compose Multiplatform ≥ 1.8.0 | at least 2.1.0 | compose-compatibility-and-versioning.html |
| iOS and web ("rapidly evolving support") | upgrade to 2.2.20 | compose-compatibility-and-versioning.html |
| Native or web targets, to use 1.11.1's latest features | upgrade to 2.3.10 | whats-new-compose-111.html |

These two pages genuinely disagree on the floor for the same platform
category (web, and native as the broader set iOS sits in): the
compatibility page's `2.2.20` and the 1.11.1 what's-new page's `2.3.10` are
not the same number. Neither page defers to the other, so both are recorded
here rather than one being silently dropped. A project on the 1.11.1 anchor
targeting native or web should treat `2.3.10` as the effective floor, since
it is the number the release the anchor is actually pinned to states — but
that is this file picking the higher, not the two pages agreeing.

A design-time note for this skill claimed a further floor — Kotlin `2.3.20`
required specifically for Kotlin/JS and Kotlin/Wasm. That number is real
(Maven Central lists a released `2.3.20` as of 2026-08-04), but it does not
appear on either page checked above: the compatibility page never mentions
it, and the only Kotlin-version sentence on the 1.11.1 what's-new page is
the `2.3.10` one quoted above, with no separate JS/Wasm figure and no
mention of `2.3.20` anywhere in that page's source. Treat `2.3.20` as
unverified against these two pages and do not write it into a catalog on
their authority.

## The catalog

```toml
# gradle/libs.versions.toml

[versions]
compose           = "1.11.1"        # verify latest: https://github.com/JetBrains/compose-multiplatform/releases
kotlin            = "2.3.10"        # verify latest: https://kotlinlang.org/docs/releases.html — this file's conservative choice for native/web targets, not a floor either page above mandates: the compatibility page's unconditional floor is 2.1.0 (2.2.20 only for iOS/web), and 2.3.10 is the what's-new-1.11.1 page's figure for "latest features," not a general requirement
lifecycle         = "2.11.0-beta01" # verify latest: https://github.com/JetBrains/compose-multiplatform/releases/tag/v1.11.1
navigation3       = "1.1.1"         # verify latest: https://github.com/JetBrains/compose-multiplatform/releases/tag/v1.11.1
navigationEvent   = "1.1.0"         # verify latest: https://github.com/JetBrains/compose-multiplatform/releases/tag/v1.11.1
savedstate        = "1.4.0"         # verify latest: https://github.com/JetBrains/compose-multiplatform/releases/tag/v1.11.1
material3Adaptive = "1.3.0-alpha07" # verify latest: https://github.com/JetBrains/compose-multiplatform/releases/tag/v1.11.1

[plugins]
kotlin-multiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
kotlin-compose        = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" } # must equal "kotlin", not "compose"
compose-multiplatform = { id = "org.jetbrains.compose", version.ref = "compose" }

[libraries]
lifecycle-viewmodel-compose = { module = "org.jetbrains.androidx.lifecycle:lifecycle-viewmodel-compose", version.ref = "lifecycle" }
navigation3-ui              = { module = "org.jetbrains.androidx.navigation3:navigation3-ui", version.ref = "navigation3" }
navigationevent-compose     = { module = "org.jetbrains.androidx.navigationevent:navigationevent-compose", version.ref = "navigationEvent" }
savedstate                  = { module = "org.jetbrains.androidx.savedstate:savedstate", version.ref = "savedstate" }
material3-adaptive-layout   = { module = "org.jetbrains.compose.material3.adaptive:adaptive-layout", version.ref = "material3Adaptive" }
```

```kotlin
// build.gradle.kts (module)

plugins {
    alias(libs.plugins.kotlin.multiplatform)
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.compose.multiplatform)
}

kotlin {
    sourceSets {
        commonMain.dependencies {
            // Plugin accessors — no version, the compose.* plugin supplies it.
            implementation(compose.runtime)
            implementation(compose.foundation)
            implementation(compose.material3)
            implementation(compose.components.resources)

            // Explicit coordinates — version comes from the anchor's release page, not
            // from each library's own "latest" tag.
            implementation(libs.lifecycle.viewmodel.compose)
            implementation(libs.navigation3.ui)
            implementation(libs.navigationevent.compose)
            implementation(libs.savedstate)
            implementation(libs.material3.adaptive.layout)
        }
    }
}
```

Material3 Adaptive is neither a plugin accessor nor an
`org.jetbrains.androidx.*` mirror — it publishes its own standalone artifacts
under `org.jetbrains.compose.material3.adaptive` (confirmed 2026-08-04,
<https://kotlinlang.org/docs/multiplatform/compose-multiplatform-jetpack-libraries.html>).
Run it through [coordinates.md](coordinates.md)'s classification procedure
if a library's layer is ever unclear; the rule that its version comes from
the anchor's release page, not its own, is unchanged either way.

## Verifying a pin

The Maven coordinate's group becomes a directory path: dots turn into
slashes, then the artifact id, then the version, then the filename. For
`org.jetbrains.androidx.lifecycle:lifecycle-viewmodel` version
`2.11.0-beta01` that path is
`org/jetbrains/androidx/lifecycle/lifecycle-viewmodel/2.11.0-beta01/`.
Dropping the version segment and requesting `maven-metadata.xml` at the
artifact directory lists every version Maven Central has ever seen:

```bash
curl -s https://repo1.maven.org/maven2/org/jetbrains/androidx/lifecycle/lifecycle-viewmodel/maven-metadata.xml \
  | grep -oE '<version>[^<]+</version>' | tail -20
```

Run against `repo1.maven.org` on 2026-08-04 for this file's own
verification, this returned real data (`HTTP 200`), tailing off with:

```
<version>2.10.0</version>
<version>2.11.0-alpha01</version>
<version>2.11.0-alpha02</version>
<version>2.11.0-alpha03</version>
<version>2.11.0-beta01</version>
<version>2.11.0-beta02</version>
<version>2.11.0-rc01</version>
<version>2.11.0</version>
```

That confirms `2.11.0-beta01` — the version this file's catalog pins for
`lifecycle` — really exists, and that a stable `2.11.0` also exists and is
still the wrong choice per the pre-release rule above. `repo1.maven.org` was
unreachable from the sandbox `coordinates.md` was verified in; it answered
directly from this one. Network reachability to Maven Central varies by
environment — if this command fails, [coordinates.md](coordinates.md)
documents the `dl.google.com` fallback for `androidx.*` coordinates, and the
same substitution (group path, artifact id, version) applies there too.

`maven-metadata.xml` only proves a version was published. It does not prove
that version belongs to the anchor's coordinated set — only the release
page's Dependencies table proves that.

## Bumping

Change the anchor first — pick the new Compose Multiplatform version. Then
re-read that release's Dependencies section in full, the same way
[`## Reading a release`](#reading-a-release) did for 1.11.1, and move every
mirrored version in the catalog to match what the new table says, including
ones that did not change. Re-check [`## Kotlin floors`](#kotlin-floors)
against the new release's own compatibility notes, since a new anchor can
raise the floor. Only then sync the catalog and rebuild.

Never bump one mirrored library alone — that recreates the exact
uncoordinated state the anchor exists to prevent. If a third-party
dependency forces a newer version of a mirrored library than the anchor
pins, that is not a bump — it's a conflict, and pinning the forced version
back down is the subject of the next file, not this one.
→ diagnosing and pinning back a forced upgrade: [conflicts.md](conflicts.md)
