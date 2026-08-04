# Deep Links Across Targets

A deep link starts as a platform-specific string — an Android `Intent`, an
iOS URL, a browser address bar — and has to end up as an `AppRoute` list on
the back stack from [nav3-core.md](nav3-core.md). This reference covers both
halves: parsing the URI once in `commonMain`, and receiving it on Android,
iOS, desktop, and web. A tab-scoped landing spot is covered at the end,
pointing at [tabs-flows.md](tabs-flows.md).

## Nav 3 Has No Deep Link API

As pinned in this skill ([nav3-core.md](nav3-core.md)'s `## Dependencies` table), Navigation 3 has
no deep-link API: no `navDeepLink`, no manifest-declared URI table, nothing
that inspects a URI at all. The app owns matching a URI to a `NavKey` and
building the back stack that key belongs on — this section adapts the
official guide (https://developer.android.com/guide/navigation/navigation-3)
and the recipes' own deep link guide
(https://github.com/android/nav3-recipes/blob/main/docs/deeplink-guide.md).
For the Nav 2 equivalent this replaces — the declarative `navDeepLink {}`
DSL — see [nav2-migration.md](nav2-migration.md). That absence is the upside, not a gap: because
the app builds the stack itself, Up from a deep-linked screen behaves
however the app decides, not however the library inferred from a graph it
never saw the user navigate.

One caveat: `navigation3-runtime` gains a `deeplink` package starting at
`1.2.0-alpha03` — confirmed against that sources jar on
`dl.google.com/android/maven2`. The stack this skill actually pins,
`org.jetbrains.androidx.navigation3:navigation3-ui:1.1.1` (a stable GA
release whose POM pulls `navigation3-runtime:1.1.1`, also GA), has none of
it. Only the newer `1.2.0-alpha0x` line ships the package, and adopting it
means moving off the pinned stable release onto an alpha one — write the
parsing yourself, as below, unless that trade is one you've made.

## Matching a URI to Keys

Keep this in `commonMain` so every target shares one implementation, and one
set of bugs to fix:

```kotlin
// commonMain
fun parseDeepLink(uri: String): List<AppRoute>? {
    val path = uri.substringAfter("://", missingDelimiterValue = "")
        .substringAfter('/', missingDelimiterValue = "")
        .substringBefore('?')

    val segments = path.split('/').filter { it.isNotEmpty() }
    return when {
        segments.isEmpty() -> listOf(Home)
        segments[0] == "item" && segments.size == 2 -> listOf(Home, Details(segments[1]))
        segments[0] == "profile" -> listOf(Home, Profile)
        else -> null   // unknown link: fall back to the default stack
    }
}
```

This matches `https://example.com/item/42` against the Android manifest
pattern below (`host="example.com"`). It's deliberately simple — a router
past a handful of patterns should move to an explicit pattern list like the
recipes' `DeepLinkPattern`/`DeepLinkMatcher`, not more nested `when` arms.

The return type is the detail worth pausing on: `parseDeepLink` returns a
*list*, the whole back stack a deep link should produce, not one
destination. `Home` beneath `Details(id)` is what makes Up and Back behave
like the user tapped their way there — return just `[Details(id)]` and the
first Back press exits the app, on a screen the user just arrived at. `null`
matters the same way: an old client can receive a link a newer release
started sending, for a route it doesn't know yet, so it falls back to
`listOf(Home)` instead of crashing.

## Building the Synthetic Back Stack

Apply the parsed stack before the user perceives the default screen:

```kotlin
@Composable
fun AppNavDisplay(initialUri: String?) {
    val backStack = rememberNavBackStack(navConfig, Home)

    LaunchedEffect(initialUri) {
        val stack = initialUri?.let(::parseDeepLink) ?: return@LaunchedEffect
        backStack.clear()
        backStack.addAll(stack)
    }

    NavDisplay(backStack = backStack, onBack = { backStack.removeLastOrNull() }, /* … */)
}
```

`LaunchedEffect(initialUri)` re-fires whenever `initialUri` changes — what a
warm-start URI arriving after composition needs. Where a platform hands over
the URI *before* Compose starts (Android's `onCreate`, before `setContent`),
resolve `parseDeepLink` there instead and pass the result straight into
`rememberNavBackStack(navConfig, *stack.toTypedArray())` — that skips
composing `Home` for a frame before the effect replaces it. Either way, the
failure this avoids is the one from above: a stack with only the
deep-linked screen exits the app on the first Back press.

## Android

The manifest declares which URIs launch the app and verifies host ownership:

```xml
<activity android:name=".MainActivity" android:exported="true">
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" android:host="example.com" />
    </intent-filter>
</activity>
```

```kotlin
// androidMain
class MainActivity : ComponentActivity() {
    private var currentUri by mutableStateOf<String?>(null)

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        currentUri = intent?.data?.toString()
        setContent { AppNavDisplay(initialUri = currentUri) }
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        setIntent(intent)
        currentUri = intent.data?.toString()
    }
}
```

`onNewIntent` is the warm-start case: a running Activity does not go through
`onCreate` again when a second deep link arrives — the system calls
`onNewIntent` on the existing instance instead. Skip it, and every link
after the first is silently dropped. `setIntent(intent)` keeps `getIntent()`
consistent afterward.

## iOS

Custom schemes go in `Info.plist` under `CFBundleURLTypes`
(`CFBundleURLSchemes: ["myapp"]`). Universal links (`https://example.com/...`)
use a different mechanism — an Associated Domains entitlement plus a
server-hosted `apple-app-site-association` file, not `Info.plist`.

SwiftUI's `onOpenURL` delivers the URL whether the app was already running
or just launched by it — one callback, not a cold/warm split like Android's.
It calls into a plain Kotlin object exported from the shared framework:

```swift
import SwiftUI
import Shared

@main
struct iOSApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .onOpenURL { url in
                    DeepLinkBridge.shared.handle(uri: url.absoluteString)
                }
        }
    }
}
```

```kotlin
// iosMain
object DeepLinkBridge {
    private val _uri = MutableStateFlow<String?>(null)
    val uri: StateFlow<String?> = _uri.asStateFlow()
    fun handle(uri: String) { _uri.value = uri }
}
```

Neither recipe repo wires a deep link through iOS end to end, so this bridge
is not a copied recipe — verify the interop call against the project's own
Kotlin/Native export setup.

## Desktop

The JVM launcher receives the URI as a process argument, or through a
protocol handler the packaging tool registers (packaging-specific — check
that tool's current docs):

```kotlin
// desktopMain
fun main(args: Array<String>) = application {
    val initialUri = args.firstOrNull()
    Window(onCloseRequest = ::exitApplication, title = "MyApp") {
        AppNavDisplay(initialUri = initialUri)
    }
}
```

What this doesn't solve alone: a second launch — another `myapp://…` click
while the app is open — starts a second JVM process and window instead of
forwarding the URI to the one already running. Fixing that needs the app to
register as single-instance (a local socket or file lock is the common
shape) and forward the URI in — no Compose Multiplatform API does this.

## Web

The other three targets receive a URI from the platform once per launch or
intent. The web target differs in kind: the browser's address bar *is* the
navigation state, continuously, and it must stay synced with `backStack` in
both directions for as long as the app runs — **app to browser**, every
`backStack` change pushes or replaces a history entry, so the address bar
reflects where the user is and reload lands them on the same screen;
**browser to app**, back/forward, a bookmark, or a typed URL arrive as a
`popstate` event or a fresh page load, and either has to re-parse the URL
and replace `backStack` to match. `history.pushState` never fires `popstate`
itself, so direction one does not loop back into direction two's handler.

`com.github.terrakok:navigation3-browser` is a **third-party** library (not
published by Google or JetBrains) that implements exactly this. Despite the
`com.github.` group id it is on Maven Central — fetchable through a normal
`mavenCentral()` repository, no JitPack step needed — `1.1.0` current
(https://central.sonatype.com/artifact/com.github.terrakok/navigation3-browser).
Its `webMain` API offers two modes: `ChronologicalBrowserNavigation` syncs
`backStack` to literal browser history with both back and forward through
the address bar; `HierarchicalBrowserNavigation` reflects only the current
entry, trading address-bar back/forward for app-like back behavior. It's a
small, single-maintainer library — weigh that before depending on it.

The hand-written alternative needs real JS interop for `window`, `History`,
and `popstate` — `navigation3-browser`'s own `BrowserApi.kt` does this with
hand-rolled `external interface` declarations resolved through an
`expect fun refBrowserWindow()`, wrapping the `popstate` listener in a
`callbackFlow`. `org.jetbrains.kotlinx:kotlinx-browser` (`0.5.0`) is an
independent, unrelated alternative to that same interop, not something
`navigation3-browser` itself depends on.

Interop is the easy part; the hazard is a naive effect that reads and
writes history in the same direction at once. Key a `LaunchedEffect` on
`backStack` to push history entries, while a separate handler also mutates
`backStack` from `popstate`, and a browser Back press updates `backStack`,
re-fires the first effect, and pushes a *new* forward-history entry in
response to what should have been a pure read — breaking the back button
the moment it's pressed twice.
`HierarchicalBrowserNavigation.kt` states the underlying browser bug
directly in a comment: calling `pushState`/`replaceState` synchronously
inside a `popstate` callback leaves Chrome and Firefox stuck in an invalid
history state. Its fix is to never do that — the `popstate`-driven listener
only ever calls `window.history.go(1)` or triggers a plain `onBack()`
callback, and every `pushState`/`replaceState` call lives in a separate
listener over the *app's* navigation state instead.
`ChronologicalBrowserNavigation.kt` solves the same problem differently: its
backstack listener compares the freshly computed stack string against
`window.history.state`, the fingerprint the browser already restored for a
`popstate`-driven change — a match means the change came from the browser
and only needs `replaceState` to correct bookkeeping, a mismatch means the
change originated in the app and gets a real `pushState`. Either shape
works; an effect with no such guard is a bug, not a simplification — write
the plumbing after reading both of those files, not from a short sketch of
one.

## Cold Start and Warm Start

| Target | Cold start (app not running) | Warm start (app already running) |
|---|---|---|
| Android | `intent?.data` read in `onCreate` | `onNewIntent` |
| iOS | `onOpenURL` (same callback as warm start) | `onOpenURL` |
| Desktop | first process argument to `main` | forwarded from the running instance, app-implemented |
| Web | `window.location.href` read at startup | `popstate`, or a fresh page load |

`parseDeepLink` is the one shared truth in that table — every row differs
only in *when* and *how* a URI string reaches it, never in what it means. A
deep link landing inside a tab has to place its key in that tab's own stack,
not a lone top-level `backStack` that no longer exists once tabs are
introduced — see [tabs-flows.md](tabs-flows.md). Which module owns each
platform's entry-point plumbing, and where `parseDeepLink` lives in a
multi-module build, is covered in
[di-modularization.md](di-modularization.md).
