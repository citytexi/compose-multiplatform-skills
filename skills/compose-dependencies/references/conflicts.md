# Conflicts — telling a coordinate problem from a version problem

A broken Compose Multiplatform build is broken in one of two unrelated ways.
Either the dependency graph is asking Gradle to resolve the wrong Maven
coordinate for a library — a `coordinates.md` problem — or the coordinate is
right and two different versions of it are colliding — a version problem,
the subject of this file. Reaching for `resolutionStrategy` on a coordinate
problem produces a build that resolves the wrong artifact; re-classifying a
library that has a genuine version skew wastes a cycle without fixing
anything. Sort the symptom first.

## Coordinate problem or version problem?

| Symptom | Class | Next |
|---|---|---|
| `Unresolved reference` on an import in `commonMain`, same import compiles in `androidMain` | Coordinate | → re-classify the library: [coordinates.md](coordinates.md) |
| `Could not find <group>:<artifact>:<version>` | Version — or a coordinate typo | check `maven-metadata.xml` first, the same way [version-lock.md](version-lock.md) verifies a pin |
| Gradle names two versions of one module in the same resolved graph | Version | see `## Reading resolution`, below |
| Compiles, fails at runtime with `NoSuchMethodError` / `NoClassDefFoundError` | Version skew | see `## Reading resolution`, below |
| The API does not exist in any version you can find, for any layer | Neither | → does the API exist at all: [api-availability.md](api-availability.md) |

## Reading resolution

Two Gradle built-in tasks answer "what did Gradle actually resolve, and
why," both confirmed against the current Gradle User Manual (read
2026-08-04, <https://docs.gradle.org/current/userguide/viewing_debugging_dependencies.html>):

```bash
./gradlew :composeApp:dependencies --configuration androidDebugRuntimeClasspath
./gradlew :composeApp:dependencyInsight \
  --configuration androidDebugRuntimeClasspath \
  --dependency org.jetbrains.androidx.lifecycle:lifecycle-viewmodel
```

Gradle's own promotion marker is a requested version rewritten as
`requested -> resolved`, for example `commons-codec:commons-codec:1.6 -> 1.7`
in the manual's own `dependencyInsight` sample output (the `dependencies`
tree sample earlier on the same page, for the same project, shows the same
module as a plain `commons-codec:commons-codec:1.6` line with no arrow — the
arrow is specific to `dependencyInsight`'s report, not the `dependencies`
tree) — that arrow is Gradle telling you conflict resolution overrode what
was asked for. In `dependencyInsight`'s output, read the requester chain: the
"Selection reasons" section states *why*, most usefully `By conflict
resolution : between versions <a> and <b>`, and the tree below it names every
dependency that requested each competing version — that tree is how you find
the third-party library actually forcing the promotion, rather than guessing.

A Kotlin Multiplatform module resolves many configurations, not one —
running `dependencies`/`dependencyInsight` against only the Android
configuration hides a conflict that only exists on another target. The JVM
target's equivalent is `jvmRuntimeClasspath`. iOS and the other Kotlin/Native
targets do not use `*RuntimeClasspath` configurations at all — Kotlin/Native
dependency resolution runs through differently-named, per-target
configurations (for example `iosSimulatorArm64CompileKlibraries`) whose exact
name shifts with the Kotlin Gradle plugin version, so don't guess one. Run
`./gradlew :composeApp:dependencies` with no `--configuration` flag first;
Gradle prints every resolvable configuration name that exists in your
project's own plugin version, and you pick the real one for the target you're
debugging from that list.

## Why highest-wins hurts here

Gradle's documented default, when the graph requests two different versions
of the same module, is numeric: "By default, it will select the highest one
out of these versions" (verified 2026-08-04,
<https://docs.gradle.org/current/userguide/dependency_constraints_conflicts.html>).
That sentence is the entire claim the citation supports — it says nothing
about Compose Multiplatform, mirrors, or runtime behavior. The rest of this
section is this skill's own reasoning about what that default does to a
`org.jetbrains.androidx.*` mirror (`coordinates.md`).

Say a third-party library's own graph requests
`org.jetbrains.androidx.lifecycle:lifecycle-viewmodel:2.11.0`, while the
project's catalog pins `lifecycle = 2.11.0-beta01` for the Compose
Multiplatform 1.11.1 anchor ([version-lock.md](version-lock.md)'s catalog).
Gradle's version comparator ranks `2.11.0` above `2.11.0-beta01` and silently
promotes every consumer of that module to `2.11.0`, project-wide. Nothing in
that step is invalid Gradle behavior: `2.11.0` is a real, published version
— version-lock.md's own `maven-metadata.xml` check already confirmed it
exists — so **resolution succeeds and compilation succeeds**. The Compose
Multiplatform 1.11.1 artifacts, though, were built and stabilized against
`2.11.0-beta01` specifically, per the anchor's own release-notes Dependencies
section; if the promoted `2.11.0` changed or removed something those
compiled artifacts call, the mismatch has nothing to trip over until that
exact call executes — `NoSuchMethodError` or `NoClassDefFoundError`, at
**runtime**, not at build or configuration time. That timing distinction
matters: it is the opposite failure mode from a wrong coordinate (a Gradle
variant-resolution failure at configuration time, because no matching
artifact exists for a target at all — see `coordinates.md`). A highest-wins
promotion is not a resolution failure; the build finishes and ships, and only
breaks once the mismatched call runs. This is the mechanism behind most
`NoSuchMethodError` reports in Compose Multiplatform projects, and it is why
the fix is to pin the module back to the anchor's version rather than accept
the promotion.

## Pinning back

The legitimate use of `strictly` / `resolutionStrategy` here is narrow: some
other dependency's graph is dragging a JetBrains mirror forward past the
anchor's pinned version, and the fix holds that one module at the anchor's
version instead of letting the promotion above happen.

Per-dependency, at the point it's declared:

```kotlin
// build.gradle.kts — commonMain, pinning a transitive mirror back to the anchor
implementation(libs.lifecycle.viewmodel.compose) {
    version { strictly(libs.versions.lifecycle.get()) }
}
```

`strictly` is Gradle's strongest version declaration: any version not
matching it is excluded from resolution outright, and if a `strictly`
constraint conflicts with another consumer's requirement, resolution fails
rather than silently choosing one (verified 2026-08-04,
<https://docs.gradle.org/current/userguide/dependency_versions.html>). That
failure is the point — it converts a silent promotion into a build you have
to look at, instead of the runtime surprise described above.

Project-wide — when more than one module needs the same pin, or the library
forcing the promotion isn't declared directly in this build file:

```kotlin
// build.gradle.kts — project-wide, holding every request for the mirrored Lifecycle
// artifacts at the anchor's pinned version
configurations.configureEach {
    resolutionStrategy.eachDependency {
        if (requested.group == "org.jetbrains.androidx.lifecycle") {
            useVersion(libs.versions.lifecycle.get())
            because("hold the JetBrains Lifecycle mirror at the Compose Multiplatform 1.11.1 anchor's pinned version")
        }
    }
}
```

(`resolutionStrategy.eachDependency` with `useVersion` and `because`,
verified 2026-08-04,
<https://docs.gradle.org/current/userguide/resolution_rules.html>.) The
`because` string is not decoration — it is what keeps this block from turning
into the first red flag below once nobody remembers which library it was for.

Two prohibitions bound this:

- Never use `strictly` / `resolutionStrategy` to make a wrong coordinate
  resolve. If the coordinate itself is wrong, forcing a version produces a
  build that resolves the wrong artifact — re-classify the library
  ([coordinates.md](coordinates.md)) instead of forcing a number onto it.
- Never force a version the anchor release never shipped against. If a
  third-party library genuinely needs a newer mirrored library than the
  anchor pins, the correct move is to raise the anchor itself
  ([version-lock.md](version-lock.md)'s bumping procedure), not to force one
  module ahead of the coordinated set it was tested with.

## Red flags

| Smell | Why it's wrong |
|---|---|
| A `resolutionStrategy` block with no comment or `because` naming the library it forces | Unexplainable six months later — indistinguishable from a forgotten hack blocking a legitimate future bump |
| A forced version that appears in no anchor release's Dependencies section | The pin no longer points at a version JetBrains ever tested the rest of the set against — it has recreated the uncoordinated state the anchor exists to prevent |
| An `Unresolved reference` "fixed" by downgrading an unrelated library | Treats a coordinate problem as a version problem — re-classify the library instead ([coordinates.md](coordinates.md)) |
| A `--configuration` check run only on `androidDebugRuntimeClasspath` for a bug reported on iOS | Hides the target the bug was actually reported on — list the real configuration names for the project and check the one that matches the platform, per `## Reading resolution` above |
