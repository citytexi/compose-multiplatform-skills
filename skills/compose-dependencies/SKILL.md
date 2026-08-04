---
name: compose-dependencies
description: Use when adding a library to a Compose Multiplatform project, writing or bumping libs.versions.toml, or debugging an unresolved reference, a Gradle version conflict, or a runtime NoSuchMethodError — pins every version from the Compose Multiplatform release notes and picks the right artifact coordinate for common code.
---

# Compose Multiplatform dependencies

The Dependencies section of the Compose Multiplatform release notes is the
single source of truth for a coordinated version set. Jetpack and Android
documentation is the source for how an API is used — never for whether it is
available, or what version to pin.

<HARD-GATE>
- NEVER pin a version read from Android/Jetpack documentation or a Jetpack
  release page. Derive it from the Compose Multiplatform release the project
  is on.
- NEVER write an `androidx.compose.*` coordinate in a `commonMain` dependency
  block for an artifact that has a `compose.*` plugin accessor (runtime, ui,
  foundation, material, material3, animation, the `compose.components.*`
  accessors, and the `compose.desktop.*` accessors) — use the accessor
  instead. An `androidx.compose.*` artifact with no accessor —
  `androidx.compose.runtime:runtime-retain` is the worked example — is not
  covered by this rule; classify it with `coordinates.md`'s Google KMP layer.
  NEVER write an `androidx.navigation.*`, `androidx.lifecycle.*`, or
  `androidx.savedstate.*` coordinate in a `commonMain` dependency block —
  these always have a JetBrains-mirror coordinate instead.
- NEVER invent a version. Every pin traces to a Compose Multiplatform release
  page or the artifact's `maven-metadata.xml`, and carries a comment naming
  the URL checked — `// verify latest` in Kotlin DSL, `#` in a TOML version
  catalog, since TOML has no `//` syntax.
</HARD-GATE>

## Checklist

Create a todo for each item and complete them in order:

1. Determine the project's Compose Multiplatform version — the anchor — and
   its current Kotlin version.
2. Read that release's Dependencies section on the Compose Multiplatform
   release notes page and write down the full coordinated set, not a subset.
3. Classify each library you need — the four Jetpack/AndroidX coordinate
   layers, or the separate third-party procedure for a library that was
   never Jetpack — before writing a single dependency line.
4. Confirm the Kotlin floor for the project's actual targets — native and
   web carry a different floor than JVM-only.
5. Write the version catalog with every pin traceable to the release page or
   `maven-metadata.xml`.
6. Verify with a Gradle sync and the `dependencies` (or `dependencyInsight`)
   task before calling the work done.

## Core Rules

### One anchor, one set
The Compose Multiplatform version a project is pinned to is the anchor;
every other version — the Kotlin floor, Lifecycle, Navigation, Navigation3,
SavedState, Material3, Material3 Adaptive, WindowManager — is read off that
release's Dependencies section, not chosen on its own merits. Bumping one
mirrored library alone recreates, one dependency at a time, exactly the
incompatibility matrix the anchor exists to remove.
→ deriving the set and the Kotlin floors: [references/version-lock.md](references/version-lock.md)

### Classify before you write a coordinate
A library found in Android documentation can reach common code as a Plugin
accessor, a JetBrains mirror, or a Google KMP artifact — or not reach it at
all, if it's Android-only. The four layers look similar enough from the
Android side that guessing is unreliable; run the numbered classification
procedure before typing any coordinate. A library that was never in Android
documentation to begin with — a third-party Kotlin Multiplatform library
such as Koin, Ktor, Coil, Turbine, or Orbit — doesn't fit these four layers
at all; it has its own, simpler check.
→ the four layers, the third-party case, and the decision procedure: [references/coordinates.md](references/coordinates.md)

### Compose artifacts go through plugin accessors, never explicit coordinates
`compose.runtime`, `compose.foundation`, `compose.ui`, `compose.material3`,
`compose.material`, `compose.animation`, and the `compose.components.*` /
`compose.desktop.*` accessors are Gradle properties the Compose Multiplatform
plugin injects — no version is written by hand, the plugin version supplies
it. An explicit `org.jetbrains.compose.*` or `androidx.compose.*` coordinate
for one of these pins a second number that can drift from the plugin's own.
→ the accessor list and its one exception: [references/coordinates.md](references/coordinates.md)

### A broken build is a coordinate problem or a version problem — decide which before changing anything
The wrong Maven coordinate fails at configuration time, often as an
`Unresolved reference` that compiles fine on `androidMain`. A version
collision between two requests for the *right* coordinate compiles and ships,
then fails at runtime with `NoSuchMethodError` or `NoClassDefFoundError`.
Sort the symptom into the right class before reaching for
`resolutionStrategy` or re-classifying a library that was never miscoded.
→ the fork and how to read Gradle resolution: [references/conflicts.md](references/conflicts.md)

### Never force a version the anchor never shipped against
`strictly` and `resolutionStrategy` legitimately hold a transitive mirror at
the anchor's pinned version when some other dependency's graph is dragging
it forward. They do not legitimately force a version that appears in no
anchor release's Dependencies section — that recreates the uncoordinated
state the anchor exists to prevent. If a third-party library genuinely needs
a newer mirrored library, raise the anchor itself instead of forcing one
module ahead of the set it was tested with.
→ when strictly and resolutionStrategy are legitimate: [references/conflicts.md](references/conflicts.md)

### Android documentation proves an API's shape, not its presence
A Jetpack Compose release is usually 1–3 months ahead of the Compose
Multiplatform release built from it, and even a matching base version
doesn't guarantee every mirrored library moved on the same day. Map the
project's anchor to its Jetpack Compose base and check the API's "Added in"
version before assuming an Android-documented call compiles in `commonMain`.
→ mapping your release to its Jetpack base: [references/api-availability.md](references/api-availability.md)

## Red Flags — STOP

| Smell | Fix |
|---|---|
| A version copied from developer.android.com | Read it from the Compose Multiplatform release notes' Dependencies section instead — Jetpack docs are never the version source |
| An `androidx.compose.*` coordinate in `commonMain` for an artifact that has a `compose.*` accessor | Use the matching accessor instead — the plugin resolves it per target. Not every `androidx.compose.*` artifact has one: `runtime-retain` doesn't, and is correctly pinned directly as Google KMP |
| Two catalog entries whose versions came from different Compose Multiplatform releases | Re-read the current anchor's Dependencies section in full and move every mirrored version to match it |
| A pin with no verify-latest comment naming the source checked (`// verify latest` in Kotlin DSL, `#` in TOML) | Every pin traces to a release page or `maven-metadata.xml` — add the comment naming what was checked |
| A pre-release version "cleaned up" to its stable spelling | The anchor was stabilized against that exact pre-release build; the tidier stable version was never tested with the rest of the set |
| `resolutionStrategy` covering a coordinate mistake | Re-classify the library instead — forcing a version onto a wrong coordinate ships the wrong artifact |
| An `Unresolved reference` fixed by downgrading an unrelated library | That's a coordinate problem treated as a version problem — reclassify the failing import instead |
| An API assumed present because the Android reference documents it | Map the anchor to its Jetpack Compose base and check the API's "Added in" version before writing the call |
