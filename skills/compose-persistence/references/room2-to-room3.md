# Moving From Room 2 to Room 3

Room 2.8.4 is current, supported, and already multiplatform — it has shipped
`setDriver()` and the `androidx.sqlite` driver APIs since 2.7.0. Nothing here
frames Room 2 as something to escape. This file is a port guide for a
project that has decided to move, written against what changes and what does
not; it is not an argument for moving, and it is not a tutorial — read
[room-setup.md](room-setup.md), [schema-dao.md](schema-dao.md), and
[migrations.md](migrations.md) for how Room 3 works once you're on it. This
file only covers the delta.

## Before You Start: The Target Blocker

Check this before anything else, because it can end the exercise before it
starts. Room 3 and Room 2 do not drop the same targets as each other, and
Paging does not drop the same targets as either — so this is two separate
lines to check, not one combined list. Both lines below were confirmed by
downloading and reading each artifact's published Gradle module file
directly — `room-runtime-2.8.4.module`, `room3-runtime-3.0.2.module`, and
`paging-common-3.5.1.module` — not a guide page and not a
`maven-metadata.xml` version index, since neither of those lists a module's
published target variants.

**`iosX64`, `macosX64`, `tvosX64`, `watchosX64`.** Room 3.0.2 publishes none
of these four native variants — the coordinate table and the per-source-set
driver table in [room-setup.md](room-setup.md) both list this as Room 3's
dropped target set. Room 2.8.4 publishes all four. So a database on an
Intel iOS simulator target, a `macosX64` build, a `tvosX64` target, or a
`watchosX64` target has a genuine reason to stay on Room 2.x: nothing
forces the coordinate change, because Room 2.8.4 still resolves there and
Room 3 does not. But Paging 3.5.1 publishes none of these four either —
staying on Room 2.x buys back the database, not the pager. A project on any
of these four targets that also depends on Paging cannot take this skill's
Paging version on that source set, regardless of which Room generation its
database runs on.

**`mingwX64`.** Neither Room generation publishes it — Room 2.8.4 has never
shipped a `mingwX64` variant, and Room 3.0.2 doesn't either. Staying on
Room 2.x is not an escape hatch here the way it is for the four targets
above: there is no Room release, old or new, that resolves on a Windows
native target, so this isn't a reason to avoid the port, it's a reason
Room isn't available at all on that target. Paging 3.5.1, by contrast, does
publish `mingwX64` — so a Windows native target can take this skill's
Paging version even though it can take no version of Room.

If none of these five targets — the four in the first line, plus
`mingwX64` in the second — are in the project's target list, the rest of
this file applies. If one is, stop here: for the first four, the resolution
is dropping that target, staying on Room 2.x for the database while
accepting no Paging there, or moving forward with everything else in this
file and accepting the target is out of scope for whichever library still
needs it; for `mingwX64`, no Room version reaches that target, so nothing
below is Room-relevant to it either way.

## Substitutions

Most of the port is mechanical — a Gradle coordinate, a package prefix, and a
processor swap, applied everywhere with no behavior change attached:

| Room 2 | Room 3 |
|---|---|
| group `androidx.room` | group `androidx.room3` |
| `room-runtime`, `room-compiler`, `room-paging`, `room-testing` | `room3-runtime`, `room3-compiler`, `room3-paging`, `room3-testing` |
| package `androidx.room.*` | package `androidx.room3.*` |
| plugin id `androidx.room` | plugin id `androidx.room3` |
| extension `room { schemaDirectory(...) }` | extension `room3 { schemaDirectory(...) }` |
| KAPT or Java annotation processing | KSP only |

Before:

```kotlin
dependencies {
    implementation("androidx.room:room-runtime:2.8.4")
    ksp("androidx.room:room-compiler:2.8.4")
}
```

After — coordinates from [room-setup.md](room-setup.md)'s table:

```kotlin
dependencies {
    implementation("androidx.room3:room3-runtime:3.0.2")
    ksp("androidx.room3:room3-compiler:3.0.2")
}
```

A codebase still on KAPT for the Room 2 compiler cannot carry that forward:
Room 3 ships no annotation processor, KSP only, so a KAPT-only module has to
adopt KSP as part of this move rather than after it, even where KAPT is
otherwise still in use for something else in the same build.

## What Has To Be Rewritten

The substitution table above is find-and-replace. These five are not — each
is a shape change substitution cannot reach, and each is already documented
in full in a sibling file:

- **`SupportSQLiteDatabase` is no longer produced by Room 3's core APIs.**
  Room 3 is built on the `androidx.sqlite` driver APIs instead. Manual
  migrations, callbacks, and raw queries all move to the driver connection
  API — see the `Migration` shape and `connection.executeSQL(...)` in
  [migrations.md](migrations.md), and `useReaderConnection` /
  `useWriterConnection` in [schema-dao.md](schema-dao.md).
- **`setDriver()` is now mandatory, on every platform, including Android.**
  Room 2's implicit Android open helper does not exist in Room 3 — omitting
  the call throws at runtime, not at compile time. The per-platform builder
  code for every target is in [room-setup.md](room-setup.md)'s "Building It"
  section; none of it is optional the way it effectively was on Android-only
  Room 2.
- **`@ConstructedBy` is now required** on a `@Database` declared in
  `commonMain`, once any non-Android target is present — the compiler
  rejects a `commonMain` `@Database` without it. The full shape, including
  the paired `expect object … RoomDatabaseConstructor` and the
  `@Suppress("KotlinNoActualForExpect")` it needs, is in
  [room-setup.md](room-setup.md)'s "The Database Class" section.
- **Blocking DAO functions must move** out of any source set that targets a
  non-Android platform. A Room 2 DAO with plain (non-suspend) function
  signatures — common on Android, where the framework tolerates blocking
  calls off the main thread — does not carry over to `iosMain`, `jvmMain`,
  or a web target unchanged; see [schema-dao.md](schema-dao.md) for the DAO
  shapes Room 3 expects.
- **Java code generation is gone.** Room 3's KSP processor emits Kotlin
  only. A build that consumed Room-generated Java from a Java module, or
  that relied on `kapt`'s Java stub generation elsewhere in the same
  module, loses that output; there is no Java-output flag to set instead.

## Staging The Move

`androidx.room3:room3-sqlite-wrapper` (confirmed present at `3.0.2` via
`room3-sqlite-wrapper`'s own `maven-metadata.xml`) exists to keep the driver
migration and the group migration from having to land in one commit. It
wraps a Room 3 `RoomDatabase` — one already built with `setDriver(...)` — in
the legacy `SupportSQLiteDatabase` surface via a `RoomDatabase.getSupportWrapper()`
extension, so call sites still written against `SupportSQLiteDatabase` keep
compiling against a Room 3 database while they're migrated on their own
schedule. It is a bridge for the *coordinate* move, not a way to avoid the
*driver* move — everything it wraps is already running on `setDriver()`.

The recommended order follows from that: move to the driver APIs first,
while still on Room 2.7 or later — `setDriver()` and the `androidx.sqlite`
driver types are available there, so a Room 2.x codebase can convert its
manual migrations, callbacks, and raw-query call sites to the connection API
without touching a single Gradle coordinate. Only once that conversion is
done and verified does the group change from `androidx.room` to
`androidx.room3` — at that point the driver-shaped code the first step
produced needs little more than an import fix, per the substitution table
above.

This staging is possible at all because the two groups can coexist in one
build: `androidx.room` and `androidx.room3` are different Gradle coordinates
with different Kotlin packages, so nothing forces an all-or-nothing switch
across every module in a multi-module project on the day of the coordinate
change. A module still on `androidx.room:room-runtime` and a module already
on `androidx.room3:room3-runtime` can both build in the same project; they
cannot, however, share a single `@Database` — the coordinate and package
change is still per-database, all-or-nothing, at the point a given database
class is ported.
