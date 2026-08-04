# Coordinates — what a library is called in Compose Multiplatform

Compose Multiplatform republishes Jetpack. This file's four layers classify
coordinates whose origin is Jetpack/AndroidX — the same API can reach common
code under three different coordinate shapes, and a fourth class of library
cannot reach it at all. Writing the coordinate found in Android documentation
without running the classification below is wrong often enough that it must
be classified rather than guessed. A third-party Kotlin Multiplatform library
(Koin, Ktor, Coil, Turbine, Orbit, kotlinx-serialization, …) was never
Jetpack to begin with and does not fit these four layers at all — see
[`## A third-party library is a different question`](#a-third-party-library-is-a-different-question)
below.

## The four layers

### Plugin accessor

`compose.*` accessors are Gradle **accessor properties** injected by the
Compose Multiplatform Gradle plugin — not Maven coordinates you write
yourself. Below is what `ComposePlugin.kt` declares, read from the `v1.11.1`
tag (verified 2026-08-04,
<https://github.com/JetBrains/compose-multiplatform/blob/v1.11.1/gradle-plugins/compose/src/main/kotlin/org/jetbrains/compose/ComposePlugin.kt>,
`ComposePlugin.Dependencies` and its nested `DesktopDependencies`,
`CommonComponentsDependencies`, `DesktopComponentsDependencies`, and
`HtmlDependencies` objects). `master`, checked the same day, declares the
identical accessor set — the only diff is an added deprecation-message
annotation on `desktop.currentOs`, not a new or removed accessor. Grouped for
readability rather than as one run-on list:

- **Core UI:** `compose.runtime`, `compose.runtimeSaveable`,
  `compose.foundation`, `compose.ui`, `compose.material`,
  `compose.material3`, `compose.material3AdaptiveNavigationSuite`,
  `compose.animation` (and `compose.animationGraphics`), and
  `compose.materialIconsExtended` (a special case: it resolves to a coordinate
  frozen at `1.7.3`, not to `composeVersion`, per the source's own
  deprecation note).
- **Test/tooling:** `compose.uiTest` (`@ExperimentalComposeLibrary`),
  `compose.uiTooling`, `compose.uiUtil`, and `compose.preview` (resolves to
  `ui-tooling-preview` — a top-level accessor, distinct from
  `compose.components.uiToolingPreview` below, which resolves to the
  separately-published `components-ui-tooling-preview` artifact).
- **`compose.components.*`:** `resources`, `uiToolingPreview`.
- **`compose.desktop.*`:** `common`, `linux_x64`, `linux_arm64`,
  `windows_x64`, `windows_arm64`, `macos_x64`, `macos_arm64`, `currentOs`,
  `uiTestJUnit4`, plus the nested `compose.desktop.components.*`: `splitPane`
  and `animatedImage` (both `@ExperimentalComposeLibrary` — not
  `compose.components.*`, which is a separate, top-level accessor group).
- **`compose.html.*`:** `core`, `svg`, `testUtils`.

The source also declares `compose.web.*` (`core`, `svg`, `testUtils`), but
that accessor carries `DeprecationLevel.ERROR` in favor of `compose.html` —
code referencing it does not compile, so treat it as removed, not as a sixth
usable group.

This is what the source declared at that tag on that date, not a claim
expected to stay true forever. Re-derive it directly rather than trusting
this list to age well: fetch `ComposePlugin.kt` at whatever tag anchors this
skill and read the `Dependencies` class plus its nested
`*ComponentsDependencies` / `*Dependencies` objects.

```bash
curl -s "https://raw.githubusercontent.com/JetBrains/compose-multiplatform/v1.11.1/gradle-plugins/compose/src/main/kotlin/org/jetbrains/compose/ComposePlugin.kt"
```

```kotlin
kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation(compose.runtime)
            implementation(compose.foundation)
            implementation(compose.ui)
            implementation(compose.material3)
            implementation(compose.components.resources)
            implementation(compose.components.uiToolingPreview)
        }
    }
}
```

Adding `compose.material3` resolves to `androidx.compose.material3:material3`
on the Android target and `org.jetbrains.compose.material3:material3` on
every other target, automatically, based on Gradle Module Metadata in the
multiplatform artifact (verified 2026-08-04,
<https://kotlinlang.org/docs/multiplatform/compose-multiplatform-and-jetpack-compose.html>).
The same redirect applies to `compose.foundation`, `compose.ui`, and
`compose.material` — JetBrains builds and publishes multiplatform versions
of these three (plus material3) under the `org.jetbrains.compose.*` group
(verified 2026-08-04,
<https://kotlinlang.org/docs/multiplatform/compose-multiplatform-jetpack-libraries.html>).

`compose.runtime` is the one exception to that redirect story, per the same
packaging page: "Among the base Compose libraries, the fundamental
`androidx.compose.runtime` is fully multiplatform. (The previously used
`org.jetbrains.compose.runtime` artifact now serves as an alias.)" Google
itself — not JetBrains — publishes `androidx.compose.runtime` with
Kotlin/Native, JS, and Wasm variants; JetBrains never built a separate
multiplatform runtime the way it did for ui/foundation/material/material3, it
just kept the old `org.jetbrains.compose.runtime` coordinate resolving to
Google's own artifact so existing builds keep working. The `compose.runtime`
accessor is still the right way to depend on it — the point is only that,
for this one artifact, "the JetBrains coordinate" and "the AndroidX
coordinate" name the same thing rather than two builds kept in lockstep.

**No version is written by hand for a `compose.*` accessor — the Compose
Multiplatform Gradle plugin version supplies each accessor's version from the
anchor release's Dependencies set, which is not always the plugin's own
version number.** Material3 is the named exception: on the Compose
Multiplatform `1.11.1` plugin, `compose.material3` resolves to
`1.11.0-alpha07`, not `1.11.1` — [version-lock.md](version-lock.md)'s own
1.11.1 anchor table lists that row, and the packaging page explains why (next
paragraph). An explicit `org.jetbrains.compose.*` or `androidx.compose.*`
coordinate for an accessor-covered artifact is still a smell even so: it pins
a second number by hand for something the plugin already resolves per
target, and that second number can drift from whatever the anchor's
Dependencies set says.

One legitimate exception, from the packaging page (verified 2026-08-04, same
URL as above):

> Unlike the others, the Material 3 library is not coupled with Compose
> Multiplatform versions. So instead of the `material3` alias, you can
> provide a direct dependency. For example, you might use an EAP version.

Depending directly on `org.jetbrains.compose.material3:material3` to take an
EAP Material3 build ahead of the one the plugin bundles is the one sanctioned
escape hatch — and it is still an escape hatch: it breaks the single-anchor
rule ([version-lock.md](version-lock.md)), so treat it as deliberate, not as
evidence that other Compose artifacts may be pinned the same way.

Not every `androidx.compose.*` coordinate has an accessor, though. Worked
counterexample: `androidx.compose.runtime:runtime-retain` (the `retain { }`
API documented in the compose-navigation skill's `results-scoping.md`) has no
`compose.*` accessor at all — it is absent from the plugin source list cited
above — and JetBrains does not publish it under `org.jetbrains.compose.runtime`
either. It is written as an explicit `androidx.compose.runtime:runtime-retain`
coordinate in `commonMain`, pinned to its own version from its own
`maven-metadata.xml`, because it belongs to the **Google KMP** layer below,
not this one — its Gradle Module Metadata publishes real Android, desktop,
iOS, and other Kotlin/Native targets (verified 2026-08-04 via the same
`.module`-fetch procedure `## Google KMP` below uses). Treating every
`androidx.compose.*` coordinate as "must be an accessor, delete it" would
flag this correct pin as the exact smell this section warns about.

### JetBrains mirror

Group `org.jetbrains.androidx.*`. Confirmed present in the Compose
Multiplatform 1.11.1 release notes (verified 2026-08-04,
<https://github.com/JetBrains/compose-multiplatform/releases/tag/v1.11.1>):
Lifecycle (`org.jetbrains.androidx.lifecycle`), Navigation
(`org.jetbrains.androidx.navigation`), Navigation 3
(`org.jetbrains.androidx.navigation3`, plus the companion
`org.jetbrains.androidx.navigationevent`), SavedState
(`org.jetbrains.androidx.savedstate`), and WindowManager Core
(`org.jetbrains.androidx.window`). These are declared explicitly in the
version catalog — the version comes from the Compose Multiplatform release
notes' Dependencies section, not from Maven Central's "latest" tag, since
JetBrains ships its mirror on its own release cadence tied to a CMP version.

The redirect, from the packaging page (verified 2026-08-04,
<https://kotlinlang.org/docs/multiplatform/compose-multiplatform-jetpack-libraries.html>):
common code depends on one unifying coordinate —
`org.jetbrains.androidx.navigation:navigation-compose` — and Gradle Module
Metadata resolves it to per-target artifacts:

> With this approach, the Android app produced by a Kotlin Multiplatform
> project with that dependency uses the original Android Navigation library.
> The iOS app, on the other hand, uses the corresponding iOS library built by
> JetBrains.

So on the Android target the unifying coordinate redirects back to Google's
own `androidx.navigation:navigation-compose`, while iOS/desktop/web targets
resolve to JetBrains-built artifacts such as
`org.jetbrains.androidx.navigation:navigation-compose-uikitarm64` and
`org.jetbrains.androidx.navigation:navigation-runtime-iossimulatorarm64`.
Both publishers' classes share identical package names
(`androidx.navigation.*`) — only the Gradle coordinate differs. Writing
`androidx.navigation:navigation-compose` directly in `commonMain` still fails
— Google's coordinate publishes no Kotlin/Native variant, so Gradle cannot
resolve a variant for the iOS source set and the build fails at
configuration/resolution time, not at runtime — but the identical package
names are exactly what make the mistake look plausible enough to reach for
in the first place.

```toml
[versions]
lifecycle  = "2.11.0-beta01" # verify latest: https://github.com/JetBrains/compose-multiplatform/releases/tag/v1.11.1
navigation3 = "1.1.1"        # verify latest: https://github.com/JetBrains/compose-multiplatform/releases/tag/v1.11.1
savedstate = "1.4.0"         # verify latest: https://github.com/JetBrains/compose-multiplatform/releases/tag/v1.11.1

[libraries]
lifecycle-viewmodel-compose = { module = "org.jetbrains.androidx.lifecycle:lifecycle-viewmodel-compose", version.ref = "lifecycle" }
navigation3-ui = { module = "org.jetbrains.androidx.navigation3:navigation3-ui", version.ref = "navigation3" }
savedstate = { module = "org.jetbrains.androidx.savedstate:savedstate", version.ref = "savedstate" }
```

All three pins above were read from the Compose Multiplatform 1.11.1 release
notes on 2026-08-04; if the live release page now differs, the live page
wins.

### Google KMP

Google's own artifacts that already publish Kotlin/Native and other non-JVM
variants, kept on their `androidx.*` coordinates — no JetBrains mirror
exists or is needed. **Here `androidx.*` in `commonMain` is correct**, which
is why a blanket "never write `androidx` in commonMain" rule is wrong and the
classification procedure below exists instead of a static list.

Confirmed by fetching each artifact's Gradle Module Metadata from
`dl.google.com` and checking for `org.jetbrains.kotlin.native.target` entries
— examples verified on 2026-08-04, not the list:

| Coordinate | Version checked | Non-JVM targets found |
|---|---|---|
| `androidx.room:room-runtime` | `2.8.4` | iOS (arm64, simulator arm64, x64), macOS, Linux, tvOS, watchOS |
| `androidx.datastore:datastore-preferences` | `1.2.1` | iOS, macOS, Linux, tvOS, watchOS |
| `androidx.paging:paging-common` | `3.5.0` | iOS (arm64, simulator arm64), macOS, Linux, mingwX64, tvOS, watchOS |
| `androidx.sqlite:sqlite` | `2.7.0` | iOS (arm64, simulator arm64), macOS, Linux, tvOS, watchOS |
| `androidx.collection:collection` | `1.6.0` | iOS, macOS, Linux, mingwX64, tvOS, watchOS |
| `androidx.annotation:annotation` | `1.10.0` | Android Native, iOS (arm64, simulator arm64), macOS, Linux, mingwX64, tvOS, watchOS |

Each row's version was the artifact's current stable release on 2026-08-04
(<https://dl.google.com/android/maven2/androidx/room/room-runtime/maven-metadata.xml>
and the equivalent `maven-metadata.xml` path for each other artifact); the
target list is what that exact `.module` file reported, run through the
command in the next section. // verify latest for each: substitute the
artifact path into the same `maven-metadata.xml` URL pattern.

### Android-only

No common-code path exists. The library goes in `androidMain`, behind an
`expect`/`actual` declaration or a DI-injected interface defined in
`commonMain`:

```kotlin
// commonMain
interface WorkScheduler { fun schedule(taskId: String) }

// androidMain
class AndroidWorkScheduler(private val context: Context) : WorkScheduler {
    override fun schedule(taskId: String) { /* androidx.work:work-runtime here */ }
}
```

`androidx.work:work-runtime` (checked at `2.11.2`, verified 2026-08-04 via
the same `.module`-fetch procedure — zero `org.jetbrains.kotlin.native.target`
lines, only `release`-variant publications) is a confirmed example: no
Kotlin/Native, JS, or Wasm variant exists, so `commonMain` cannot reference
`androidx.work.WorkManager` at all. For the DI-injected-interface side of this
boundary — binding `WorkScheduler`'s platform implementation without an
`expect`/`actual` — see the compose-di skill.

These four layers are all about libraries whose Jetpack/AndroidX origin is
the reason classification is needed at all. A library that was never Jetpack
— the next section — doesn't fit any of them.

## A third-party library is a different question

Koin, Ktor, Coil, Turbine, Orbit, and kotlinx-serialization are not Jetpack
libraries, so the four layers above — which exist to sort out *how a Jetpack
API reaches common code* — don't apply to them. The question for a
third-party coordinate is simpler and answered the same way the **Google
KMP** layer above answers it for `androidx.*`: fetch the artifact's own
Gradle Module Metadata from Maven Central and check whether it declares
Kotlin/Native (or other non-JVM) targets.

```bash
curl -s "https://repo1.maven.org/maven2/io/insert-koin/koin-core/4.2.2/koin-core-4.2.2.module" \
  | grep -o '"org.jetbrains.kotlin.native.target": *"[^"]*"' | sort -u
```

Run against `io.insert-koin:koin-core:4.2.2` on 2026-08-04, this returned 16
distinct Kotlin/Native targets (iOS, macOS, Linux, mingwX64, tvOS, watchOS —
arm and simulator/device variants of each). Koin is declared directly in
`commonMain`, at its own version, traced to its own
`maven-metadata.xml` — there is no Compose Multiplatform anchor release that
covers a third-party library's version, because JetBrains never mirrors or
republishes it. The same check, run the same way, is why this repository's
other skills tell a reader to depend on Ktor, Coil, Turbine, Orbit, and
kotlinx-serialization directly in `commonMain`.

This is not a fifth coordinate layer — it's a documented case that falls
**outside** the four layers above, because those four layers only exist to
sort out Jetpack's multiple publishing paths, and a third-party library was
never published through any of them. Answering "does this library reach
`commonMain` at all" this way is also a different question from "which
version of this library's own plugins/companions is compatible with which"
— Koin's `koin-annotations` / `koin-ksp-compiler` pairing is a real example
of that second question, and it is the compose-di skill's question to
answer, not this file's: this file only says whether the artifact's own
Gradle Module Metadata reaches non-JVM targets, never what a third-party
library's own internal compatibility contract requires.

## Classifying a library

A numbered decision procedure, for a coordinate found in Jetpack/AndroidX
documentation. Each step is independently verifiable — run it, don't guess
it. (If the library was never in Jetpack/AndroidX documentation to begin
with — a third-party Kotlin Multiplatform library — use
[`## A third-party library is a different question`](#a-third-party-library-is-a-different-question)
above instead of this procedure.)

1. Does the artifact have a `compose.*` plugin accessor? The accessors are
   `compose.runtime`, `compose.runtimeSaveable`, `compose.foundation`,
   `compose.ui`, `compose.uiTest`, `compose.uiTooling`, `compose.uiUtil`,
   `compose.preview`, `compose.material`, `compose.material3`,
   `compose.material3AdaptiveNavigationSuite`, `compose.materialIconsExtended`,
   `compose.animation` (and `compose.animationGraphics`), the
   `compose.components.*` accessors, the `compose.desktop.*` accessors
   (including nested `compose.desktop.components.*`), and the
   `compose.html.*` accessors — read from the plugin's own source at the tag
   cited in `### Plugin accessor` above; re-derive from `ComposePlugin.kt`
   rather than assuming this list stays current. → **Plugin accessor**. Stop.
   An `androidx.compose.*` coordinate with **no** matching accessor —
   `androidx.compose.runtime:runtime-retain` is the worked counterexample
   above — is not this layer; continue to step 2.
2. Does the current Compose Multiplatform release notes' Dependencies
   section name it (<https://github.com/JetBrains/compose-multiplatform/releases>)?
   → **JetBrains mirror**; the release notes also give the version to pin.
   Stop.
3. Search `org.jetbrains.androidx` on Maven Central
   (<https://central.sonatype.com/search?q=g:org.jetbrains.androidx>) for the
   artifact name. Found? → **JetBrains mirror**. Stop.
4. Is the coordinate's group `androidx.*`? Fetch the Google artifact's
   Gradle Module Metadata and look for non-JVM variants. `repo1.maven.org`
   refuses plain `.module` fetches from some networks (it did in this
   skill's own verification run — a 404/403 with no body); Google's own
   artifact host does not, and it is the authoritative publisher for
   `androidx.*` anyway:
   ```bash
   curl -s "https://dl.google.com/android/maven2/androidx/room/room-runtime/2.8.4/room-runtime-2.8.4.module" \
     | grep -o '"org.jetbrains.kotlin.native.target": *"[^"]*"' | sort -u
   ```
   Substitute the artifact's own group path, artifact id, and exact version
   in both the URL directory and the filename. Any output → **Google KMP**.
   Empty output → continue.
5. Still `androidx.*` with no non-JVM variants found → **Android-only**.
   A coordinate whose group is **not** `androidx.*` should never reach this
   step — a non-`androidx.*` coordinate was never Jetpack, so steps 1–4 above
   will already have fallen through without matching, and the answer is
   [`## A third-party library is a different question`](#a-third-party-library-is-a-different-question),
   not "Android-only." Concluding "Android-only" for a third-party group is
   the exact false negative this file exists to prevent — Koin, Ktor, Coil,
   Turbine, and Orbit all fail steps 1–4 (they're in no Compose Multiplatform
   release notes and dl.google.com doesn't publish them) but are genuinely
   multiplatform libraries the third-party procedure confirms directly.

Steps 2–4 are the load-bearing ones. The example lists in this file are
**examples verified on 2026-08-04, not the list** — a library added to any
layer after that date, or a library this file never mentions, will only be
placed correctly by running steps 1–5, not by pattern-matching against the
tables above.

## Red flags

| Smell | Why it's wrong |
|---|---|
| An `androidx.compose.*` coordinate in `commonMain` for an artifact that has a `compose.*` accessor | Bypasses the plugin's version anchor for that accessor — use the accessor instead. Not every `androidx.compose.*` artifact has one: `runtime-retain` doesn't, and belongs in `commonMain` via the Google KMP layer instead |
| An `org.jetbrains.androidx` coordinate pinned to a version not in the current release notes | The mirror's version cadence is tied to a Compose Multiplatform release, not to Maven Central's "latest" |
| A library placed in `commonMain` because "the docs said it's multiplatform," with no `.module` check run | "Multiplatform" in prose can mean two Android flavors, not iOS/desktop/web — the classification procedure exists because the claim alone doesn't say which targets |
| An `androidx.*` coordinate deleted from `commonMain` on the assumption it must be wrong | It may be **Google KMP** — the fourth layer exists precisely so `androidx.*` isn't reflexively treated as an Android-only smell |
| A non-`androidx.*` third-party coordinate (Koin, Ktor, Coil, Turbine, Orbit, …) run through steps 1–5 and concluded "Android-only" because it matched no accessor, release note, or `dl.google.com` fetch | Steps 1–5 classify Jetpack-origin coordinates only; a third-party group falling through all of them is a sign to use `## A third-party library is a different question`, not evidence the library is Android-only |
