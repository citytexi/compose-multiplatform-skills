# API availability — is this function in my release?

The failure this file prevents doesn't look like a dependency bug: an API is
found in the Android API reference, written straight into `commonMain`, and
the build fails — not because the coordinate is wrong ([coordinates.md](coordinates.md))
and not because two versions of a module collided ([conflicts.md](conflicts.md)),
but because the Compose Multiplatform release the project is pinned to
predates the API entirely. Nothing here is a resolution problem; Gradle never
gets a chance to complain, because the call itself doesn't exist yet in the
version of the framework the project ships.

## Compose Multiplatform to Jetpack Compose

Snapshot read 2026-08-04 from
<https://kotlinlang.org/docs/multiplatform/compose-compatibility-and-versioning.html>
(page last modified 02 June 2026), section "Jetpack Compose artifacts used" —
"the following table lists Jetpack Compose artifact versions used by each
version of Compose Multiplatform." This is the complete table as that page
published it on the date read, newest first, not a subset picked for this
file:

| Compose Multiplatform version | Jetpack Compose version |
|---|---|
| 1.11.1 | 1.11.2 |
| 1.10.3 | 1.10.5 |
| 1.9.3 | 1.9.4 |
| 1.8.2 | 1.8.2 |
| 1.7.3 | 1.7.6 |
| 1.7.1 | 1.7.5 |
| 1.7.0 | 1.7.1 |
| 1.6.11 | 1.6.7 |
| 1.6.10 | 1.6.7 |
| 1.6.2 | 1.6.4 |
| 1.6.1 | 1.6.3 |
| 1.6.0 | 1.6.1 |
| 1.5.12 | 1.5.4 |
| 1.5.11 | 1.5.4 |
| 1.5.10 | 1.5.4 |
| 1.5.1 | 1.5.0 |
| 1.5.0 | 1.5.0 |
| 1.4.3 | 1.4.3 |
| 1.4.1 | 1.4.3 |
| 1.4.0 | 1.4.0 |
| 1.3.1 | 1.3.3 |
| 1.3.0 | 1.3.3 |
| 1.2.1 | 1.2.1 |
| 1.2.0 | 1.2.1 |
| 1.1.1 | 1.1.0 |
| 1.1.0 | 1.1.0 |
| 1.0.1 | 1.1.0-beta02 |
| 1.0.0 | 1.1.0-beta02 |

**The anchor** row for this skill's running example, `1.11.1`, maps to
Jetpack Compose `1.11.2` — that pairing is what the rest of this file works
through. Treat this table as a snapshot, not a source: check the live page
before trusting a row, since JetBrains adds a new one with every Compose
Multiplatform release and this copy will not.

## Why there is a lag

The same page, section "Jetpack Compose and Compose Multiplatform release
cycles" (read 2026-08-04), states the process as four steps taken "when a new
version of Jetpack Compose is released":

1. "Use the release commit as a base for the next Compose Multiplatform
   version."
2. "Add support for new platform features."
3. "Stabilize all platforms."
4. "Release a new version of Compose Multiplatform."

And the cadence, quoted exactly, not rounded: "The gap between a Compose
Multiplatform release and a Jetpack Compose release is usually 1–3 months."

Steps 2 and 3 are where the lag lives — a Jetpack Compose API exists from the
moment its release commit lands in step 1, but nothing built on Compose
Multiplatform can call it until steps 2–4 finish, months later. Flatly:
**"the Android docs say it exists" is not evidence that it exists in your
project.** The Android API reference documents Jetpack Compose, not Compose
Multiplatform; the two move on the same track but never at the same speed.

## The check

1. Read the project's Compose Multiplatform plugin version — the anchor.
2. Map it through [`## Compose Multiplatform to Jetpack Compose`](#compose-multiplatform-to-jetpack-compose)
   above to its Jetpack Compose base version.
3. Find the API in the Android API reference and read its "Added in" version.
4. Added-in version newer than the base → it is not in your project.

Worked through once, with a function actually looked up on the Android API
reference rather than invented for the example:

- Anchor: Compose Multiplatform `1.11.1`.
- Step 2: from the table above, `1.11.1` maps to Jetpack Compose base
  `1.11.2`.
- Step 3: `FlexBoxConfig.then(other: FlexBoxConfig): FlexBoxConfig` — the
  infix operator that merges two flex-layout configs — is documented as
  "Added in 1.12.0-rc01," artifact `androidx.compose.foundation:foundation-layout`
  (verified 2026-08-04,
  <https://developer.android.com/reference/kotlin/androidx/compose/foundation/layout/FlexBoxConfig>).
  The enclosing `FlexBoxConfig` class itself is older — "Added in 1.11.0" on
  the same page — so the class compiles on this anchor; this one member does
  not.
- Step 4: `1.12.0-rc01` is newer than the base `1.11.2` → **no**, `then()` is
  not available on the 1.11.1 anchor. A project on this anchor that calls it
  in `commonMain` fails to resolve, and the fix is not a coordinate change —
  the function doesn't exist yet in any artifact this anchor pulls in.

## Two traps the check misses

- **The API exists in the mirrored artifact but is Android-only inside it** —
  present on the Android target and absent, or non-functional, everywhere
  else. This is not hypothetical: the compose-navigation skill documents a
  live instance of exactly this shape, in its Navigation 3 back-stack setup —
  a no-argument-config overload of the back-stack constructor that survives
  configuration change and process death is declared only in `androidMain`
  inside the `org.jetbrains.androidx.navigation3` mirror; every other
  target, desktop JVM included, must use a different overload that takes an
  explicit configuration object. The mirrored coordinate resolves on every
  platform — the specific member does not.
- **The API exists on the Jetpack side and was simply not part of what
  JetBrains republished for this release.** The Jetpack Compose base version
  in the table above names what the *runtime and layout* code was built
  against; it says nothing about whether every mirrored library
  ([coordinates.md](coordinates.md)'s JetBrains-mirror layer — Lifecycle,
  Navigation, Navigation3, SavedState, WindowManager) picked up the matching
  Jetpack release on the same day. A mirror can lag its own Jetpack
  counterpart independently of where the runtime/foundation/ui version sits.

Both traps are settled faster by the release's own "What's new" page than by
reading source or guessing from the table above. The URL pattern (confirmed
2026-08-04 by requesting both and reading the page title): drop the patch
number and the dots from the Compose Multiplatform version and splice it into
`https://kotlinlang.org/docs/multiplatform/whats-new-compose-<major><minor>.html`
— `1.11.1` → `whats-new-compose-111.html` (`HTTP 200`), `1.10.3` →
`whats-new-compose-110.html` (`HTTP 200`). That page enumerates what shipped
for the release by platform, which is a direct answer to both traps above
instead of an inference from a single version pair.

## When it is not there

In recommendation order, cheapest first:

1. **Wait for the next Compose Multiplatform release.** No code changes tomorrow, no divergence from the anchor. Costs calendar time — per the cadence
   above, usually 1–3 months from the Jetpack Compose release the missing API
   shipped in, and Compose Multiplatform's own stabilization work (steps 2–3
   above) can push it further out than that.
2. **Implement the behavior in common code yourself.** Stays on the anchor,
   so nothing else in the coordinated set moves. Costs ongoing maintenance:
   the hand-rolled version has to be tracked and retired once the real API
   arrives, or it silently diverges from what the Android docs describe.
3. **Move to a Compose Multiplatform dev build.** JetBrains publishes these
   to `https://redirector.kotlinlang.org/maven/compose-dev`, with version
   strings shaped like `1.8.2+dev2544` (both the repository URL and the
   version shape confirmed 2026-08-04 on the same compatibility page cited
   above). Explicitly unstable, and not something to ship: moving to a dev
   build changes the anchor itself, which means the entire mirrored set —
   Lifecycle, Navigation, Navigation3, SavedState, Material3, Material3
   Adaptive, WindowManager — has to be re-derived against it, not just the
   one function you needed.
   → re-deriving the set after moving the anchor: [version-lock.md](version-lock.md)
