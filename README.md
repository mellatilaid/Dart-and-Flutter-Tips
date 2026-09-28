---
title: 38 Dart & Flutter Tips That Actually Make a Difference in Production
published: true
tags: flutter, dart, mobile, performance
---

After 4+ years of building Flutter apps, I've learned that the biggest improvements come from small decisions repeated every day: a safer way to handle a nullable value, a cleaner way to build a list, knowing when `compute()` is worth it, proving a memory leak instead of guessing at it.

This list has two parts.

**Part 1** is my take on ideas from LeanCode's excellent article [24 Flutter & Dart Code Hacks & Best Practices](https://leancode.co/blog/flutter-coding-best-practices) (by Agnieszka Forajter and the LeanCode team). I read it, kept coming back to it, and tested these ideas on a production tourism app. Every tip in Part 1 is tagged *via LeanCode*. The examples and commentary are mine, but the credit for the ideas belongs to them, and you should read the original too.

**Part 2** is tips I collected myself from code reviews, production bugs, and mistakes I made the hard way.

For every tip I try to say what it does, why it matters, and when *not* to use it.

> Version notes: null-aware list elements (`?item`) need Dart 3.8+, dot shorthand (`.min`) needs Dart 3.10+.

---

# Part 1: Habits worth stealing (via LeanCode)

## Dart language habits

### 1. Bind while you check, don't assert afterwards *(via LeanCode)*

```dart
// Fragile
if (booking.guest != null) {
  showWelcome(booking.guest!.firstName);
}

// Safer
if (booking.guest case final guest?) {
  showWelcome(guest.firstName);
}
```

A local variable gets promoted to non-null after a check, but a class field or getter does not, because something else could change it before the next line. That is why you end up writing `!`. A pattern captures the value into a fresh local at the moment of the check, so the check and the use can never drift apart.

You can also reach into the object: `if (booking case Booking(guest: Guest(:final firstName?)))`.

> **My story:** *[Add a real crash or near-miss where a `!` broke after a refactor.]*

### 2. Describe a list, don't mutate one *(via LeanCode)*

```dart
final chips = [
  const AllDestinationsChip(),
  ?selectedRegionChip,
  if (isOffline) ...cachedChips else ...remoteChips,
];
```

The alternative is an empty list plus a handful of `add` and `addAll` calls, some inside `if` blocks. In review, the literal lets you read the final shape top to bottom instead of replaying mutations in your head.

### 3. Let `switch` do the branching *(via LeanCode)*

```dart
Widget homeFor(Session session) => switch (session) {
  GuestSession() => const BrowseScreen(),
  MemberSession(isProfileComplete: true) => const DashboardScreen(),
  MemberSession() => const OnboardingScreen(),
};
```

Made `Session` a `sealed` class and the compiler will refuse to build if a new session type appears and this switch doesn't handle it. That is the real gain, more than the shorter syntax.

### 4. Unpack in one line *(via LeanCode)*

```dart
final (lat, lng) = await locationService.current();

if (json case {'city': String city, 'country': String country}) {
  label = '$city, $country';
}
```

Destructuring shines with records and JSON maps. The second example checks the shape *and* extracts the values in a single step, which replaces a pile of `is String` checks.

### 5. Drop `async`/`await` on pure pass-throughs *(via LeanCode)*

```dart
Future<List<Hotel>> search(String query) => _api.searchHotels(query);
```

If the function only forwards a `Future`, the keywords add nothing. Keep them the moment you need `try/catch`, want to transform the result, or await two things in order. Without `await`, an exception thrown by the inner call escapes your `try` block instead of being caught by it.

### 6. Write the reason next to every `ignore` *(via LeanCode)*

```dart
// Payload comes from a platform channel and is validated in fromMap().
// ignore: avoid_dynamic_calls
final id = payload['id'];
```

An unexplained `ignore` makes the next reader ask "is this a bug or a decision?" A one-line reason answers that for good.

## Testing and logging

### 7. Use matchers that say what you mean *(via LeanCode)*

```dart
expect(results, hasLength(3));
expect(prices, everyElement(greaterThan(0)));
expect(total, closeTo(99.9, 0.01));
expect(() => cart.checkout(), throwsStateError);
```

Compare the failure messages, not just the code. `expect(x.length == 3, true)` fails with "Expected: true, Actual: false". `hasLength(3)` tells you how long the list actually was.

### 8. Keep your app's own widgets out of unrelated tests *(via LeanCode)*

If you are testing that `AppBottomSheet` shows its content, put a plain `Text` inside it, not your themed `AppLabel`. Otherwise the test also depends on your theme, localization, and any fallback logic in `AppLabel`. When it fails, you want the cause to be the bottom sheet, not something three layers away. A custom widget that swallows errors can even make a broken test pass.

### 9. Give your logger the error and the stack trace separately *(via LeanCode)*

```dart
// Everything squashed into one string
logger.e('Payment sync failed: $error\n$stackTrace');

// Structured (check your logger's exact signature)
logger.e('Payment sync failed', error: error, stackTrace: stackTrace);
```

Error trackers like Sentry can group and search structured errors. A stack trace buried in a message string is just text.

## Widgets and layout

### 10. Prefix widgets that return slivers *(via LeanCode)*

```dart
class SliverTripsHeader extends StatelessWidget { ... }
```

Putting a sliver inside a `Column` fails at runtime, far from where you wrote it, and the analyzer won't warn you. A `Sliver` prefix in the name makes the mistake visible at a glance. LeanCode's lint package also ships a rule for this, so a linter can enforce it for you.

### 11. Pick the small widget when you only need one thing *(via LeanCode)*

```dart
Padding(padding: const EdgeInsets.all(12), child: child);
ColoredBox(color: surface, child: child);
SizedBox(width: 48, height: 48, child: child);
```

Small widgets state their intent, and they have `const` constructors, which `Container` doesn't. But once you need padding *and* colour *and* a border, `Container` is easier to read than three nested widgets. It also handles border insets differently from `DecoratedBox` plus `Padding`, so switching between them can shift your layout slightly.

### 12. Use dot shorthand where the type is obvious *(via LeanCode)*

```dart
Row(
  mainAxisAlignment: .spaceBetween,
  children: [
    Text(title, overflow: .ellipsis, textAlign: .start),
  ],
);
```

Less noise, same behavior. I only use it where the expected type is clear from the parameter. If a reader has to guess what `.foo` refers to, write the full name.

### 13. Keep expensive work out of `build()` *(via LeanCode)*

```dart
late List<MonthGroup> _groups;

@override
void initState() {
  super.initState();
  _groups = groupByMonth(widget.bookings);
}

@override
void didUpdateWidget(covariant BookingList old) {
  super.didUpdateWidget(old);
  if (!identical(widget.bookings, old.bookings)) {
    _groups = groupByMonth(widget.bookings);
  }
}
```

`build()` can run many times per second. Do the grouping once, when the input changes, and let `build()` only describe the UI. Better still, do this work in your state layer (Cubit, BLoC, provider) so the widget never sees it.

### 14. Prefer widget classes to helper methods that return widgets *(via LeanCode)*

A `_buildPriceRow()` method rebuilds every time its parent does. A `const PriceRow(...)` widget class lets Flutter skip the whole subtree when its inputs haven't changed. It also gets its own name in DevTools and can be tested alone. Don't split every three-line snippet into a class, though. Extract when a piece is reused, has its own state, or deserves its own test.

## Performance and readability

### 15. Reach for `compute()` for heavy, one-off work, and only then *(via LeanCode)*

```dart
final destinations = await compute(_parseDestinations, rawJson);

List<Destination> _parseDestinations(String raw) => ...; // top-level or static
```

Parsing a multi-megabyte JSON response on the main isolate can freeze scrolling. But data crossing isolates is copied, so for small payloads the overhead can cost more than the work. For repeated work, see the long-lived isolate tip in Part 2.

### 16. Replace nested ternaries with `if` or `switch` *(via LeanCode)*

```dart
if (isOffline) return const OfflineBanner();

return switch (state) {
  .loading => const Spinner(),
  .empty => const EmptyTrips(),
  .data => TripList(trips),
};
```

A ternary is great for one condition. Past two, you are hiding the decision inside punctuation.

### 17. A tiny `let()` extension for nullable transformations *(via LeanCode)*

```dart
extension Let<T extends Object> on T {
  R let<R>(R Function(T) transform) => transform(this);
}

final label = prefs.getString('lastCity')?.let(City.parse).let(cityLabel);
```

It removes temporary variables from a chain of transformations. If the chain gets harder to read than four plain lines, use the four plain lines.

### 18. Make the analyzer strict, and let it do the nagging *(via LeanCode)*

```yaml
analyzer:
  language:
    strict-casts: true
    strict-inference: true
    strict-raw-types: true
```

Run `dart analyze` in CI and use `dart fix --apply` to clear the easy warnings in bulk. Every style comment the analyzer makes is one a teammate doesn't have to. See also tip 37 in Part 2.

### 19. Compose lists with spread and `if` *(via LeanCode)*

```dart
final actions = [
  const ShareAction(),
  ...?extraActions,
  if (booking.canCancel) CancelAction(booking.id),
  ...adminActions,
];
```

Compare that with `+`, `?? []`, and ternaries that return either a one-item list or an empty one. The null-aware spread `...?` skips a null list for you. The win is readability, not speed.

### 20. Name numbers that carry meaning *(via LeanCode)*

```dart
static const _otpResendCooldown = Duration(seconds: 45);
static const _maxPhotosBeforeGallery = 5;
```

A bare `45` says nothing about why it's 45. A name is the cheapest documentation you can write. `padding: 16` doesn't need one, but a number that encodes a product or accessibility rule does.

### 21. Be honest about what `castOrNull` hides *(via LeanCode)*

```dart
extension CastOrNull on Object? {
  T? castOrNull<T>() => this is T ? this as T : null;
}

// API sometimes sends 4 and sometimes 4.5 for "rating"
final rating = (json['rating'] as num?)?.toDouble();
```

The extension turns a loud `TypeError` into a quiet `null`, which is only right when "wrong type" and "missing" should be handled the same way. Otherwise you have hidden a backend bug. For int-or-double fields, `as num?` is usually the better tool, because the value is valid and only its type varies.

### 22. Extension types can give any value a domain API *(via LeanCode)*

```dart
extension type DeepLink(Uri _uri) {
  String? get bookingId => _uri.queryParameters['booking'];
  bool get isPromo => _uri.path.startsWith('/promo');
}
```

There is no wrapper object at runtime, just the `Uri`. The catch is that the type exists only at compile time. A runtime `is` check or cast sees the underlying `Uri`, and an extension type can't hold extra fields.

### 23. Let `build_runner` watch for changes *(via LeanCode)*

```bash
dart run build_runner watch
```

Keep it running in its own terminal tab and generated code (`freezed`, `json_serializable`, routers) stays current while you work. It's an ergonomics win, not a speed win: the incremental `build` is just as fast, you simply stop forgetting to run it.

### 24. Use hooks when lifecycle boilerplate repeats *(via LeanCode)*

```dart
class SearchField extends HookWidget {
  const SearchField({super.key, required this.onSearch});
  final ValueChanged<String> onSearch;

  @override
  Widget build(BuildContext context) {
    final controller = useTextEditingController();
    final query = useValueListenable(controller).text;
    final debounced = useDebounced(query, const Duration(milliseconds: 400));

    useEffect(() {
      if (debounced != null && debounced.isNotEmpty) onSearch(debounced);
      return null;
    }, [debounced]);

    return TextField(controller: controller);
  }
}
```

Creating, listening to, and disposing a controller normally spreads across `initState` and `dispose`. Hooks put it in one place. Keep each hook to a single job. A widget with ten unrelated effects is as hard to follow as the `StatefulWidget` it replaced.

---

# Part 2: Tips from my own production experience

## Testing that catches real problems

### 25. Golden tests for catching visual regressions

Golden tests (`matchesGoldenFile`) capture a reference image of a widget and diff it against a fresh render on every CI run. The payoff: you catch a visual regression, like shifted padding or a colour that changed by accident, before it ever reaches human review, let alone production. The cost: you need to pin them down carefully, because font rendering differences across CI environments produce false positives. We only run them inside one standardized CI image to avoid that.

### 26. Use `mocktail` over `mockito` for new projects, and `fake_async` for time-dependent tests

`mocktail` skips code generation entirely, which saves real build time on larger projects. `fake_async` lets you test code involving `Future.delayed` or `Timer` without actually waiting in real time. You get faster, deterministic tests, instead of a real `await Future.delayed()` that slows down CI and occasionally produces flaky results.

## Performance beyond the basics

### 27. Stop worrying about shader jank, but know why

If you've been carrying around `--cache-sksl` shader warm-up routines from an older project, you can retire them. Impeller, Flutter's rewritten rendering engine, compiles shaders ahead-of-time at build time instead of the first time they're used at runtime, which is what caused that old "stutters once, then runs smooth forever" bug. Impeller is now the default rendering engine across the current stable line: default on iOS since Flutter 3.16, and on Android since Flutter 3.22. If you're still hand-rolling shader warm-up code, it's dead weight. The advanced move here isn't a hack. It's knowing the old hack is obsolete and trusting the default.

### 28. Use long-lived isolates for repeated work, not `compute()` in a loop

`compute()` (built on `Isolate.run`) is convenient, but it spins up and tears down a fresh isolate every single call. If you're doing the same kind of computation repeatedly, that spawn-and-copy overhead adds up, and you'll get better throughput from an isolate that stays alive and receives multiple messages over time via `SendPort`/`ReceivePort`. This is the pattern usually called a "background worker" isolate. Reach for it when you're processing a stream of items (image thumbnails, incoming socket messages) rather than one one-off blob of JSON.

```dart
// One-off: fine
final result = await compute(parseJson, jsonString);

// Repeated: spin up once, reuse
final worker = await Worker.spawn(); // keeps a SendPort/ReceivePort alive
for (final item in incomingStream) {
  worker.send(item);
}
```

If an isolate is finishing with a large object to hand back (a decoded image buffer, say), use `Isolate.exit()` instead of a normal return. It transfers ownership of the object to the receiving isolate instead of copying it, which matters a lot for large payloads.

### 29. Reach for a custom `RenderObject` only when layout widgets genuinely can't do it

This is a legitimate escape hatch, not a toy. The honest criteria: you need it when existing layout widgets like Row, Column, Stack, or Wrap can't express the layout you need, when you need fully custom painting like a chart or a game element, when you need custom hit-testing logic, or when you need to skip the Widget/Element overhead entirely for performance-critical rendering. If none of those apply, you don't need it. `CustomPaint` combined with a `CustomMultiChildLayout` covers most "custom layout" needs without touching the rendering pipeline directly.

## Release builds and debugging

### 30. `--obfuscate --split-debug-info`, and actually keep the output

This one's less about runtime performance and more about a production hack almost nobody sets up correctly on the first try:

```bash
flutter build appbundle --obfuscate --split-debug-info=./debug-info/1.4.2
```

This strips Dart symbol names from the release binary while keeping a private mapping file for symbolication. The part people miss: you need that split-debug-info output to de-obfuscate stack traces from real crash reports. If you don't archive that folder per release version, every Crashlytics or Sentry stack trace you get from that build is permanently unreadable gibberish. Tag the folder with the exact version/build number and store it somewhere durable. One caveat worth knowing: obfuscation doesn't encrypt string literals. API keys hardcoded in Dart are still fully readable in the binary, so this isn't a substitute for actually not shipping secrets in source.

### 31. Prove memory leaks instead of guessing at them

"Did I actually leak that controller?" is usually answered by intuition. `WeakReference` and `Finalizer` let you answer it with evidence: attach a `Finalizer` to an object, and it fires a callback when the GC actually collects it. If the callback never fires after you navigate away and force a GC, you have a confirmed leak, not a suspicion.

```dart
final _finalizer = Finalizer<String>((label) {
  debugPrint('Collected: $label');
});

class MyController {
  MyController() {
    _finalizer.attach(this, runtimeType.toString(), detach: this);
  }
}
```

This is dev/debug-only tooling, not something you ship, but it turns "I think this page leaks" into "I proved this page leaks, and here's the exact class."

## Dart language: genuinely advanced features

### 32. Sealed classes for state that can't be invalid by construction

```dart
sealed class AuthState {}
class Loading extends AuthState {}
class Authenticated extends AuthState { final User user; Authenticated(this.user); }
class Error extends AuthState { final String message; Error(this.message); }

return switch (state) {
  Loading() => const CircularProgressIndicator(),
  Authenticated(:final user) => HomePage(user: user),
  Error(:final message) => ErrorView(message: message),
};
```

The point isn't the `switch`. It's that `bool isLoading` + `bool hasError` + `User? user` as separate fields allows logically impossible combinations (`isLoading == true` *and* `hasError == true`). Sealed classes make the invalid states unrepresentable, and the compiler's exhaustiveness check means a new state can't silently go unhandled somewhere.

### 33. Extension types for zero-cost domain types

```dart
extension type UserId(String value) {
  bool get isValid => value.length == 24;
}

void fetchUser(UserId id) { ... }
```

The bug this prevents: passing a raw `String` meant as a `userId` into a function expecting a `productId`. The compiler stays silent because both are just `String`. `extension type` gives you a distinct compile-time type that's impossible to mix up, with genuinely zero runtime cost. It's identical to the underlying `String` in memory, unlike a wrapper class, which needs its own allocation.

### 34. Deferred imports for real code-splitting

```dart
import 'package:my_app/heavy_feature.dart' deferred as heavy;

Future<void> openHeavyFeature() async {
  await heavy.loadLibrary();
  heavy.launch();
}
```

Most Flutter devs never touch this because most apps don't need it. But if you've got a rarely-used, code-heavy feature (a PDF editor, an in-app video processor), deferred imports keep it out of the initial bundle entirely and only load it when actually invoked. This changes your app's cold-start size, not just a single frame's render time.

## Habits that paid off on my team

### 35. Tag every hardcoded string so localization never loses one

Every multi-language app hits this: someone hardcodes a string during development ("just temporary"), it ships, and six months later nobody remembers it was never wired into the localization files. I solved this with a tiny extension:

```dart
extension StringExtension on String? {
  bool isNullOrEmpty() => this == null || this == "";
  String get hardcoded => this ?? "";
}
```

Instead of writing `Text("Loading...")`, I write `Text("Loading...".hardcoded)`. Functionally it does nothing, it just returns the same string. But it means every intentionally-hardcoded string in the codebase is tagged with a unique, greppable marker. When it's time to localize a new batch of strings, I just search the whole project for `.hardcoded` and get a complete, accurate list of every string that still needs to move into the localization files. No missed strings, no relying on memory or a design doc that's gone stale.

### 36. Put `BuildContext` lookups behind extensions

`Theme.of(context).textTheme`, `Theme.of(context).colorScheme`, `AppLocalizations.of(context)`: every Flutter codebase ends up typing these dozens of times per screen. I wrap them in extensions on `BuildContext` instead:

```dart
extension ThemeExtension on BuildContext {
  ThemeData get theme => Theme.of(this);
  TextTheme get textTheme => theme.textTheme;
  ColorScheme get colorScheme => theme.colorScheme;
  AppDimensionsTheme get dimensionsTheme =>
      theme.extension<AppDimensionsTheme>()!;
}

extension AppLocalizationsExtension on BuildContext {
  AppLocalizations get l10n => AppLocalizations.of(this);
}
```

So `Theme.of(context).colorScheme.primary` becomes `context.colorScheme.primary`, and `AppLocalizations.of(context).welcomeMessage` becomes `context.l10n.welcomeMessage`. It's a small thing, but it compounds: less visual noise in every build method, one place to change if a lookup's signature ever changes, and it reads closer to how the rest of your app's context-based APIs (`context.push`, `context.read`, if you're using go_router or Riverpod) already feel, so it doesn't stick out as a one-off convention.

### 37. Strict `analysis_options.yaml`, with CI failing on any warning

We set this up on the team from day one: any lint warning fails CI, not just errors. The result: code review conversations shifted from repetitive style nitpicks to actual logic and design discussion.

### 38. Keep business logic completely separate from widgets

When your repository/usecase layer is fully decoupled from widgets, you test it with plain unit tests, no `WidgetTester` needed. That's dozens of times faster than widget tests. On our team, this separation cut a significant chunk of our CI test suite runtime from minutes down to seconds.

---

## The takeaway

Thirty-eight tips, one rule underneath all of them: understand *why* before you apply a tip. Good code isn't code that follows every rule. It's code written by someone who knows when a rule applies and when it doesn't.

Thanks again to the LeanCode team for the ideas behind Part 1. If you have a tip you rely on that isn't here, I'd like to hear it.
