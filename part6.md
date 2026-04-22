# PART 6 -- E2E MOBILE IN DETAIL

<div class="chapter-intro">
Part 5 turned you into a productive desktop E2E engineer. Part 6 does the same for mobile. If you have never written a line of React Native, never touched Xcode, and have no idea what an AVD is, you are in the right place. These first two chapters are the foundation — we teach React Native from zero (Chapter 32) and walk through every tool you need to install on your laptop to run a mobile E2E test (Chapter 33). After this, the rest of Part 6 will plug Detox, page objects, Maestro, and device bridges on top of a solid base.
</div>

---

## React Native Primer for TypeScript Engineers

<div class="chapter-intro">
If you already know TypeScript and a bit of React, you are 70% of the way to understanding React Native. The remaining 30% is about what happens under the hood — how JavaScript ends up driving a native iOS or Android UI. This chapter explains that gap so that, by the time we introduce Detox, you understand exactly <em>what</em> Detox is automating and <em>why</em> the <code>testID</code> prop is the single most important line of code a QA engineer will ever add to a component.
</div>

### 32.1 What Is React Native?

**React Native (RN)** is a framework from Meta that lets you write native mobile apps using TypeScript/JavaScript and React. The "native" part is not marketing — your `<View>` in JSX really does become a `UIView` on iOS and an `android.view.ViewGroup` on Android. It is **not** a web browser, not a webview, not a PWA.

**Key facts:**
- Language: TypeScript/JavaScript + React component model
- Created by: Meta (Facebook), 2015
- What it is: a JavaScript runtime + a bridge to the host platform's native UI toolkit
- Used by: Meta, Shopify, Discord, Microsoft Teams, and **Ledger Live Mobile**
- Not to be confused with: Cordova, Ionic, Capacitor (all of which render inside a `WebView`)

#### The bridge model (Old Architecture) vs the New Architecture

Classic RN apps run on two threads:

- **JS thread** — executes your bundled JavaScript in a JavaScript engine (Hermes by default since RN 0.70). This is where React reconciliation, your `useState`, your business logic, and your Redux store all live.
- **UI thread (main thread)** — the platform's native thread. Draws the actual pixels. On iOS this is driven by UIKit, on Android by the Android View system (or Jetpack Compose for newer widgets).

These two threads communicate through a **bridge** (old arch) or a **JSI + TurboModules** layer (new arch):

```
+---------------------------+          +-------------------------------+
|      JS Thread (Hermes)   |          |   UI / Main Thread (Native)   |
|  ----------------------   |  bridge  |   --------------------------  |
|   React tree              | <======> |   UIView tree (iOS)           |
|   setState / reconciler   |  (JSON   |   ViewGroup tree (Android)    |
|   Your TypeScript code    |  or JSI) |   Native modules (Camera,     |
|   Redux / Zustand state   |          |   USB, Bluetooth, Keychain)   |
+---------------------------+          +-------------------------------+
              ^                                        ^
              |                                        |
              | (in dev) Metro bundler serves JS       | (always) platform SDK
              | over http://localhost:8081             | drives real pixels
```

Every time your JS calls `setState`, RN diffs the virtual tree and sends a batch of "mutate this view, set this prop" instructions across the bridge. The UI thread applies them.

> **Note:** The New Architecture (JSI, Fabric, TurboModules) replaces JSON serialization over the bridge with direct C++ function calls exposed to JS. From a QA perspective the mental model is the same — there is still a JS side and a native side — but it is faster and the error messages look a bit different.

#### React Native vs web React

| Aspect | Web React | React Native |
|--------|-----------|--------------|
| Rendering target | DOM (`<div>`, `<span>`) | Native views (`UIView`, `ViewGroup`) |
| Styling | CSS, class names | `StyleSheet` objects (a JS subset of CSS) |
| Layout engine | Browser's CSS engine | Yoga (Flexbox-only, no block/inline) |
| Routing | `react-router`, `next/router` | `react-navigation` |
| File loading | Webpack / Vite | Metro |
| DOM APIs | `document.querySelector` | None — there is no DOM |
| Events | `onClick`, `onChange` | `onPress`, `onChangeText` |

> **Gotcha:** There is **no DOM**. `document` does not exist. `window` exists but is mostly empty. If a library assumes a browser, it does not work in RN.

#### React Native vs Electron vs Flutter

- **Electron** (what Desktop Ledger Live uses): ships Chromium + Node.js. Every UI element is an HTML element. No native views.
- **Flutter**: ships its own rendering engine (Skia). It does *not* use native views — it paints pixels itself. Fast, but "looks native" rather than "is native".
- **React Native**: uses the host platform's actual native views. When you put a `<Text>` on screen, it is a real `UILabel` / `TextView`. This is why testing frameworks like Detox can find it by the platform's accessibility ID.

### 32.2 Core Primitives

Instead of HTML elements, RN exposes a small set of built-in components. These are the only ones you need to read 90% of Ledger Live Mobile source.

#### `View` — the `<div>` of React Native

```tsx
import { View, StyleSheet } from "react-native";

export function Card(): JSX.Element {
  return (
    <View style={styles.card}>
      {/* children */}
    </View>
  );
}

const styles = StyleSheet.create({
  card: {
    padding: 16,
    borderRadius: 8,
    backgroundColor: "#fff",
  },
});
```

A `View` is a flex container. By default `flexDirection` is `column` (not `row` like on the web) and every `View` is itself a flex container (no need to set `display: flex`).

#### `Text` — **the only** way to put text on screen

```tsx
import { Text } from "react-native";

<Text style={{ fontSize: 16 }}>Hello, Ledger</Text>;
```

> **Gotcha:** You cannot put a string directly inside a `View`. `<View>Hello</View>` throws at runtime. Every visible string must be wrapped in `<Text>`. This trips up almost every new RN developer.

#### `Pressable` — the modern tap handler

```tsx
import { Pressable, Text } from "react-native";

<Pressable
  testID="portfolio-add-account-button"
  onPress={() => handleAdd()}
  style={({ pressed }) => [styles.btn, pressed && styles.btnPressed]}
>
  <Text>Add account</Text>
</Pressable>;
```

`Pressable` is the recommended way to make anything tappable. It exposes the `pressed` state and supports `onPressIn`, `onPressOut`, `onLongPress`, and hit-slop configuration. See [`reactnative.dev/docs/pressable`](https://reactnative.dev/docs/pressable).

#### `TouchableOpacity` — the older, still-everywhere alternative

```tsx
import { TouchableOpacity, Text } from "react-native";

<TouchableOpacity testID="continue-button" onPress={onContinue}>
  <Text>Continue</Text>
</TouchableOpacity>;
```

`TouchableOpacity` fades the button when pressed. It predates `Pressable`. Ledger Live Mobile still uses it extensively in legacy screens. Functionally equivalent for QA purposes.

#### `ScrollView` — scrolls any children

```tsx
import { ScrollView } from "react-native";

<ScrollView contentContainerStyle={{ padding: 16 }}>
  <LongForm />
</ScrollView>;
```

Use `ScrollView` for a **small, bounded** list of items. It renders every child immediately — bad for 500 accounts.

#### `FlatList` — virtualized lists

```tsx
import { FlatList } from "react-native";

type Account = { id: string; name: string };

<FlatList<Account>
  data={accounts}
  keyExtractor={a => a.id}
  renderItem={({ item }) => (
    <AccountRow testID={`account-row-${item.id}`} account={item} />
  )}
/>;
```

`FlatList` only renders the rows currently on screen — essential for the accounts list, operations history, market cap list, etc. See [`reactnative.dev/docs/flatlist`](https://reactnative.dev/docs/flatlist).

> **Gotcha:** If you set `testID` on the `FlatList` itself, it lands on the outer scroll container, not the rows. Row `testID`s must be set **inside `renderItem`** — on the row component — and the row component must actually forward the prop to its top-level `View`/`Pressable`. See 32.3 for why this matters.

#### Other primitives you will see

- `TextInput` — the native text field (`onChangeText`, not `onChange`).
- `Image` — renders a `UIImageView` / `ImageView`. Source is `{ uri: "https://..." }` or `require("./logo.png")`.
- `SafeAreaView` — pads children to avoid the notch / home indicator.
- `Modal` — renders a native modal on top of the app window.
- `ActivityIndicator` — the spinning wheel (`UIActivityIndicator` / `ProgressBar`).

> **Further reading:** [`reactnative.dev/docs/components-and-apis`](https://reactnative.dev/docs/components-and-apis) lists every built-in component and API.

### 32.3 The `testID` Prop — Your Best Friend

`testID` is the prop you will encounter, add, and debate more than any other. **It is the single most important prop for mobile QA.**

```tsx
<Pressable testID="portfolio-add-account-button" onPress={handleAdd}>
  <Text>Add account</Text>
</Pressable>;
```

#### What `testID` becomes on each platform

React Native translates `testID` into the platform's accessibility identifier:

| Platform | Native attribute | Queried by |
|----------|------------------|------------|
| iOS      | `accessibilityIdentifier` on the backing `UIView` | XCUITest, Detox `by.id()` |
| Android  | `resource-id` (tag on the `View`) | UIAutomator / Espresso, Detox `by.id()` |

Detox's `by.id("portfolio-add-account-button")` resolves to `accessibilityIdentifier == "portfolio-add-account-button"` on iOS and `resource-id == "portfolio-add-account-button"` on Android. See [`reactnative.dev/docs/view#testid`](https://reactnative.dev/docs/view#testid).

#### Why not use text or accessibility labels?

Same reason as desktop: text changes with i18n, layout, and copy reviews. `testID` is added **for tests**, stays stable across translations, and does not affect the rendered UI.

#### Common gotchas

1. **`testID` must be on a real view** — not on a `React.Fragment`, not on a `<>`, not on a pure JS object. Some components (like `react-native-svg`'s `<Svg>`) accept it but do not always propagate it to the backing view.

2. **Row-level `testID` requires forwarding**. If you write a reusable component:

   ```tsx
   type Props = { testID?: string; label: string };
   export function MyRow({ testID, label }: Props) {
     return (
       <View testID={testID}>        {/* <- must forward! */}
         <Text>{label}</Text>
       </View>
     );
   }
   ```

   If you forget `testID={testID}` on the outer `View`, Detox will never find the row even though the prop is set on the parent.

3. **Duplicate `testID`s** break Detox with an ambiguous-match error. Make them unique with a stable suffix: `` `account-row-${accountId}` ``.

4. **`FlatList` cells**. The outer `FlatList` is a scroll container; only rows rendered inside `renderItem` carry row-level `testID`s. To wait for a specific row, scroll the list first with Detox's `scrollTo()`.

5. **Dynamic vs static**. Prefer `testID={currency}` over `testID={\`row-${isSelected ? "sel" : "idle"}\`}` — a `testID` that changes on interaction is a flaky test waiting to happen.

> **Note:** In Ledger Live Mobile, `testID` values are kebab-case and read like a breadcrumb: `settings-general-language-row`, `send-recipient-input`, `modal-confirm-button`. Follow the existing convention when adding new ones.

### 32.4 Platform Module & Cross-Platform Branching

A single RN codebase produces two native apps. Sometimes you need to branch per platform:

```tsx
import { Platform } from "react-native";

if (Platform.OS === "ios") {
  // iOS-specific logic
}

// Or, cleaner:
const headerPaddingTop = Platform.select({
  ios: 20,    // under the notch
  android: 0, // status bar is handled by the system
  default: 0,
});
```

`Platform.OS` is `"ios" | "android" | "windows" | "macos" | "web"`. For Ledger Live it is always `"ios"` or `"android"`. See [`reactnative.dev/docs/platform`](https://reactnative.dev/docs/platform).

#### Why E2E tests often branch on platform

Detox exposes `device.getPlatform()` so tests can branch at runtime. Patterns you will see in Ledger Live Mobile E2E:

```ts
export const isAndroid = (): boolean => device.getPlatform() === "android";
export const isIos = (): boolean => device.getPlatform() === "ios";
```

Why? A handful of reasons recur:

- **Keyboard**: Android auto-dismisses the keyboard on `Enter`; iOS does not. Tests may need `device.pressBack()` on Android only.
- **Permissions dialogs**: On iOS, push/biometric dialogs are system alerts handled by the iOS alert API; on Android they are shown by the app itself.
- **Scrolling behavior**: iOS supports bounce/over-scroll; Android does not. Some tests calibrate pixel offsets.
- **File/attachment picker**: completely different implementations.
- **Back button**: only Android has a hardware-equivalent back button.

> **Gotcha:** Avoid scattering `if (isAndroid())` through tests. Put the branching inside a page-object method so tests stay clean.

### 32.5 Metro — The JavaScript Bundler

Metro is RN's dedicated bundler. It exists because Webpack was too heavy and too web-centric when RN was created.

**What Metro does:**
- Walks your `import` graph starting from `index.js`
- Transpiles TypeScript/JSX with Babel
- Produces a **single** JavaScript bundle (no code-splitting by default)
- Serves the bundle over HTTP at `http://localhost:8081/index.bundle`
- Hot-reloads modules in dev mode

#### Debug vs release builds

| Mode | Who serves JS? | When used |
|------|----------------|-----------|
| **Debug** | Metro dev server on port 8081 | Local development, Detox `*.debug` configs |
| **Release** | JS is bundled into the `.ipa` / `.apk` at build time | TestFlight, Play Store, Detox `*.release` configs |

A debug build loads the JS over HTTP each time the app starts. A release build has the JS baked in as a static file inside the app bundle. This is why you can edit code and reload without rebuilding in debug, but not in release.

> **Note:** For E2E we use both. Debug builds iterate fast locally. Release builds are what CI runs to catch bundler-only issues (dead-code elimination, Hermes bytecode). See [`reactnative.dev/docs/metro`](https://reactnative.dev/docs/metro).

### 32.6 Deep Links & the `Linking` API

A **deep link** is a URL that opens a specific screen inside your app. Ledger Live Mobile uses the `ledgerlive://` scheme.

```
ledgerlive://portfolio
ledgerlive://account?currency=bitcoin
ledgerlive://wc?uri=wc:abc123...
ledgerlive://discover/swap
```

When the OS sees a URL with a registered scheme, it launches the app and hands the URL to it. In RN you subscribe with the `Linking` module:

```tsx
import { Linking } from "react-native";

useEffect(() => {
  const sub = Linking.addEventListener("url", ({ url }) => {
    navigateFromDeeplink(url);
  });
  return () => sub.remove();
}, []);
```

See [`reactnative.dev/docs/linking`](https://reactnative.dev/docs/linking).

#### How deep links are used in E2E

Deep links are the fastest way to jump a test to a specific screen without tapping through navigation. Detox exposes `device.openURL()`, and Ledger Live Mobile wraps it in an `openDeeplink()` helper:

```ts
await openDeeplink("ledgerlive://account?currency=bitcoin");
```

This skips onboarding, the portfolio tab, the account list — straight to the Bitcoin account screen. Huge time saver and fewer flaky steps.

### 32.7 Dev Menu & React DevTools

When you run a debug build on a simulator/emulator, you can open the **dev menu**:

- iOS simulator: **Device → Shake** or **⌘D**
- Android emulator: **⌘M** (macOS) / **Ctrl+M**

From here you can reload JS, enable/disable Fast Refresh, toggle the performance overlay, and open the in-app element inspector.

The **element inspector** is the RN equivalent of the web's DOM inspector: tap a component on screen and the menu shows its props, source location, and accessibility info. This is how you discover which component owns which `testID` — critical when designing new E2E tests.

**React DevTools** (a standalone app: `npx react-devtools`) attaches over the network and shows the full component tree. Use it when designing page objects to understand the rendered hierarchy. See [`reactnative.dev/docs/debugging`](https://reactnative.dev/docs/debugging) and [`reactnative.dev/docs/react-devtools`](https://reactnative.dev/docs/react-devtools).

> **Gotcha:** The dev menu is only available in debug builds. Release builds strip it.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://reactnative.dev/">React Native</a> — official landing page</li>
<li><a href="https://reactnative.dev/docs/components-and-apis">Core Components and APIs</a> — every built-in primitive</li>
<li><a href="https://reactnative.dev/docs/view#testid">View: testID prop</a> — how testID maps to native platform attributes</li>
<li><a href="https://reactnative.dev/docs/pressable">Pressable</a> — modern touchable component</li>
<li><a href="https://reactnative.dev/docs/flatlist">FlatList</a> — virtualized list rendering</li>
<li><a href="https://reactnative.dev/docs/platform">Platform</a> — detect iOS vs Android at runtime</li>
<li><a href="https://reactnative.dev/docs/metro">Metro</a> — the RN bundler</li>
<li><a href="https://reactnative.dev/docs/linking">Linking</a> — deep linking API</li>
<li><a href="https://reactnative.dev/docs/debugging">Debugging</a> — dev menu and inspectors</li>
<li><a href="https://reactnative.dev/docs/react-devtools">React DevTools</a> — component tree inspection</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> React Native runs your TypeScript on a JS thread and drives real native views on the UI thread via a bridge. The <code>testID</code> prop becomes <code>accessibilityIdentifier</code> on iOS and <code>resource-id</code> on Android — which is what Detox queries. Master <code>View</code>, <code>Text</code>, <code>Pressable</code>, <code>FlatList</code>, the <code>Platform</code> module, Metro, and deep links, and you can read any screen in Ledger Live Mobile.
</div>

### 32.8 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — React Native for TypeScript Engineers</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> What does the <code>testID</code> prop on a <code>&lt;Pressable&gt;</code> become at runtime on the native side?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A CSS class on the Chromium DOM node</button>
<button class="quiz-choice" data-value="B">B) A custom React context consumed by Detox</button>
<button class="quiz-choice" data-value="C">C) <code>accessibilityIdentifier</code> on the backing UIView on iOS, and <code>resource-id</code> on the Android View</button>
<button class="quiz-choice" data-value="D">D) Nothing — it is stripped at build time in release mode</button>
</div>
<p class="quiz-explanation">React Native forwards <code>testID</code> to the platform's native accessibility identifier. Detox's <code>by.id()</code> query resolves to those native attributes, which is why the same test locator works on both platforms.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> Why does <code>&lt;View&gt;Hello&lt;/View&gt;</code> crash at runtime?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>View</code> is self-closing and cannot take children</button>
<button class="quiz-choice" data-value="B">B) Plain strings must be wrapped in <code>&lt;Text&gt;</code> — <code>View</code> is not a text container</button>
<button class="quiz-choice" data-value="C">C) Only <code>ScrollView</code> accepts string children</button>
<button class="quiz-choice" data-value="D">D) It works — the question is wrong</button>
</div>
<p class="quiz-explanation">Unlike a <code>&lt;div&gt;</code> in the DOM, <code>&lt;View&gt;</code> does not render text. Every visible string must be inside <code>&lt;Text&gt;</code>, which maps to <code>UILabel</code> / <code>TextView</code>.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> What happens if you put <code>testID="account-row"</code> directly on a <code>&lt;FlatList&gt;</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Every row receives the same <code>testID</code></button>
<button class="quiz-choice" data-value="B">B) Only the first row receives the <code>testID</code></button>
<button class="quiz-choice" data-value="C">C) It is ignored entirely</button>
<button class="quiz-choice" data-value="D">D) It lands on the outer scroll container, not on rows. Row <code>testID</code>s must be set inside <code>renderItem</code></button>
</div>
<p class="quiz-explanation"><code>FlatList</code> is a virtualized scroll container. Its own <code>testID</code> identifies the list. To address a row, pass a unique <code>testID</code> inside <code>renderItem</code> — and ensure the row component forwards it to its outer <code>View</code>.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> What is the role of the <strong>bridge</strong> (or JSI in the new architecture) in React Native?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It connects the JS thread (Hermes) to the UI / native thread so state changes in JS become native view mutations</button>
<button class="quiz-choice" data-value="B">B) It connects the app to Metro over HTTP</button>
<button class="quiz-choice" data-value="C">C) It is Detox's test runner channel</button>
<button class="quiz-choice" data-value="D">D) It converts JSX to HTML</button>
</div>
<p class="quiz-explanation">The bridge (old arch) serialized calls between the JS thread and the UI thread. The new architecture replaces that with JSI — direct synchronous function calls exposed to JS via C++ — but the role is the same: move data and commands between JS and native.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> How does a debug build differ from a release build with respect to JavaScript?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Debug uses Hermes, release uses JavaScriptCore</button>
<button class="quiz-choice" data-value="B">B) Debug loads the JS bundle from Metro over HTTP at app start; release bakes the JS bundle into the app package</button>
<button class="quiz-choice" data-value="C">C) Debug has a dev menu, release also has a dev menu</button>
<button class="quiz-choice" data-value="D">D) There is no difference — both download from Metro</button>
</div>
<p class="quiz-explanation">In debug the app fetches <code>index.bundle</code> from Metro on launch (which is why you need Metro running). In release the bundle is embedded in the <code>.ipa</code> / <code>.apk</code>, the dev menu is removed, and you cannot hot-reload.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> A test needs to jump directly to the Bitcoin account screen. What is the idiomatic way?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Tap through Portfolio → Accounts → Bitcoin</button>
<button class="quiz-choice" data-value="B">B) Call a native Detox API to push a screen onto the stack</button>
<button class="quiz-choice" data-value="C">C) Use a deep link such as <code>ledgerlive://account?currency=bitcoin</code> via <code>openDeeplink()</code></button>
<button class="quiz-choice" data-value="D">D) Edit the React navigation state from the test runner</button>
</div>
<p class="quiz-explanation">Deep links skip the tap-through navigation and cut flakiness. Detox's <code>device.openURL()</code> is wrapped by an <code>openDeeplink()</code> helper in Ledger Live Mobile E2E. The <code>Linking</code> API on the app side receives and routes the URL.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Mobile Toolchain & Environment Setup

<div class="chapter-intro">
Unlike desktop E2E — where <code>pnpm i</code> is roughly enough — mobile E2E requires you to install a small operating system's worth of native tooling: Xcode, Android Studio, CocoaPods, Ruby, Bundler, Java, Gradle, adb, simctl, applesimutils, a handful of emulators, and at least one simulator. This chapter is the definitive checklist. If anything breaks during the rest of Part 6, come back here.
</div>

### 33.1 The Toolchain at a Glance

Here is every moving part you will touch, grouped by role:

```
                +---------------------------------------------+
                |             YOUR LAPTOP (macOS)             |
                |                                             |
                |   proto / mise  ──>  Node, pnpm, Ruby       |
                |         │                                   |
                |         ├─> pnpm  ──>  monorepo packages    |
                |         │                                   |
                |         └─> Bundler  ──>  CocoaPods (gem)   |
                |                           │                 |
                |                           v                 |
                |                     Podfile / Pods          |
                |                                             |
                |   Xcode ──┬─> iOS Simulator                 |
                |           ├─> simctl (CLI to sim)           |
                |           └─> applesimutils (QA CLI)        |
                |                                             |
                |   Android Studio ──┬─> AVD Manager          |
                |                    ├─> emulator (CLI)       |
                |                    ├─> adb (device bridge)  |
                |                    └─> Gradle / JDK         |
                |                                             |
                |   Metro (localhost:8081)                    |
                |        ▲                                    |
                |        │ http (debug only)                  |
                |        │                                    |
                |   +----+-----------------+                  |
                |   |  Simulator / Emulator                   |
                |   |   runs Ledger Live Mobile .ipa / .apk   |
                |   +------------------------------------+    |
                +---------------------------------------------+
```

Pay attention to the arrows. Metro is a **dev-time** HTTP server that the app connects to **only for debug builds**. On iOS the simulator shares `localhost` with the host automatically. On Android the emulator does *not* — you must use `adb reverse` to forward port 8081 (see 33.5).

### 33.2 macOS-Only Reality (for iOS)

Apple legally restricts iOS development tooling to macOS:

- **iOS builds require a Mac** running macOS + Xcode. There is no supported way around this.
- **Android builds** work on macOS, Linux, and Windows.
- Linux contributors to Ledger Live Mobile can work on Android E2E only; the CI runs iOS jobs on macOS runners.

Windows is not supported for iOS. If you are on a Ledger-issued MacBook you are fine.

### 33.3 Version Management — `proto` and `mise`

A monorepo this big needs pinned tool versions. Ledger Live uses [`mise`](https://mise.jdx.dev/) (and historically [`proto`](https://moonrepo.dev/proto)) as the single source of truth.

At the repo root:

```
.prototools      (minimal: node, npm, pnpm)
mise.toml        (full: node, pnpm, npm, bun, gitleaks, pkl, plus plugins)
```

Example `mise.toml` entries (paraphrased from the actual repo):

```toml
min_version = '2026.3.1'

[plugins]
bundler   = "jonathanmorley/asdf-bundler"
cocoapods = "mise-plugins/mise-cocoapods"

[tools]
node = '24.14.0'
pnpm = '10.24.0'
npm  = '10.3.0'
```

Install `mise` once (`brew install mise`), then run `mise install` at the repo root. It reads the file and pulls down the exact versions, putting shims on your `PATH`. You never `nvm use` or `rbenv global` again — `mise` does it automatically when you `cd` into the repo.

> **Note:** `mise` can also manage Ruby, Bundler, CocoaPods, and Java via plugins. The current repo comments out Ruby management (install Ruby via Homebrew), but the plugins are declared so the team can flip them on. Check `mise.toml` on the branch you are working on.

> **Further reading:** [`mise.jdx.dev`](https://mise.jdx.dev/) and [`moonrepo.dev/proto`](https://moonrepo.dev/proto).

### 33.4 iOS Side — Xcode to CocoaPods

#### Xcode

Install from the Mac App Store. Version should match what the repo expects (check the `ios/` project's build settings). After install, run **once**:

```
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
sudo xcodebuild -license accept
xcodebuild -downloadPlatform iOS
```

This accepts the license and downloads the required iOS simulator platform. See [`developer.apple.com/documentation/xcode`](https://developer.apple.com/documentation/xcode).

#### iOS Simulator

Bundled with Xcode. Run it via **Xcode → Open Developer Tool → Simulator**, or from the CLI through `xcrun simctl`:

```
xcrun simctl list devices         # list all simulators
xcrun simctl boot "iPhone 15"     # boot a specific one
xcrun simctl shutdown all         # stop all
```

Reference: [`developer.apple.com/documentation/xcode/xcrun`](https://developer.apple.com/documentation/xcode/xcrun) and [`developer.apple.com/documentation/xcode/running-your-app-in-simulator-or-on-a-device`](https://developer.apple.com/documentation/xcode/running-your-app-in-simulator-or-on-a-device).

#### applesimutils

The Apple `simctl` CLI is powerful but misses a few operations Detox needs — granting permissions, clearing keychains, resetting push tokens. [`applesimutils`](https://github.com/wix/AppleSimulatorUtils) from the Wix team (who also maintain Detox) fills those gaps:

```
brew tap wix/brew
brew install applesimutils
```

If you skip this, Detox will complain at startup on iOS.

#### Ruby + Bundler + CocoaPods

CocoaPods is the iOS dependency manager. It is a **Ruby gem**. You do not install it globally — you use Bundler to install the exact version pinned by the repo's `Gemfile`:

```ruby
# apps/ledger-live-mobile/Gemfile (excerpt)
source "https://rubygems.org"

gem "fastlane", '~> 2.228.0'
gem 'cocoapods', '>= 1.13', '!= 1.15.0', '!= 1.15.1', '!= 1.16.0'
gem 'xcodeproj', '< 1.26.0'
```

Workflow:

```
cd apps/ledger-live-mobile
bundle install                # installs all gems locally (vendor/bundle)
bundle exec pod install --project-directory=ios
```

- `bundle install` reads the `Gemfile` + `Gemfile.lock` and installs the exact gem versions.
- `bundle exec pod install` runs CocoaPods via Bundler, so you use the pinned version instead of whatever is on your machine.
- `pod install` reads `ios/Podfile`, resolves all native iOS dependencies (React Native itself, Firebase SDK, WalletConnect, etc.), and generates `ios/Pods/` plus the `ios/ledgerlivemobile.xcworkspace`.

> **Gotcha:** After a big dependency bump, you will often need to run `pod install` again. Symptom: the app builds but crashes on launch with a missing class. Running `bundle exec pod install --project-directory=ios` and rebuilding almost always fixes it.

> **Further reading:** [`bundler.io/guides/getting_started.html`](https://bundler.io/guides/getting_started.html), [`cocoapods.org`](https://cocoapods.org/), [`guides.cocoapods.org/using/getting-started.html`](https://guides.cocoapods.org/using/getting-started.html), [`guides.cocoapods.org/using/the-podfile.html`](https://guides.cocoapods.org/using/the-podfile.html).

### 33.5 Android Side — Studio to Gradle to adb

#### Android Studio

Install from [`developer.android.com/studio`](https://developer.android.com/studio). The first launch walks you through installing:

- A JDK (Android Studio ships with one; you can use it)
- The Android SDK
- A default system image (pick API 34 or higher — check the repo's `android/build.gradle` for the target SDK)
- At least one AVD (Android Virtual Device)

Set two environment variables in your shell profile (`.zshrc`):

```
export ANDROID_HOME="$HOME/Library/Android/sdk"
export PATH="$PATH:$ANDROID_HOME/platform-tools:$ANDROID_HOME/emulator"
```

`platform-tools` is where `adb` lives. `emulator` is where the `emulator` CLI lives.

#### AVD Manager and the `emulator` CLI

AVDs are created in Android Studio: **Device Manager → Create device**. Name each one consistently, e.g. `Pixel_7_API_34`. List them from the CLI:

```
emulator -list-avds
emulator -avd Pixel_7_API_34                 # boot one
emulator -avd Pixel_7_API_34 -no-snapshot    # cold boot, useful after crashes
```

See [`developer.android.com/studio/run/managing-avds`](https://developer.android.com/studio/run/managing-avds) and [`developer.android.com/studio/run/emulator-commandline`](https://developer.android.com/studio/run/emulator-commandline).

#### `adb` — Android Debug Bridge

`adb` is the CLI that talks to connected devices and running emulators. Core commands:

```
adb devices                          # list attached devices / emulators
adb install app.apk                  # install an APK
adb shell pm clear com.ledger.live   # clear app data
adb logcat                           # stream device logs
adb reverse tcp:8081 tcp:8081        # <-- KEY for Metro
```

Reference: [`developer.android.com/tools/adb`](https://developer.android.com/tools/adb).

##### Why `adb reverse tcp:8081 tcp:8081` is critical

On iOS, `localhost` inside the simulator is the same as `localhost` on your Mac — so a debug build automatically reaches Metro on `http://localhost:8081/index.bundle`.

On Android it is not. The Android emulator runs in its own network namespace and its `localhost` is the emulator itself, not your machine. Without help, the app opens a connection to `localhost:8081` and gets nothing.

`adb reverse tcp:8081 tcp:8081` tells adb: "any TCP connection from the device to `localhost:8081` should be forwarded to `localhost:8081` on the host." Metro then answers and the debug bundle loads.

In Ledger Live Mobile's E2E setup, this call happens automatically in Detox's `setup.ts`:

```ts
await device.reverseTcpPort(8081);   // Metro
await device.reverseTcpPort(wsPort); // Device Bridge (default 8099)
```

See [`developer.android.com/tools/adb#forwardports`](https://developer.android.com/tools/adb#forwardports).

> **Gotcha:** If Android debug builds crash with "Unable to load script" or an endless red screen, your first instinct should be: "did `adb reverse` run? is Metro running on 8081?"

#### Gradle

Gradle is Android's build system. It resolves Maven dependencies, compiles Kotlin/Java, runs annotation processors, and packages the final `.apk`. You rarely invoke it directly — RN's scripts do. When you do:

```
cd apps/ledger-live-mobile/android
./gradlew assembleDebug
./gradlew assembleRelease
```

The Android Gradle Plugin version must match the Gradle version, and both must match the Kotlin version the RN release expects. When in doubt, follow the versions the repo pins. See [`docs.gradle.org/current/userguide/userguide.html`](https://docs.gradle.org/current/userguide/userguide.html) and [`developer.android.com/build/releases/gradle-plugin`](https://developer.android.com/build/releases/gradle-plugin).

#### System images and API levels

An AVD is "a device config + a system image." The system image is a specific Android version (e.g. "Android 14, API 34, x86_64 with Google Play"). Install system images from **SDK Manager → SDK Platforms** in Android Studio. For E2E, pick the API level the Android project targets (check `android/build.gradle`'s `compileSdk`). See [`developer.android.com/tools/releases/platforms`](https://developer.android.com/tools/releases/platforms).

### 33.6 The Ledger Live Monorepo Relevance

All of the above feeds into one location in the monorepo:

```
ledger-live/
  apps/
    ledger-live-mobile/           <-- the RN app
      android/                    <-- Gradle project, AndroidManifest
      ios/                        <-- Xcode project, Podfile
      src/                        <-- TypeScript source
      e2e/                        <-- Detox + page objects (covered in Ch 34+)
      detox.config.js             <-- Detox build & device matrix
      Gemfile                     <-- Bundler-managed gems (CocoaPods)
      package.json                <-- scripts: e2e:build, e2e:test
```

Part 1 Chapter 4 explained the monorepo layout at a high level (apps vs libs vs tools). Zoom in here: everything mobile-specific lives under `apps/ledger-live-mobile/`. Shared code (LLM logic, account model, countervalues) lives in `libs/` and is consumed through the monorepo workspace links, exactly like Desktop.

### 33.7 `ENVFILE` — Which Firebase Project Do You Mock Against?

Ledger Live Mobile reads its environment from a dotenv file pointed at by the `ENVFILE` variable. Two files matter for QA:

| File | Firebase / feature-flag backend | When to use |
|------|--------------------------------|-------------|
| `.env.mock`             | **Staging** Firebase (feature flags for dev + QA) | Default for local E2E and CI |
| `.env.mock.prerelease`  | **Production** Firebase (what real users see)     | Pre-release smoke tests only |

The `mock` suffix does not mean "fake everything." It means "run the app against **mocked** Ledger Sync, Countervalues, and Speculos-driven device flows" — the rest (Firebase, deep links, navigation) is still real. We mock because tests must be deterministic and must not spend real crypto or depend on external API availability.

Typical usage:

```
ENVFILE=.env.mock pnpm mobile e2e:build -c ios.sim.debug
```

The build scripts read `ENVFILE` and inject it into the native build, so both JS and native code see the same values. The file itself lives at the app root (`apps/ledger-live-mobile/.env.mock`) and is not checked in — see the internal `mobile-env.md` for the list of required keys.

> **Gotcha:** Running an E2E build without `ENVFILE` set produces an app that crashes on first launch with "Firebase config missing." Always specify it.

### 33.8 Build Commands — `pnpm mobile e2e:build`

From the repo root, the main entry points are:

```
pnpm mobile e2e:build -c ios.sim.debug
pnpm mobile e2e:build -c ios.sim.release
pnpm mobile e2e:build -c android.emu.debug
pnpm mobile e2e:build -c android.emu.release
```

Breaking those down:

- `pnpm mobile` is a monorepo script that `cd`s into `apps/ledger-live-mobile/`.
- `e2e:build` calls `pnpm detox build` inside that directory.
- `-c <config>` selects a configuration from `detox.config.js`. The full list exposed in the repo is:

| Config                   | Platform | Build type | Firebase / flags source      |
|--------------------------|----------|------------|------------------------------|
| `ios.sim.debug`          | iOS      | Debug      | `.env.mock` (staging)        |
| `ios.sim.staging`        | iOS      | Staging    | Staging Firebase             |
| `ios.sim.release`        | iOS      | Release    | Production-like, mock flags  |
| `ios.sim.prerelease`     | iOS      | Prerelease | `.env.mock.prerelease` (prod)|
| `android.emu.debug`      | Android  | Debug      | `.env.mock` (staging)        |
| `android.emu.staging`    | Android  | Staging    | Staging Firebase             |
| `android.emu.release`    | Android  | Release    | Production-like, mock flags  |
| `android.emu.prerelease` | Android  | Prerelease | `.env.mock.prerelease` (prod)|

Rules of thumb:
- For iterating on tests locally → `ios.sim.debug` or `android.emu.debug`. Fast, hot-reloadable.
- For reproducing a CI failure → the same `release` config CI uses. Slower but bytecode-accurate.
- For smoke-testing a release candidate before store submission → `prerelease`. Runs against production Firebase, so you see the flags real users will see.

`e2e:test` then runs the already-built app:

```
pnpm mobile e2e:test -c ios.sim.debug
```

Chapter 34 will dig into the Detox runner side. Here we only care that the **build** step produces an artifact (`.app` or `.apk`) that Detox can install.

### 33.9 Verifying Your Setup — A Checklist

Before you ever try to run a real test, go through this list. Each step has an expected output; if yours differs, stop and fix it before moving on.

**1. Tool versions**

```
mise --version            # >= 2026.3.1
node -v                   # 24.14.x
pnpm -v                   # 10.24.x
ruby -v                   # 3.x (via Homebrew or mise)
bundle -v                 # 2.x
pod --version             # 1.13+
```

**2. Xcode & simulators**

```
xcodebuild -version       # Xcode <pinned version>
xcrun simctl list devices | grep Booted    # at least one simulator or nothing yet
applesimutils --list      # prints simulators; means applesimutils works
```

**3. Android tools**

```
adb version               # Android Debug Bridge version
adb devices               # List of devices — empty is fine, not missing
emulator -list-avds       # Prints at least one AVD name
echo $ANDROID_HOME        # Prints /Users/you/Library/Android/sdk
```

**4. Repo-level install**

From the monorepo root:

```
pnpm install              # Installs all workspace packages
cd apps/ledger-live-mobile
bundle install            # Installs Ruby gems (CocoaPods, fastlane)
bundle exec pod install --project-directory=ios
```

Expected: `pnpm install` completes without errors, `bundle install` prints "Bundle complete!", `pod install` prints "Pod installation complete!"

**5. Metro can start**

```
pnpm mobile start         # Starts Metro on http://localhost:8081
```

Visit `http://localhost:8081/status` in a browser — you should see `packager-status:running`.

**6. Build a debug app**

```
ENVFILE=.env.mock pnpm mobile e2e:build -c ios.sim.debug
```

Expected: `xcodebuild` runs for several minutes, ends with "BUILD SUCCEEDED" and the built `.app` lands inside the Detox build output directory.

If all six pass, you are ready for Chapter 34 where we finally introduce Detox and wire the first test.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://developer.apple.com/documentation/xcode">Xcode documentation</a></li>
<li><a href="https://developer.apple.com/documentation/xcode/xcrun">xcrun / simctl</a></li>
<li><a href="https://developer.apple.com/documentation/xcode/running-your-app-in-simulator-or-on-a-device">Running your app in Simulator</a></li>
<li><a href="https://developer.android.com/tools/adb">adb reference</a></li>
<li><a href="https://developer.android.com/tools/adb#forwardports">adb: port forwarding / reverse</a></li>
<li><a href="https://developer.android.com/studio/run/managing-avds">Managing AVDs</a></li>
<li><a href="https://developer.android.com/studio/run/emulator-commandline">emulator CLI</a></li>
<li><a href="https://developer.android.com/tools/releases/platforms">Android platforms &amp; API levels</a></li>
<li><a href="https://cocoapods.org/">CocoaPods</a> and <a href="https://guides.cocoapods.org/using/getting-started.html">Getting Started</a> / <a href="https://guides.cocoapods.org/using/the-podfile.html">The Podfile</a></li>
<li><a href="https://bundler.io/">Bundler</a> and <a href="https://bundler.io/guides/getting_started.html">Getting Started</a></li>
<li><a href="https://docs.gradle.org/current/userguide/userguide.html">Gradle user guide</a> and <a href="https://developer.android.com/build/releases/gradle-plugin">Android Gradle Plugin</a></li>
<li><a href="https://moonrepo.dev/proto">proto</a> and <a href="https://mise.jdx.dev/">mise</a></li>
<li><a href="https://github.com/wix/AppleSimulatorUtils">applesimutils</a></li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Mobile E2E needs an entire platform toolchain on top of Node — Xcode + simulator + applesimutils + CocoaPods/Ruby/Bundler on the iOS side, and Android Studio + AVDs + adb + Gradle on the Android side. <code>mise</code> pins versions; <code>ENVFILE</code> picks which Firebase backend you test against; <code>pnpm mobile e2e:build -c &lt;config&gt;</code> compiles the app for Detox. On Android, <code>adb reverse tcp:8081 tcp:8081</code> is what makes debug builds able to reach Metro — without it, you get a red screen and a very confused developer.
</div>

### 33.10 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Mobile Toolchain &amp; Environment Setup</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> Why does <code>adb reverse tcp:8081 tcp:8081</code> exist in the Android E2E setup?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) To expose Detox's runner to the emulator</button>
<button class="quiz-choice" data-value="B">B) To reset the emulator's network stack</button>
<button class="quiz-choice" data-value="C">C) The emulator's <code>localhost</code> is not the host's — this command forwards the device's <code>localhost:8081</code> to the host's Metro on <code>localhost:8081</code></button>
<button class="quiz-choice" data-value="D">D) It is only used in release builds</button>
</div>
<p class="quiz-explanation">Android emulators have their own network namespace. Debug builds need to load the JS bundle from Metro on the host machine. <code>adb reverse</code> forwards the emulator's <code>localhost:8081</code> to the host's <code>localhost:8081</code>. iOS simulators share the host network so this is not needed.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q2.</strong> Why install CocoaPods via <code>bundle install</code> rather than <code>gem install cocoapods</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Bundler installs the exact gem versions pinned in <code>Gemfile</code> / <code>Gemfile.lock</code>, ensuring every engineer uses the same CocoaPods version</button>
<button class="quiz-choice" data-value="B">B) <code>gem install</code> does not work on macOS</button>
<button class="quiz-choice" data-value="C">C) Bundler installs faster</button>
<button class="quiz-choice" data-value="D">D) CocoaPods can only be installed through Bundler</button>
</div>
<p class="quiz-explanation">The repo pins specific versions (e.g. excluding known-broken CocoaPods 1.15.0 / 1.15.1 / 1.16.x). Bundler reads <code>Gemfile.lock</code> and installs those exact versions, so <code>bundle exec pod install</code> uses a reproducible CocoaPods on every machine and in CI.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q3.</strong> What is the practical difference between <code>.env.mock</code> and <code>.env.mock.prerelease</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) One enables mocks, the other disables them</button>
<button class="quiz-choice" data-value="B">B) <code>.env.mock</code> points to the staging Firebase project; <code>.env.mock.prerelease</code> points to the production Firebase project</button>
<button class="quiz-choice" data-value="C">C) <code>.env.mock</code> is for iOS, <code>.env.mock.prerelease</code> is for Android</button>
<button class="quiz-choice" data-value="D">D) They are identical — the suffix is cosmetic</button>
</div>
<p class="quiz-explanation">Both activate the same mocking strategy for Ledger Sync, countervalues, and device flows. They differ in which Firebase project (therefore which feature-flag set) the app reads from. Day-to-day E2E uses staging; pre-release smoke tests validate against production feature flags before shipping to users.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> What does <code>pnpm mobile e2e:build -c ios.sim.release</code> produce, and what is it used for?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A debug <code>.app</code> ready for hot reload</button>
<button class="quiz-choice" data-value="B">B) A Firebase release tag</button>
<button class="quiz-choice" data-value="C">C) A signed IPA for TestFlight</button>
<button class="quiz-choice" data-value="D">D) A release-mode iOS <code>.app</code> for the simulator — JS is pre-bundled (no Metro), ideal for reproducing CI failures</button>
</div>
<p class="quiz-explanation">The <code>*.release</code> Detox configs build a release-mode app: Hermes bytecode, bundled JS, no dev menu. This matches what CI runs, so it is the configuration to reach for when reproducing a CI-only bug locally.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> You skipped installing <code>applesimutils</code>. What symptom will you see first?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The iOS build fails in <code>xcodebuild</code></button>
<button class="quiz-choice" data-value="B">B) Detox fails at startup on iOS because it cannot grant permissions / reset the simulator</button>
<button class="quiz-choice" data-value="C">C) Metro refuses to start</button>
<button class="quiz-choice" data-value="D">D) The tests pass but Allure produces no report</button>
</div>
<p class="quiz-explanation"><code>applesimutils</code> supplies operations that plain <code>simctl</code> does not cover — granting permissions, clearing keychains, resetting push tokens. Detox calls it during test setup. Without it installed, the iOS run aborts almost immediately.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q6.</strong> What role does <code>mise</code> (or <code>proto</code>) play in the toolchain?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It pins exact versions of Node, pnpm, and optionally Ruby / CocoaPods / Java, and activates them automatically when you <code>cd</code> into the repo</button>
<button class="quiz-choice" data-value="B">B) It is a replacement for Gradle</button>
<button class="quiz-choice" data-value="C">C) It is the iOS build system</button>
<button class="quiz-choice" data-value="D">D) It manages Android AVDs</button>
</div>
<p class="quiz-explanation"><code>mise</code> reads <code>mise.toml</code> / <code>.prototools</code> at the repo root and shims the right versions of each tool onto your <code>PATH</code>. This removes the class of bugs where "works on my machine" means "works on <em>my</em> Node version."</p>
</div>

<div class="quiz-score"></div>
</div>

---
## Detox from Zero: Core Concepts

<div class="chapter-intro">
Detox is to React Native what Playwright is to the web — a grey-box end-to-end framework that drives a real application running on a real device (or simulator/emulator). If you have never touched it, this chapter takes you from "what is Detox?" to reading and writing tests confidently. Every concept is anchored in Ledger Live Mobile code. After this chapter you will understand the three-verb grammar of Detox (matcher, action, expectation), the synchronization model, and why mobile E2E feels different from web E2E even though the surface APIs look similar.
</div>

### 34.1 What Is Detox?

Detox is an open-source **grey-box** end-to-end testing framework for React Native, created and maintained by **Wix** since 2016. Wix built it because, at scale, their React Native app was flaky under every black-box tool available (Appium in particular). Their insight: a test runner that has **privileged, in-process knowledge** of the app under test can detect when the app is busy — and wait deterministically — rather than guessing with arbitrary `sleep()` calls.

**Key facts:**
- Language: TypeScript/JavaScript
- Created by: Wix (the website builder)
- First release: 2016
- Targets: React Native on iOS and Android only
- Runs on top of: **EarlGrey 2** (iOS) and **Espresso** (Android) — both are Google-maintained native UI test drivers
- Killer feature: **synchronization** — Detox knows when the app's JS bridge, dispatch queues, animations, timers, and network calls are idle, and it automatically waits until the app is idle before performing the next action

**Grey-box vs black-box — what the difference actually means:**

| Aspect | Black-box (Appium) | Grey-box (Detox) |
|--------|--------------------|------------------|
| How it drives the app | Talks to an external OS-level agent (WebDriverAgent on iOS, UIAutomator2 on Android) over HTTP | Injects a native test library into the app itself (EarlGrey/Espresso) |
| Knowledge of app state | None — sees only what the OS accessibility tree exposes | Full — sees the JS bridge, dispatch queues, animations, timers, network |
| Waiting strategy | Poll with timeouts (`implicitWait`) | Wait for app to be idle, then act |
| Flakiness | High — tests need explicit waits and retries everywhere | Low — idle detection eliminates most race conditions |
| Speed | Slow — every command is an HTTP round-trip to an agent | Fast — commands run inside the app process |
| Cross-platform real devices | Yes, on real devices + simulators | Simulators and emulators primarily (real device support is experimental) |

> **Why this matters for Ledger Live:** our React Native app talks to a Ledger device over BLE/USB, downloads feature flags, imports Redux state, dispatches mock device events from the bridge — a huge amount happens asynchronously on startup. Black-box testing this with arbitrary waits would be miserable. Detox waits for the bridge to be idle before it does anything.

**Detox vs Playwright — same verbs, very different model:**

| | Playwright (web/Electron) | Detox (React Native) |
|---|---|---|
| Driver protocol | Chrome DevTools Protocol (in-process) | Native test library injected into the app |
| Locator primitive | `Locator` — a lazy, retryable query | `Element` — a matcher reference, not a node |
| Auto-waiting | Actionability checks (attached, visible, stable, enabled) | App-wide idle detection (bridge + queues + timers + animations) |
| Assertion syntax | `expect(locator).toBeVisible()` | `expect(element(by.id("x"))).toBeVisible()` |
| Parallelism | Workers share a browser or spawn many | Each worker needs its own device/simulator |
| Cleanup granularity | `context` per test | `device` per worker, state management is manual |

The two tools look deceptively similar on the page. The underlying model is very different: Playwright watches the DOM; Detox watches the whole app runtime.

### 34.2 The Matcher–Action–Expectation Triad

Every Detox test boils down to three kinds of calls:

1. **Matcher** — how do I describe the element I want? (`by.id("send-button")`)
2. **Action** — what do I do to it? (`.tap()`)
3. **Expectation** — what should be true after? (`expect(...).toBeVisible()`)

The three are composed through `element()`:

```typescript
import { by, element, expect } from "detox";

await element(by.id("send-button")).tap();                 // matcher + action
await expect(element(by.text("Send"))).toBeVisible();      // matcher + expectation
```

**Side-by-side examples:**

```typescript
// (1) Tap a button identified by testID
await element(by.id("onboarding-skip")).tap();

// (2) Type into a text field then assert visibility of a result
await element(by.id("search-input")).typeText("Bitcoin");
await expect(element(by.text("Bitcoin (BTC)"))).toBeVisible();

// (3) Scroll a list until an element appears
await waitFor(element(by.text("Solana")))
  .toBeVisible()
  .whileElement(by.id("currencies-list"))
  .scroll(200, "down");

// (4) Long-press then assert a context menu opened
await element(by.id("account-row-bitcoin")).longPress();
await expect(element(by.id("account-context-menu"))).toExist();

// (5) Replace text (faster than typeText) and assert value
await element(by.id("amount-input")).replaceText("0.001");
await expect(element(by.id("amount-input"))).toHaveValue("0.001");
```

> **Rule:** every Detox statement is either "find + do" (matcher + action) or "find + assert" (matcher + expectation). If a line isn't one of those, you probably want `device.*` (see 34.8) or a `waitFor(...)` polling construct (see 34.7).

### 34.3 Matchers — `by.*`

Matchers are small description objects that tell Detox "the element is the one whose *X* equals *Y*". They do **not** find anything yet. They just describe.

| Matcher | What it matches | Android | iOS |
|---------|-----------------|---------|-----|
| `by.id("send-btn")` | React Native `testID` prop | `android:tag` | `accessibilityIdentifier` |
| `by.text("Send")` | Visible text content | `android:text` | label/text |
| `by.label("Send button")` | Accessibility label | `contentDescription` | `accessibilityLabel` |
| `by.type("RCTTextInput")` | Native view class name | Full Java class name | Full ObjC class name |
| `by.traits(["button"])` | iOS accessibility traits | not supported | `accessibilityTraits` |

**`by.id` is the idiomatic choice**, for the same reason `getByTestId` dominates in Playwright: it is stable, short, and injected by developers specifically for tests. The React Native source looks like this:

```tsx
<Pressable testID="onboarding-skip" onPress={handleSkip}>
  <Text>Skip</Text>
</Pressable>
```

And the test:

```typescript
await element(by.id("onboarding-skip")).tap();
```

**Composition** — when a test ID alone is not unique (e.g. many rows in a list), compose matchers with `.and(...)`, `.withAncestor(...)`, `.withDescendant(...)`:

```typescript
// Combine: the element with testID "row" AND containing text "Bitcoin"
await element(by.id("row").and(by.text("Bitcoin"))).tap();

// Ancestor: the "delete" button whose ancestor has testID "row-bitcoin"
await element(by.id("delete").withAncestor(by.id("row-bitcoin"))).tap();

// Descendant: the row that contains a child with testID "badge-selected"
await element(by.id("row").withDescendant(by.id("badge-selected"))).tap();
```

When multiple elements match, use `atIndex(n)`:

```typescript
await element(by.id("account-row")).atIndex(0).tap();  // first account row
```

This is exactly what Ledger Live does in `e2e/helpers/elementHelpers.ts`:

```typescript
getElementById(id: string | RegExp, index = 0) {
  return element(by.id(id)).atIndex(index);
},
```

> **Reference:** https://wix.github.io/Detox/docs/api/matchers

### 34.4 Actions — `element(...).*`

Once you have a matcher, you pick an action. Actions return promises — always `await` them.

| Action | What it does | Typical use |
|--------|--------------|-------------|
| `tap()` | Single tap at the element's centre | Buttons, list rows |
| `longPress(duration?)` | Press-and-hold | Context menus, reordering |
| `multiTap(times)` | Tap N times in quick succession | Double-tap to zoom |
| `tapAtPoint({ x, y })` | Tap at a specific offset inside the element | Click on a map, on a specific part of a custom view |
| `typeText(text)` | Simulate per-character keystrokes | Inputs that react to each character (autocomplete) |
| `replaceText(text)` | Replace the whole value atomically (no keystrokes) | Fast form filling — preferred when you don't need the per-char handlers |
| `clearText()` | Clear an input | Reset before typing |
| `scroll(amount, direction, startX?, startY?)` | Scroll the element by N pixels | Lists, scroll views |
| `scrollTo(edge)` | Scroll to `"top"`, `"bottom"`, `"left"`, `"right"` | Jump to list edges |
| `swipe(direction, speed?, percentage?)` | Swipe gesture on the element | Dismiss rows, pull-to-refresh |
| `pinch(scale, speed?, angle?)` | Pinch zoom gesture | Maps, images |

**Real examples:**

```typescript
// Tap a button
await element(by.id("continue-button")).tap();

// Type a BIP39 word, expect autocomplete to react
await element(by.id("word-input-1")).typeText("abandon");

// Replace text — faster, skips per-character events
await element(by.id("amount-input")).replaceText("0.00123");

// Clear then replace
await element(by.id("amount-input")).clearText();
await element(by.id("amount-input")).replaceText("1.0");

// Scroll a list 200px down, starting from 80% of its height (Android gotcha)
await element(by.id("account-list")).scroll(200, "down", NaN, 0.8);

// Swipe left on a row to reveal delete
await element(by.id("account-row-bitcoin")).swipe("left", "fast", 0.75);

// Pull-to-refresh
await element(by.id("portfolio-scroll")).swipe("down", "slow", 0.9);
```

> **Android gotcha (from `elementHelpers.ts`):** on Android, starting a scroll at `y = 0.5` (middle) often triggers a long-press on a list row. The Ledger Live helpers use `startPositionY = 0.8`:
> ```typescript
> const startPositionY = 0.8; // Needed on Android to scroll views: https://github.com/wix/Detox/issues/3918
> ```

> **Reference:** https://wix.github.io/Detox/docs/api/actions

### 34.5 Expectations — `expect(...).*`

Expectations retry-wait for up to a small internal timeout (≈ 1s by default, but synchronization-gated — see 35.3) and then throw if the condition is false.

| Expectation | Meaning |
|-------------|---------|
| `toBeVisible()` | Element is rendered AND visible on screen (intersection > 75% by default) |
| `toExist()` | Element exists in the hierarchy (may be off-screen) |
| `toHaveText(text)` | The element's text content equals `text` |
| `toHaveValue(value)` | The element's value equals `value` (inputs, switches) |
| `toHaveLabel(label)` | Accessibility label equals `label` |
| `toHaveId(id)` | testID equals `id` |
| `toBeFocused()` | Element currently has input focus |
| `toHaveSliderPosition(n)` | Slider is at position `n` |

Every expectation can be negated with `.not`:

```typescript
await expect(element(by.id("loading-spinner"))).not.toBeVisible();
await expect(element(by.id("error-banner"))).not.toExist();
```

**Mental model — `toBeVisible` vs `toExist`:**

- `toExist()` only checks the view hierarchy. A detached `View` deep under a collapsed scroll view counts as existing.
- `toBeVisible()` requires the element's visible area to exceed 75% of its size — it's much stricter, and it is the right default for "the user can see this".

```typescript
// Correct — assert the user can see the button
await expect(element(by.id("send-button"))).toBeVisible();

// Correct — assert the row is in the DOM even if off-screen (common for virtualized lists)
await expect(element(by.id("account-row-42"))).toExist();
```

> **Reference:** https://wix.github.io/Detox/docs/api/expect

### 34.6 `element()` Is Not an Element

This is the number-one source of confusion for people coming from Playwright, Selenium, or DOM manipulation.

```typescript
const btn = element(by.id("send-button"));
```

`btn` is **not a reference to a UI node**. It is a reference to a *query* — a matcher wrapped in a proxy — that Detox will resolve **every time** you call an action or expectation on it. There is no `await`, no round-trip to the device here. Nothing has happened yet.

This means:

- You can declare `element()` proxies eagerly at the top of a page object — it is free.
- The same proxy can resolve to different physical views across navigations — you don't need to re-create it.
- If the matcher is ambiguous, `element(...)` itself doesn't throw; the action/expectation does.

**Compare to Playwright's `Locator`:** same idea. A `Locator` is also lazy and re-evaluates on each action. The mental model carries over cleanly.

```typescript
// Declare once, use many times
const amountInput = element(by.id("amount-input"));
await amountInput.clearText();
await amountInput.replaceText("0.001");
await expect(amountInput).toHaveValue("0.001");
```

> **Reference:** https://wix.github.io/Detox/docs/api/the-element-object

### 34.7 `waitFor(...)` — Polling With a Timeout

Expectations have a short internal wait. When you need to wait longer — for example, for a screen to finish loading, or for a list to scroll until an item comes into view — use `waitFor(...)`:

```typescript
import { waitFor } from "detox";

// Wait up to 10s for the portfolio screen to appear
await waitFor(element(by.id("portfolio-screen")))
  .toBeVisible()
  .withTimeout(10000);
```

The chain is:
1. `waitFor(element(...))` — what to poll
2. `.toBeVisible()` / `.toExist()` / `.toHaveText(...)` / etc. — the condition
3. `.withTimeout(ms)` — the overall deadline

**`whileElement(...)` — scroll until visible:**

One of Detox's most useful patterns is "keep scrolling this list until the target is visible":

```typescript
await waitFor(element(by.text("Solana")))
  .toBeVisible()
  .whileElement(by.id("currencies-list"))
  .scroll(200, "down");
```

This says: poll for `by.text("Solana")` to be visible; while it isn't, scroll the list 200px down, then re-check. It is the idiomatic way to find items in long lists.

The Ledger Live wrapper is simply a tiny default-timeout helper:

```typescript
// e2e/helpers/elementHelpers.ts
waitForElementById(id: string | RegExp, timeout: number = DEFAULT_TIMEOUT) {
  return waitFor(element(by.id(id)))
    .toBeVisible()
    .withTimeout(timeout);
},
```

With `DEFAULT_TIMEOUT = 60000` — 60 seconds. Long, because mobile startup + Speculos handshake takes time.

> **Reference:** https://wix.github.io/Detox/docs/api/waitfor

### 34.8 The `device` Global

`device` is the process-wide handle to the simulator/emulator. It is imported from `detox`:

```typescript
import { device } from "detox";
```

| Method | What it does |
|--------|--------------|
| `device.launchApp(opts?)` | Launch/relaunch the app under test with optional `launchArgs`, `permissions`, `languageAndLocale`, `newInstance`, etc. |
| `device.reloadReactNative()` | Reload the JS bundle without restarting the native side. Fast test isolation for pure-RN state. |
| `device.terminateApp()` | Kill the app process cleanly |
| `device.sendToHome()` | Send the app to background (home screen). Used to test backgrounding |
| `device.takeScreenshot(name)` | Capture a screenshot, attached to the Detox artifacts folder |
| `device.generateViewHierarchyXml()` | Dump the entire native view tree as XML — gold for debugging "why doesn't this match?" |
| `device.openURL({ url })` | Open a deeplink (e.g. `ledgerlive://portfolio`) |
| `device.sendUserNotification(payload)` | Simulate a push notification tap |
| `device.getPlatform()` | `"ios"` or `"android"` — use to branch test logic |
| `device.reverseTcpPort(port)` | Android only: forward `emulator:port → host:port` (see 35.7) |
| `device.disableSynchronization()` | Turn off idle detection for stubborn screens (see 35.4) |

**Real usage from Ledger Live `commonHelpers.ts`:**

```typescript
const BASE_DEEPLINK = "ledgerlive://";

/** @param path the part after "ledgerlive://", e.g. "portfolio" */
export async function openDeeplink(path?: string) {
  await device.openURL({ url: BASE_DEEPLINK + path });
}

export function isAndroid() {
  return device.getPlatform() === "android";
}
```

And the launch call:

```typescript
await device.launchApp({
  launchArgs: {
    wsPort: port,                                  // which port the bridge server is on
    detoxURLBlacklistRegex: '\\(...\\)',           // URLs to NOT wait on (analytics, etc.)
    mock: getEnv("MOCK") ? getEnv("MOCK") : "1",   // mock mode flag
    IS_TEST: true,
  },
  languageAndLocale: { language: "en-US", locale: "en-US" },
  permissions: { camera: "YES" },                  // pre-grant iOS permissions
});
```

The `launchArgs` object is turned into CLI-style flags that the native code reads on boot (via `react-native-launch-arguments`). This is how the bridge client learns which WebSocket port to connect to.

> **Reference:** https://wix.github.io/Detox/docs/api/device

### 34.9 Your First Detox Test — Line by Line

Here is the minimal possible Detox spec. Nothing Ledger-specific, just the framework:

```typescript
// firstTest.spec.ts
import { by, device, element, expect } from "detox";

describe("First Detox test", () => {
  beforeAll(async () => {
    await device.launchApp();               // (1)
  });

  beforeEach(async () => {
    await device.reloadReactNative();       // (2)
  });

  it("shows the welcome screen on boot", async () => {
    await expect(element(by.id("welcome"))).toBeVisible();  // (3)
  });

  it("navigates to settings when the gear is tapped", async () => {
    await element(by.id("gear-button")).tap();              // (4)
    await expect(element(by.text("Settings"))).toBeVisible();
  });
});
```

**Walk-through:**

1. `device.launchApp()` — installs (if needed) and launches the app on the configured device. This happens once before all tests in the file.
2. `device.reloadReactNative()` — between every test, reload only the JS bundle. Much faster than a full `launchApp()` and enough to reset RN state for most tests. (When you need a full reset — e.g. the bridge client has to reconnect — use `device.launchApp({ newInstance: true })`.)
3. Matcher + expectation: find the element whose `testID === "welcome"`, assert it is visible. Detox synchronizes internally: if the splash screen is still animating, it waits.
4. Matcher + action: tap the gear button. Detox waits for the app to be idle (no pending animations, no pending JS, no in-flight network in the non-blacklisted set), then taps, then waits again.

There is no `sleep()`, no `waitForTimeout()`, no polling loop. That is the entire point of Detox.

### 34.10 The Platform Split — EarlGrey vs Espresso

Detox is a thin-ish cross-platform façade over two very different native drivers:

- **iOS:** [EarlGrey 2](https://github.com/google/EarlGrey) — Google's iOS UI test driver. Runs in the same process as the app. Hooks into `CADisplayLink`, `NSURLSession`, `NSOperationQueue`, main dispatch queue, etc., to detect idle.
- **Android:** [Espresso](https://developer.android.com/training/testing/espresso) — Google's Android UI test driver. Runs in an instrumented test process. Hooks into the main looper, `IdlingResource`s, view tree observer.

**Consequences you will hit in practice:**

| Symptom | iOS (EarlGrey) | Android (Espresso) |
|---------|----------------|--------------------|
| Test IDs resolve to | `accessibilityIdentifier` | `android:tag` |
| `by.text` matches against | `UILabel`/`UITextView` text | `android:text` (excludes children's text unless the view is a merged node) |
| `scroll()` starting position | Safe at `(NaN, 0.5)` | Often fails — use `(NaN, 0.8)` (see 34.4 gotcha) |
| Long lists | Virtualized `FlatList` rows exist when scrolled near, so `toExist` works after reaching them | Same |
| Networking idle | Watches `NSURLSession` | Watches OkHttp via an IdlingResource |
| Per-worker isolation | One Simulator per worker (simulator shutdown is cheap) | One Emulator per worker (emulator boot is expensive) |

You rarely need to branch by platform in test logic, but you *do* have to remember that:

- **`reverseTcpPort` is Android-only.** On iOS simulator, `localhost` in the app === `localhost` on the host, so no port forwarding is needed (see 35.7).
- **Android APKs ship as a pair**: the app APK plus a `testBinaryPath` (the instrumentation APK that hosts Espresso). The `detox.config.js` declares both. iOS ships a single `.app` bundle.

> **Why this matters for flakiness:** most "my test works on iOS but fails on Android" bugs trace back to one of: (a) wrong `by.text` match because Android excludes nested text, (b) scroll gesture rejected because it started on a pressable, (c) a missing `device.reverseTcpPort()` so the bridge client can't reach the host.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://wix.github.io/Detox/docs/introduction/getting-started">Detox: Getting Started</a> — official onboarding doc</li>
<li><a href="https://wix.github.io/Detox/docs/api/matchers">Detox: Matchers API</a> — full <code>by.*</code> reference</li>
<li><a href="https://wix.github.io/Detox/docs/api/actions">Detox: Actions API</a> — every action with options</li>
<li><a href="https://wix.github.io/Detox/docs/api/expect">Detox: Expectations API</a> — assertion reference</li>
<li><a href="https://wix.github.io/Detox/docs/api/the-element-object">Detox: The element() Object</a> — why it's a proxy, not a node</li>
<li><a href="https://wix.github.io/Detox/docs/api/waitfor">Detox: waitFor API</a> — polling and <code>whileElement</code></li>
<li><a href="https://wix.github.io/Detox/docs/api/device">Detox: device API</a> — the process-wide device handle</li>
<li><a href="https://github.com/google/EarlGrey">EarlGrey 2</a> — the iOS driver Detox wraps</li>
<li><a href="https://developer.android.com/training/testing/espresso">Espresso</a> — the Android driver Detox wraps</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Detox is a grey-box React Native E2E framework. Every test reduces to three verbs — matcher, action, expectation — composed through a lazy <code>element()</code> proxy. It synchronizes automatically with the app's internal queues, so you almost never write <code>sleep()</code>. Under the hood, iOS runs on EarlGrey and Android on Espresso, and the small platform seams (scroll start position, <code>reverseTcpPort</code>, APK vs app bundle) are the ones you'll hit in practice.
</div>

### 34.11 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Detox Core Concepts</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> What does "grey-box" mean in the context of Detox?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It uses a greyscale screenshot diffing algorithm</button>
<button class="quiz-choice" data-value="B">B) The test driver is injected into the app process itself, giving it privileged knowledge of the app's runtime state (JS bridge, dispatch queues, timers, network) to synchronize automatically</button>
<button class="quiz-choice" data-value="C">C) It is neither open-source nor fully proprietary</button>
<button class="quiz-choice" data-value="D">D) It only runs on physical devices, never on simulators</button>
</div>
<p class="quiz-explanation">Grey-box means the test framework has inside knowledge of the app under test. Appium is black-box — it sees only the OS accessibility tree. Detox ships a native library that is linked into the app, so it can detect when the app is idle and wait deterministically.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> What does <code>element(by.id("send-button"))</code> actually return?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A reference to the native view — fails immediately if no match</button>
<button class="quiz-choice" data-value="B">B) A promise resolving to the native view's coordinates</button>
<button class="quiz-choice" data-value="C">C) A lazy matcher proxy — no lookup happens until an action or expectation is called on it, and the lookup re-runs every time</button>
<button class="quiz-choice" data-value="D">D) An array of all matching elements</button>
</div>
<p class="quiz-explanation">Just like Playwright's <code>Locator</code>, <code>element()</code> is lazy. It stores the matcher and re-resolves against the current UI hierarchy on each action. This is why you can declare element proxies eagerly in page objects without worrying about screen transitions.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> Which statement is TRUE about <code>toBeVisible()</code> vs <code>toExist()</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>toBeVisible()</code> requires the element to be on screen with &gt;75% of its area visible, while <code>toExist()</code> only requires the element to be in the view hierarchy (possibly off-screen)</button>
<button class="quiz-choice" data-value="B">B) They are synonyms</button>
<button class="quiz-choice" data-value="C">C) <code>toBeVisible()</code> is iOS-only</button>
<button class="quiz-choice" data-value="D">D) <code>toExist()</code> asserts the element's parent exists, not the element itself</button>
</div>
<p class="quiz-explanation"><code>toBeVisible()</code> is the stricter, user-facing check — "the user can actually see this". <code>toExist()</code> is a hierarchy check — useful for virtualized lists where rows exist but may be scrolled off. Use <code>toBeVisible</code> by default.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> What is the correct way to "scroll this list until the target row is visible"?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Write a <code>for</code> loop that scrolls 200px and checks <code>isVisible</code></button>
<button class="quiz-choice" data-value="B">B) Use <code>device.scrollUntilVisible()</code></button>
<button class="quiz-choice" data-value="C">C) Add an arbitrary <code>await new Promise(r =&gt; setTimeout(r, 3000))</code> before the assertion</button>
<button class="quiz-choice" data-value="D">D) Use <code>waitFor(target).toBeVisible().whileElement(listId).scroll(200, "down")</code></button>
</div>
<p class="quiz-explanation">The <code>whileElement(...).scroll(...)</code> combinator is Detox's idiomatic polling-scroll. It re-checks the condition between each scroll step and gives up at <code>withTimeout</code>.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> What is the difference between <code>typeText("hello")</code> and <code>replaceText("hello")</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>typeText</code> is iOS-only, <code>replaceText</code> is Android-only</button>
<button class="quiz-choice" data-value="B">B) <code>typeText</code> simulates per-character keystrokes (slower, fires per-char handlers), <code>replaceText</code> sets the value atomically in one shot (faster, skips per-char handlers)</button>
<button class="quiz-choice" data-value="C">C) They are identical — one is just an alias</button>
<button class="quiz-choice" data-value="D">D) <code>replaceText</code> clears the field first, <code>typeText</code> appends</button>
</div>
<p class="quiz-explanation">Use <code>replaceText</code> for fast form filling when the app does not care about keystroke-by-keystroke events. Use <code>typeText</code> when you need autocomplete, validation, or analytics hooks that fire per character to run.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> Which of the following is a platform-specific Detox gotcha you will hit?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) iOS requires <code>reverseTcpPort</code> to reach <code>localhost</code> from the simulator</button>
<button class="quiz-choice" data-value="B">B) Android does not support <code>by.id</code>; you must use <code>by.label</code></button>
<button class="quiz-choice" data-value="C">C) On Android, scrolling from the default vertical start position (0.5) may be interpreted as a long-press on a list row, so Ledger Live's helpers use <code>startPositionY = 0.8</code></button>
<button class="quiz-choice" data-value="D">D) <code>device.takeScreenshot()</code> is unsupported on Android</button>
</div>
<p class="quiz-explanation">This is an infamous upstream issue (wix/Detox#3918). On Android, gestures starting on a pressable row can be eaten as long-presses. Ledger Live's <code>elementHelpers.ts</code> sets <code>startPositionY = 0.8</code> to start scrolls from near the bottom of the container.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Detox Advanced: Config, Synchronization & Bridge

<div class="chapter-intro">
Chapter 34 taught you the grammar. This chapter teaches you the machinery: how Detox is configured, how it stays in sync with the app, how Jest drives it, and — critically — how Ledger Live's custom WebSocket bridge injects Redux state, feature flags, and mock device events into the running mobile app. By the end you will be able to read every line of <code>detox.config.js</code>, <code>jest.config.js</code>, and the bridge server/client, and you will know when to reach for <code>device.disableSynchronization()</code> (spoiler: almost never).
</div>

### 35.1 `detox.config.js` — The Top-Level Map

Every Detox project has one configuration file at its root. It has six top-level sections:

- `testRunner` — which runner to use (Jest here) and its options
- `apps` — the *things* to install (builds, binaries)
- `devices` — the *things* to run on (simulators, emulators)
- `configurations` — pairings of an app with a device (`ios.sim.debug`, etc.)
- `behavior` — lifecycle defaults (reinstall? clean up? expose globals?)
- `artifacts`, `session`, `logger` — I/O defaults

Here is the Ledger Live file, annotated section by section.

**File: `apps/ledger-live-mobile/detox.config.js`**

```js
// lines 1-3 — architecture constants
const iosArch = "arm64";
// NOTE: Pass CI=1 if you want to build locally when you don't have a mac M1.
const androidArch = process.env.CI ? "x86_64" : "arm64-v8a";
```

**Why this matters:** CI runners are x86_64, Apple Silicon dev machines are arm64. Gradle builds an ABI-specific APK, and picking the wrong one causes `INSTALL_FAILED_NO_MATCHING_ABIS`. The `CI=1` escape hatch lets you force x86 on a laptop (useful if you want to share a build with a CI debug artefact).

```js
// lines 5-17 — testRunner
testRunner: {
  $0: "jest",                              // the test runner executable
  args: {
    config: "e2e/jest.config.js",          // path to jest config
  },
  jest: {
    setupTimeout: 500000,                  // 500s before giving up on a worker's setup
  },
  retries: 0,                              // no Detox-level retries (we rely on CI-level retries)
  forwardEnv: true                         // pass process env into the worker subprocess
},
```

**Why `setupTimeout: 500000`?** Each Jest worker does `device.launchApp()` in `beforeAll` — on Android that can mean booting a cold emulator, installing the APK, and starting Metro. Half a second is fine on CI; half a millennium (500s) is the safety net.

```js
// lines 18-31 — logger + behavior
logger: {
  level: process.env.DEBUG_DETOX ? "trace" : "info",
},
behavior: {
  init: {
    reinstallApp: true,                    // blow away prior install + data each worker
    exposeGlobals: false,                  // do NOT inject by/element/expect as globals
  },
  launchApp: "auto",                       // auto-launch before first test of each worker
  cleanup: {
    shutdownDevice: process.env.CI ? true : false,  // kill simulator/emulator in CI
  },
},
```

- `reinstallApp: true` — a **fresh install** per worker boot. Eliminates state bleed between CI runs.
- `exposeGlobals: false` — Detox can inject `by`, `element`, `expect`, `waitFor` as globals. Ledger Live disables that and imports them explicitly — better TypeScript ergonomics and stricter linting.
- `launchApp: "auto"` — before the first test in each file, call `device.launchApp()`. Our own `setup.ts` in `beforeAll` then calls `launchApp()` again (with our extra args + bridge port); the second call is a no-op if we pass the same args — here we actually relaunch with the bridge port.
- `shutdownDevice` — only kill the simulator at end of run on CI. Locally, leave it running to amortise boot time across re-runs.

```js
// lines 32-72 — apps
apps: {
  "ios.debug": {
    type: "ios.app",
    build: `export ENVFILE=.env.mock && xcodebuild ARCHS=${iosArch} ONLY_ACTIVE_ARCH=YES \
            -workspace ios/ledgerlivemobile.xcworkspace -scheme ledgerlivemobile \
            -configuration Debug -sdk iphonesimulator -derivedDataPath ios/build`,
    binaryPath: "ios/build/Build/Products/Debug-iphonesimulator/ledgerlivemobile.app",
  },
  // ... ios.staging, ios.release, ios.prerelease ...
  "android.debug": {
    type: "android.apk",
    build: `cd android && ENVFILE=.env.mock ./gradlew app:assembleDebug app:assembleAndroidTest \
            -DtestBuildType=debug -PreactNativeArchitectures=${androidArch} && cd ..`,
    binaryPath: `android/app/build/outputs/apk/debug/app-${androidArch}-debug.apk`,
    testBinaryPath: "android/app/build/outputs/apk/androidTest/debug/app-debug-androidTest.apk",
  },
  // ... android.release, android.prerelease ...
},
```

**Key fields:**

- `type` — `"ios.app"` or `"android.apk"`. Determines the install pipeline.
- `build` — shell command that Detox will run when you pass `detox build`. Can be anything; here it is `xcodebuild` on iOS and Gradle on Android. `ENVFILE=.env.mock` switches the RN config to mock mode (no real backend calls).
- `binaryPath` — where the artefact lands after build, so Detox knows where to find it when you pass `detox test` without rebuilding.
- `testBinaryPath` (Android only) — the separate instrumentation APK (Espresso test harness).

There are four iOS variants and three Android variants because each `.env` profile (`.env.mock`, `.env.mock.prerelease`) produces a different bundle, and each Xcode configuration (`Debug`/`Staging`/`Release`) enables different compiler flags.

```js
// lines 73-116 — devices
devices: {
  simulator: { type: "ios.simulator", device: { name: "iOS Simulator" } },
  simulator2: { type: "ios.simulator", device: { name: "iOS Simulator 2" } },
  simulator3: { type: "ios.simulator", device: { name: "iOS Simulator 3" } },
  emulator: {
    type: "android.emulator",
    device: { avdName: "Android_Emulator" },
    gpuMode: "swiftshader_indirect",
    headless: !!process.env.CI,
  },
  emulator2: { /* same but avdName: "Android_Emulator_2" */ },
  emulator3: { /* same but avdName: "Android_Emulator_3" */ },
},
```

**Why three simulators and three emulators?** The mobile suite can run with `JEST_MAX_WORKERS=3`. Each worker needs its own device — you cannot share a simulator between parallel Jest workers (two tests writing to the same RN state at the same moment would be chaos). The `jest.environment.ts` (see 35.6) looks at `JEST_WORKER_ID` and swaps the device to `simulator${id}` / `emulator${id}`.

- `gpuMode: "swiftshader_indirect"` — software GPU. On headless CI there is no real GPU; SwiftShader gives predictable rendering.
- `headless: !!process.env.CI` — no window in CI.

```js
// lines 117-148 — configurations (the things you actually pass to `detox test`)
configurations: {
  "ios.sim.debug":        { device: "simulator", app: "ios.debug" },
  "ios.sim.staging":      { device: "simulator", app: "ios.staging" },
  "ios.sim.release":      { device: "simulator", app: "ios.release" },
  "ios.sim.prerelease":   { device: "simulator", app: "ios.prerelease" },
  "android.emu.debug":    { device: "emulator",  app: "android.debug" },
  "android.emu.release":  { device: "emulator",  app: "android.release" },
  "android.emu.prerelease": { device: "emulator", app: "android.prerelease" },
},
```

These are the names you pass on the command line: `detox test -c ios.sim.release`. Each one is just a (device, app) pair. The `jest.environment.ts` post-processes the chosen device to swap it for `simulator2`, `simulator3`, etc., when running with multiple workers.

> **What's missing here?** `artifacts` and `session` are both absent, so Detox uses defaults. `artifacts` would control where screenshots/videos/logs go (the Allure adapter in `jest.config.js` handles screenshots instead). `session` would pin WebSocket ports between Detox and the app — we don't pin them because our own bridge uses a dynamic free port (see `commonHelpers.ts`).

> **Reference:** https://wix.github.io/Detox/docs/config/overview

### 35.2 The `detox test` CLI

Once configured, you run:

```bash
# Build the iOS debug variant
detox build -c ios.sim.debug

# Run tests on the iOS debug variant
detox test -c ios.sim.debug

# Run one spec only
detox test -c ios.sim.debug e2e/specs/deeplinks.spec.ts

# Keep the simulator booted and the app installed (faster iteration locally)
detox test -c ios.sim.debug --reuse --cleanup=false

# Run with 3 Jest workers (needs simulator2/simulator3 available)
JEST_MAX_WORKERS=3 detox test -c ios.sim.debug

# Write artefacts to a custom folder
detox test -c ios.sim.debug --artifacts-location ./my-artifacts
```

**Important flags:**

| Flag | Purpose |
|------|---------|
| `-c, --configuration <name>` | Which configuration from `detox.config.js` to use |
| `--reuse` | Don't reinstall the app; reuse the simulator's existing install. Huge iteration-time win |
| `--cleanup` | Kill the device after tests. Invert with `--cleanup=false` (default depends on `behavior.cleanup`) |
| `--artifacts-location <path>` | Where screenshots, videos, logs go |
| `--record-videos all`, `--record-logs all` | Force artefact capture regardless of pass/fail |
| `--loglevel <trace\|debug\|info\|warn\|error>` | Detox logger verbosity. Equivalent to `DEBUG_DETOX=1` for the `trace` level in our config |
| `--device-launch-args "..."` | Extra launch args appended to the app |
| `--workers <n>` | Shorthand for `JEST_MAX_WORKERS=n` in Ledger Live's setup |
| `--headless` | Override the config's headless flag (Android) |

> **Local iteration tip:** once you have the app installed once, drop `--cleanup` and pass `--reuse`:
> ```bash
> detox test -c ios.sim.debug --reuse --cleanup=false e2e/specs/send/send.spec.ts
> ```
> This skips the ~30s reinstall and ~10s simulator boot.

> **Reference:** https://wix.github.io/Detox/docs/guide/running-tests

### 35.3 How Detox Synchronizes — Idle Detection

The single most important thing to understand about Detox is **what it watches to decide "the app is idle, I can act now"**.

Before any action or expectation, Detox waits for all of the following to reach an idle state:

1. **The JavaScript thread** — the RN JS bundle must be done with all currently queued tasks (the RN JS runtime exposes hooks for this).
2. **The native dispatch queues** — iOS main queue + registered serial queues; Android main looper. No pending tasks.
3. **The native animation frame pump** — `CADisplayLink` on iOS, `Choreographer` on Android. No animations in flight.
4. **Active timers** — `setTimeout`, `setInterval`, `NSTimer`, Android `Handler.postDelayed`. Detox waits for short timers to fire; long ones are ignored (configurable).
5. **In-flight network requests** — any HTTP request not on the URL blacklist must complete.
6. **The React Native bridge** — all bridge-queued calls between JS and native must drain.

When all six queues drain, the app is *idle* and Detox performs the action. If any queue never drains within the test timeout, the action fails with:

> Test Failed: App is busy. Trying to synchronize...

This is informative, not a bug. The error message usually includes which queue is busy:

> App synchronization debug: {dispatch_queue: com.example.long_running_queue has 1 job(s); ui: 2 animations in progress}

**The URL blacklist — the single most important escape hatch.** If your app makes a long-polling or streaming request (e.g., analytics beacon, push notifications, feature flag streaming), Detox will wait forever for it to finish. The fix is to mark those URLs as "don't synchronize on me":

```ts
// Ledger Live — e2e/helpers/commonHelpers.ts (inside launchApp)
detoxURLBlacklistRegex:
  '\\(".*sdk.*.braze.*",".*.googleapis.com/.*",".*clients3.google.com.*",' +
  '".*tron.coin.ledger.com/wallet/getBrokerage.*"\\)',
```

That ugly string is the list of regexes Detox will NOT wait for. The format is a Detox-specific escaped tuple: `\("regex1","regex2",...\)`. Four regexes here:

- `.*sdk.*.braze.*` — Braze analytics SDK endpoints
- `.*.googleapis.com/.*` — Google APIs (push notification token registration, fonts)
- `.*clients3.google.com.*` — Google Play captive-portal detection (Android)
- `.*tron.coin.ledger.com/wallet/getBrokerage.*` — a specific long-polling endpoint

You can also set this at runtime:

```ts
await device.setURLBlacklist([".*analytics.*", ".*telemetry.*"]);
```

> **Reference:** https://wix.github.io/Detox/docs/articles/how-detox-works

### 35.4 Synchronization Troubles — When to Give Up

Sometimes a screen has an infinite animation, or a custom native module that Detox can't see, and the app is simply never idle. Symptoms:

- `App is busy. Trying to synchronize...` errors that persist even after blacklisting
- Tests that hang at the same screen every time
- `detox.log` showing the same queue stuck for >30s

**The nuclear option** — turn off synchronization entirely:

```ts
await device.disableSynchronization();
// ... interact with the screen manually using waitFor() ...
await device.enableSynchronization();
```

While disabled, Detox will not wait for anything before an action. You must manually `waitFor(...).toBeVisible().withTimeout(...)` every element. This is strictly worse than letting Detox synchronize — it brings back all the flakiness Detox was designed to eliminate — so do it narrowly, around exactly the offending screen, and re-enable immediately.

**Diagnostic recipe:**

```ts
await device.enableSynchronization();       // ensure on
try {
  await waitFor(element(by.id("infinite-spinner-screen")))
    .toBeVisible().withTimeout(5000);
} catch (e) {
  // capture state for debugging
  await device.takeScreenshot("busy-sync");
  console.log(await device.generateViewHierarchyXml());
  throw e;
}
```

If the view hierarchy shows an animated Lottie loader that never stops: add its URL to the blacklist (if it fetches remote JSON), wrap the interaction in `disableSynchronization`, or fix the animation in the app to terminate.

> **Reference:** https://wix.github.io/Detox/docs/troubleshooting/synchronization

### 35.5 Jest as the Runner — `jest.config.js`

Detox does not run tests itself; it delegates to a runner. Ledger Live uses **Jest**, because (a) the React Native ecosystem already ships with it, (b) it handles TypeScript via SWC, and (c) the `jest-allure2-reporter` plugin gives us rich Allure reports for free.

**File: `apps/ledger-live-mobile/e2e/jest.config.js`** — the important bits:

```js
// lines 35-80 — the config itself
module.exports = async () => ({
  rootDir: "..",                                                           // (1)
  maxWorkers: process.env.JEST_MAX_WORKERS
    ? parseInt(process.env.JEST_MAX_WORKERS, 10)
    : 1,                                                                   // (2)
  transform: {
    "^.+\\.m?jsx?$": "babel-jest",
    "^.+\\.(ts|tsx)?$": ["@swc/jest", {                                    // (3)
      jsc: {
        target: "es2022",
        parser: {
          syntax: "typescript",
          tsx: true,
          decorators: true,                                                // (4)
          dynamicImport: true,
        },
        transform: { react: { runtime: "automatic" } },
      },
      sourceMaps: "inline",
      module: { type: "commonjs" },
    }],
  },
  setupFilesAfterEnv: ["<rootDir>/e2e/setup.ts"],                          // (5)
  testTimeout: 150000,                                                     // (6)
  testMatch: ["<rootDir>/e2e/specs/**/*.spec.ts"],                         // (7)
  reporters: [
    "detox/runners/jest/reporter",                                         // (8)
    ["jest-allure2-reporter", jestAllure2ReporterOptions],
  ],
  globalSetup: "<rootDir>/e2e/jest.globalSetup.ts",                        // (9)
  testEnvironment: "<rootDir>/e2e/jest.environment.ts",                    // (10)
  testEnvironmentOptions: {
    eventListeners: [
      "jest-metadata/environment-listener",
      "jest-allure2-reporter/environment-listener",
      ["detox-allure2-adapter", detoxAllure2AdapterOptions],
    ],
  },
  transformIgnorePatterns: [
    `node_modules/.pnpm/(?!(${transformIncludePatterns.join("|")}))`,       // (11)
  ],
  verbose: true,
  resetModules: true,                                                      // (12)
});
```

Line by line:

1. **`rootDir: ".."`** — points at `apps/ledger-live-mobile/` (the config lives in `e2e/`, so `..` hops up). All other `<rootDir>` references resolve from there.
2. **`maxWorkers`** — **default is `1`**. Parallel E2E is off by default because each worker needs its own simulator/emulator, and booting three emulators locally is painful. Set `JEST_MAX_WORKERS=3` on CI where three emulators are pre-booted.
3. **`@swc/jest`** — not `ts-jest` (too slow). SWC compiles TypeScript to JS 20× faster. `target: es2022` keeps async/await native.
4. **`decorators: true`** — enables `@Step` (see 35.8). Without this flag, TypeScript decorators are a syntax error.
5. **`setupFilesAfterEnv: ["<rootDir>/e2e/setup.ts"]`** — runs `setup.ts` *after* Jest's globals (`describe`, `it`, `expect`) are injected. This is where Ledger Live wires up its globals: `global.app`, `global.Step`, the page-object helpers (see 35.6).
6. **`testTimeout: 150000`** — 150 seconds per test. Mobile is slow: launching the app, importing accounts over the bridge, mocking device events, rendering onboarding screens — it adds up.
7. **`testMatch`** — any `*.spec.ts` under `e2e/specs/`.
8. **`reporters`** — Detox's Jest reporter logs action traces; `jest-allure2-reporter` generates the Allure XML/JSON that populates the reporting dashboard.
9. **`globalSetup`** — runs *once* per process, before all workers. Here it just delegates to Detox's own setup (boot the device, install the app if needed). See 35.6.
10. **`testEnvironment`** — Ledger Live ships a **custom test environment** that extends Detox's. This is what lets us support multiple workers without hard-coding three copies of the config (see 35.6).
11. **`transformIgnorePatterns`** — by default Jest does NOT transform `node_modules`. This regex carves an exception for packages published as ESM (`ky`, `@mysten+ledgerjs-hw-app-sui`) so Jest can compile them.
12. **`resetModules: true`** — between tests, clear the module registry. This prevents Redux store singletons from leaking state across tests.

> **Reference:** https://jestjs.io/docs/configuration

### 35.6 `jest.globalSetup.ts` and `jest.environment.ts`

Jest distinguishes two lifecycle hooks:

- **`globalSetup`** — a single async function, runs **once** before the whole test run begins, in the master process.
- **`testEnvironment`** — a class instantiated **per worker**, `setup()` runs before each test file on that worker, `teardown()` after.

**`jest.globalSetup.ts`** is tiny:

```ts
// e2e/jest.globalSetup.ts
import { globalSetup } from "detox/runners/jest";

export default async () => {
  await globalSetup();
};
```

Detox's built-in `globalSetup` handles the heavy lifting: reading `detox.config.js`, deciding which device to reuse, deciding whether to install the app, etc. We just call it.

**`jest.environment.ts`** is where the interesting Ledger Live customization happens:

```ts
// e2e/jest.environment.ts
import { config as detoxConfig } from "detox/internals";
// @ts-expect-error detox doesn't provide type declarations for this module
import DetoxEnvironment from "detox/runners/jest/testEnvironment";

import { logMemoryUsage } from "./helpers/commonHelpers";

export default class TestEnvironment extends DetoxEnvironment {
  declare global: typeof globalThis;

  async setup() {
    const workerId = Number(process.env.JEST_WORKER_ID ?? 1);            // (1)
    if (workerId > 1) await this.setupDeviceForSecondaryWorker(workerId); // (2)
    await super.setup();                                                  // (3)
  }

  private async setupDeviceForSecondaryWorker(workerId: number) {
    const configName = process.env.DETOX_CONFIGURATION;
    if (!configName) throw new Error("Missing DETOX_CONFIGURATION...");

    const detoxConfigModule = await import("../detox.config.js");
    const detoxConfigFile =
      "default" in detoxConfigModule ? detoxConfigModule.default : detoxConfigModule;
    const targetConfig = detoxConfigFile.configurations[configName];
    if (!targetConfig || !targetConfig.device) {
      throw new Error(`Detox configuration "${configName}" not found...`);
    }

    const deviceName = `${targetConfig.device}${workerId}`;               // (4) "simulator" -> "simulator2"
    const targetDevice = detoxConfigFile.devices[deviceName];
    if (!targetDevice) {
      throw new Error(`Device configuration not found for ${deviceName}...`);
    }

    Object.assign(detoxConfig, { device: targetDevice });                 // (5) monkey-patch Detox's runtime config
  }

  async handleTestEvent(event: any, state: any) {
    await super.handleTestEvent(event, state);

    if (["hook_failure", "test_fn_failure"].includes(event.name)) {
      this.global.IS_FAILED = true;                                       // (6)
    }
    if (event.name === "run_start") {
      await logMemoryUsage();                                             // (7)
    }
  }
}
```

What it does:

1. **`JEST_WORKER_ID`** — Jest sets this env var on each worker subprocess. Worker 1 uses the default device from config; workers 2+ need to swap.
2. For workers 2+, we load the config file, look up the configuration (e.g. `ios.sim.debug`), find its device (`simulator`), and ...
3. ... swap the runtime device to `simulator2` / `simulator3` (line 4) by monkey-patching `detox/internals.config.device` (line 5). Then call `super.setup()` so Detox boots the correct device.
4. **The naming convention is load-bearing:** `simulator + workerId` → `simulator2`, `simulator3`. This is why `detox.config.js` has exactly `simulator`, `simulator2`, `simulator3`. You can extend to 4+ workers by adding devices and AVDs.
5. **`IS_FAILED`** — on any hook failure or test failure, set a global boolean. `setup.ts` reads it in `afterAll` and attaches app logs to Allure only when the test failed (writing logs on every test would drown the report).
6. **`logMemoryUsage()`** — at the start of each test run, `top` is sampled for the current PID and memory is attached to Allure. Debugging memory leaks in a long-running E2E CI run is much easier with this timeline.

> **Why extend vs wrap?** Detox's `testEnvironment` already plugs into Jest's lifecycle, reads `detox.config.js`, and manages the Detox client/connection. Extending via `super.setup()` preserves all that behavior; we only inject the worker-ID device swap and the two observability hooks. Wrapping or re-implementing would fight Detox's internal state machine.

> **Reference:** https://jestjs.io/docs/configuration#testenvironment-string

### 35.7 The WebSocket Bridge — Why Detox Alone Isn't Enough

Detox drives the UI. It does not know about Redux, feature flags, or Ledger's hardware device events. For most RN apps, that is fine — you seed state by tapping through the UI. For Ledger Live it isn't, because:

- Onboarding the app from scratch every test would take 2 minutes (and require BLE pairing).
- Feature flags need to be overridden per test without a real Firebase fetch.
- Device events (USB plug, APDU responses) must be **mocked**, not driven by a real Ledger.
- Redux state (accounts, settings) must be pre-populated with test fixtures.

So Ledger Live layers a **custom WebSocket bridge** on top of Detox:

- **Server** — runs in the test process (Jest worker). Started in `commonHelpers.ts#launchApp` on a free port (default 8099). Sends commands.
- **Client** — linked into the RN app under `src/e2e/`. Connects to the server on boot. Receives commands, dispatches into Redux / emits native events / calls the navigation controller.

```
┌──────────────────────┐            ┌───────────────────────┐
│ Jest worker (Node)   │  WebSocket │ RN App on device      │
│                      │ <────────> │                       │
│ bridge/server.ts     │   ws://... │ bridge/client.ts      │
│  - postMessage()     │            │  - dispatch(importX)  │
│  - pendingAcks       │            │  - navigate()         │
│  - ws.Server         │            │  - setOverride(flag)  │
└──────────────────────┘            └───────────────────────┘
         │                                    │
         │          Test code calls:          │
         │     await importAccounts([...])    │
         │     await mockDeviceEvent(...)     │
         │     await setFeatureFlags({...})   │
         ▼                                    ▼
```

#### 35.7.1 Server side

**File: `apps/ledger-live-mobile/e2e/bridge/server.ts`** (excerpt):

```ts
// lines 61-83 — init: open a WebSocket server
export function init(port = 8099, onConnection?: () => void) {
  webSocket.wss = new WebSocket.Server({ port });
  webSocket.messages = {};
  log(`Start listening on localhost:${port}`);

  webSocket.wss.on("connection", (ws: WebSocket) => {
    log(`Client connected`);
    onConnection && onConnection();
    webSocket.ws?.close();           // close any prior client (reconnect case)
    webSocket.ws = ws;
    ws.on("message", onMessage);
    ws.on("close", () => { webSocket.ws = undefined; });
    // Re-send any messages that were queued while disconnected
    if (Object.keys(webSocket.messages).length !== 0) {
      Object.values(webSocket.messages).forEach(m => postMessage(m));
    }
  });
}
```

One bridge server per test process. The `global.webSocket` object (declared in `setup.ts`) holds the singleton so every helper in `bridge/server.ts` can reach it.

**The ACK pattern — every message waits for a reply:**

```ts
// lines 237-251 — postMessageAndWaitForAck
function postMessageAndWaitForAck(message: MessageData, timeout = RESPONSE_TIMEOUT): Promise<void> {
  return new Promise((resolve, reject) => {
    const timeoutId = setTimeout(() => {
      pendingAcks.delete(message.id);
      reject(new Error(`Timeout while waiting for ACK for ${message.type} (${message.id})`));
    }, timeout);

    pendingAcks.set(message.id, { resolve, timeoutId });
    postMessage(message);
  });
}

// lines 253-280 — onMessage handles the ACK reply
function onMessage(messageStr: string) {
  const msg: ServerData = JSON.parse(messageStr);
  switch (msg.type) {
    case "ACK": {
      delete webSocket.messages[msg.id];
      const pendingAck = pendingAcks.get(msg.id);
      if (pendingAck) {
        clearTimeout(pendingAck.timeoutId);
        pendingAcks.delete(msg.id);
        pendingAck.resolve();
      }
      break;
    }
    // ...
  }
}
```

**Why ACKs matter:** without them, `await setFeatureFlag(...)` would resolve as soon as the message left the server's socket — *before* the RN client had actually run the Redux dispatch. The next test line would race against the dispatch. With ACKs, the client only sends `{ type: "ACK", id }` *after* running the handler (see `bridge/client.ts` line 199), so the server's promise only resolves after the state change is live.

**The message catalogue** (`bridge/types.ts`):

| Message | Direction | Purpose |
|---------|-----------|---------|
| `open` | server → client | Open a BLE connection to a mock device |
| `acceptTerms` | server → client | Mark T&Cs as accepted in Redux |
| `importSettings` | server → client | Seed settings state |
| `importAccounts` | server → client | Seed accounts state from a fixture |
| `importBle` | server → client | Seed known BLE devices |
| `addUSB` | server → client | Emit a fake USB device-connect native event |
| `addKnownSpeculos` | server → client | Add a Speculos-emulated device to the known list and set DEVICE_PROXY_URL |
| `removeKnownSpeculos` | server → client | Remove a Speculos device, clear DEVICE_PROXY_URL |
| `overrideFeatureFlag` | server → client | Override a single feature flag (merge into existing overrides) |
| `overrideFeatureFlags` | server → client | Replace all feature flag overrides at once |
| `setGlobals` | server → client | Inject globals into the RN runtime (rarely used) |
| `navigate` | server → client | Call the React Navigation navigator programmatically |
| `mockDeviceEvent` | server → client | Push one or more device events into the mock subject (APDU responses, connect events, etc.) |
| `getLogs` / `appLogs` | ↔ | Request log dump; client replies with `appLogs` payload |
| `getFlags` / `appFlags` | ↔ | Request resolved feature flags; client replies |
| `getEnvs` / `appEnvs` | ↔ | Request live-env config; client replies |
| `swapSetup` / `swapSetupDone` | ↔ | Set up Swap mock env (sync handshake) |
| `ACK` | client → server | Acknowledge any server message after handling |

#### 35.7.2 Client side

**File: `apps/ledger-live-mobile/e2e/bridge/client.ts`** (excerpt):

```ts
// lines 34-74 — init: connect to the server from inside the app
export function init() {
  const wsPort = LaunchArguments.value()["wsPort"] || "8099";                   // (1)
  const mock = LaunchArguments.value()["mock"];
  const disable_broadcast = LaunchArguments.value()["disable_broadcast"];

  if (mock == "0") {
    setEnv("MOCK", ""); setEnv("MOCK_COUNTERVALUES", ""); Config.MOCK = "";
  }
  setEnv("DISABLE_TRANSACTION_BROADCAST", disable_broadcast != "0");

  if (ws) ws.close();

  const ipAddress = Platform.OS === "ios" ? "localhost" : "10.0.2.2";           // (2)
  const path = `${ipAddress}:${wsPort}`;
  ws = new WebSocket(`ws://${path}`);

  ws.onopen = () => { retryCount = 0; };
  ws.onerror = error => {
    if (retryCount < maxRetries) {
      retryCount++;
      const retryTimeout = retryDelay * Math.pow(2, retryCount);                // (3) exponential backoff
      setTimeout(init, retryTimeout);
    }
  };
  ws.onmessage = onMessage;
}
```

1. **`LaunchArguments.value()["wsPort"]`** — reads the `--wsPort=NNNN` launch argument that `device.launchApp({ launchArgs: { wsPort } })` set from the host side. This is how the RN app learns which port to connect to.
2. **The iOS/Android IP split** — **the single most important line in this file**. On iOS simulator, `localhost` inside the simulated device resolves to the host Mac's loopback, because the simulator runs as a macOS process sharing the same network stack. On Android emulator, the guest has its own kernel and network; `10.0.2.2` is the magic IP that the QEMU NIC maps to the host's loopback. Without this split, the bridge can't connect on Android.
3. **Exponential backoff** — the app might boot before the server is ready (race with `commonHelpers.launchApp`, which starts the server *before* calling `device.launchApp`, but still). Five retries, doubling delay each time.

**The message handler:**

```ts
// lines 76-203 (excerpt) — onMessage dispatches the right side effect
switch (msg.type) {
  case "acceptTerms":
    acceptGeneralTerms(store);
    break;
  case "importAccounts":
    store.dispatch(await importAccountsRaw({ active: msg.payload }));
    break;
  case "mockDeviceEvent":
    msg.payload.forEach(e => mockDeviceEventSubject.next(e));  // push into RxJS subject
    break;
  case "importSettings":
    store.dispatch(importSettings(msg.payload));
    break;
  case "overrideFeatureFlag": {
    const { id, value } = msg.payload as { id: FeatureId; value: Feature | undefined };
    store.dispatch(setOverride({ key: id, value }));
    break;
  }
  case "navigate":
    navigate(msg.payload, {});
    break;
  case "addUSB":
    DeviceEventEmitter.emit("onDeviceConnect", msg.payload);   // native event
    break;
  case "addKnownSpeculos": {
    const { address, model } = JSON.parse(msg.payload);
    await disconnectAllSpeculosSessions();
    // ... update known devices + set DEVICE_PROXY_URL env ...
    setEnv("DEVICE_PROXY_URL", address);
    break;
  }
  // ... many more ...
}
postMessage({ type: "ACK", id: msg.id });     // ALWAYS ACK last
```

Three side-effect shapes:

- **Redux dispatch** — `importAccounts`, `importSettings`, `overrideFeatureFlag`, etc. Modifies app state directly.
- **RxJS subject emission** — `mockDeviceEvent`. The real-app device action pipeline subscribes to `mockDeviceEventSubject`, so pushing events into it is indistinguishable from a real device sending APDU responses.
- **Native event emission** — `addUSB` uses `DeviceEventEmitter.emit("onDeviceConnect", ...)`, the exact same native event a real USB-connect would fire.

Every handler ends by posting an `ACK` back to the server with the original message id.

#### 35.7.3 `device.reverseTcpPort` — the Android-only glue

In `setup.ts`:

```ts
beforeAll(async () => {
  const port = await launchApp();
  await device.reverseTcpPort(8081);      // Metro bundler
  await device.reverseTcpPort(port);      // bridge server
  await device.reverseTcpPort(52619);     // dummy app for webview tests
  // ...
});
```

`device.reverseTcpPort(p)` tells the emulator: "when the guest connects to `localhost:p`, route that connection to the host's `localhost:p`". It is an `adb reverse tcp:p tcp:p` under the hood.

**Why only Android?**

- iOS simulator shares the host's network, so `localhost` already works. `reverseTcpPort` is a no-op on iOS (it's either missing in the iOS driver or silently skipped — we call it regardless and it's harmless).
- Android emulator has an isolated network; without `adb reverse`, `localhost` inside the emulator points at the emulator itself.
- On Android we also call `10.0.2.2` from the client as a second path (see client line 52). With `reverseTcpPort` established, both `10.0.2.2:port` and `localhost:port` work; Ledger Live standardized on `10.0.2.2`.

**Three forwarded ports in Ledger Live:**

| Port | Used for |
|------|----------|
| `8081` | Metro bundler (development-only, for `--devmenu` style reloads) |
| `port` (dynamic) | The bridge WebSocket server — port chosen by `findFreePort()` |
| `52619` | A local "dummy app" used by WebView tests |

### 35.8 `@Step` — Allure Step Decorators on POM Methods

Ledger Live uses the [Page Object Model](#pom) pattern. Each page object has methods like `tapSendButton()`, `waitForPortfolioToLoad()`. For Allure reporting, we want each method to show up as a named step with its own duration and sub-assertions.

The `jest-allure2-reporter/api` package exposes a `Step` decorator factory:

```ts
import { Step } from "jest-allure2-reporter/api";

class SendPage {
  @Step("Open the Send flow")
  async openSendFlow() {
    await tapById("portfolio-send-button");
    await waitForElementById("send-flow-screen");
  }

  @Step("Enter amount $0")
  async enterAmount(amount: string) {
    await replaceTextById("amount-input", amount);
  }
}
```

What happens at runtime:

- The decorator wraps the method in `allure.step(name, async () => originalMethod.apply(this, args))`.
- The step name supports positional parameter interpolation — `$0` = first arg, `$1` = second, etc.
- In the Allure report, the test tree shows the step hierarchy with timings and nested assertions.

**Why this needs `experimentalDecorators: true`.** Open `e2e/tsconfig.test.json`:

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "allowJs": true,
    "experimentalDecorators": true
  }
}
```

TypeScript's "legacy" decorator syntax (`@decorator`) is still behind an experimental flag. The newer stage-3 decorators have a different signature and are not what `jest-allure2-reporter` ships. Mixing the two would be a compile error. Ledger Live extends the base app `tsconfig.json` but layers this flag for the `e2e/` subtree only — tests see decorators, app code does not.

The corresponding SWC flag is in `jest.config.js`:

```js
parser: {
  syntax: "typescript",
  tsx: true,
  decorators: true,           // <- must match tsconfig
  dynamicImport: true,
},
```

Both flags must be on for the transform to emit decorator metadata.

> **In `setup.ts`**, `Step` is also exposed as a **global** so page objects can use it without importing — see `Object.defineProperty(globalThis, "Step", { value: Step, ... })`. This keeps page-object files terse.

### 35.9 `$TmsLink("B2CQA-XXX")` — Jira/Xray Linkage

Every spec and every `it` block in the Ledger Live mobile suite has one or more TMS (Test Management System) links. These are the Xray test case IDs in Jira:

```ts
// e2e/specs/deeplinks.spec.ts
import { knownDevices } from "../models/devices";

$TmsLink("B2CQA-1837");                         // (1) test-file-level link
describe("DeepLinks Tests", () => {
  beforeAll(async () => {
    await app.init({ userdata: "...", /* ... */ });
  });

  it("should open My Ledger page", async () => {
    await app.manager.openViaDeeplink();
    await app.manager.expectManagerPage();
  });
  // ...
});
```

1. **`$TmsLink("B2CQA-1837")`** — a global helper (set up in `setup.ts`) that attaches a TMS link to all subsequent tests in the file via Allure's `allure.link()` API. The Jira ticket prefix `B2CQA-` identifies the B2C QA project; the number is the Xray test case ID.

The `jest-allure2-reporter` config in `jest.config.js` (lines 1-23) turns that raw ID into a clickable URL:

```js
testCase: {
  links: {
    issue: "https://ledgerhq.atlassian.net/browse/{{name}}",
    tms:   "https://ledgerhq.atlassian.net/browse/{{name}}",
  },
  // ...
},
```

`{{name}}` is the template placeholder that the reporter replaces with the value passed to `$TmsLink`. So `$TmsLink("B2CQA-1837")` produces a clickable link to `https://ledgerhq.atlassian.net/browse/B2CQA-1837` in the Allure report.

**Why does this matter for QA workflows?**

- The Xray test case in Jira has the manual test steps written by QA.
- When the automated test runs, Xray receives the pass/fail via the Allure-Jira bridge.
- Traceability is complete: a failing test in the suite links directly back to the Xray case, which links to the product requirement.

> **In practice:** every new spec starts with `$TmsLink("B2CQA-NNNN")`, and every Xray case has a matching automation. If you add a test without a TMS link, Xray has no idea the automation exists — the case stays "manual" forever.

<div class="resource-box">
<h4>Resources</h4>
<ul>
<li><a href="https://wix.github.io/Detox/docs/config/overview">Detox: Configuration Overview</a> — every field in <code>detox.config.js</code></li>
<li><a href="https://wix.github.io/Detox/docs/guide/running-tests">Detox: Running Tests</a> — the <code>detox test</code> CLI</li>
<li><a href="https://wix.github.io/Detox/docs/articles/how-detox-works">Detox: How Detox Works</a> — idle detection deep dive</li>
<li><a href="https://wix.github.io/Detox/docs/troubleshooting/synchronization">Detox: Troubleshooting Synchronization</a> — <code>disableSynchronization</code> and the URL blacklist</li>
<li><a href="https://jestjs.io/docs/configuration">Jest: Configuration</a> — every Jest config option</li>
<li><a href="https://jestjs.io/docs/configuration#testenvironment-string">Jest: testEnvironment</a> — how to write your own</li>
<li><a href="https://github.com/wix/Detox/blob/master/docs/APIRef.Configuration.md#behavior-configuration">Detox: Behavior Configuration</a> — <code>reinstallApp</code>, <code>cleanup</code>, <code>launchApp</code></li>
<li><a href="https://www.typescriptlang.org/docs/handbook/decorators.html">TypeScript: Decorators</a> — legacy vs stage-3 decorator syntax</li>
<li><a href="https://github.com/wix/Detox/issues/3918">wix/Detox#3918</a> — the Android scroll long-press gotcha</li>
</ul>
</div>

<div class="chapter-outro">
<strong>Key takeaway:</strong> Detox's configuration lives in three files working together — <code>detox.config.js</code> defines builds and devices, <code>jest.config.js</code> drives the runner, and <code>jest.environment.ts</code> customizes per-worker lifecycle. Detox synchronizes automatically with the JS bridge, dispatch queues, timers, and network — with the URL blacklist as your escape hatch and <code>disableSynchronization()</code> as the nuclear option. On top of Detox, Ledger Live runs a WebSocket bridge that injects Redux state, feature flags, and mock device events into the running app, with every message waiting for an ACK to guarantee state has applied before the test proceeds. <code>reverseTcpPort</code>, the iOS/Android IP split, <code>@Step</code>, and <code>$TmsLink</code> are the seams where cross-platform and reporting concerns live — master them and you can read any spec in the suite.
</div>

### 35.10 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Detox Advanced</h3>
<p class="quiz-subtitle">7 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> In <code>detox.config.js</code>, what is the purpose of the <code>configurations</code> section?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It defines build commands for iOS and Android</button>
<button class="quiz-choice" data-value="B">B) It lists the test files Detox should run</button>
<button class="quiz-choice" data-value="C">C) It pairs an <code>app</code> (a build) with a <code>device</code> (a simulator/emulator) into a named target that you pass on the CLI with <code>-c</code></button>
<button class="quiz-choice" data-value="D">D) It stores environment variables for the app</button>
</div>
<p class="quiz-explanation"><code>apps</code> defines what to install, <code>devices</code> defines what to run on, and <code>configurations</code> pairs them. You run <code>detox test -c ios.sim.debug</code>, which resolves to the <code>simulator</code> device plus the <code>ios.debug</code> app.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q2.</strong> Why does Ledger Live's <code>jest.config.js</code> default <code>maxWorkers</code> to <code>1</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Each worker needs its own simulator/emulator, and the config only guarantees one device per platform by default; parallel workers are opt-in via <code>JEST_MAX_WORKERS</code> on CI where additional devices are pre-booted</button>
<button class="quiz-choice" data-value="B">B) Jest does not support parallel execution</button>
<button class="quiz-choice" data-value="C">C) Detox is single-threaded by nature</button>
<button class="quiz-choice" data-value="D">D) Parallel E2E tests are forbidden by the test policy</button>
</div>
<p class="quiz-explanation">Mobile E2E parallelism requires one device per worker. The config exposes <code>simulator</code>, <code>simulator2</code>, <code>simulator3</code> and matching emulators; the custom <code>jest.environment.ts</code> swaps the device based on <code>JEST_WORKER_ID</code>. But booting three emulators locally is expensive, so the default is 1.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> What does Detox wait on before performing each action (idle detection)?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Only network requests</button>
<button class="quiz-choice" data-value="B">B) Only the React Native JS thread</button>
<button class="quiz-choice" data-value="C">C) Only UI animations</button>
<button class="quiz-choice" data-value="D">D) All of: JS thread, native dispatch queues, animation frame pump, active timers, in-flight non-blacklisted network requests, and the RN bridge</button>
</div>
<p class="quiz-explanation">This is why Detox is called grey-box. It synchronizes against the app's whole runtime — not just the DOM-equivalent. When any of these queues is busy, the action waits.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q4.</strong> What is the <code>detoxURLBlacklistRegex</code> and why does Ledger Live set it?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A list of URLs the app is forbidden from calling</button>
<button class="quiz-choice" data-value="B">B) A list of URL regexes that Detox will NOT treat as in-flight network requests for synchronization — used to exclude long-polling or always-on endpoints (analytics, Google APIs) that would otherwise keep Detox waiting forever</button>
<button class="quiz-choice" data-value="C">C) A list of URLs that Jest will intercept and mock</button>
<button class="quiz-choice" data-value="D">D) A CSP (Content Security Policy) for the RN WebView</button>
</div>
<p class="quiz-explanation">Detox's network synchronization waits for all in-flight requests to complete before acting. Analytics beacons and Google connectivity checks never complete in the test's lifetime, so they must be excluded or every test would hang.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q5.</strong> In the WebSocket bridge, why does every server-to-client message wait for an ACK from the client?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) WebSockets don't guarantee delivery without ACKs</button>
<button class="quiz-choice" data-value="B">B) To encrypt the message in transit</button>
<button class="quiz-choice" data-value="C">C) Without an ACK, <code>await setFeatureFlag(...)</code> would resolve as soon as the message was sent — before the client actually ran the Redux dispatch. The ACK is sent only after the client handler completes, so the next test line only runs once the state change is live</button>
<button class="quiz-choice" data-value="D">D) To measure network latency</button>
</div>
<p class="quiz-explanation">The ACK is a handler-completion signal. Without it, tests would race against the async state updates they just requested. See <code>bridge/client.ts</code>: every case in the switch statement ends with <code>postMessage({ type: "ACK", id: msg.id })</code>.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q6.</strong> Why does <code>bridge/client.ts</code> use <code>localhost</code> on iOS but <code>10.0.2.2</code> on Android?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) iOS simulator shares the host macOS network stack so <code>localhost</code> resolves to the host; Android emulator has an isolated guest network and <code>10.0.2.2</code> is the QEMU-defined alias for the host's loopback</button>
<button class="quiz-choice" data-value="B">B) <code>10.0.2.2</code> is Android's public DNS server</button>
<button class="quiz-choice" data-value="C">C) iOS blocks <code>10.0.2.2</code> at the firewall</button>
<button class="quiz-choice" data-value="D">D) Historical reasons; both should resolve identically today</button>
</div>
<p class="quiz-explanation"><code>10.0.2.2</code> is a well-known Android emulator alias documented by Google. The bridge must use the correct IP for each platform; <code>reverseTcpPort</code> additionally enables <code>localhost</code> on Android, but the client is coded defensively to use the canonical alias.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q7.</strong> What does the <code>@Step("Enter amount $0")</code> decorator do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Pauses the test at that line, like a debugger breakpoint</button>
<button class="quiz-choice" data-value="B">B) Wraps the decorated method in an <code>allure.step(name, fn)</code> call so the method appears as a named step in the Allure report, with <code>$0</code>/<code>$1</code> substituted by the method's arguments. Requires <code>experimentalDecorators: true</code> in tsconfig and <code>decorators: true</code> in SWC</button>
<button class="quiz-choice" data-value="C">C) Marks the step as a blocking checkpoint in CI</button>
<button class="quiz-choice" data-value="D">D) Registers the step with Jira/Xray as a new test case</button>
</div>
<p class="quiz-explanation"><code>@Step</code> is a reporting decorator from <code>jest-allure2-reporter/api</code>. It's a zero-runtime-overhead way to make page-object methods show up as hierarchical steps in Allure. The positional placeholders interpolate the method's arguments into the step name.</p>
</div>

<div class="quiz-score"></div>
</div>
## Mobile Codebase Deep Dive: Every File Explained

<div class="chapter-intro">
This is your reference chapter for the mobile side. It catalogues every file under <code>apps/ledger-live-mobile/e2e/</code>, shows the important ones verbatim, and explains how each piece connects to the rest of the suite. Keep this chapter open when you land in an unfamiliar mobile test file — it will tell you what the file does, why it exists, and where to look next. It is the mobile counterpart of Part 5 Chapter 27.
</div>

### 36.1 The `e2e/` Directory Tree

Unlike desktop — where tests live under `apps/ledger-live-desktop/tests/` — the mobile suite is rooted at `apps/ledger-live-mobile/e2e/`. Every file in this chapter is relative to that root. Here is the full tree:

```
apps/ledger-live-mobile/e2e/
├── babel.config.detox.js            # Babel config used by the Detox runner (decorators + module mapping)
├── jest.config.js                   # Jest runner config (Detox adapter + Allure reporter)
├── jest.environment.ts              # Custom Jest environment (extends DetoxEnvironment)
├── jest.globalSetup.ts              # Boots the Detox global setup once per run
├── setup.ts                         # Per-file setup: globals, beforeAll/afterAll, bridge init
├── tsconfig.test.json               # TS config for the test tree (decorators + path aliases)
│
├── bridge/                          # WebSocket bridge between the test process and the app
│   ├── server.ts                    # (303 lines) test-side server: open/close, loadConfig, mockDeviceEvent, ...
│   ├── client.ts                    # (248 lines) app-side client: receives messages, dispatches into Redux
│   ├── types.ts                     # (128 lines) ServerData / MessageData discriminated unions
│   └── start-server.ts              # (43 lines) CLI helper: `pnpm mobile e2e:loadConfig <name>`
│
├── helpers/                         # Low-level UI helpers, shared by every POM
│   ├── commonHelpers.ts             # delay, openDeeplink, isAndroid, launchApp, setupEnvironment, logMemoryUsage
│   └── elementHelpers.ts            # ElementHelpers object: 28 Detox primitives (tapById, waitForElementById, ...)
│
├── mocks/
│   └── react-native-blur.js         # Jest module mock so BlurView does not crash in Detox
│
├── models/                          # Domain models used by POMs and specs
│   ├── DeviceAction.ts              # Mock device action controller (selectMockDevice, openApp, silentSign, ...)
│   ├── devices.ts                   # knownDevices = { nanoX, nanoSP, nanoS, stax, europa } + USB descriptors
│   ├── currencies.ts                # initTestAccounts, formattedAmount, account helpers
│   └── liveApps.ts                  # startLiveApp / stopServer for the dummy wallet-app
│
├── page/                            # Page Object Model layer
│   ├── index.ts                     # Application class — hub with lazy-init getters
│   ├── common.page.ts               # Shared UI: search, close, selectAccount, addDeviceViaBluetooth/USB
│   ├── passwordEntry.page.ts        # Simplest POM — password lock screen
│   ├── accounts/                    # accounts.page, addAccount.drawer, assetAccounts.page
│   ├── discover/                    # discover.page (Discover tab)
│   ├── drawer/                      # modular.drawer
│   ├── liveApps/                    # cryptoDrawer, dummyWalletApp.webView, walletAPIReceive, walletAPISignMessage
│   ├── manager/                     # manager.page
│   ├── market/                      # market.page
│   ├── onboarding/                  # onboardingSteps.page
│   ├── postOnboarding/              # postOnboarding.page
│   ├── settings/                    # settings.page, settingsGeneral.page
│   ├── stax/                        # customLockscreen.page
│   ├── trade/                       # send.page, receive.page, swap.page, stake.page, operationDetails.page
│   └── wallet/                      # portfolio.page, walletTabNavigator.page, transferMenu.drawer
│
├── specs/                           # Test specs — the files Jest/Detox actually runs
│   ├── deeplinks.spec.ts
│   ├── languageChange.spec.ts
│   ├── manager.spec.ts
│   ├── market.spec.ts
│   ├── onboarding.spec.ts
│   ├── onboardingReadOnly.spec.ts
│   ├── password.spec.ts
│   ├── unknown-currency-resilience.spec.ts
│   ├── wallet-api.spec.ts
│   ├── addAccounts/
│   ├── delegate/
│   ├── receive/
│   ├── send/
│   └── swap/
│       └── dexSwap.spec.ts
│
├── userdata/                        # Redux state snapshots loaded via the bridge
│   ├── skip-onboarding.json                       # Onboarding complete, no accounts
│   ├── 1accountEth.json                           # 1 ETH account
│   ├── 1Account1NFTReadOnlyFalse.json             # 1 account with an NFT
│   ├── 1AccountBTC1AccountETHReadOnlyFalse.json   # 1 BTC + 1 ETH account
│   └── EthAccountXrpAccountReadOnlyFalse.json     # ETH + XRP accounts
│
└── artifacts/                       # Test run output (created at runtime, gitignored)
    ├── screenshots/                 # PNGs attached to Allure on failure
    ├── logs/                        # Detox / Metro / device logs
    └── allure-results/              # Raw Allure JSON consumed by `allure generate`
```

The three load-bearing folders are `bridge/`, `helpers/`, and `page/`. Everything else (models, specs, userdata) either feeds them or is driven by them.

### 36.2 `setup.ts` — The Per-File Bootstrap

`setup.ts` is wired in through `jest.config.js` as `setupFilesAfterEnv`, which means Jest runs it **once per spec file**, after the test environment is ready but before `describe` blocks execute. This is where the whole test world is assembled. Read it in full:

```typescript
/* eslint-disable no-var */
import { registerAllCoins } from "@ledgerhq/live-common/coin-modules/load-all-coins";
import { LiveConfig } from "@ledgerhq/live-config/LiveConfig";
import { liveConfig } from "@ledgerhq/live-common/config/sharedConfig";
import { device } from "detox";
import { getLogs, close as closeBridge } from "./bridge/server";
import { launchApp, setupEnvironment } from "./helpers/commonHelpers";
import { Step } from "jest-allure2-reporter/api";
import { ServerData } from "./bridge/types";
import { Subject } from "rxjs";
import { Currency } from "@ledgerhq/live-common/e2e/enum/Currency";
import { Delegate } from "@ledgerhq/live-common/e2e/models/Delegate";
import { Account } from "@ledgerhq/live-common/e2e/enum/Account";
import { Transaction } from "@ledgerhq/live-common/e2e/models/Transaction";
import { Fee } from "@ledgerhq/live-common/e2e/enum/Fee";
import { AppInfos } from "@ledgerhq/live-common/e2e/enum/AppInfos";
import { Swap } from "@ledgerhq/live-common/e2e/models/Swap";
import { ElementHelpers } from "./helpers/elementHelpers";
import expect from "expect";
import { Application } from "./page";
```

What the imports reveal:

- `registerAllCoins`, `LiveConfig`, `liveConfig` — the same coin-module boot sequence the production app runs, so every coin is known to live-common before any test line executes.
- `device` from `detox` — the Detox device proxy (launch, reverseTcpPort, openURL, ...).
- `getLogs`, `closeBridge` from `./bridge/server` — used in `afterAll` for the failure log attachment.
- `launchApp`, `setupEnvironment` from `./helpers/commonHelpers` — see 36.6.
- `Step` from `jest-allure2-reporter/api` — the decorator factory used in every POM method annotated `@Step(...)`.
- `Currency`, `Account`, `Delegate`, `Transaction`, `Fee`, `AppInfos`, `Swap` — shared enums/models from `@ledgerhq/live-common/e2e/`. Same ones used by desktop (see Part 5 Ch 27.9), so the two suites stay consistent.
- `ElementHelpers` — the 28-method primitive catalogue (see 36.6).
- `Application` — the POM hub (see 36.8).

```typescript
Object.defineProperty(globalThis, "Step", {
  value: Step,
  writable: true,
  configurable: true,
  enumerable: true,
});
```

`Step` is installed on `globalThis` so that decorators like `@Step("Tap on X")` inside POM files can resolve without importing it in every file.

```typescript
type StepType = typeof Step;
type expectType = typeof expect;
// ...
declare global {
  var IS_FAILED: boolean;
  var app: Application;
  var Step: StepType;
  var jestExpect: expectType;
  var Currency: CurrencyType;
  var Delegate: DelegateType;
  var Account: AccountType;
  var Transaction: TransactionType;
  var Fee: FeeType;
  var AppInfos: AppInfosType;
  var Swap: SwapType;

  var waitForElementById: typeof ElementHelpers.waitForElementById;
  // ... 25+ more ElementHelpers promoted to global
}
```

This is the TypeScript side of the global namespace. Spec files like `onboarding.spec.ts` can write `await app.onboarding.startOnboarding()` and `await tapById("connect-with-bluetooth")` without importing anything — because everything is on `globalThis`.

```typescript
registerAllCoins();
LiveConfig.setConfig(liveConfig);
setupEnvironment();

global.IS_FAILED = false;
global.app = new Application();
global.webSocket = {
  wss: undefined,
  ws: undefined,
  messages: {},
  e2eBridgeServer: new Subject<ServerData>(),
};
global.jestExpect = expect;
global.Currency = Currency;
global.Delegate = Delegate;
global.Account = Account;
// ... etc
global.waitForElementById = ElementHelpers.waitForElementById;
global.tapById = ElementHelpers.tapById;
// ... 25+ more
```

- `global.IS_FAILED = false` — reset at the top of each file, flipped to `true` by `jest.environment.ts` on any hook/test failure. Used in `afterAll` to decide whether to attach app logs.
- `global.app = new Application()` — the single hub instance. Note: this is per file; nothing persists between spec files.
- `global.webSocket = { wss, ws, messages, e2eBridgeServer }` — the bridge plumbing. `e2eBridgeServer` is an RxJS Subject that the app pushes into and that helpers like `await app.dummyWalletApp.getResOutput()` subscribe to.

```typescript
beforeAll(
  async () => {
    const port = await launchApp();
    await device.reverseTcpPort(8081);
    await device.reverseTcpPort(port);
    await device.reverseTcpPort(52619); // To allow the android emulator to access the dummy app
    const testFileName = expect.getState().testPath?.replace(/^.*\/(.+?)(?:\.spec)?\.[^.]+$/, "$1");
    allure.description("Test file : " + testFileName);
  },
  process.env.CI ? 150000 : 300000,
);
```

Three `reverseTcpPort` calls bind the host's ports onto the emulator's `localhost`, which is how Android reaches services running on the developer machine:

- `8081` — Metro bundler (required for JS bundle + hot reload in debug builds).
- `port` — the bridge WS port chosen by `findFreePort()` inside `launchApp()`. Matches what the app reads from `launchArgs.wsPort`.
- `52619` — the dummy Wallet-API Live App started by `app.dummyWalletApp.startApp()` (see 36.7 and `wallet-api.spec.ts`).

The timeout is 150s on CI, 300s locally — the extra budget locally covers cold-start of a fresh emulator.

```typescript
afterAll(async () => {
  if (IS_FAILED && process.env.CI) {
    await allure.attachment("App logs", await getLogs(), "application/json");
  }
  closeBridge();
});
```

Only on CI, only when a test failed, we attach the app logs to the Allure report. Always, we close the WS bridge so the next spec file starts clean.

There is no explicit `beforeEach` / `afterEach` in `setup.ts` — spec files add their own when they need them (see `onboarding.spec.ts` in 36.9).

### 36.3 `jest.config.js`, `jest.environment.ts`, `jest.globalSetup.ts`

Chapter 35 already covered the Jest-Detox adapter model. Here are the three files as they actually are in the mobile repo, with the Ledger-Live-specific notes.

#### `jest.config.js`

```javascript
/** @type {import('jest-allure2-reporter').ReporterOptions} */
const jestAllure2ReporterOptions = {
  resultsDir: "artifacts",
  testCase: {
    links: {
      issue: "https://ledgerhq.atlassian.net/browse/{{name}}",
      tms: "https://ledgerhq.atlassian.net/browse/{{name}}",
    },
    labels: {
      host: process.env.RUNNER_NAME,
    },
  },
  overwrite: false,
  environment: async ({ $ }) => {
    return {
      path: process.cwd(),
      "version.node": process.version,
      "version.jest": await $.manifest("jest", ["version"]),
      "package.name": await $.manifest(m => m.name),
      "package.version": await $.manifest(["version"]),
    };
  },
};

const transformIncludePatterns = ["ky", "@mysten+ledgerjs-hw-app-sui"];

const detoxAllure2AdapterOptions = {
  deviceLogs: false,
  deviceScreenshots: true,
  deviceVideos: false,
  deviceViewHierarchy: false,
  onError: "warn",
};

module.exports = async () => ({
  rootDir: "..",
  maxWorkers: process.env.JEST_MAX_WORKERS ? parseInt(process.env.JEST_MAX_WORKERS, 10) : 1,
  transform: {
    "^.+\\.m?jsx?$": "babel-jest",
    "^.+\\.(ts|tsx)?$": [
      "@swc/jest",
      {
        jsc: {
          target: "es2022",
          parser: { syntax: "typescript", tsx: true, decorators: true, dynamicImport: true },
          transform: { react: { runtime: "automatic" } },
        },
        sourceMaps: "inline",
        module: { type: "commonjs" },
      },
    ],
  },
  setupFilesAfterEnv: ["<rootDir>/e2e/setup.ts"],
  testTimeout: 150000,
  testMatch: ["<rootDir>/e2e/specs/**/*.spec.ts"],
  reporters: ["detox/runners/jest/reporter", ["jest-allure2-reporter", jestAllure2ReporterOptions]],
  globalSetup: "<rootDir>/e2e/jest.globalSetup.ts",
  testEnvironment: "<rootDir>/e2e/jest.environment.ts",
  testEnvironmentOptions: {
    eventListeners: [
      "jest-metadata/environment-listener",
      "jest-allure2-reporter/environment-listener",
      ["detox-allure2-adapter", detoxAllure2AdapterOptions],
    ],
  },
  transformIgnorePatterns: [`node_modules/.pnpm/(?!(${transformIncludePatterns.join("|")}))`],
  verbose: true,
  resetModules: true,
});
```

Ledger-Live-specific choices worth noting:

- **`resetModules: true`** — Jest wipes the module registry between every test. Without this, the `loadConfig` userdata sent to `app.init(...)` would leak across tests because the in-memory Redux store module would be the same instance. Combined with `setupFilesAfterEdv: ["<rootDir>/e2e/setup.ts"]`, this means every test gets a fresh `app = new Application()` and a fresh bridge.
- **Reporter stack** — `["detox/runners/jest/reporter", ["jest-allure2-reporter", ...]]`. Detox's reporter prints progress to the console; `jest-allure2-reporter` writes Allure results under `artifacts/`. The TMS links are pre-wired to `ledgerhq.atlassian.net/browse/{{name}}`, which is how `$TmsLink("B2CQA-1803")` in a spec turns into a clickable Jira link in the Allure HTML.
- **`detox-allure2-adapter`** — bridges Detox events into the Allure reporter. `deviceScreenshots: true` means every failure captures a screenshot; `deviceVideos: false` keeps artifact size manageable.
- **`transformIgnorePatterns`** — normally `node_modules` is not transformed, but a few ESM-only packages (`ky`, `@mysten+ledgerjs-hw-app-sui`) are allow-listed.
- **`maxWorkers`** — defaults to `1`. Parallelism is driven by `JEST_MAX_WORKERS` on CI; `jest.environment.ts` handles device suffixing for secondary workers.

#### `jest.environment.ts`

```typescript
import { config as detoxConfig } from "detox/internals";
// @ts-expect-error detox doesn't provide type declarations for this module
import DetoxEnvironment from "detox/runners/jest/testEnvironment";

import { logMemoryUsage } from "./helpers/commonHelpers";

export default class TestEnvironment extends DetoxEnvironment {
  declare global: typeof globalThis;

  async setup() {
    const workerId = Number(process.env.JEST_WORKER_ID ?? 1);
    if (workerId > 1) await this.setupDeviceForSecondaryWorker(workerId);
    await super.setup();
  }

  private async setupDeviceForSecondaryWorker(workerId: number) {
    const configName = process.env.DETOX_CONFIGURATION;
    if (!configName) throw new Error("Missing DETOX_CONFIGURATION environment variable");

    const detoxConfigModule = await import("../detox.config.js");
    const detoxConfigFile =
      "default" in detoxConfigModule ? detoxConfigModule.default : detoxConfigModule;
    const targetConfig = detoxConfigFile.configurations[configName];
    if (!targetConfig || !targetConfig.device) {
      throw new Error(
        `Detox configuration "${configName}" not found or missing device, check your detox config file`,
      );
    }

    const deviceName = `${targetConfig.device}${workerId}`;
    const targetDevice = detoxConfigFile.devices[deviceName];
    if (!targetDevice) {
      throw new Error(`Device configuration not found for ${deviceName}, check your detox config file`);
    }

    Object.assign(detoxConfig, { device: targetDevice });
  }

  async handleTestEvent(event: any, state: any) {
    await super.handleTestEvent(event, state);

    if (["hook_failure", "test_fn_failure"].includes(event.name)) {
      this.global.IS_FAILED = true;
    }
    if (event.name === "run_start") {
      await logMemoryUsage();
    }
  }
}
```

Two Ledger-Live specifics:

- **Secondary workers pick a numbered device.** When `JEST_MAX_WORKERS=4`, worker 1 uses the default device, workers 2-4 get `<deviceName>2`, `<deviceName>3`, `<deviceName>4`. The `detox.config.js` must declare those numbered devices up front (e.g., `iPhone15_2`, `iPhone15_3`).
- **`IS_FAILED` flag.** This is what flips the global set in `setup.ts`. Later `afterAll` reads it to decide whether to attach app logs.

#### `jest.globalSetup.ts`

```typescript
import { globalSetup } from "detox/runners/jest";

export default async () => {
  await globalSetup();
};
```

Three lines. All it does is call Detox's own `globalSetup` — which builds/installs the app if needed and warms up the runner. Per-file setup still runs `setup.ts`.

### 36.4 `tsconfig.test.json` — Path Aliases for Tests

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "allowJs": true,
    "experimentalDecorators": true
  }
}
```

Four keys that do a lot:

- **`extends: "../tsconfig.json"`** inherits the app's path aliases. `~/*` maps to `apps/ledger-live-mobile/src/*`, `LLM/*` is the same thing with a different prefix. This is why `bridge/server.ts` can write `import { BleState, DeviceLike } from "../../src/reducers/types";` or a POM can do `import { DeviceLike } from "~/reducers/types";` — same target, just shorter.
- **`allowJs: true`** lets TypeScript include `babel.config.detox.js`, `jest.config.js`, and the `.js` mock in `mocks/` without conversion.
- **`experimentalDecorators: true`** is the switch that makes `@Step("...")` compile. Without it, every POM method that uses the decorator would be a compile error.

The `include`/`exclude` live in the parent `tsconfig.json`, not here; this file only adjusts the two flags that differ for the test tree.

### 36.5 The `bridge/` Layer

The bridge is the single component that separates the mobile suite from a pure Detox test. Chapter 35 walked through the full message catalogue and sequence; here we tour the four files and pin down what is in each.

| File | Lines | Runs on | Role |
|------|-------|---------|------|
| `bridge/server.ts` | 303 | Test process (Node) | Hosts the WS server; exposes `init`, `close`, `open`, `loadConfig`, `loadAccounts`, `loadBleState`, `setFeatureFlags`, `addDevicesBT`, `addDevicesUSB`, `mockDeviceEvent`, `getLogs`, `findFreePort` |
| `bridge/client.ts` | 248 | App process (React Native) | Connects to the WS URL given by `launchArgs.wsPort`, receives `MessageData` frames, dispatches them into the Redux store and the device-action event stream |
| `bridge/types.ts` | 128 | Both | `ServerData` and `MessageData` discriminated unions — the wire format |
| `bridge/start-server.ts` | 43 | Dev CLI only | Standalone server used by `pnpm mobile e2e:loadConfig <name>` to push a userdata file into a running debug build |

**`server.ts`** exports the API specs call via `app.init({ userdata, ... })` and via `addDevicesBT(...)`, `mockDeviceEvent(...)`, etc. Its `findFreePort()` opens and closes a socket on port 0, letting the OS assign a port, which is why the WS port is different every run. `init(port, onConnection)` then spawns a `WebSocket.Server` on that port and wires the connection/message handlers.

**`client.ts`** lives in the app bundle and is gated by `IS_TEST=true`. On connection it reads `launchArgs.wsPort`, opens a `new WebSocket(...)`, and dispatches every received `MessageData` into the Redux store (`loadConfig` → replaces the state, `loadAccounts` → hydrates accounts, `mockDeviceEvent` → pushed into the device-action event `Subject`).

**`types.ts`** is where you look when a message name surprises you. `ServerData` is what the app sends back to the tests (`walletAPIResponse`, `appLogs`, `appFile`, `appFlags`, `appEnvs`, `ACK`, `swapSetupDone`, `swapLiveAppReady`, `earnLiveAppReady`). `MessageData` is the larger union that covers every device-action event type the app can replay.

**`start-server.ts`** is not part of the Detox run — it is a dev-loop tool. You start Metro, run `pnpm mobile e2e:loadConfig 1AccountBTC1AccountETHReadOnlyFalse`, and then launch the app with `MOCK=1`. The app connects to the bridge, receives the userdata, and you can iterate on UI code against a known state without writing a test.

For the full message catalogue, refer back to Chapter 35.

### 36.6 The `helpers/` Layer

Two files, both used on every line of every spec.

#### `commonHelpers.ts` (verbatim)

```typescript
import { findFreePort, close as closeBridge, init as initBridge } from "../bridge/server";
import { getEnv, setEnv } from "@ledgerhq/live-env";
import { exec } from "child_process";
import { device, log } from "detox";
import { allure } from "jest-allure2-reporter/api";

const BASE_DEEPLINK = "ledgerlive://";

export const currencyParam = "?currency=";

/**
 * Waits for a specified amount of time
 * /!\ Do not use it to wait for a specific element, use waitFor instead.
 * @param {number} ms
 */
export async function delay(ms: number) {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve("delay complete");
    }, ms);
  });
}

/** @param path the part after "ledgerlive://", e.g. "portfolio", or "discover?param=123"  */
export async function openDeeplink(path?: string) {
  await device.openURL({ url: BASE_DEEPLINK + path });
}

export function isAndroid() {
  return device.getPlatform() === "android";
}

export async function launchApp() {
  const port = await findFreePort();
  closeBridge();
  initBridge(port);
  await device.launchApp({
    launchArgs: {
      wsPort: port,
      detoxURLBlacklistRegex:
        '\\(".*sdk.*.braze.*",".*.googleapis.com/.*",".*clients3.google.com.*",".*tron.coin.ledger.com/wallet/getBrokerage.*"\\)',
      mock: getEnv("MOCK") ? getEnv("MOCK") : "1",
      IS_TEST: true,
    },
    languageAndLocale: {
      language: "en-US",
      locale: "en-US",
    },
    permissions: {
      camera: "YES", // Give iOS permissions for the camera
    },
  });
  return port;
}

export function setupEnvironment() {
  setEnv("DISABLE_APP_VERSION_REQUIREMENTS", true);
  setEnv("MOCK", "1");
}

export const logMemoryUsage = async () => {
  const pid = process.pid;
  const isLinux = process.platform !== "darwin";
  exec(
    `top ${isLinux ? "-b -n 1 -p" : "-l 1 -pid"} ${pid} | grep "${pid}" | awk '{print ${isLinux ? "$6" : "$8"}}'`,
    async (error, stdout, stderr) => {
      if (error || stderr) {
        console.error(`Error getting memory usage:\n Error: ${error}\n Stderr: ${stderr}`);
        return;
      }
      const logMessage = `📦 Detox Memory Usage: ${stdout.trim()}`;
      await allure.attachment("Memory Usage Details", logMessage, "text/plain");
      log.warn(logMessage);
    },
  );
};
```

Notes:

- **`delay(ms)`** — the explicit warning in the JSDoc says it all. If you find yourself reaching for `delay` to wait for an element, use `waitForElementById` or Detox's `waitFor(...).withTimeout(...)` instead. `delay` exists only for rare timing issues (e.g., animation tail after a scroll).
- **`openDeeplink(path)`** — the mobile equivalent of clicking a URL. `ledgerlive://send`, `ledgerlive://discover?param=123`, etc. Every screen has a deeplink; using them skips navigation and is the fastest way to a specific screen.
- **`isAndroid()`** — Detox's `device.getPlatform()` returns `ios` or `android`. Most platform-specific branches in POMs use this.
- **`launchApp()`** — the centerpiece. It finds a free port, resets the bridge, then calls `device.launchApp({ launchArgs: { wsPort, mock: "1", IS_TEST: true, detoxURLBlacklistRegex } })`. The regex blacklists sdk/braze/googleapis endpoints so the idle/sync pollers in Detox don't hang on them. Returns the port, which `setup.ts` then passes to `device.reverseTcpPort(port)`.
- **`setupEnvironment()`** — sets `DISABLE_APP_VERSION_REQUIREMENTS` and `MOCK=1` before anything else. Called both from `setup.ts` and from `Application` constructor logic — running it twice is harmless.
- **`logMemoryUsage()`** — emits Detox's own memory footprint into Allure at the start of every run. Cross-platform `top` invocation.

#### `elementHelpers.ts` — The 28 Primitives

This is the file that gives specs their global vocabulary. Every exported method on `ElementHelpers` is promoted to `global.*` in `setup.ts`. Here is the full table:

| Method | Signature | Purpose |
|--------|-----------|---------|
| `waitForElementById` | `(id, timeout=60000)` | Wait for a native element by testID to be visible |
| `waitForElementByText` | `(text, timeout=60000)` | Wait by visible text |
| `getElementsById` | `(id)` | Get the raw matcher (no index) |
| `getElementById` | `(id, index=0)` | Get element at index |
| `getElementByText` | `(text, index=0)` | Get by text at index |
| `getWebElementById` | `(id, index=0)` | Get a web element (webview) by id |
| `getWebElementByTag` | `(tag, index=0)` | Get a web element by HTML tag |
| `getWebElementByTestId` | `(id)` | Get a web element by `data-testid` |
| `getWebElementsByIdAndText` | `(id, text)` | XPath combining testid and text |
| `getWebElementText` | `(id, index=0)` | Read text from a web element |
| `getWebElementValue` | `(id, index=0)` | Read `value` from a web input |
| `waitWebElementByTestId` | `(id, timeout=60000)` | Poll a web element until present |
| `tapWebElementByTestId` | `(id, index=0)` | Tap a web element by testid |
| `typeTextByWebTestId` | `(id, text)` | Type into a web input via DOM setter |
| `IsIdVisible` | `(id) => Promise<boolean>` | 1-second soft check — never throws |
| `tapById` | `(id, index=0)` | Tap native element by id |
| `tapByText` | `(text, index=0)` | Tap by visible text |
| `tapByElement` | `(elem)` | Tap an already-resolved element |
| `typeTextById` | `(id, text, closeKeyboard=true, focus=true)` | Type text into input by id |
| `typeTextByElement` | `(elem, text, closeKeyboard, focus)` | Type into a resolved element |
| `clearTextByElement` | `(elem)` | Clear a text field |
| `performScroll` | `(matcher, scrollViewId?, pixels=300, direction="down")` | Scroll until visible |
| `scrollToText` | `(text, scrollViewId?, pixels?, direction?)` | Scroll helper by text |
| `scrollToId` | `(id, scrollViewId?, pixels?, direction?)` | Scroll helper by id |
| `getTextOfElement` | `(id, index=0)` | Read text attribute |
| `getIdOfElement` | `(element, index=0)` | Read identifier attribute |
| `getIdByRegexp` | `(regex, index=0)` | Resolve a regex match to its concrete id |

The five most-used methods, verbatim:

```typescript
waitForElementById(id: string | RegExp, timeout: number = DEFAULT_TIMEOUT) {
  return waitFor(element(by.id(id)))
    .toBeVisible()
    .withTimeout(timeout);
},

async tapById(id: string | RegExp, index = 0) {
  return await ElementHelpers.getElementById(id, index).tap();
},

async IsIdVisible(id: string | RegExp) {
  try {
    await waitFor(element(by.id(id)))
      .toBeVisible()
      .withTimeout(1000);
    return true;
  } catch {
    return false;
  }
},

async typeTextByElement(
  elem: Detox.NativeElement,
  text: string,
  closeKeyboard = true,
  focus = true,
) {
  if (focus) await ElementHelpers.tapByElement(elem);
  await elem.replaceText(text);
  if (closeKeyboard) await elem.typeText("\n"); // ' \n' close keyboard if open
},

async performScroll(
  elementMatcher: Detox.NativeMatcher,
  scrollViewId?: string | RegExp,
  pixels = 300,
  direction: Direction = "down",
) {
  const scrollViewMatcher = scrollViewId
    ? by.id(scrollViewId)
    : by.type(isAndroid() ? "android.widget.ScrollView" : "RCTEnhancedScrollView");
  await waitFor(element(elementMatcher))
    .toBeVisible()
    .whileElement(scrollViewMatcher)
    .scroll(pixels, direction, NaN, startPositionY);
  if (isAndroid()) await delay(30); // Issue on tap after scroll on Android : https://github.com/wix/Detox/issues/3637
},
```

Three details the desktop folks will appreciate:

- `DEFAULT_TIMEOUT` is **60 seconds** — much more than desktop. Emulators are slow and flaky, and React Native's navigation animations often take 200-500ms.
- `IsIdVisible` has a hardcoded 1-second timeout so it can be used as a cheap predicate. That is why POMs like `CommonPage.openApp()` do `if (await IsIdVisible(rememberedDeviceRowId)) { ... }` without slowing the test down.
- `performScroll` uses `RCTEnhancedScrollView` on iOS and `android.widget.ScrollView` on Android. The `startPositionY = 0.8` works around a known Detox scroll bug ([wix/Detox#3918](https://github.com/wix/Detox/issues/3918)).

### 36.7 The `models/` Layer

Four files, each a small domain model used by POMs and specs.

#### `models/DeviceAction.ts` (231 lines)

The class that fakes a hardware device. Every time a spec needs the app to "see" a device event — plugged in, app opened, genuine check OK, signature returned — it calls a method on this class, and the method emits a `mockDeviceEvent` over the bridge.

Class skeleton:

```typescript
export default class DeviceAction {
  device: Device;

  constructor(input: DeviceLike | Device) { ... }

  async selectMockDevice() {
    await addDevicesBT(this.deviceToDeviceLike(this.device));
    const elId = "device-item-" + this.device.deviceId;
    await waitForElementById(elId);
    await tapById(elId);
  }

  async waitForSpinner() { await waitForElementById("device-action-loading"); }

  async openApp() {
    await addDevicesBT(this.deviceToDeviceLike(this.device));
    const rememberedDeviceRowId = `device-item-${this.device.deviceId}`;
    const scannedDeviceRowId = `device-scanned-${this.device.deviceId}`;
    if (await IsIdVisible(rememberedDeviceRowId)) await tapById(rememberedDeviceRowId);
    else if (await IsIdVisible(scannedDeviceRowId)) await tapById(scannedDeviceRowId);
    await this.waitForSpinner();
    await mockDeviceEvent({ type: "opened" });
  }

  async genuineCheck(appDesc = "Bitcoin", installedDesc = "Bitcoin") { ... }
  async accessManager(appDesc, installedDesc) { ... }
  async accessManagerWithL10n(...) { ... }
  async complete() { await mockDeviceEvent({ type: "complete" }); }
  async initiateLanguageInstallation() { ... }
  async add50ProgressToLanguageInstallation() { ... }
  async installSetOfAppsMocked(progress, itemProgress, currentAppOp, installQueue) { ... }
  async resolveDependenciesMocked(installQueue) { ... }
  async completeLanguageInstallation() { ... }
  async requestImageLoad() { ... }
  async loadImageWithProgress(progress: number) { ... }
  async requestImageCommit() { ... }
  async confirmImageLoaded(imageSize: number, imageHash: string) { ... }

  async silentSign() {
    await this.waitForSpinner();
    await mockDeviceEvent({ type: "opened" }, { type: "complete" });
  }
}
```

Key methods in plain English:

- **`selectMockDevice()`** — emits a Bluetooth scan hit with this device's id, then taps the resulting row. Used in setup code before any action that needs a device.
- **`openApp()`** — the "device is plugged in, app is open" moment. Handles both the remembered-device path (one-tap reconnect) and the fresh-scan path.
- **`genuineCheck()` / `accessManager()`** — emit the `deviceChange → listingApps → result → complete` sequence that live-common expects. `mockListAppsResult(appDesc, installedDesc, deviceInfo)` builds the payload.
- **`silentSign()`** — the simplest sign. Emits `opened` then `complete`. Used everywhere `deviceAction.silentSign()` appears in a spec (e.g., `wallet-api.spec.ts` message.sign test).
- **Custom-lockscreen and language-install methods** — granular progress events used by the Stax/Europa tests.

#### `models/devices.ts` (verbatim)

```typescript
export const knownDevices = {
  nanoX: { name: "Nano X", id: "mock_1", modelId: DeviceModelId.nanoX },
  nanoSP: { name: "Nano SP", id: "mock_2", modelId: DeviceModelId.nanoSP },
  nanoS: { name: "Nano S", id: "mock_3", modelId: DeviceModelId.nanoS },
  stax: { name: "Stax", id: "mock_4", modelId: DeviceModelId.stax },
  europa: { name: "Flex", id: "mock_5", modelId: DeviceModelId.europa },
};

export type ModelId = "nanoX" | "nanoSP" | "nanoS" | "stax";

export function getUSBDevice(model: ModelId): DeviceUSB { ... }

export const nanoX_USB: DeviceUSB = {
  deviceId: "1002",
  deviceName: "Ledger Nano X",
  productId: 16401,
  vendorId: 11415,
  wired: true,
  modelId: DeviceModelId.nanoX,
};
// ... nanoSP_USB, nanoS_USB, stax_USB
```

Two shapes: the BLE-style `knownDevices` (what gets passed to `app.init({ knownDevices: [knownDevices.nanoX] })`) and the USB descriptors (what `common.addDeviceViaUSB("nanoX")` uses, with the real USB product/vendor IDs so the bridge can emit a plausible USB event).

Note `europa` maps to the display name "Flex" — the product was renamed but the code symbol stays.

#### `models/currencies.ts`

The key export is `initTestAccounts(currencyIds: string[])`:

```typescript
export function initTestAccounts(currencyIds: string[]) {
  setSupportedCurrencies(currencyIds as CryptoCurrencyId[]);
  return currencyIds.map((currencyId: string) =>
    genAccount("mock" + currencyId, { currency: getCryptoCurrencyById(currencyId) }),
  );
}
```

When a spec passes `testedCurrencies: ["bitcoin", "ethereum"]` to `app.init(...)`, this function is called; it restricts the supported currency list and generates mock accounts (`genAccount` from live-common's mock module) that are then sent over the bridge via `loadAccounts`. Also exports `formattedAmount`, `getAccountUnit`, `getAccountName`, and the Cosmos safety constants.

#### `models/liveApps.ts` (verbatim)

```typescript
import { startDummyServer, stopDummyServer } from "@ledgerhq/test-utils";

export async function startLiveApp(liveAppDirectory: string, liveAppPort = 3000) {
  try {
    const port = await startDummyServer(`${liveAppDirectory}/dist`, liveAppPort);
    const url = `http://localhost:${port}`;
    const response = await fetch(url);
    if (response.ok) {
      console.info(`========> Dummy Wallet API app successfully running on port ${port}! <=========`);
      return true;
    } else {
      throw new Error("Ping response != 200, got: " + response.status);
    }
  } catch (error) {
    console.warn(`========> Dummy test app not running! <=========`);
    console.error(error);
    return false;
  }
}

export async function stopServer() {
  await stopDummyServer();
}
```

`startLiveApp` is how `wallet-api.spec.ts` stands up a local web server serving the dummy wallet-app on port 3000 (or, on Android emulator, the reverse-tcp-port 52619 from `setup.ts` bridges the emulator's view of `localhost:52619` to the host's `localhost:3000`). After this runs, the app's webview can load the dummy app and the test can drive it like any other Wallet API consumer.

### 36.8 The `page/` Layer — The Centrepiece

Desktop uses class inheritance (`AppPage` extends `Component` extends `PageHolder`) and Playwright fixtures. Mobile uses a **single `Application` hub with lazy-initialised POM getters**. Same spirit, different mechanism.

#### `page/index.ts` — The Hub

```typescript
import AssetAccountsPage from "./accounts/assetAccounts.page";
import AccountsPage from "./accounts/accounts.page";
import AddAccountDrawer from "./accounts/addAccount.drawer";
import CommonPage from "./common.page";
import CryptoDrawer from "./liveApps/cryptoDrawer";
// ... 20+ more imports

import type { Account } from "@ledgerhq/types-live";
import { DeviceLike } from "~/reducers/types";
import { loadAccounts, loadBleState, loadConfig, setFeatureFlags } from "../bridge/server";
import { initTestAccounts } from "../models/currencies";
import { setupEnvironment } from "../helpers/commonHelpers";
import type { PartialFeatures } from "@shared/feature-flags";

setupEnvironment();

type ApplicationOptions = {
  userdata?: string;
  knownDevices?: DeviceLike[];
  testedCurrencies?: string[];
  featureFlags?: PartialFeatures;
};

const lazyInit = <T>(PageClass: new () => T) => {
  let instance: T | null = null;
  return () => {
    if (!instance) instance = new PageClass();
    return instance;
  };
};

export class Application {
  public testAccounts: Account[] = [];
  private assetAccountsPageInstance = lazyInit(AssetAccountsPage);
  private accountsPageInstance = lazyInit(AccountsPage);
  private addAccountDrawerInstance = lazyInit(AddAccountDrawer);
  private commonPageInstance = lazyInit(CommonPage);
  // ... 20+ more

  public async init({ userdata, knownDevices, testedCurrencies, featureFlags }: ApplicationOptions) {
    userdata && (await loadConfig(userdata, true));
    knownDevices && (await loadBleState({ knownDevices }));
    if (testedCurrencies) {
      this.testAccounts = initTestAccounts(testedCurrencies);
      await loadAccounts(this.testAccounts);
    }
    if (featureFlags) {
      await setFeatureFlags(featureFlags);
    }
  }

  public get assetAccountsPage() { return this.assetAccountsPageInstance(); }
  public get accounts() { return this.accountsPageInstance(); }
  public get addAccount() { return this.addAccountDrawerInstance(); }
  public get common() { return this.commonPageInstance(); }
  public get cryptoDrawer() { return this.cryptoDrawerInstance(); }
  public get customLockscreen() { return this.customLockscreenPageInstance(); }
  public get discover() { return this.discoverPageInstance(); }
  public get dummyWalletApp() { return this.dummyWalletAppInstance(); }
  public get walletAPIReceive() { return this.walletAPIReceivePageInstance(); }
  public get walletAPISignMessage() { return this.walletAPISignMessagePageInstance(); }
  public get manager() { return this.managerPageInstance(); }
  public get market() { return this.marketPageInstance(); }
  public get onboarding() { return this.onboardingPageInstance(); }
  public get postOnboarding() { return this.postOnboardingPageInstance(); }
  public get operationDetails() { return this.operationDetailsPageInstance(); }
  public get passwordEntry() { return this.passwordEntryPageInstance(); }
  public get portfolio() { return this.portfolioPageInstance(); }
  public get receive() { return this.receivePageInstance(); }
  public get send() { return this.sendPageInstance(); }
  public get settings() { return this.settingsPageInstance(); }
  public get settingsGeneral() { return this.settingsGeneralPageInstance(); }
  public get stake() { return this.stakePageInstance(); }
  public get swap() { return this.swapPageInstance(); }
  public get transferMenu() { return this.transferMenuDrawerInstance(); }
  public get walletTabNavigator() { return this.walletTabNavigatorPageInstance(); }
  public get modularDrawer() { return this.modularDrawerPageInstance(); }
}
```

Walk the `lazyInit` closure line by line:

```typescript
const lazyInit = <T>(PageClass: new () => T) => {
  let instance: T | null = null;       // 1. private to the closure
  return () => {                        // 2. getter returned to the field
    if (!instance) instance = new PageClass();  // 3. construct on first access
    return instance;                    // 4. always return the same instance
  };
};
```

The `T | null = null` is captured by the returned function; each `lazyInit(PageClass)` call produces its own closure with its own `instance` slot. Field assignment stores the getter function on the instance. Calling `this.portfolioPageInstance()` — which the `get portfolio()` accessor does — materialises the POM only if needed.

Why lazy-init matters:

- A spec that never touches the swap screen never constructs `SwapPage`. Jest's `resetModules: true` still clears modules per test, but within a test you avoid building a tree of POMs that would never run.
- The pattern plays well with `resetModules` because each fresh module load rebuilds the `Application` class and all closures.
- It keeps the API shape (`app.portfolio.method(...)`) consistent — specs don't care that construction is deferred.

`app.init({...})` is the single knob that prepares state before tests:

- **`userdata`** — `await loadConfig(userdata, true)` reads `e2e/userdata/<name>.json` and pushes it over the bridge; the app replaces its persisted state with that blob. The `true` flag means "overwrite, don't merge".
- **`knownDevices`** — `loadBleState({ knownDevices })` makes the device reducer treat those devices as previously paired. Enables one-tap reconnect in the "remembered device" flow.
- **`testedCurrencies`** — builds fresh mock accounts and loads them (see `models/currencies.ts`).
- **`featureFlags`** — `setFeatureFlags` is the mobile mirror of desktop's `E2E_FEATURE_FLAGS_JSON`.

#### `page/common.page.ts` (verbatim)

```typescript
import { DeviceUSB, ModelId, getUSBDevice, knownDevices } from "../models/devices";
import { expect } from "detox";
import { Step } from "jest-allure2-reporter/api";
import DeviceAction from "../models/DeviceAction";
import { open, addDevicesBT, addDevicesUSB } from "../bridge/server";
import { delay } from "../helpers/commonHelpers";

export default class CommonPage {
  searchBarId = "common-search-field";
  searchBar = () => getElementById(this.searchBarId);
  successCloseButtonId = "enabled-success-close-button";
  closeButton = () => getElementById("NavigationHeaderCloseButton");

  accountCardPrefix = "account-card-";
  accountCardId = (id: string) => this.accountCardPrefix + id;

  accountItemId = "account-item-";
  accountItemRegExp = (id = ".*(?<!-name)$") => new RegExp(`${this.accountItemId}${id}`);
  accountItem = (id: string) => getElementById(this.accountItemRegExp(id));
  accountItemName = (accountId: string) => getElementById(`${this.accountItemId + accountId}-name`);

  addDeviceButton = () => getElementById("connect-with-bluetooth");
  scannedDeviceRow = (id: string) => `device-scanned-${id}`;
  pluggedDeviceRow = (nano: DeviceUSB) => `device-item-usb|${JSON.stringify(nano)}`;
  blePairingLoadingId = "ble-pairing-loading";

  @Step("Perform search")
  async performSearch(text: string) {
    await waitForElementById(this.searchBarId);
    await typeTextByElement(this.searchBar(), text);
  }

  async expectSearch(text: string) {
    await expect(this.searchBar()).toHaveText(text);
  }

  async closePage() {
    await tapByElement(this.closeButton());
  }

  async successClose() {
    await waitForElementById(this.successCloseButtonId);
    await tapById(this.successCloseButtonId);
  }

  async selectAccount(accountId: string) {
    const id = this.accountCardId(accountId);
    await waitForElementById(id);
    await tapById(id);
  }

  async selectAddDevice() {
    await tapByElement(this.addDeviceButton());
  }

  async addDeviceViaBluetooth(device = knownDevices.nanoX) {
    const deviceAction = new DeviceAction(device);
    await addDevicesBT(device);
    await waitForElementById(this.scannedDeviceRow(device.id));
    await tapById(this.scannedDeviceRow(device.id));
    await waitForElementById(this.blePairingLoadingId);
    await open();
    await deviceAction.accessManager();
  }

  async addDeviceViaUSB(device: ModelId) {
    const nano = getUSBDevice(device);
    const deviceAction = new DeviceAction(nano);
    await addDevicesUSB(nano);
    await delay(1000);
    await scrollToId(this.pluggedDeviceRow(nano));
    await waitForElementById(this.pluggedDeviceRow(nano));
    await tapById(this.pluggedDeviceRow(nano));
    await deviceAction.accessManager();
  }
}
```

Five things worth calling out:

- **`@Step("Perform search")`** — every Step-decorated method becomes a visible step in the Allure report. In the timeline you'll see "Perform search" with its nested actions. Note that not all methods are decorated — `closePage`, `successClose`, etc. don't appear as named steps.
- **`accountItemRegExp`** uses a negative lookbehind `(?<!-name)$` so it matches `account-item-xyz` but not `account-item-xyz-name`. That's how `getIdByRegexp(this.accountItemRegExp(), 0)` in `AddAccountDrawer.expectAccountDiscovery` retrieves the dynamic account id assigned by the app.
- **`pluggedDeviceRow`** serializes the full `DeviceUSB` object into the testID: `device-item-usb|{"deviceId":"1002",...}`. Ugly, but it uniquely identifies the row without requiring the app to emit a stable id.
- **`addDeviceViaBluetooth()`** is the canonical BT pairing flow: emit a scan hit → wait for the row → tap → wait for pairing spinner → emit `opened` → emit the manager app list.
- **`addDeviceViaUSB()`** has a one-second `delay` because the USB-plug animation ships before the list updates; this is one of the rare justified uses of `delay`.

#### `page/passwordEntry.page.ts` (verbatim, the simplest POM — 26 lines)

```typescript
import { expect } from "detox";

export default class PasswordEntryPage {
  getPasswordTextInput = () => getElementById("password-text-input");
  getLogin = () => getElementByText("Log in");

  async enterPassword(password: string) {
    await tapByElement(this.getPasswordTextInput()); //prevent flakiness with Log in button not appearing
    await typeTextByElement(this.getPasswordTextInput(), password, false);
  }

  async login() {
    await tapByElement(this.getLogin());
  }

  async expectLock() {
    await expect(this.getPasswordTextInput()).toBeVisible();
    await expect(this.getPasswordTextInput()).toBeVisible();
  }

  async expectNoLock() {
    await expect(this.getPasswordTextInput()).not.toBeVisible();
    await expect(this.getPasswordTextInput()).not.toBeVisible();
  }
}
```

Anatomy of a minimal POM:

- Element getters (arrow functions, not properties — keeps them lazy so they re-query every call).
- A few imperatives (`enterPassword`, `login`).
- A few assertions (`expectLock`, `expectNoLock`).

No constructor, no base class, no dependency injection. Even `expect` is imported directly rather than taken from a fixture. The extra `.toBeVisible()` duplication in `expectLock`/`expectNoLock` is a minor quirk for Android, which can briefly lie about visibility during keyboard transitions — a second assertion smooths it out.

#### Sub-domain POMs

One POM per subfolder, shown minimally to illustrate the conventions.

**`page/accounts/accounts.page.ts`** (the entire file):

```typescript
import CommonPage from "../common.page";

export default class AccountsPage extends CommonPage {
  baseLink = "accounts";
  listTitle = "accounts-list-title";

  async waitForAccountsPageToLoad() {
    await waitForElementById(this.listTitle);
  }
}
```

Ten lines. It extends `CommonPage` to inherit the search/account-card helpers, and adds a single waiter.

**`page/accounts/addAccount.drawer.ts`** — a drawer POM with `@Step` decorators everywhere:

```typescript
export default class AddAccountDrawer extends CommonPage {
  baseLink = "add-account";
  deselectAllButtonId = "add-accounts-deselect-all";
  accountId = (currency: string, index: number) => `mock:1:${currency}:MOCK_${currency}_${index}:`;
  modalButtonId = "add-accounts-modal-add-button";
  continueButtonId = "enabled-add-accounts-continue-button";
  // ...

  @Step("Open add account via deeplink")
  async openViaDeeplink() {
    await openDeeplink(this.baseLink);
  }

  @Step("Click on 'Import with your Ledger' button")
  async importWithYourLedger() {
    await waitForElementById(this.modalButtonId);
    await tapById(this.modalButtonId);
  }

  @Step("Wait for accounts discovery")
  async waitAccountsDiscovery() {
    await waitForElementById(this.continueButtonId, 240000);
    await delay(1000); // element is animated with delay
  }

  @Step("Expect account discovered")
  async expectAccountDiscovery(currencyName: string, currencyId: string, index = 0) {
    const accountName = `${currencyName} ${index + 1}`;
    await expect(this.accountItem(this.accountId(currencyId, index))).toBeVisible();
    const accountId = (await getIdByRegexp(this.accountItemRegExp(), index)).replace(
      this.accountItemId,
      "",
    );
    await expect(this.accountItemName(accountId)).toHaveText(accountName);
    return accountId;
  }

  // ... tapCloseAddAccountCta, addAccountAtIndex, tapAddFunds, tapReceiveinActionDrawer
}
```

Note the 240-second wait in `waitAccountsDiscovery` — account discovery on testnet can really take that long, and the framework's 60s default would fail spuriously.

**`page/trade/send.page.ts`** (excerpt):

```typescript
export default class SendPage {
  baseLink = "send";
  summaryAmountId = "send-summary-amount";
  recipientInputId = "recipient-input";
  amountInputId = "amount-input";
  // ...

  async sendViaDeeplink(currencyLong?: string) {
    const link = currencyLong ? this.baseLink + currencyParam + currencyLong : this.baseLink;
    await openDeeplink(link);
  }

  @Step("Set recipient and memo tag")
  async setRecipient(address: string, memoTag?: string) {
    await typeTextById(this.recipientInputId, address);
    if (memoTag && memoTag !== "noTag") {
      await typeTextById(this.memoTagInputId, memoTag);
    }
  }

  @Step("Set the amount and return the value")
  async setAmount(amount: string) {
    if (amount === "max") await tapByElement(this.amountMaxSwitch());
    else {
      const element = getElementById(this.amountInputId);
      await element.replaceText(amount);
      await element.tapReturnKey();
    }
    return await getTextOfElement(this.amountInputId);
  }

  @Step("Select account")
  async selectAccount(account: string) {
    await TradePageUtil.selectAccount(account);
  }
  // ... expectSummaryAmount, expect*Error helpers
}
```

**`page/onboarding/onboardingSteps.page.ts`** (excerpt, 153 lines total):

```typescript
export default class OnboardingStepsPage {
  getStartedButtonId = "onboarding-getStarted-button";
  acceptAnalyticsButtonId = "enabled-accept-analytics-button";
  setupLedger = "onboarding-setupLedger";
  selectDevice = (device: ModelId) => `onboarding-device-selection-${device}`;
  // ... 30+ testID definitions

  async startOnboarding() {
    await waitForElementById(this.getStartedButtonId);
    await tapByElement(this.onboardingGetStartedButton());
    await waitForElementById(new RegExp(`${this.setupLedger}|${this.acceptAnalyticsButtonId}`));
    try {
      await tapByElement(this.acceptAnalyticsButton());
    } catch {
      // Analytics prompt not enabled
    }
  }

  async chooseSetupLedger() { await tapById(this.setupLedger); }

  async chooseDevice(device: ModelId) {
    await scrollToId(this.selectDevice(device), this.scrollListContainer);
    await tapById(this.selectDevice(device));
  }

  async goesThroughRestorePhrase() {
    await tapById(this.recoveryPhrase);
    await tapById(this.seedWarning);
    await tapById(this.importRecoveryPhraseCta);
    await tapById(this.importRecoveryPhraseWarning);
    await tapById(this.stepSetupDeviceCta);
    await tapById(this.pinCodeCta); // disabled — no-op
    await tapById(this.pinCodeSwitch);
    await tapById(this.pinCodeCta); // enabled
    await tapById(this.pinCodeSetupCta);
    // ... continues through recovery phrase import
  }
  // ... goesThroughCreateWallet (the happy-path wallet creation carousel)
}
```

`OnboardingStepsPage` is the longest POM because onboarding has the most screens. It is not decorated with `@Step` everywhere — larger orchestration methods like `goesThroughRestorePhrase` are composite steps and it is up to the spec to decide if that's a single step or should be unwrapped.

### 36.9 The `specs/` Layer

Every spec lives under `e2e/specs/` and matches the pattern `*.spec.ts`. Jest's `testMatch` in `jest.config.js` is `<rootDir>/e2e/specs/**/*.spec.ts`, so subfolders like `specs/swap/`, `specs/send/`, `specs/delegate/` are picked up automatically.

Naming convention: one file per feature area. Shared flows (like "create an account, then receive on it") live in dedicated subfolder specs (`specs/receive/*.spec.ts`). Cross-cutting concerns go in root-level files (`deeplinks.spec.ts`, `languageChange.spec.ts`, `password.spec.ts`).

#### `specs/onboarding.spec.ts` — the canonical spec

```typescript
import { device } from "detox";
import { isAndroid, launchApp } from "../helpers/commonHelpers";

describe("Onboarding", () => {
  let isFirstTest = true;

  beforeEach(async () => {
    if (!isFirstTest) {
      await device.uninstallApp();
      await device.installApp();
      await launchApp();
    } else isFirstTest = false;
  });

  $TmsLink("B2CQA-1803");
  it("does the Onboarding and choose to access wallet", async () => {
    await app.onboarding.startOnboarding();
    await app.onboarding.chooseToAccessYourWallet();
    await app.onboarding.chooseToConnectYourLedger();
    await app.common.selectAddDevice();
    await app.common.addDeviceViaBluetooth();
    await app.portfolio.waitForPortfolioPageToLoad();
    await app.portfolio.expectPortfolioEmpty();
  });

  $TmsLink("B2CQA-1802");
  it("does the Onboarding and choose to restore a Nano X", async () => {
    await app.onboarding.startOnboarding();
    await app.onboarding.chooseSetupLedger();
    await app.onboarding.chooseDevice("nanoX");
    await app.onboarding.goesThroughRestorePhrase();
    await app.common.selectAddDevice();
    await app.common.addDeviceViaBluetooth();
    await app.postOnboarding.passThroughPostOnboarding();
    await app.portfolio.waitForPortfolioPageToLoad();
    await app.portfolio.expectPortfolioEmpty();
  });

  $TmsLink("B2CQA-1800");
  $TmsLink("B2CQA-1833");
  it("does the Onboarding and choose to restore a Nano SP", async () => { ... });

  $TmsLink("B2CQA-1799");
  it("does the Onboarding and choose to setup a new Nano X", async () => { ... });
});
```

Top-to-bottom:

- `import { device } from "detox"` — used for the uninstall/install cycle. Nothing else is imported because `app`, `tapById`, etc. are already global.
- `import { isAndroid, launchApp } from "../helpers/commonHelpers"` — platform branch and the app-relaunch helper.
- `describe("Onboarding", ...)` — one describe per file, standard Jest.
- **`let isFirstTest = true` + `beforeEach`** — the first test uses whatever `setup.ts` already launched. Every subsequent test uninstalls, reinstalls, and relaunches the app. This is required for onboarding tests because onboarding can only run once per install; you cannot reset the onboarding flow from inside a running app.
- **`$TmsLink("B2CQA-1803")`** — a label attached by `jest-allure2-reporter`. `$TmsLink` is injected into the file scope by the reporter; it registers the Jira ID with the next `it(...)` so the Allure report links the test to `ledgerhq.atlassian.net/browse/B2CQA-1803`. Multiple `$TmsLink` calls before one `it(...)` attach multiple links (see the "Nano SP" test).
- **`it(...)`** — each test is a short choreography of POM method calls. No low-level Detox in sight. Steps visible in the Allure timeline come from the POM methods' `@Step` decorators.
- **No `afterEach` / `afterAll` overrides** — the defaults in `setup.ts` are enough.

#### `specs/wallet-api.spec.ts` — a live-app example

```typescript
import DeviceAction from "../models/DeviceAction";
import { knownDevices } from "../models/devices";

describe("Wallet API methods", () => {
  const knownDevice = knownDevices.nanoX;
  let deviceAction: DeviceAction;

  beforeAll(async () => {
    await app.init({
      userdata: "1AccountBTC1AccountETHReadOnlyFalse",
      knownDevices: [knownDevice],
    });
    await app.dummyWalletApp.startApp();

    await app.portfolio.waitForPortfolioPageToLoad();
    await app.dummyWalletApp.openApp();
    await app.dummyWalletApp.expectApp();
    deviceAction = new DeviceAction(knownDevice);
  });

  afterAll(async () => {
    await app.dummyWalletApp.stopApp();
  });

  afterEach(async () => {
    await app.dummyWalletApp.clearStates();
  });

  it("account.request", async () => {
    await app.dummyWalletApp.sendRequest();
    await app.cryptoDrawer.selectCurrencyFromDrawer("Bitcoin");
    await app.cryptoDrawer.selectAccountFromDrawer("Bitcoin 1 (legacy)");

    const res = await app.dummyWalletApp.getResOutput();
    expect(res).toMatchObject({
      id: "2d23ca2a-069e-579f-b13d-05bc706c7583",
      address: "1xeyL26EKAAR3pStd7wEveajk4MQcrYezeJ",
      balance: "35688397",
      currency: "bitcoin",
      name: "Bitcoin 1 (legacy)",
      spendableBalance: "35688397",
    });
  });

  it("account.receive", async () => {
    await app.dummyWalletApp.sendAccountReceive();
    await app.walletAPIReceive.continueWithoutDevice();
    await app.walletAPIReceive.cancelNoDevice();
    await app.walletAPIReceive.continueWithoutDevice();
    await app.walletAPIReceive.confirmNoDevice();

    const res = await app.dummyWalletApp.getResOutput();
    expect(res).toBe("1xeyL26EKAAR3pStd7wEveajk4MQcrYezeJ");
  });

  it("message.sign", async () => {
    const account = "Ethereum 1";
    const message = "Hello World! This is a test message for signing.";

    await app.dummyWalletApp.setAccountId("e86e3bc1-49e1-53fd-a329-96ba6f1b06d3");
    await app.dummyWalletApp.setMessage(message);
    await app.dummyWalletApp.messageSign();

    await app.walletAPISignMessage.expectSummary(account, message);
    await app.walletAPISignMessage.summaryContinue();
    await deviceAction.selectMockDevice();
    await deviceAction.silentSign();

    const res = await app.dummyWalletApp.getResOutput();
    expect(res).toBe("mockedSignature");
  });

  // ... several xit()-skipped tests for transaction.sign (various chains)
});
```

What makes this spec different from `onboarding.spec.ts`:

- **`beforeAll` does the heavy lifting** — one-shot setup with userdata, then launches the dummy live app. Every `it` builds on this shared state.
- **`afterEach` clears transient live-app state** so one test's request state doesn't pollute the next.
- **`DeviceAction` is constructed explicitly** — this spec needs to call `selectMockDevice()` and `silentSign()` directly because the wallet API flow routes through the device-action modal.
- **`app.dummyWalletApp.getResOutput()`** reads back whatever the live app responded to the Wallet API call. The bridge `ServerData.walletAPIResponse` frame is how that value gets here.

#### `specs/swap/dexSwap.spec.ts` (verbatim — 29 lines, skipped)

```typescript
// SKIP same as swap.spec.ts - it uses the old swap interface and it's not working on iOS
// with new-arch because of the infinite re-render loop
describe.skip("DEX Swap", () => {
  beforeAll(async () => {
    await app.init({ userdata: "1AccountBTC1AccountETHReadOnlyFalse" });

    await app.portfolio.waitForPortfolioPageToLoad();
    await app.swap.openViaDeeplink();
    await app.swap.expectSwapPage();
  });

  it("should be able to generate a quote with DEX providers available", async () => {
    await app.swap.openSourceAccountSelector();
    await app.swap.selectAccount("Ethereum 2");
    await app.swap.openDestinationAccountSelector();
    await app.swap.selectAccount("Tether USD");
    await app.swap.enterSourceAmount("1");
    await app.swap.goToProviderSelection();
    await app.swap.chooseProvider("1inch");
  });

  // FIXME site unavailable on Android CI
  it("should be able to navigate to a DEX with the correct params", async () => {
    await app.swap.startExchange();

    await app.discover.expectApp("1inch");
    await app.discover.expect1inchParams();
  });
});
```

Even skipped, this spec is the blueprint for Swap tests on mobile. We'll reuse its shape in Chapter 40 when filling the mobile swap coverage. Notice:

- `describe.skip` (not `xit`) — the whole block is off, but the structure is preserved as reference.
- Same `app.init({ userdata })` pattern as the other specs.
- Deeplink navigation (`openViaDeeplink`) is the preferred entry, not menu traversal.

### 36.10 The `userdata/` Directory

`userdata/*.json` files are **Redux state snapshots**. The shape mirrors the app's persisted state: top-level `data` key containing `settings`, `accounts`, `bleDevices`, etc.

```json
{
  "data": {
    "SPECTRON_RUN": {
      "localStorage": {
        "acceptedTermsVersion": "2019-12-04"
      }
    },
    "settings": {
      "hasCompletedOnboarding": true,
      "counterValue": "USD",
      "language": "en",
      ...
    },
    ...
  }
}
```

The key files:

| File | Size | What it seeds |
|------|------|---------------|
| `skip-onboarding.json` | 1.5 KB | Onboarding complete, no accounts. Starting state for anything that needs the portfolio empty. |
| `1accountEth.json` | 127 KB | Onboarding complete + one Ethereum account. |
| `1Account1NFTReadOnlyFalse.json` | 21 KB | One account plus one NFT. Used by NFT-visibility tests. |
| `1AccountBTC1AccountETHReadOnlyFalse.json` | 1.1 MB | One BTC + one ETH account with realistic balances and history. The workhorse for Swap/Send/Wallet-API specs. |
| `EthAccountXrpAccountReadOnlyFalse.json` | 444 KB | ETH + XRP accounts. Used by tests that need two independent chains without BTC. |

"ReadOnlyFalse" is a suffix that comes from the desktop-era naming convention (`readOnlyModeEnabled: false`); it means the store is set up for a real-device flow, not the no-device "read-only" mode.

**The bridge call path.** When a spec calls:

```typescript
await app.init({ userdata: "1AccountBTC1AccountETHReadOnlyFalse" });
```

`Application.init()` invokes `loadConfig(userdata, true)` from `bridge/server.ts`. That function:

1. Resolves `e2e/userdata/<name>.json` from disk.
2. Reads it with `fs.readFileSync`.
3. Sends a `{ type: "loadConfig", payload: <parsed json>, id: <uuid> }` WS message to the app.
4. Waits for the matching `ACK`.

The client (`bridge/client.ts`, in the app) receives the message, dispatches a Redux action that replaces the entire persisted state with the payload, and ACKs. By the time `app.init()` resolves, the app has restarted its screens with the new state — which is why the next line in a spec is usually `await app.portfolio.waitForPortfolioPageToLoad()`.

### 36.11 How a Test Actually Runs

ASCII sequence from CLI to artifact:

```
  detox test --configuration ios.release.iphone15
              │
              ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │ 1. Detox CLI                                                      │
  │    - Reads detox.config.js                                         │
  │    - Boots simulator/emulator                                      │
  │    - Installs the app binary                                       │
  │    - Spawns Jest with --config e2e/jest.config.js                  │
  └─────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │ 2. Jest (once per run)                                             │
  │    - Executes jest.globalSetup.ts → detox globalSetup()            │
  │    - Loads jest.environment.ts for every worker                    │
  │    - Registers reporters: detox reporter + allure2 reporter         │
  └─────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │ 3. Spec file start (e.g. onboarding.spec.ts)                       │
  │    setupFilesAfterEnv → setup.ts runs:                              │
  │       • registerAllCoins() / LiveConfig.setConfig()                 │
  │       • setupEnvironment() (MOCK=1, disable version checks)         │
  │       • global.app = new Application()                              │
  │       • globals wired: Currency, Account, ElementHelpers, Step, ... │
  └─────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │ 4. beforeAll (in setup.ts):                                        │
  │       const port = await launchApp();                              │
  │          ├─ findFreePort()                                         │
  │          ├─ closeBridge(); initBridge(port);                       │
  │          └─ device.launchApp({ launchArgs: { wsPort, MOCK, ... }}) │
  │       device.reverseTcpPort(8081)   // Metro                       │
  │       device.reverseTcpPort(port)   // Bridge WS                   │
  │       device.reverseTcpPort(52619)  // Dummy wallet-app            │
  │                                                                    │
  │       app launches → bridge/client.ts connects to ws://:<port>     │
  │       → WS handshake complete                                      │
  └─────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │ 5. Test beforeAll (in spec):                                       │
  │       await app.init({ userdata, knownDevices, ... })              │
  │          ├─ loadConfig("<name>", true)  // over WS                 │
  │          │     app replaces Redux state → ACK                      │
  │          ├─ loadBleState({ knownDevices })                         │
  │          └─ setFeatureFlags(flags)                                 │
  └─────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │ 6. it(...) runs:                                                   │
  │       POM methods called (app.onboarding.startOnboarding, ...)    │
  │       each @Step-decorated method emits a step to Allure           │
  │       each tapById / waitForElementById → Detox native action      │
  │       DeviceAction methods → mockDeviceEvent over WS               │
  │       Dummy live app responses → ServerData.walletAPIResponse      │
  └─────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │ 7. jest.environment.ts handleTestEvent:                            │
  │       on "test_fn_failure" or "hook_failure" → IS_FAILED = true    │
  │       on "run_start" → logMemoryUsage()                            │
  │       detox-allure2-adapter captures screenshot on failure         │
  └─────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │ 8. afterAll (in setup.ts):                                         │
  │       if (IS_FAILED && CI) allure.attachment("App logs", getLogs())│
  │       closeBridge()                                                │
  └─────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │ 9. Jest final teardown:                                            │
  │       - reporters flush artifacts/allure-results/*.json            │
  │       - Detox shuts down the app                                   │
  │       - CI job runs `allure generate artifacts/allure-results`     │
  └───────────────────────────────────────────────────────────────────┘
```

### 36.12 Cross-Reference with Part 5 Chapter 27

What is **the same** between mobile and desktop:

- **Same shared enums.** Both suites import `Currency`, `Account`, `Team`, `AppInfos`, `Fee`, `Delegate`, `Transaction`, `Swap` from `@ledgerhq/live-common/e2e/`. Changes propagate to both at once.
- **Same Allure reporter stack.** Both emit `artifacts/allure-results/` and both use `$TmsLink("B2CQA-XXXX")` to link tests to Jira.
- **Same POM philosophy.** Each screen/drawer/dialog has a dedicated class with element getters and imperative/assertion methods.
- **Same userdata idea.** Desktop has `.json` files under `tests/userdata/`; mobile has `.json` files under `e2e/userdata/`. Both represent persisted state snapshots that bypass onboarding or account setup.
- **Same Step decorator pattern.** `@Step("...")` produces nested timeline entries in Allure on both suites.

What is **different**:

- **Runner.** Desktop = Playwright + Electron. Mobile = Detox + Jest + native runtimes (iOS simulator / Android emulator).
- **No fixture file.** Desktop hangs everything on Playwright test fixtures in `tests/fixtures/common.ts`. Mobile has no fixtures — it uses a single global `app = new Application()` and promotes primitives to `globalThis`.
- **No class hierarchy.** Desktop has `PageHolder → Component → AppPage`. Mobile POMs either stand alone or extend `CommonPage` for the search/add-device bits. No `PageHolder`, no `Component` base.
- **Lazy-init getters instead of eager construction.** `page/index.ts` builds POMs on first access via the `lazyInit<T>` closure, rather than constructing them all in a fixture.
- **Bridge instead of direct launch args.** Desktop pushes config by setting env vars before Electron starts. Mobile has a persistent WebSocket bridge and pushes state at any time during a test.
- **DeviceAction class instead of SpeculosPage.** Desktop talks to a real Speculos (BLE/USB-simulated device). Mobile emits mock device events directly over the bridge via `DeviceAction`.
- **No dual-path modular selector.** The desktop `getModularSelector()` pattern has no direct mobile analogue — modular UI detection on mobile happens inside specific POMs (`ModularDrawer`) rather than as a generic utility.
- **Looser typing on globals.** Desktop relies on fixture parameters typed at the test signature. Mobile declares globals in `setup.ts` with a giant `declare global { ... }` block. Less ceremony, less help from the compiler.
- **Screenshots by default, videos off.** Desktop can capture trace files via Playwright. Mobile captures screenshots via the `detox-allure2-adapter` but keeps videos off by default for artifact size.

<div class="chapter-outro">
<strong>Key takeaway:</strong> the mobile suite is leaner than desktop — one global <code>app</code>, one global pool of primitives, one <code>Application</code> class with lazy getters. The complexity that desktop expresses through Playwright fixtures and class hierarchies is replaced on mobile by the bridge. When you are stuck, start at <code>e2e/setup.ts</code> (global wiring) → <code>e2e/page/index.ts</code> (<code>Application</code> hub) → the specific POM you need. The bridge and <code>DeviceAction</code> are the only genuinely mobile-only concepts; everything else maps onto a Part 5 Chapter 27 concept.
</div>

### 36.13 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz -- Mobile Codebase Deep Dive</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> In <code>setup.ts</code>, why does <code>beforeAll</code> call <code>device.reverseTcpPort(8081)</code>, <code>device.reverseTcpPort(port)</code>, and <code>device.reverseTcpPort(52619)</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) To open firewall holes on the developer machine</button>
<button class="quiz-choice" data-value="B">B) To bind host-side services (Metro on 8081, the bridge on <code>port</code>, the dummy wallet-app on 52619) to the emulator's <code>localhost</code> so the app can reach them</button>
<button class="quiz-choice" data-value="C">C) To randomize ports for security</button>
<button class="quiz-choice" data-value="D">D) To proxy network traffic through Speculos</button>
</div>
<p class="quiz-explanation">Android emulators see their own <code>localhost</code>, not the host's. <code>reverseTcpPort</code> forwards the emulator's <code>localhost:X</code> to the host's <code>localhost:X</code>. We need three: Metro (8081), the WebSocket bridge (a free port found by <code>findFreePort</code>), and the dummy wallet-app (52619).</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> What does the <code>lazyInit&lt;T&gt;</code> helper in <code>page/index.ts</code> do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Asynchronously loads POM files from disk</button>
<button class="quiz-choice" data-value="B">B) Returns a new POM instance on every access</button>
<button class="quiz-choice" data-value="C">C) Returns a getter that constructs the POM on first access and caches it in a closure for subsequent calls</button>
<button class="quiz-choice" data-value="D">D) It decorates POMs with <code>@Step</code></button>
</div>
<p class="quiz-explanation"><code>lazyInit</code> creates a closure over <code>let instance: T | null = null</code>. The first call to the returned function constructs the POM, later calls return the same instance. A spec that never touches Swap never builds <code>SwapPage</code>.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> Where do globals like <code>tapById</code>, <code>waitForElementById</code>, and <code>app</code> come from in a spec file? The file imports nothing from <code>helpers/</code>.</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) They are installed on <code>globalThis</code> by <code>e2e/setup.ts</code>, which is wired as <code>setupFilesAfterEnv</code> in <code>jest.config.js</code></button>
<button class="quiz-choice" data-value="B">B) They are auto-imported by Detox</button>
<button class="quiz-choice" data-value="C">C) They are injected by the Babel transformer</button>
<button class="quiz-choice" data-value="D">D) They are registered by <code>jest.globalSetup.ts</code></button>
</div>
<p class="quiz-explanation"><code>setup.ts</code> runs once per spec file before the tests, and it promotes every <code>ElementHelpers</code> method plus <code>app</code>, <code>Currency</code>, <code>Account</code>, etc. to <code>globalThis</code>. The TypeScript side is the <code>declare global { ... }</code> block at the top.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> A test calls <code>await app.init({ userdata: "1AccountBTC1AccountETHReadOnlyFalse" })</code>. What happens next?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The app is relaunched from scratch</button>
<button class="quiz-choice" data-value="B">B) A new Redux store is created on the test side</button>
<button class="quiz-choice" data-value="C">C) The JSON is committed to the device's filesystem</button>
<button class="quiz-choice" data-value="D">D) <code>loadConfig</code> reads the file, sends it over the WS bridge; the app receives it in <code>bridge/client.ts</code> and dispatches a Redux action that replaces the persisted state — then ACKs</button>
</div>
<p class="quiz-explanation">Userdata is a Redux state snapshot. The test sends the blob over the bridge; the app replaces its state with that blob. This is why the next line in a spec is typically <code>app.portfolio.waitForPortfolioPageToLoad()</code> — the UI has re-rendered on the new state.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> Why does <code>onboarding.spec.ts</code> have a <code>beforeEach</code> that uninstalls, reinstalls, and relaunches the app for every test except the first?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) To test install reliability</button>
<button class="quiz-choice" data-value="B">B) Onboarding can only run once per install — a fresh install is the only way to re-enter the onboarding flow</button>
<button class="quiz-choice" data-value="C">C) Because <code>resetModules: true</code> is broken</button>
<button class="quiz-choice" data-value="D">D) To force Allure to capture screenshots</button>
</div>
<p class="quiz-explanation">Once onboarding completes, the app persists <code>hasCompletedOnboarding: true</code> and never shows the onboarding flow again. The only way to retest onboarding from scratch is a full uninstall + install + launch. The first test skips the reinstall because <code>setup.ts</code> already launched the app fresh.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q6.</strong> Which structural difference with the desktop suite is the most consequential for how you read mobile tests?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Mobile has no fixture file and no class hierarchy — a single <code>app</code> hub plus <code>globalThis</code> primitives; desktop uses Playwright fixtures injected into each test signature</button>
<button class="quiz-choice" data-value="B">B) Mobile uses Jest, desktop uses Mocha</button>
<button class="quiz-choice" data-value="C">C) Mobile has no Allure reporting</button>
<button class="quiz-choice" data-value="D">D) Mobile POMs are written in JavaScript, not TypeScript</button>
</div>
<p class="quiz-explanation">Desktop tests start with <code>test("...", async ({ page, app, speculos }) =&gt; ...)</code> — the fixture system provides the objects. Mobile tests start with <code>it("...", async () =&gt; ...)</code> and use <code>app.*</code> and <code>tapById</code> directly from globals. Once you internalise this, every other difference (lazy init, bridge, DeviceAction) is incremental.</p>
</div>

<div class="quiz-score"></div>
</div>

---


---

## Running & Debugging Mobile E2E Tests

<div class="chapter-intro">
Chapters 34 and 35 taught you how Detox finds elements, how the bridge mocks a device, and how a spec file is structured. Chapter 36 grounded you in the actual repository layout. This chapter turns that knowledge into muscle memory: every flag on every command, every environment file toggle, every way a mobile test can go wrong — and exactly how to recover. By the end you should be able to boot a simulator, reproduce a failing scheduled CI run on your laptop, attach a debugger, and read the artifacts like a native speaker.
</div>

### 37.1 Prerequisites Recap

Before running a single `detox test`, confirm the toolchain from Chapter 33 is in place. Unlike desktop Playwright — where launching the Electron binary is essentially self-contained — mobile E2E depends on **three** external processes that must all be alive simultaneously: the simulator/emulator, the Metro bundler (for debug builds), and the Detox test-host process.

**iOS checklist (on macOS only):**
- Xcode 15+ installed (`xcode-select -p` should point to `/Applications/Xcode.app/Contents/Developer`)
- Command-line tools: `xcode-select --install`
- A booted simulator matching the Detox config: typically `iPhone 15` on iOS 17.x. List them with `xcrun simctl list devices`.
- `applesimutils` installed (`brew tap wix/brew && brew install applesimutils`) — Detox uses this to set permissions.
- CocoaPods synced (Chapter 33 `pod install` in `apps/ledger-live-mobile/ios`).

**Android checklist (macOS or Linux):**
- Android Studio installed, or at least the `cmdline-tools` package via `sdkmanager`.
- An AVD created matching the config name — typically `Android_Emulator` API 35 `pixel_7_pro`.
- `ANDROID_HOME` exported and `$ANDROID_HOME/platform-tools` on `PATH` (so `adb` resolves).
- Emulator booted and visible: `adb devices` should list exactly one device in `device` state.

**Universal checklist:**
- `pnpm` installed and `pnpm i` run at the monorepo root.
- `pnpm --filter ledger-live-mobile-e2e-tests i` (Chapter 36 explained why the e2e workspace pulls its own deps).
- Speculos coin apps accessible — either a local `COINAPPS` directory or the `selectMockDevice` bridge helper (see 37.10).

> **Tip:** Run `pnpm mobile exec detox --version` from the repo root before anything else. If that command fails, nothing else will work. A non-zero exit here almost always means `pnpm i` was never run inside the mobile workspace.

A running simulator or emulator is **not optional**. Detox does not spin one up for you the way Playwright spins up Chromium. If you forget this step you will see `DetoxRuntimeError: Failed to run application on the device` within the first ten seconds.

### 37.2 Build Commands in Depth

Detox separates **build** from **test**. The build step compiles a binary wired for Detox (`detox.frameworkBundle` injected, testability enabled). The test step installs and drives that binary. You almost never want to rebuild on every run — it takes 5–15 minutes for iOS, 3–8 minutes for Android.

#### iOS debug build

```bash
pnpm mobile e2e:build -c ios.sim.debug
```

**What each piece does:**
- `pnpm mobile` — alias defined in the root `package.json` that expands to `pnpm --filter live-mobile`.
- `e2e:build` — script in `apps/ledger-live-mobile/package.json`. Internally: `detox build -c <configuration>`.
- `-c ios.sim.debug` — selects a configuration from `.detoxrc.js`. The name encodes platform (`ios`), target (`sim` = simulator), and variant (`debug`).

A **debug** build is compiled with Metro serving JS at runtime. Reloads are fast (edit code → `r` in Metro → hot reload), but you pay the cost of a live Metro server on port 8081 during the test run.

#### iOS release build

```bash
pnpm mobile e2e:build -c ios.sim.release
```

A **release** build bundles JS into `main.jsbundle` at build time, inlines it into the `.app`, and runs Hermes bytecode. No Metro needed. This is what CI uses because it is faster per-test (no Metro round-trips) and more representative of production.

Use `release` when:
- You are debugging a flaky test that you suspect is timing-related.
- You want to reproduce a CI failure exactly.
- You have finished iterating and want a final validation pass.

Use `debug` when:
- You are authoring a new test and want to tweak the spec and re-run without rebuilding.
- You want to use React DevTools or the Metro logs to inspect app state.

#### Android builds

```bash
pnpm mobile e2e:build -c android.emu.debug
pnpm mobile e2e:build -c android.emu.release
```

Same split: `debug` uses Metro, `release` uses a bundled APK. The emulator config expects an AVD named exactly `Android_Emulator` — look at `.detoxrc.js` for the `avdName` field.

#### The prerelease variant

```bash
pnpm mobile e2e:build -c ios.sim.prerelease
pnpm mobile e2e:build -c android.emu.prerelease
```

`prerelease` is a third variant used for pre-release validation against production services. It compiles the app with `ENVFILE=.env.mock.prerelease` (see below), which points Firebase Remote Config at the **production** project instead of staging. You rarely need this locally; CI uses it when `production_firebase: true` is set on the workflow (Chapter 38.12).

#### The `ENVFILE` toggle

Every React Native build reads an `.env` file at compile time via `react-native-config`. Detox variants pick different env files:

| Variant | ENVFILE | Firebase | Intended use |
|---------|---------|----------|--------------|
| `*.debug`, `*.release` | `.env.mock` | Staging project | Normal dev & scheduled CI |
| `*.prerelease` | `.env.mock.prerelease` | Production project | Release gating, dogfood validation |

**What changes downstream when you flip the file:**
- `FIREBASE_*` keys (API key, project ID, app ID) swap to production.
- Remote Config feature flags are read from the production namespace — a flag disabled in staging may be enabled in prod or vice versa.
- Analytics endpoints change (Segment write keys, etc.), though the E2E harness typically stubs these.
- The bundle ID of the installed `.app` / `.apk` changes (e.g., `com.ledger.live.dev.detox` vs `com.ledger.live.detox.prerelease`). This matters because the test can only drive the binary whose bundle ID matches the Detox config.

> **Warning:** If you rebuild with a different `ENVFILE` without uninstalling the previous binary, the simulator may still launch the old one — Detox resolves apps by bundle ID. Run `xcrun simctl uninstall booted <bundleId>` between variant switches, or simply `device.uninstallApp()` at the start of a spec.

#### Build caching

The native build artifact is large (several hundred MB) and its content is deterministic per the combination of:
- The `ios/` or `android/` source tree (hashed).
- The selected variant.
- The `ENVFILE` content.

CI keys its S3 cache off `hashFiles('apps/ledger-live-mobile/ios')` — locally, you get the same effect by not touching native code between runs. If you only edit TypeScript specs, the previously-built `.app` or `.apk` is reused and you just re-run tests.

### 37.3 Test Execution

Once a build exists, test execution is invoked via:

```bash
pnpm mobile e2e:test -c ios.sim.debug
pnpm mobile e2e:test -c android.emu.debug
```

The script expands to `detox test -c <configuration> [...]`. Anything after the config passes through to Detox and thence to Jest.

#### Filtering which tests run

Jest supplies the selection primitives; Detox just forwards them.

**By test name (`-t` or `--testNamePattern`):**

```bash
pnpm mobile e2e:test -c ios.sim.debug -- -t "Portfolio"
pnpm mobile e2e:test -c ios.sim.debug -- --testNamePattern "B2CQA-604"
```

This matches anything in the concatenation of `describe` + `it` titles. Since every `it()` in the suite embeds a `$TmsLink("B2CQA-604")` tag in its title (Chapter 36.5), `-t "B2CQA-604"` is the single-ticket filter you will use most often.

**By file path (`--testPathPattern`):**

```bash
pnpm mobile e2e:test -c ios.sim.debug -- --testPathPattern specs/portfolio
pnpm mobile e2e:test -c ios.sim.debug -- --testPathPattern "addAccount|send"
```

This is a regex against the absolute path of each spec file. It runs before the test files are loaded, so it is cheaper than `-t` when you know you only want a subset.

**Combining them:**

```bash
pnpm mobile e2e:test -c ios.sim.debug -- --testPathPattern specs/history -t "filter"
```

Runs only spec files under `specs/history/`, and within those only `it()` names matching `/filter/`.

**Excluding paths (`--testPathIgnorePatterns`):**

```bash
pnpm mobile e2e:test -c ios.sim.debug -- --testPathIgnorePatterns specs/experimental
```

Handy when a directory is known-broken and you want to skip it wholesale without editing `jest.config.js`.

#### Worker and lifecycle flags

```bash
pnpm mobile e2e:test -c ios.sim.debug -- --maxWorkers=1 --cleanup --reuse
```

- `--maxWorkers=1` — run tests serially in a single process. This is actually the **default** (and only supported) configuration for mobile E2E; the codebase sets `maxWorkers: 1` in `jest.config.js` because Detox can only drive one device per worker and the bridge port would collide with parallel workers. CI achieves parallelism across **shards**, not workers (Chapter 38).
- `--cleanup` — uninstall the app at the end of the run. Without this, the app stays on the simulator and subsequent runs start faster (but may see stale Redux persist state).
- `--reuse` — skip the install step and reuse the binary already on the device. Fastest iteration mode when nothing has been rebuilt.
- `--forceExit` — kill Jest's process even if there are unhandled handles. Used in CI to avoid timing out on lingering WebSocket connections.
- `--retries 2` — retry a failed `it()` up to 2 times. The CI orchestrator sets this; locally you usually want `0` so you see the true failure.
- `--record-logs failing`, `--take-screenshots failing` — Detox artifact policies. Keep logs and screenshots only for failing tests (see 37.6).
- `--headless` — boot the simulator/emulator without a visible window. CI uses this; locally you probably want the window so you can watch.
- `--loglevel warn` — Detox log verbosity. Values: `trace`, `debug`, `info`, `warn`, `error`. For a frustrating failure, bump to `trace` and grep the output.

#### Putting it together

A realistic local iteration loop on a single ticket:

```bash
# One-time: build (or rebuild if native code changed)
pnpm mobile e2e:build -c ios.sim.debug

# Many times: run just the failing test with verbose logging and no retries
pnpm mobile e2e:test -c ios.sim.debug \
  -- -t "B2CQA-604" \
  --reuse \
  --retries 0 \
  --loglevel trace \
  --record-logs all \
  --take-screenshots all
```

### 37.4 The `scripts/e2e-ci.mjs` Orchestrator

Opening the workflow YAML you saw in Chapter 36, the actual command every shard runs is not `detox test` directly — it is a `zx` script at `apps/ledger-live-mobile/scripts/e2e-ci.mjs`. Read it verbatim; it is small (~210 lines) and explains itself:

```js
#!/usr/bin/env zx
// apps/ledger-live-mobile/scripts/e2e-ci.mjs (excerpt)
let platform, test, build, bundle, bundleSize;
let testType = "mock";
let cache = true;
let shard = "";
let target = "release";
let filter = "";
let outputFile = "";
```

**Flags it accepts:**

| Flag | Meaning |
|------|---------|
| `-p <ios\|android>` | Which platform to target. Required. |
| `-t` | Run the `test` step. |
| `-b` | Run the `build` step (native compile). |
| `--bundle` | Run the JS bundling step (produces `main.jsbundle`). |
| `--bundle-size` | Produce a minified bundle for size reporting only (not used for tests). |
| `--cache / --no-cache` | On iOS, whether to copy the JS bundle into the cached `Release-iphonesimulator` directory so subsequent cache hits are valid. |
| `--e2e` | Switch `testType` from `mock` (default) to `e2e` — flips between `pnpm mobile mock:test` and `pnpm mobile e2e:test`. |
| `--shard N/M` | Forwarded to Detox/Jest as `--shard N/M`. |
| `--production` | Switch the Detox target from `release` to `prerelease`. |
| `--filter <pattern>` | Not forwarded directly; present so the orchestrator can strip it from trailing args. |
| `-o / --outputFile <path>` | Appends `--json --outputFile=<path>` to the Jest command for timing-data collection. |

**How it invokes Detox.** The key production-test call is:

```js
await $`pnpm mobile ${testType}:test\
    -c ios.sim.${target} \
    --loglevel warn \
    --record-logs failing \
    --take-screenshots failing \
    --forceExit \
    --headless \
    --retries 2 \
    --cleanup \
    ${filteredArgs}`.nothrow();
```

`filteredArgs` is every trailing positional argument that isn't one of the recognized flags — in practice, the list of spec file paths computed by the sharding step (Chapter 38.5).

**The mock/e2e split.** `mock:test` uses the bridge mock device (no Speculos required — see Chapter 35 and 37.10). `e2e:test` uses a real Speculos container. CI almost always uses `--e2e` because scheduled runs exercise real device flows; `mock` is kept as an option for speed-critical smoke passes.

**The bundle-then-build dance.** Notice `bundle_ios_with_cache` copies `main.jsbundle` into the already-built `Release-iphonesimulator/` directory. This lets CI cache the native build (which is expensive and rarely changes) and only re-bundle JS (which is cheap and changes every commit).

**Metro and the bridge.** For `release`/`prerelease` builds, there is no Metro — the JS is already inside the `.app`/`.apk`. The Detox bridge still runs (that is the WebSocket that ships mock device commands from spec to app), on port 8099 by default. For `debug` builds, Metro runs on 8081 and the bridge on 8099 — two distinct processes with two distinct ports.

### 37.5 Debugging Cookbook

This section is a field guide. Each entry is a symptom, a diagnosis, and the exact command or code change that resolves it.

#### Symptom: "App is busy"

```
DetoxRuntimeError: Timed out while waiting for the app to become idle
```

**Diagnosis:** Detox synchronizes with the app — it won't execute the next action until the JS thread, animations, pending network requests, and timers have all settled. If your app has a never-closing WebSocket or a repeating poller, Detox waits forever.

**Fix:** Blacklist the URL or disable synchronization locally.

```typescript
// Globally blacklist a URL from the sync check
await device.setURLBlacklist([".*/polling/.*", ".*/sse/.*"]);

// Or disable synchronization for a block of actions
await device.disableSynchronization();
try {
  await Tap(button).click();
  await AnimationThatTakesForever.waitFor({ timeout: 5000 });
} finally {
  await device.enableSynchronization();
}
```

See Detox's [synchronization troubleshooting guide](https://wix.github.io/Detox/docs/troubleshooting/synchronization) — the full list of strategies includes disabling specific modules (`device.setSyncSettings`) and whitelisting vs blacklisting.

#### Symptom: "Element not found"

```
Error: Test Failed: No elements found for “MATCHER(id == 'send-button')”
```

**Diagnosis:** The `testID` you expect isn't on the screen. Either (a) the screen is not rendered yet, (b) the element uses a different `testID`, or (c) it's inside a `FlatList` that hasn't virtualized it into view yet.

**Fix:** Dump the view hierarchy at the moment of failure.

```typescript
// In the failing test, just before the action:
const xml = await device.generateViewHierarchyXml();
console.log(xml);
// Or to a file:
await device.generateViewHierarchyXml({ path: "/tmp/hierarchy.xml" });
```

The XML contains every `testID`, `accessibilityLabel`, and `text` value currently on screen. Grep it for a partial match. If the element is genuinely missing, the preceding action must not have completed — walk back through the spec.

If the element is offscreen in a list:

```typescript
await waitFor(element(by.id("item-42")))
  .toBeVisible()
  .whileElement(by.id("scrollable-list"))
  .scroll(200, "down");
```

#### Symptom: Metro port collision

```
error Metro is already running on port 8081
```

**Diagnosis:** A previous run left Metro alive, or another project is using 8081.

**Fix:**

```bash
# Find the process
lsof -i :8081
# Kill it
kill -9 <pid>
# Or kill every node process listening on 8081
lsof -t -i :8081 | xargs kill -9
```

For Android specifically, even if Metro is running on the host, the emulator must be told to forward its own `localhost:8081` to the host's. This happens automatically when Detox installs a debug APK, but if you sideload manually:

```bash
adb reverse tcp:8081 tcp:8081
```

See Android's [adb forwardports docs](https://developer.android.com/tools/adb#forwardports) — `adb reverse` is the opposite direction (device → host) and is what React Native's Metro bridge depends on.

#### Symptom: Bridge WebSocket not connecting

```
DetoxRuntimeError: The app has not started communicating with the test runner
```

**Diagnosis:** The Detox bridge runs on port 8099. Either the port is taken, the app build didn't include the Detox framework, or the device cannot reach the host.

**Fix checklist:**
1. `lsof -i :8099` — if something else is there, kill it.
2. Rebuild with `-c <config>` matching the one you're testing against. A common mistake is building `ios.sim.debug` and trying to test with `ios.sim.release` — the binary on the simulator is the old one.
3. Check `.detoxrc.js` and the app's `AppDelegate.mm` — confirm `wsPort: 8099` matches on both sides. The bridge host and port are injected at launch via `launchArgs`.
4. On Android, verify `adb reverse tcp:8099 tcp:8099` is active.

#### Symptom: Timeouts

```
thrown: "Exceeded timeout of 120000 ms for a test"
```

**Diagnosis:** Jest's per-test timeout is exhausted. Mobile tests are slow — a default 30s is far too short.

**Fix:** `apps/ledger-live-mobile/e2e/jest.config.js` sets the default.

```js
// Typical config
module.exports = {
  testTimeout: 300000,    // 5 minutes per it()
  maxWorkers: 1,          // Detox constraint
  resetModules: true,
  setupFilesAfterEnv: ["<rootDir>/setup.ts"],
};
```

For a specific slow action, use `waitFor(...).withTimeout()`:

```typescript
await waitFor(element(by.id("sync-complete-banner")))
  .toBeVisible()
  .withTimeout(180000); // 3 minutes
```

#### Symptom: Flaky test ordering

Test A passes alone but fails when run after Test B.

**Diagnosis:** State leaking between tests — usually Redux state persisted across `it()` blocks.

**Fix:** The harness already sets `resetModules: true`. What you likely need is `device.reloadReactNative()` in a `beforeEach` — it reboots the JS runtime without reinstalling the native side. If that's not enough, `device.launchApp({ delete: true })` reinstalls fresh.

Do not attempt to run tests in parallel by bumping `maxWorkers`. The bridge port alone makes that impossible; the codebase pins `maxWorkers: 1` on purpose.

### 37.6 Screenshots, Videos, View Hierarchy Dumps

Detox's `artifacts` section in `.detoxrc.js` controls automated captures.

```js
artifacts: {
  rootDir: "./artifacts",
  plugins: {
    log: "failing",
    screenshot: {
      enabled: true,
      shouldTakeAutomaticSnapshots: true,
      keepOnlyFailedTestsArtifacts: true,
      takeWhen: { testStart: true, testDone: true, testFailure: true },
    },
    video: "failing",
    instruments: "failing",
    uiHierarchy: "enabled",
  },
}
```

**What lands where:**
- Screenshots: `./artifacts/<run-id>/<spec-name>/<test-name>/<step>.png`
- Videos: `./artifacts/<run-id>/<spec-name>/<test-name>/test.mp4` (iOS only; Android uses screen recording when the emulator supports it)
- View hierarchy: `.viewhierarchy` files on iOS, XML dumps on Android
- Logs: `device.log`, `app.log`, `jest.log`

The `failing` policy means artifacts are only retained for failed tests — passing tests' captures are deleted at the end of the run to save disk.

**Allure pickup.** The Jest Allure reporter (configured in `jest.config.js`) writes one JSON per `it()` to `./allure-results/`. When the Detox run emits an artifact for a failing test, the reporter attaches the file path as an Allure attachment so the eventual HTML report embeds the screenshot and video inline.

### 37.7 Allure Output Locally

After a run, the raw Allure JSON sits at:

```
apps/ledger-live-mobile/e2e/allure-results/
```

Each `it()` produces one `<uuid>-result.json` plus an `-attachment.json` file for each attached artifact.

To view the report in a browser, install the CLI once and serve it:

```bash
# One-time install (use pnpm dlx or brew)
brew install allure
# Or: pnpm dlx allure-commandline --version

# Serve the report — opens a browser at http://localhost:<random>
allure serve apps/ledger-live-mobile/e2e/allure-results
```

`allure serve` generates the HTML on the fly and starts a tiny web server; it is the fastest feedback loop. To produce a static report (e.g. to attach to a ticket):

```bash
allure generate apps/ledger-live-mobile/e2e/allure-results -o /tmp/report --clean
open /tmp/report/index.html
```

Cross-reference Part 5 Chapter 26 for the full Allure internals — the mobile setup reuses the same reporter conventions as desktop, so the mental model is identical. For the official specification, see the [Allure documentation](https://allurereport.org/docs/).

### 37.8 Logs on Failure

The fixture file `e2e/mobile/bridge/setup.ts` installs an `afterAll` hook that, on failure, pulls three diagnostic bundles and attaches them to Allure:

- **App logs** — the output of the React Native app's `console.log` and native logs, captured by Detox's `device.log` plugin.
- **Bridge logs** — the WebSocket traffic between the spec runner and the bridged app. Indispensable for "why did `selectMockDevice` fail".
- **Memory snapshots** — on iOS only, a compact memory graph dumped via `instruments` when the app crashes.

They land in `apps/ledger-live-mobile/e2e/artifacts/<run-id>/`. Open the Allure report and click into the failing test — the attachments tab lists each one by name.

When a CI failure is reported in Slack, your first move should be: download the artifact zip, open `app.log`, then open `bridge.log`. Three out of four failures become obvious from those two files alone.

### 37.9 Running a Single `it()` by TmsLink

Every `it()` in the Ledger suite is tagged with a Jira ticket via `$TmsLink`:

```typescript
it(
  `@B2CQA-604 • Send BTC - valid address, broadcast disabled`,
  async () => {
    // ...
  },
);
```

To run only that one test:

```bash
pnpm mobile e2e:test -c ios.sim.debug -- -t "B2CQA-604"
```

The `-t` pattern is a regex (case-insensitive by default for Jest), so `B2CQA-604` will match exactly one test unless there are duplicates — in which case `--testPathPattern` narrows it further.

**Performance tip:** combine with `--testPathPattern` to skip loading spec files that can't contain the test. Jest loads every matching file and then filters by name; if you know the ticket is in `specs/send/`, adding `--testPathPattern send` saves 2–5 seconds of file loading.

### 37.10 Speculos on Mobile

Chapter 24 (desktop) and Chapter 35 (mobile) both introduced Speculos. Mobile E2E has two modes:

**Mode A: `selectMockDevice` (bridge mock — no Speculos needed).**

```typescript
import { selectMockDevice } from "@bridge/client";

beforeAll(async () => {
  await selectMockDevice({ nano: "nanoX" });
});
```

The bridge intercepts all DMK calls from the app and answers them with canned signed payloads. This is the fastest mode and is used for flows where you don't care about on-device confirmation screens (e.g., portfolio display, settings).

**Mode B: Real Speculos container.**

A Speculos container (the same Docker image from Part 4) runs on the host exposing HTTP/WS on a port, typically `52619`. The app's DMK transport must be redirected to that host/port.

- **iOS simulator** — shares the host's network, so `http://localhost:52619` works from inside the simulator without any port-forwarding dance.
- **Android emulator** — has its own network namespace. `localhost` from the emulator is the emulator itself, not the host. You must forward:

```typescript
await device.reverseTcpPort(52619);
// or from a shell: adb reverse tcp:52619 tcp:52619
```

`device.reverseTcpPort()` is Detox's wrapper around `adb reverse`. After this call, the emulator's `localhost:52619` points at the host's Speculos container.

**When to pick which:**
- Use **`selectMockDevice`** when you're testing app-side logic and any valid-looking response is sufficient. 80% of specs are this.
- Use **real Speculos** when the test asserts against on-device screens (confirming an address matches what the device shows), when you need signature randomness to match a specific seed, or when you're validating a transaction-signing flow end-to-end.

The Chapter 38 CI workflow has a `speculos_device` input to pick which device firmware Speculos emulates — `nanoX`, `stax`, `flex`, etc.

### 37.11 Chapter 37 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Running & Debugging Mobile E2E Tests</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> You change a single line in a spec file and want to re-run it against the simulator as fast as possible. Which command best minimizes iteration time?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>pnpm mobile e2e:build -c ios.sim.debug && pnpm mobile e2e:test -c ios.sim.debug</code></button>
<button class="quiz-choice" data-value="B">B) <code>pnpm mobile e2e:test -c ios.sim.debug -- -t "B2CQA-604" --reuse --retries 0</code></button>
<button class="quiz-choice" data-value="C">C) <code>pnpm mobile e2e:test -c ios.sim.release -- --maxWorkers=4</code></button>
<button class="quiz-choice" data-value="D">D) <code>pnpm mobile e2e:build -c ios.sim.prerelease</code></button>
</div>
<p class="quiz-explanation">When only TypeScript changes, the native binary does not need to be rebuilt. <code>--reuse</code> skips re-installing the app, <code>-t</code> runs just the target test, and <code>--retries 0</code> surfaces the true failure instead of silently retrying. Option C is invalid because <code>maxWorkers</code> is pinned to 1 by the bridge constraint.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> What is the difference between <code>.env.mock</code> and <code>.env.mock.prerelease</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>.env.mock.prerelease</code> uses Metro bundling while <code>.env.mock</code> uses Hermes</button>
<button class="quiz-choice" data-value="B">B) Only the bundle ID changes; Firebase keys are identical</button>
<button class="quiz-choice" data-value="C">C) <code>.env.mock</code> points Firebase at the staging project; <code>.env.mock.prerelease</code> points it at the production project — Remote Config flags, analytics endpoints, and bundle IDs all change</button>
<button class="quiz-choice" data-value="D">D) They are the same file under two names, kept for legacy reasons</button>
</div>
<p class="quiz-explanation">The ENVFILE toggle is the mechanism by which CI validates against production Firebase during release gating (<code>production_firebase: true</code> in the workflow). Everything keyed off <code>FIREBASE_*</code> and the bundle ID flips at compile time.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> A test fails with “App is busy” even though nothing is visibly animating. What is the most likely cause and fix?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A long-poll or SSE connection keeps the network idle-check from ever settling — blacklist the URL with <code>device.setURLBlacklist([...])</code></button>
<button class="quiz-choice" data-value="B">B) The simulator is out of memory — reboot it</button>
<button class="quiz-choice" data-value="C">C) The spec is using <code>getByText</code> — switch to <code>by.id</code></button>
<button class="quiz-choice" data-value="D">D) Metro is not running — start it with <code>pnpm mobile start</code></button>
</div>
<p class="quiz-explanation">Detox waits for the app to be idle before each action. Persistent network sockets never become idle. <code>setURLBlacklist</code> tells the synchronization engine to ignore traffic matching the given patterns.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> You need to drive the app against a real Speculos Nano X on an Android emulator. Which step is required but easy to forget?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Rebuild the app with <code>-c android.emu.prerelease</code></button>
<button class="quiz-choice" data-value="B">B) Set <code>maxWorkers: 2</code> so Speculos and Detox don't share a worker</button>
<button class="quiz-choice" data-value="C">C) Call <code>selectMockDevice({ nano: "nanoX" })</code> before the test</button>
<button class="quiz-choice" data-value="D">D) Reverse the Speculos port to the emulator with <code>device.reverseTcpPort(52619)</code> (or <code>adb reverse tcp:52619 tcp:52619</code>)</button>
</div>
<p class="quiz-explanation">Android emulators have their own network namespace — <code>localhost</code> is the emulator itself. <code>adb reverse</code> (wrapped by Detox's <code>device.reverseTcpPort</code>) forwards the emulator's <code>localhost:52619</code> to the host, where Speculos is listening. iOS simulators share the host network, so no forwarding is needed there. Option C is the <em>opposite</em> of using real Speculos — it is the mock-bridge mode.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> Where do Allure results from a local mobile run land, and how do you view them?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>apps/ledger-live-mobile/coverage/</code> — open <code>index.html</code> directly</button>
<button class="quiz-choice" data-value="B">B) <code>apps/ledger-live-mobile/e2e/allure-results/</code> — view with <code>allure serve &lt;that path&gt;</code></button>
<button class="quiz-choice" data-value="C">C) They are only produced on CI; local runs don't generate Allure data</button>
<button class="quiz-choice" data-value="D">D) In <code>~/.allure/</code>, indexed by run timestamp</button>
</div>
<p class="quiz-explanation">The Jest Allure reporter writes per-test JSON plus attachment metadata into <code>e2e/allure-results/</code>. <code>allure serve</code> spins up a local web server that renders the report on the fly — the fastest feedback loop. <code>allure generate</code> produces a static site for sharing.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> Why does <code>jest.config.js</code> pin <code>maxWorkers: 1</code> for mobile E2E, and what is the CI's parallelism strategy instead?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Jest does not support multi-worker runs — it is a framework limitation</button>
<button class="quiz-choice" data-value="B">B) Detox can use multiple workers, but TypeScript compilation is single-threaded</button>
<button class="quiz-choice" data-value="C">C) Each Detox worker needs its own device and bridge port; parallel workers would collide on port 8099 and on the single booted simulator. CI parallelizes across <em>shards</em> (separate GitHub runners), not workers</button>
<button class="quiz-choice" data-value="D">D) It is a historical artifact — the value could be raised but nobody has tested it</button>
</div>
<p class="quiz-explanation">Mobile E2E is constrained at the device level: one simulator per process, one bridge WebSocket per simulator. Horizontal scaling therefore happens at the runner level — each CI shard is its own runner with its own simulator/emulator(s) (Chapter 38).</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Mobile CI & Test Sharding

<div class="chapter-intro">
A full mobile E2E pass serially on one device takes roughly 45–60 minutes — and that is only iOS. Running both platforms sequentially on every PR would add an hour to the feedback loop and frustrate every contributor. The solution is <strong>test sharding</strong>: split the test corpus into N independent buckets, run them on N parallel GitHub runners, then merge the Allure reports at the end. This chapter walks through the reusable workflow, the sharding math, the matrix strategy, and how to drive it all from your laptop with <code>gh workflow run</code>.
</div>

### 38.1 Mobile CI Overview

Mobile CI at Ledger Live is composed of three layers. Cross-reference Part 4 Chapter 17 for the general workflow framework; this chapter drills into the mobile specifics.

**Layer 1: PR / push workflows.**
- `test-mobile-e2e.yml` — the wrapper that fires on PR and dispatches the reusable workflow with a smoke-test filter.
- `mobile-build.yml` — produces APK/IPA artifacts for every push.

**Layer 2: The reusable core.**
- `test-mobile-e2e-reusable.yml` — called from both PR workflows and release workflows. This is where all the real work happens. The filename literally tells you this is a reusable workflow per GitHub's convention (see [reusable workflows docs](https://docs.github.com/en/actions/using-workflows/reusing-workflows)).

**Layer 3: Scheduled / manual triggers.**
- Cron trigger on the reusable workflow itself — runs the full suite Monday–Friday at 03:00 UTC against `develop`.
- `workflow_dispatch` trigger on the reusable workflow — what you use when you need to manually validate a branch (see [workflow_dispatch events](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_dispatch)).

Mental model: every mobile E2E run, regardless of origin, eventually calls `test-mobile-e2e-reusable.yml` with the same set of inputs. The difference is only which inputs each caller passes.

### 38.2 `test-mobile-e2e-reusable.yml` Walkthrough

Open `.github/workflows/test-mobile-e2e-reusable.yml`. The file has three distinct parts: **triggers + inputs** (top), **jobs** (middle), **reporting** (bottom).

#### The header

```yaml
name: "[Mobile] - E2E Only - Scheduled/Manual"
run-name: >
  @Mobile E2E • ${{ github.event_name == 'workflow_dispatch' && 'Manual' || github.event_name == 'schedule' && 'Scheduled' || github.event_name }} • ${{ inputs.tests_type || 'All' }} • ${{ inputs.ref || github.ref_name }} • ${{ github.actor }}

concurrency:
  group: >-
    mobile-e2e-${{ github.workflow_ref }}-${{ inputs.ref || github.sha }}-${{ github.event_name == 'schedule' && github.event.schedule || inputs.tests_type }}-${{ inputs.test_filter || '' }}
  cancel-in-progress: false
```

The `run-name` template surfaces trigger, scope, ref, and actor in the Actions UI — so a quick glance at the runs list tells you who ran what. The `concurrency` group is intentionally granular (workflow + ref + test type + filter) with `cancel-in-progress: false` — two manual runs against the same branch with the same filter are serialized rather than cancelling each other.

#### Inputs

There are two trigger blocks: `workflow_dispatch` (manual) and `workflow_call` (invoked by another workflow). They mirror each other with minor differences:

| Input | Type | Default | What it toggles |
|-------|------|---------|------------------|
| `ref` | string | current branch | Git ref to checkout and test. Leave blank to test the branch hosting the workflow file. |
| `test_filter` | string | `""` | Test filter pattern. Comma or pipe separates alternatives (e.g. `"@bitcoin,@family-evm"` or `"addAccount deeplinks"`). Passed through to Jest's name/path filter logic. |
| `tests_type` | choice | `iOS & Android` | Which platforms to run. Options: `Android Only`, `iOS Only`, `iOS & Android`. |
| `speculos_device` | choice | `nanoX` | Which Speculos device firmware to emulate. Options: `nanoS`, `nanoSP`, `nanoX`, `stax`, `flex`, `nanoGen5`. `nanoS` pins to a specific legacy Speculos image tag. |
| `production_firebase` | boolean | `false` | Build with `.env.mock.prerelease` to target production Firebase. Required for release gating. |
| `enable_broadcast` | boolean | `false` | Unset `DISABLE_TRANSACTION_BROADCAST`. When false, the app's transaction-submission path is stubbed (no real chain hits). |
| `export_to_xray` | boolean | `false` | At the end of the run, POST Allure results to Xray as a test execution. |
| `test_execution_android` | string | `""` | Existing Xray test execution key to attach Android results to (otherwise a new one is created). |
| `test_execution_ios` | string | `""` | Same, for iOS. |
| `smoke_tests` | boolean | `false` | Prepend `@smoke` to the test filter so only smoke-tagged tests run. |
| `enable_wallet40` | boolean | `true` | Sets `E2E_ENABLE_WALLET40` env var to `1` (default) or `0` — flips the Wallet 4.0 feature flag used inside the app. |
| `generate_ai_artifacts` | boolean | `false` | Produce an AI-analysis bundle after the run (used by a separate summarization job). |

**What each toggles downstream.** The YAML then computes env vars from these inputs at the top of the jobs:

```yaml
env:
  SPECULOS_IMAGE_TAG: ghcr.io/ledgerhq/speculos:${{ inputs.speculos_device == 'nanoS' && '1be65b91a8e0691866f880fd437ac05fce78af9d' || 'latest' }}
  SPECULOS_DEVICE: ${{ inputs.speculos_device || 'nanoX' }}
  E2E_ENABLE_WALLET40: ${{ inputs.enable_wallet40 == false && '0' || '1' }}
  ANDROID_APK_PATH: apps/ledger-live-mobile/android/app/build/outputs/apk/detox${{ inputs.production_firebase == true && 'PreRelease' || '' }}/app-x86_64-detox${{ inputs.production_firebase == true && 'PreRelease' || '' }}.apk
```

Notice how `production_firebase` changes the **path** to the APK — the `PreRelease` suffix in the filename maps directly to the Gradle build type that was produced with `ENVFILE=.env.mock.prerelease`.

### 38.3 Triggers

Three concrete ways this workflow runs:

**1. Scheduled cron.**

```yaml
on:
  schedule:
    - cron: "0 3 * * 1-5"   # Mon–Fri at 03:00 UTC
```

Runs against `develop` (or whichever branch the workflow file lives on), full suite, Wallet 4.0 enabled. The nightly sanity check.

**2. Manual `workflow_dispatch`.** Opened from the Actions tab UI or via `gh workflow run` (see 38.9). You fill in the inputs; most commonly you pick a `ref` (branch), a `tests_type`, and sometimes a `test_filter`.

**3. Called from release workflows.** Release and hotfix branches invoke the workflow via `workflow_call` with `production_firebase: true` to gate on production-Firebase behavior.

The concurrency group key differentiates these so that a scheduled run does not cancel a manual run testing the same ref.

### 38.4 Environment Variables Table

Every interesting variable that flows from workflow inputs down to the Detox process:

| Variable | Source | Read by | Effect |
|----------|--------|---------|--------|
| `SEED` | Repo secret `SEED_QAA_B2C` | App (via bridge) | The 24-word seed the emulated Speculos uses. Determines every derived address. |
| `MOCK` | Set on the test step | App runtime | Enables mock mode (skip real crypto network calls). Desktop concept; mobile uses `selectMockDevice` in specs. |
| `PRODUCTION` | Input `production_firebase` | `e2e-ci.mjs` | When `true`, orchestrator flips target from `release` to `prerelease`. |
| `SHARD_INDEX` | `matrix.shard` | Detox / Jest | 1-based index of the current shard (used for naming artifacts and for Jest's `--shard N/M` if used). |
| `SHARD_TOTAL` | `matrix.total` | Detox / Jest | Total number of shards in the matrix. |
| `SPECULOS_DEVICE` | Input `speculos_device` | Speculos container | Which device firmware image Speculos loads. |
| `SPECULOS_API_PORT` | Computed per-shard | Speculos container, bridge | Port the Speculos HTTP/WS API listens on. Typically `52619`; per-shard variants prevent collisions on multi-emulator runners. |
| `SPECULOS_IMAGE_TAG` | Computed from `speculos_device` | Docker pull | `latest` for most devices; pinned SHA for legacy Nano S. |
| `E2E_ENABLE_WALLET40` | Input `enable_wallet40` | App feature-flag loader | `"1"` enables the Wallet 4.0 UI; `"0"` keeps the legacy flow. |
| `E2E_ENABLE_BROADCAST` | Derived from `enable_broadcast` (inverse of `DISABLE_TRANSACTION_BROADCAST`) | App transaction layer | When disabled, `sendTransaction` no-ops rather than broadcasting. |
| `ENVFILE` | Computed in Gradle/Xcode build from `production_firebase` | Native build | Picks `.env.mock` vs `.env.mock.prerelease`. |
| `INPUTS_TEST_FILTER` | `steps.test-filter.outputs.filter` | Spec runner | The resolved filter string after optional `@smoke` prefix. |
| `SWAP_API_BASE` | Repo env | App runtime | Points the swap feature at a stubbed backend. |
| `DEVICE_INFO` | Computed per-platform | Allure metadata | Human-readable device string (e.g. `iPhone 15 (iOS 17.4)`), attached as an Allure parameter. |

The flow from input to test process is literally:

```
workflow_dispatch inputs
      ↓
env: block in reusable workflow
      ↓
job-level env: block
      ↓
step-level env: block (for `run-ios`, `run-android`)
      ↓
pnpm mobile e2e:ci (spawned child process)
      ↓
detox test (spawned child process)
      ↓
Jest (runs specs)
      ↓
process.env accessed in setup.ts / bridge / specs
```

### 38.5 Sharding

#### Why sharding exists

A full mobile E2E run serialized on one device takes ~45–60 minutes. Running both platforms on every PR would cost 90+ minutes of wall time. Sharding splits the N test files across K runners so wall time becomes `ceil(N/K) × per-test-time` — for K=12, that is roughly 5–8 minutes per shard, all running in parallel.

Sharding at the runner (job) level also sidesteps the `maxWorkers: 1` constraint from Chapter 37 — each runner is its own OS, its own simulator/emulator, its own bridge port. The pieces are isolated cleanly.

#### The `generate-shards-matrix` composite action

`tools/actions/composites/generate-shards-matrix/action.yml` (~230 lines) computes:
1. The list of spec files to run (after applying the filter).
2. How many shards to use (`SHARD_COUNT_IOS`, `SHARD_COUNT_ANDROID`).
3. The per-shard file assignments.
4. A sensible per-shard timeout.

**Finding the test files.**

```bash
TEST_FILES=$(node apps/ledger-live-mobile/scripts/shard-tests.mjs "${{ inputs.test_filter }}" "${{ inputs.test_directory }}")
```

The script walks `e2e/mobile` recursively for `*.spec.ts` files, excluding `*.skip.spec.ts`, and returns them space-separated.

**The base-shard formula.**

```bash
TEST_COUNT=$(echo "${{ steps.test-files.outputs.test_files_for_sharding }}" | wc -w)
EMULATORS_PER_RUNNER=3
BASE_SHARD=$(( TEST_COUNT > 0 ? (TEST_COUNT + EMULATORS_PER_RUNNER - 1) / EMULATORS_PER_RUNNER : 1 ))
```

Read: `BASE_SHARD = ceil(TEST_COUNT / 3)`. The `3` is the number of simulators/emulators each runner boots in parallel (one runner drives three devices — see the iOS boot step in the workflow). So one shard corresponds to one runner, and one runner can chew through roughly 3 specs concurrently if Jest were allowed to parallelize — but since `maxWorkers: 1`, the 3 simulators are currently used sequentially (reserved for future parallelism).

**The caps.**

```bash
if [ "${{ inputs.event_name }}" = "schedule" ]; then
  SHARD_COUNT_IOS=$(( BASE_SHARD < 12 ? BASE_SHARD : 12 ))
  SHARD_COUNT_ANDROID=$(( BASE_SHARD < 12 ? BASE_SHARD : 12 ))
elif [[ "${{ inputs.ref }}" == *"release"* || "${{ inputs.ref }}" == *"hotfix"* ]]; then
  SHARD_COUNT_IOS=$(( BASE_SHARD < 12 ? BASE_SHARD : 12 ))
  SHARD_COUNT_ANDROID=$(( BASE_SHARD < 12 ? BASE_SHARD : 12 ))
else
  SHARD_COUNT_IOS=$(( BASE_SHARD < 3 ? BASE_SHARD : 3 ))
  SHARD_COUNT_ANDROID=$(( BASE_SHARD < 12 ? BASE_SHARD : 12 ))
fi
```

Two tiers:
- **Scheduled or release/hotfix:** up to 12 shards each for iOS and Android. Maximum parallelism when wall-time matters most.
- **Manual dispatch on a regular branch:** up to 3 iOS shards (macOS runners are scarce and expensive), up to 12 Android shards (Linux runners are abundant and cheap).

So the concrete table reads:

| Event | iOS cap | Android cap | Why |
|-------|---------|-------------|-----|
| `schedule` | 12 | 12 | Nightly full pass, wall time matters most |
| Manual on `release/*` or `hotfix/*` | 12 | 12 | Release gating needs fast feedback |
| Manual on any other ref | 3 | 12 | Save macOS runner budget for release branches |

#### The `shard-tests.mjs` algorithm

`apps/ledger-live-mobile/scripts/shard-tests.mjs` implements **greedy bin-packing by past duration**. The full logic:

```js
// Sort files by estimated duration (from timing data, longest first)
filesWithTiming.sort((a, b) => {
  if (b.duration !== a.duration) {
    return b.duration - a.duration;
  }
  return compareStrings(a.file, b.file);  // Determinism on ties
});

// Separate into "known timing" and "zero timing" buckets
const testsWithTiming = filesWithTiming.filter(f => f.duration > 0);
const testsWithZeroTiming = filesWithTiming.filter(f => f.duration === 0);

// Initialize empty shards
const shards = Array.from({ length: shardTotal }, () => ({ files: [], totalDuration: 0 }));

// Greedy: assign each known-timing test to the shard with the minimum total time so far
for (const { file, duration } of testsWithTiming) {
  let minShardIndex = 0;
  let minTotalTime = Infinity;
  for (let i = 0; i < shardTotal; i++) {
    if (shards[i].totalDuration < minTotalTime) {
      minTotalTime = shards[i].totalDuration;
      minShardIndex = i;
    }
  }
  shards[minShardIndex].files.push(file);
  shards[minShardIndex].totalDuration += duration;
}

// Round-robin the zero-timing tests (new files, no history yet)
for (let i = 0; i < testsWithZeroTiming.length; i++) {
  const shardToUse = i % shardTotal;
  shards[shardToUse].files.push(testsWithZeroTiming[i].file);
}

return shards[shardIndex - 1]?.files || [];
```

**Why this beats naive round-robin.** A naive "test #N goes to shard N % K" ignores duration. If specs 1, 5, 9 are 10-minute behemoths and everything else is 2 minutes, round-robin piles all three onto shard 1 and leaves shards 2 and 3 idle for eight minutes. Greedy longest-first minimizes the max-shard duration, which is the critical-path wall time of the whole run.

**Where timing data comes from.** Each previous run of the shard uploads `e2e-test-results-<platform>-shard-N.json` (from Jest's `--json --outputFile`). The generate-shards action restores these from an S3 cache keyed by the spec-directory hash and merges them. On the first run after adding new specs, those specs are in the "zero timing" bucket and get round-robin'd; the next run picks up their measured durations.

**Determinism.** Two runs with the same test set and the same timing data produce identical shard assignments, because ties break on file path. This is essential for artifact naming to be stable.

#### A concrete example: 27 specs, scheduled run

- `TEST_COUNT = 27`, `BASE_SHARD = ceil(27/3) = 9`.
- Scheduled → cap is 12, so `SHARD_COUNT_IOS = min(9, 12) = 9`.
- Assume timing data: 3 specs at 8 min each, 6 specs at 4 min, 18 specs at 2 min.

Greedy walk:
1. Sort descending by duration.
2. First iteration: three 8-min specs go to shards 1, 2, 3 (each empty → tie-broken by lowest index).
3. Then six 4-min specs: each picks the lightest shard. After iteration, shards 1–3 have 8 min, shards 4–9 have 0 min → the six 4-min specs fill shards 4–9 first.
4. After step 3, shards 1–3 are at 8, shards 4–9 are at 4. Eighteen 2-min specs then fill from lightest: shards 4–9 get two each (4→6→8), then two specs go each to shards 1–3 bringing them to 12 min.

Final per-shard: 9 shards each at ~10–12 minutes. The unbalanced naive version (9 shards of 3 specs each, round-robin) would have one shard at 24 min and another at 6.

### 38.6 Matrix Strategy

The matrix is generated as JSON by the composite action and consumed by the iOS and Android jobs:

```yaml
detox-tests-ios:
  strategy:
    fail-fast: false
    matrix:
      include: ${{ fromJSON(needs.determine-builds.outputs.matrix_ios) }}
```

`matrix_ios` is a JSON array where each element is an object like:

```json
{ "shard": 3, "total": 9, "files": "e2e/mobile/specs/send/btc.spec.ts e2e/mobile/specs/send/eth.spec.ts" }
```

`include` expands each object into a separate job run, and fields become accessible as `matrix.shard`, `matrix.total`, `matrix.files`. Inside the step:

```yaml
- name: Set shard test files from pre-computed matrix
  run: |
    echo "SHARD_TEST_FILES<<EOF" >> $GITHUB_ENV
    echo "${{ matrix.files }}" >> $GITHUB_ENV
    echo "EOF" >> $GITHUB_ENV
```

Then the actual Detox invocation:

```yaml
- name: Run iOS Detox shard ${{ matrix.shard }}/${{ matrix.total }}
  id: run-ios
  timeout-minutes: ${{ fromJSON(needs.determine-builds.outputs.ios_timeout) }}
  env:
    SHARD_INDEX: ${{ matrix.shard }}
    # ... other env
  run: pnpm run mobile e2e:ci -p ios -t --e2e $([[ "$PRODUCTION" == "true" ]] && printf %s '--production') $SHARD_TEST_FILES --outputFile=artifacts/e2e-test-results-ios-shard-${{ matrix.shard }}.json
```

**`fail-fast: false` rationale.** A failure in shard 3 should not cancel shards 4–12. The test corpus is independent per-shard, and you want the full signal — "which 27 tests failed out of 300?" — not just the first one. Downstream, the `aggregate-shard-results` composite collects per-shard statuses (`status_1` through `status_12`) and computes a single final status.

### 38.7 Per-Shard Lifecycle

Each iOS shard runs these steps (Android is analogous with AVD-specific setup):

```
┌──────────────────────────────────────────────────────────┐
│ iOS Shard N/M                                            │
├──────────────────────────────────────────────────────────┤
│ 1. actions/checkout @ inputs.ref                         │
│ 2. setup-caches (pnpm, mise, nx, AWS credentials)        │
│ 3. Set SHARD_TEST_FILES from matrix.files                │
│ 4. Install scrcpy (Android screen capture tool)          │
│ 5. Boot 3 iOS simulators in background (nohup)           │
│ 6. pnpm install (filtered to mobile + cli packages)      │
│ 7. Detox post-install                                    │
│ 8. Download native build from S3 (from build-ios job)    │
│ 9. Download JS bundle from S3                            │
│ 10. Copy jsbundle into native app directory              │
│ 11. Build @ledgerhq/live-cli (Speculos helper)           │
│ 12. Pull Speculos image + coin apps                      │
│ 13. Set DISABLE_TRANSACTION_BROADCAST from inputs        │
│ 14. Compute DEVICE_INFO (simulator model + iOS version)  │
│ 15. pnpm run mobile e2e:ci -p ios -t --e2e $SHARD_FILES  │
│ 16. Upload test artifacts (Allure JSON, screenshots...)  │
│ 17. Upload shard timing JSON for next run's caching      │
│ 18. Set job output status_<N>                            │
│ 19. Delete created simulators                            │
└──────────────────────────────────────────────────────────┘
```

Step 15 is where the actual testing happens; everything before it is setup, everything after is cleanup and reporting. Note that step 5 boots **three** simulators — the infrastructure is ready for per-shard worker parallelism if the bridge constraint is ever lifted.

### 38.8 Artifact Flow

Artifacts fan out from shards, get merged, then get published.

**Per-shard outputs:**
- `ios-test-artifacts-N` (zip of `e2e/mobile/artifacts/`) — contains Allure JSON, screenshots, videos, logs.
- A timing-data JSON uploaded as a separate artifact named `<ios_timing_cache_key>-N` — consumed by the next run's sharding step.

**Merge step (`allure-report-ios` job):**

```yaml
- name: Download Allure Report
  uses: actions/download-artifact@...
  with:
    path: ios-test-artifacts
    pattern: ios-test-artifacts*
    merge-multiple: true
- uses: LedgerHQ/ledger-live/tools/actions/composites/upload-allure-report@develop
  with:
    platform: ios-e2e
    path: ios-test-artifacts
```

`merge-multiple: true` is the glue — all 12 per-shard artifacts get extracted into the same directory, effectively concatenating their Allure result JSONs. The composite action then:
1. Generates the Allure HTML report.
2. Uploads it to the internal Allure server (accessed via `ALLURE_USERNAME` / `ALLURE_LEDGER_LIVE_PASSWORD`).
3. Outputs a `report-url` that gets posted to Slack.

**Xray sync (optional, `upload-to-xray` job):**

If `export_to_xray: true`, the same Allure JSONs are reformatted by `e2e/mobile/xray.formater.sh`, authenticated to the Xray Cloud API, and POSTed to `/api/v2/import/execution`. The resulting Xray test execution key is written to the job summary.

**Timing cache update.** The per-shard timing uploads get restored on the next run — closing the feedback loop on the greedy bin-packer.

### 38.9 Kicking Off a Custom Run

Three paths, each with its own friction level.

**1. GitHub Actions UI.** Browse to Actions → "[Mobile] - E2E Only - Scheduled/Manual" → "Run workflow". Fill in the inputs. Slowest but obvious.

**2. The `gh` CLI.** The fastest path for repeat runs:

```bash
gh workflow run test-mobile-e2e-reusable.yml \
  -f ref=feat/qaa-702-add-cosmos-staking \
  -f tests_type="iOS & Android" \
  -f test_filter='@bitcoin Cosmos' \
  -f speculos_device=nanoX \
  -f smoke_tests=false
```

Watch it from the terminal:

```bash
gh run watch
```

**3. Common recipes.**

Run only one ticket's tests on iOS against a PR branch:

```bash
gh workflow run test-mobile-e2e-reusable.yml \
  -f ref=feat/qaa-702 \
  -f tests_type="iOS Only" \
  -f test_filter='-t "B2CQA-604"'
```

> **Note:** the `test_filter` here is passed verbatim to the shard-tests script, which uses the filter as a regex over file paths and contents. Jest-specific flags like `-t` make it through because `shard-tests.mjs` also greps the file content for the pattern.

Validate a release candidate against production Firebase:

```bash
gh workflow run test-mobile-e2e-reusable.yml \
  -f ref=release/3.50.0 \
  -f tests_type="iOS & Android" \
  -f production_firebase=true \
  -f enable_broadcast=false
```

Smoke-only run after a dependency bump:

```bash
gh workflow run test-mobile-e2e-reusable.yml \
  -f ref=chore/bump-dmk \
  -f tests_type="iOS & Android" \
  -f smoke_tests=true
```

### 38.10 Reading a Failed Run

The recipe for triaging a red mobile CI:

1. Click the failing run in GitHub Actions. Find the red shard (e.g., "iOS E2E Tests (4, 12)").
2. Scroll to the `Run iOS Detox shard 4/12` step. The exit code tells you it was a test failure (vs a setup error). Note the `DEVICE_INFO` line.
3. Check the job summary for the Allure report URL (posted by the `allure-report-ios` job if it completed). If that job ran, click straight into Allure.
4. In Allure, navigate to "Failed" and find the specific `it()`. Open it.
5. Attachments panel: `app.log`, `bridge.log`, `device.log`, the failure screenshot, the video (iOS) or screen recording (Android).
6. If Allure didn't render (e.g., the shard timed out entirely), go back to the run, download the `ios-test-artifacts-4` zip directly. Unzip locally — you get the same Allure inputs plus raw Detox artifacts.
7. Reproduce locally:

```bash
pnpm mobile e2e:build -c ios.sim.release
pnpm mobile e2e:test -c ios.sim.release \
  -- -t "<exact it() name from Allure>" \
  --loglevel trace
```

8. If the test passes locally but fails in CI: suspect sharding ordering (state bleed from a test earlier in the shard). Look at which specs ran before yours in the shard's output — they're listed in the step log — and re-run them locally in the same order.

### 38.11 Smoke vs Full

The `smoke_tests` input works via Jest's name filter:

```yaml
- name: Prepare test filter
  id: test-filter
  run: |
    BASE_FILTER="${{ inputs.test_filter }}"
    if [ -n "$BASE_FILTER" ]; then
      BASE_FILTER="${BASE_FILTER//,/|}"
    fi
    if [ "${{ inputs.smoke_tests }}" = "true" ]; then
      if [ -n "$BASE_FILTER" ]; then
        echo "filter=@smoke $BASE_FILTER" >> $GITHUB_OUTPUT
      else
        echo "filter=@smoke" >> $GITHUB_OUTPUT
      fi
    else
      echo "filter=$BASE_FILTER" >> $GITHUB_OUTPUT
    fi
```

Any `it()` whose title contains `@smoke` runs; everything else is skipped. The tag is a convention maintained in the spec source:

```typescript
it(
  `@smoke @B2CQA-604 • Send BTC - valid address, broadcast disabled`,
  async () => { /* ... */ },
);
```

Smoke runs typically contain 10–15% of the full corpus — chosen for breadth (one test per coin family, one per major feature) rather than depth. Wall time drops from ~45 min per platform to ~10 min per platform. PRs run smoke only; scheduled runs do the full corpus.

`describe.skip` patterns are an older mechanism that is still present in a few files — they unconditionally skip a whole describe block. Prefer tags over `describe.skip` for new work because tags compose (you can combine `@smoke @bitcoin` to target a specific slice), whereas `describe.skip` is binary.

### 38.12 Production vs Staging Firebase in CI

`production_firebase: true` flips four things:
1. Build config: `ENVFILE=.env.mock.prerelease` → production Firebase keys in the binary.
2. Binary location: Gradle outputs `app-x86_64-detoxPreRelease.apk` instead of `app-x86_64-detox.apk`; the workflow env paths use a ternary to pick the right one.
3. Cache key suffix: `prod-2` instead of `3` — separate native-build cache so production and staging binaries don't overwrite each other.
4. Orchestrator target: `--production` flag passed to `e2e-ci.mjs` → Detox uses the `*.prerelease` config.

**When you need it:**
- **Release / hotfix gating.** Before shipping a release, you need confidence that the binary that runs against production Firebase still passes. A staging-only green is insufficient because Remote Config flag values differ between projects.
- **Production-only feature validation.** Some features (e.g., Remote Config-controlled rollouts) only activate when production Remote Config returns specific values.
- **Incident reproduction.** If a user reports a bug that cannot be reproduced against staging Firebase, switch to production to see if it's Remote Config driven.

**When you do not need it:**
- **Every PR.** The CI cost is 2× (separate native build, separate cache line) and staging is representative for 95% of changes. PR workflows never set `production_firebase: true`.
- **Pure UI work.** If the change doesn't touch anything Firebase-driven, staging suffices.

### 38.13 End-to-End Sharding Pipeline — ASCII Sequence

The full cross-job handoff from trigger to published report:

```
  DEVELOPER            GITHUB ACTIONS                          DETOX                ALLURE / XRAY
  ─────────            ──────────────                          ─────                ─────────────
     │                         │                                  │                       │
     │  gh workflow run -f …   │                                  │                       │
     ├────────────────────────►│                                  │                       │
     │                         │                                  │                       │
     │                 ┌───────┴───────────┐                      │                       │
     │                 │ determine-builds  │                      │                       │
     │                 │ (ledger-live-med) │                      │                       │
     │                 │                   │                      │                       │
     │                 │  checkout ref     │                      │                       │
     │                 │  compute filter   │                      │                       │
     │                 │  shard-tests.mjs  │──► read test dir     │                       │
     │                 │                   │◄── N spec files      │                       │
     │                 │  restore timing   │──► S3 cache          │                       │
     │                 │  cache (AWS)      │◄── per-spec durations│                       │
     │                 │  greedy bin-pack  │                      │                       │
     │                 │  emit matrix_ios  │                      │                       │
     │                 │  emit matrix_and  │                      │                       │
     │                 └───────┬───────────┘                      │                       │
     │                         │                                  │                       │
     │                 ┌───────┴──────────────┐                   │                       │
     │                 │ build-ios (macOS)    │                   │                       │
     │                 │ build-android (linux)│                   │                       │
     │                 │                      │                   │                       │
     │                 │  compile + bundle    │                   │                       │
     │                 │  upload to S3 cache  │                   │                       │
     │                 └───────┬──────────────┘                   │                       │
     │                         │                                  │                       │
     │                 ┌───────┴──────────────────────┐           │                       │
     │                 │ detox-tests-ios matrix       │           │                       │
     │                 │ [shard 1/9][shard 2/9] … [9] │           │                       │
     │                 │ fail-fast: false             │           │                       │
     │                 │ (9 parallel macOS runners)   │           │                       │
     │                 │                              │           │                       │
     │                 │  per shard:                  │           │                       │
     │                 │   ├─ checkout + install      │           │                       │
     │                 │   ├─ download binary from S3 │           │                       │
     │                 │   ├─ boot 3 simulators       │           │                       │
     │                 │   ├─ pull speculos image     │           │                       │
     │                 │   ├─ run e2e-ci.mjs          │──────────►│ detox test            │
     │                 │   │                          │           │  run subset           │
     │                 │   │                          │           │  capture artifacts    │
     │                 │   │                          │◄──────────│ exit code N           │
     │                 │   ├─ upload ios-test-arts-K  │           │                       │
     │                 │   ├─ upload timing JSON      │           │                       │
     │                 │   ├─ set status_<K> output   │           │                       │
     │                 │   └─ delete simulators       │           │                       │
     │                 └───────┬──────────────────────┘           │                       │
     │                         │                                  │                       │
     │                 ┌───────┴────────────────┐                 │                       │
     │                 │ allure-report-ios      │                 │                       │
     │                 │ allure-report-android  │                 │                       │
     │                 │                        │                 │                       │
     │                 │  download all shard    │                 │                       │
     │                 │  artifacts, merge      │                 │                       │
     │                 │  generate HTML         │                 │                       │
     │                 │  upload to server      │────────────────────────────────────────►│ Allure web UI
     │                 │  aggregate status_1…12 │                 │                       │
     │                 └───────┬────────────────┘                 │                       │
     │                         │                                  │                       │
     │                 ┌───────┴────────────────┐                 │                       │
     │                 │ upload-to-xray         │                 │                       │
     │                 │ (if export_to_xray)    │                 │                       │
     │                 │                        │                 │                       │
     │                 │  format allure→xray    │                 │                       │
     │                 │  POST /import/execution│────────────────────────────────────────►│ Xray test exec
     │                 └───────┬────────────────┘                 │                       │
     │                         │                                  │                       │
     │                 ┌───────┴────────────────┐                 │                       │
     │                 │ report-on-slack        │                 │                       │
     │                 │   post summary + URLs  │                 │                       │
     │                 └───────┬────────────────┘                 │                       │
     │                         │                                  │                       │
     │ ◄───────────────────────┤  notification in #qa-results     │                       │
     │                         │                                  │                       │
```

Follow the arrows top-to-bottom: trigger → determine matrix → build → sharded test → merge → report. The critical insight is that **all nine iOS shards (or twelve for scheduled/release) run truly in parallel** — the wall time of the detox-tests-ios job is the wall time of the slowest shard, not the sum of all shards. Hence the greedy bin-packer's job: minimize the maximum.

### 38.14 Chapter 38 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Mobile CI & Test Sharding</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> 27 specs are present, a manual run is triggered on <code>feat/qaa-702</code> (not a release or hotfix branch). How many iOS shards and Android shards will the matrix contain?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) iOS 27, Android 27 — one shard per spec</button>
<button class="quiz-choice" data-value="B">B) iOS 3, Android 9 — iOS is capped to 3 for non-release manual runs; Android uses BASE_SHARD = ceil(27/3) = 9</button>
<button class="quiz-choice" data-value="C">C) iOS 9, Android 9 — BASE_SHARD applies equally to both</button>
<button class="quiz-choice" data-value="D">D) iOS 12, Android 12 — the hard cap is always used</button>
</div>
<p class="quiz-explanation">On manual runs on non-release branches, iOS is capped at 3 (to preserve scarce macOS runners), Android at 12. BASE_SHARD = ceil(TEST_COUNT / 3) = 9. So iOS = min(9, 3) = 3, Android = min(9, 12) = 9.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> Why does <code>shard-tests.mjs</code> sort tests by duration descending before placing them into shards?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Alphabetical stability for reproducible CI logs</button>
<button class="quiz-choice" data-value="B">B) Jest requires tests to be provided in a specific order</button>
<button class="quiz-choice" data-value="C">C) Greedy bin-packing is most effective when the largest items are placed first — this minimizes the maximum shard duration, which is the wall-clock critical path</button>
<button class="quiz-choice" data-value="D">D) It optimizes cache locality on the runner</button>
</div>
<p class="quiz-explanation">Classic greedy bin-packing optimization: place the largest items first so small items can fill the remaining gaps. Placing a 10-minute test on a nearly-full shard adds 10 min to its total; placing it on an empty shard starts it fresh. Always starting with the largest test makes the result near-optimal.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> In the matrix strategy for the iOS tests job, why is <code>fail-fast: false</code> set?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Shards are independent; one shard failing should not cancel the others because QA needs the full failure signal across the whole corpus</button>
<button class="quiz-choice" data-value="B">B) GitHub's default for reusable workflows is true and the team prefers the opposite</button>
<button class="quiz-choice" data-value="C">C) Fail-fast is incompatible with Detox</button>
<button class="quiz-choice" data-value="D">D) It is required for <code>matrix.include</code> to work</button>
</div>
<p class="quiz-explanation">If fail-fast were true, a shard-3 failure would cancel shards 4–12 mid-run, and the Allure report would be missing their data. Aggregating all statuses and rendering a full report is more valuable than saving a few minutes of runner time.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> Which combination of inputs should you use to validate a release candidate branch against production Firebase, running the full suite on both platforms?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>ref=release/3.50.0, tests_type="iOS Only", smoke_tests=true</code></button>
<button class="quiz-choice" data-value="B">B) <code>ref=develop, production_firebase=true</code></button>
<button class="quiz-choice" data-value="C">C) <code>ref=release/3.50.0, tests_type="iOS & Android"</code> and no other changes</button>
<button class="quiz-choice" data-value="D">D) <code>ref=release/3.50.0, tests_type="iOS & Android", production_firebase=true, smoke_tests=false</code></button>
</div>
<p class="quiz-explanation">Release gating needs the full suite (no smoke), both platforms, and production Firebase (<code>.env.mock.prerelease</code>). Because the ref contains <code>release</code>, both platforms automatically get the 12-shard cap — fast feedback for the release.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> How does the workflow route a value from <code>workflow_dispatch</code> input down into the actual Detox process environment?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Via a shared GitHub Actions secret that Detox reads at runtime</button>
<button class="quiz-choice" data-value="B">B) Through a chain: <code>inputs.*</code> → workflow-level <code>env</code> → job-level <code>env</code> → step-level <code>env</code> → child process <code>process.env</code>, each layer referencing the previous</button>
<button class="quiz-choice" data-value="C">C) Automatically — GitHub Actions injects all inputs as env vars by default</button>
<button class="quiz-choice" data-value="D">D) Only via <code>GITHUB_ENV</code> file writes; <code>env:</code> blocks are not supported for reusable workflows</button>
</div>
<p class="quiz-explanation">Each layer is explicit. The top-level <code>env:</code> block uses <code>${{ inputs.* }}</code> expressions; step-level <code>env:</code> blocks reference those plus matrix values. The spawned Detox process inherits the step's environment. Inputs are not auto-exposed as env — you must wire them.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> A test passes locally on your machine but fails consistently in CI shard 7. What is the most likely cause, and what is the first thing to check?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) GitHub runners are slower; bump the test timeout and retry</button>
<button class="quiz-choice" data-value="B">B) Allure is dropping events; re-run the workflow</button>
<button class="quiz-choice" data-value="C">C) Another spec earlier in shard 7 leaves state that breaks your test; look at the shard's step log to see which specs ran before yours, then reproduce them locally in the same order</button>
<button class="quiz-choice" data-value="D">D) Speculos is running a different firmware on CI; you cannot reproduce locally without that firmware</button>
</div>
<p class="quiz-explanation">The greedy bin-packer assigns specs deterministically but not alphabetically — which specs share a shard with yours depends on all their durations. State bleed (residual Redux state, undelivered async actions, device state not reset) is the #1 cause of "green locally, red in shard N". The shard's step log lists the spec file order; replay it locally to trigger the same interaction.</p>
</div>

<div class="quiz-score"></div>
</div>

---
## Your Daily Mobile Workflow: From Ticket to PR

<div class="chapter-intro">
You have learned the React Native primitives, the Detox toolchain, the matcher DSL, the bridge protocol, the singleton page-object pattern, the sharding algorithm, the Allure pipeline, and the CI graph. This chapter stitches all of that into the <strong>workflow</strong> — the exact, repeatable sequence you will execute every single day as an E2E Mobile engineer at Ledger. It mirrors Chapter 28 (desktop) but tracks the mobile realities: a Jira QAA ticket, a B2CQA Xray test, a Detox build, a Metro bundle, an Allure upload, and an iOS/Android CI matrix. Follow it every time. The steps do not change just because the ticket seems small.
</div>

### 39.1 The Ticket Lifecycle at Ledger Live

Every piece of work you do on the mobile E2E suite starts in Jira and ends in an Xray traceability update. The lifecycle has **three ticket shapes** — learn to distinguish them on sight.

| Ticket type | Project | Example | Purpose |
|---|---|---|---|
| **QAA ticket** | `QAA` | `QAA-702` | A work item you pick up in your sprint — "automate X", "cover regression Y", "fix flake in Z". |
| **Xray test case** | `B2CQA` | `B2CQA-604` | A test scenario in Xray — reusable across platforms (LLD, LLM, LLC). |
| **Bug** | `LIVE` (tagged with `LLM` / `LLD` labels) | `LIVE-12345` | A product defect. Rarely your starting point, but you link it in `$BugLink(...)` when reproducing. |

The **QAA ticket tells you what to do**. The **B2CQA ticket tells you what to verify**. These are never the same ticket. A single QAA-XXX can cover one B2CQA, or it can bundle five. Read carefully.

```
QAA-702  ─────linked to─────►  B2CQA-604  (Swap history export — ERC20 receive)
 "meta: automate this test"      "scenario: after a swap, user clicks 'Export
                                  history' and CSV contains ERC20 row"
```

The traceability chain below is the reason this distinction exists:

```
QAA-702 (sprint ticket)
   │
   │ links to
   ▼
B2CQA-604 (Xray test case)          ◄── PM and QA leads see coverage here
   │
   │ automated in
   ▼
e2e/specs/swap/swapHistoryExport…spec.ts
   │
   │ runs in CI, uploads results
   ▼
Allure report + Xray execution update
   │
   │ stakeholders trace
   ▼
"Is B2CQA-604 passing on main?"  ◄── single source of truth
```

If you skip the Xray link (`$TmsLink("B2CQA-604")`), your test runs but Xray sees nothing. It is **invisible coverage** — a cardinal sin of the onboarding guide's Part 5 and still a cardinal sin here. The B2CQA ID must appear in the spec.

### 39.2 Picking Up a Ticket

You have been assigned `QAA-702`. Before touching code, do the **read-only pass**:

1. **Open the ticket in Jira.** Read the description end-to-end. Twice.
2. **Acceptance Criteria (AC).** If the ticket has an AC block, copy it into your branch notes. If it does not, go back to your lead and ask — a ticket without AC is a ticket you cannot prove done.
3. **Linked Xray tests.** Look at the "Tests" panel on the right. Click through each B2CQA. Note the `preconditions`, `steps`, and `expected result`. These become the `@Step` descriptions in your POM.
4. **Feature area.** Map the ticket to a source tree:
   - Portfolio, Market, Accounts → `apps/ledger-live-mobile/src/screens/Portfolio`, `.../Market`, `.../Accounts`
   - Trade: Swap / Buy / Sell → `.../Swap`, `.../Exchange`
   - Onboarding → `.../Onboarding`
   - Live Apps (DApps, discover) → `.../Platform` and `live-apps` package
   - Settings / Manager / Sync → `.../Settings`, `.../Manager`
5. **CODEOWNERS.** Before you write a line, look up who must review your PR:

   ```bash
   cat .github/CODEOWNERS | grep -E "ledger-live-mobile/e2e|src/screens/Swap"
   # apps/ledger-live-mobile/e2e/   @ledgerhq/wallet-xp
   # apps/ledger-live-mobile/src/screens/Swap/   @ledgerhq/ptx
   ```

   For a swap-area test you will end up with `@ledgerhq/wallet-xp` (owns the e2e folder) and, if you touch any React code, `@ledgerhq/ptx`.

6. **Estimate scope.** Most QAA tickets fit in 0.5–2 days of focused work. If your first estimate exceeds 3 days, the ticket is probably two tickets in a trench coat. Ask your lead to split it.

### 39.3 Branching

Branch names must describe the scope and reference the Jira ID. The global `git-workflow.md` rules apply verbatim:

| Prefix | Use for |
|---|---|
| `feat/` | New feature coverage, new POM, new CI job. |
| `bugfix/` | Fixing a real product-triggered flake, fixing a broken spec. |
| `support/` | Refactor, renames, test-data updates, config tweaks. |
| `chore/` | Dependency bumps, tooling config, lint rules. |

Mobile-scope conventions:

```bash
# New coverage
git checkout -b feat/llm-qaa-702-swap-history-erc20-export

# Bug fix in an existing spec
git checkout -b bugfix/llm-qaa-814-fix-send-ton-flake

# Refactor shared POM
git checkout -b support/llm-extract-modal-base-page
```

The `llm-` segment stands for **Ledger Live Mobile** — it tells the reviewer the platform without opening the diff. Desktop branches use `lld-`, live-common uses `llc-`.

**Conventional Commits** for mobile:

```
test(mobile): add swap history ERC20 export test (QAA-702)
feat(mobile): add SwapHistoryPage POM with @Step decorators
fix(mobile): debounce swap history refresh to stabilize e2e
refactor(mobile): extract bridge CSV listener into helper
chore(mobile): bump detox to 20.x
```

One logical change per commit. A diff that adds a POM, adds userdata, adds a spec, and bumps Detox is four commits, not one. Rebase and polish before you open the PR — the global rules say "rebase before PR to keep history clean."

### 39.4 Orientation Pass — Mapping the Terrain

You now open your editor. Before writing, you **recon the repo**. This pass alone catches 80 % of the "I wrote it twice" mistakes.

```bash
# 1. Is there already a spec for this feature?
ls apps/ledger-live-mobile/e2e/specs/swap/

# 2. Is there an existing POM?
ls apps/ledger-live-mobile/e2e/page/trade/

# 3. Which testIDs already exist on screen?
rg "testID=" apps/ledger-live-mobile/src/screens/Swap/History/

# 4. Which userdata fixtures look similar?
ls apps/ledger-live-mobile/e2e/userdata/ | rg -i "swap|exchange"

# 5. Which Xray IDs already reference this feature?
rg "B2CQA-6(0[0-9]|1[0-9])" apps/ledger-live-mobile/e2e/
```

The answers shape the rest of the day:

- **Spec exists** → add an `it()` inside it, do not create a new file.
- **POM exists** → add one or two `@Step` methods, do not create a parallel POM.
- **testID missing on screen** → stop. Open a product ticket, tag `@ledgerhq/ptx`, ask them to add it. **Never** select by text or accessibility label as a substitute — it will break on the next i18n change.
- **Userdata exists** → reuse it. Duplicated fixtures are a maintenance tax.

Write the outcome of the recon in 5 bullets in the Jira ticket. Your reviewer will read those bullets and immediately know why your diff looks the way it does.

### 39.5 Designing the Flow

Before writing `it()`, sketch the user flow **top to bottom** as a sequence of POM method calls. Use a scratch file, a whiteboard, or the comment header of your future spec:

```
// QAA-702: swap history export with ERC20 receive
// 1. Launch app with userdata = swapHistoryWithERC20
// 2. PortfolioPage.openSwapTab()
// 3. SwapPage.openHistoryTab()
// 4. SwapHistoryPage.expectOperationRowVisible(swapId)
// 5. SwapHistoryPage.expectERC20Received(swapId, "USDT")
// 6. SwapHistoryPage.tapExportButton()
// 7. Bridge: await sendFile event
// 8. Assert CSV contains USDT row with correct "To account address"
```

Each bullet is one `@Step`. If you cannot express a bullet with an existing method, you have two choices:

1. **Extend the existing POM** — preferred. Adds a `@Step` method to `swap.page.ts`.
2. **Create a new POM** — only when the feature is a new screen with no POM yet (our case for `SwapHistoryPage`).

**Rule of thumb.** If the screen has its own route and its own set of testIDs, it deserves its own POM. If it is a modal over an existing screen, extend the parent POM.

### 39.6 Implementation Order

Do not write the spec first. You will re-edit it six times. Instead:

1. **Fixtures (`userdata/*.json`)** — what does the app state need to be at `beforeAll`?
2. **POM methods (`page/*.page.ts`)** — `@Step` decorators, return types, no assertions on the caller's behalf unless the step name says "Expect".
3. **Spec (`specs/**/*.spec.ts`)** — stitch the POM calls into an `it()`.
4. **Dry-run locally** — `pnpm mobile e2e:test -t "<name>"`.
5. **Fix** — the first run fails. That is normal. Read the failure, open Allure, patch the POM or spec.
6. **Push** to a branch.
7. **CI green** — wait for the full matrix. If flaky, rerun once; if flaky twice, treat as failure and investigate.
8. **Allure check** — open the uploaded report, confirm `@Step` hierarchy and `$TmsLink`.
9. **PR** — follow 39.10.

Do not skip step 8. A green CI with a broken Allure report is a hidden regression — Xray will see "passed", reviewers will see no steps, and debugging when it breaks next month will be miserable.

### 39.7 Running Locally

Two pnpm scripts do 95 % of your work:

```bash
# ONCE per working session — build the app bundle
pnpm mobile e2e:build -c ios.sim.debug

# TIGHT LOOP — run one spec by test-name filter
pnpm mobile e2e:test -c ios.sim.debug -t "swap history containing an ERC20"
```

`-c` picks a Detox configuration (see Chapter 33 on `detox.config.js`). Common values:

| Configuration | Target |
|---|---|
| `ios.sim.debug` | iOS simulator with Metro debug bundle — fast reload. |
| `ios.sim.release` | iOS simulator with production bundle — slower, closer to CI. |
| `android.emu.debug` | Android emulator with Metro debug bundle. |
| `android.emu.release` | Android emulator with production bundle — what CI runs. |

`-t "<pattern>"` filters by the `it()` description. Detox passes it straight to Jest's `--testNamePattern`, so **use the exact string** that will appear in your `it()` title.

If you work on Android, keep an emulator warm:

```bash
emulator -avd Pixel_7_API_34 -no-snapshot-save -no-audio &
```

iOS simulators auto-boot via `applesimutils` — you do not need to pre-launch them.

### 39.8 Adding Allure Steps

Every public POM method should be decorated:

```typescript
@Step("Tap the export history button")
async tapExportButton() {
  await this.exportButton().tap();
}

@Step("Verify ERC20 receive for swap $0 with ticker $1")
async verifyERC20Received(swapId: string, tokenTicker: string) {
  await expect(this.toAmount(swapId)).toHaveText(new RegExp(tokenTicker));
}
```

The `@Step` decorator is imported from `jest-allure2-reporter/api`:

```typescript
import { Step } from "jest-allure2-reporter/api";
```

Behavior:

- `@Step("text")` wraps the method in an Allure step with that name.
- `$0`, `$1`, … are substituted with the method's arguments at call time — same semantics as Chapter 26.2's desktop `@step("Click on asset $0")`.
- If the method throws, the step is marked `failed` and the screenshot is attached automatically by the global Jest retry handler (Chapter 33.6).

**Do not** nest two `@Step` methods silently. If `verifyERC20Received()` internally calls `tapExportButton()`, you will get a two-level tree. That is usually fine, but if it produces a 40-line single-step trace in the report, flatten it.

### 39.9 Xray Linkage

The `$TmsLink` wrapper is how a mobile spec announces its B2CQA coverage. Two equivalent forms depending on your Jest-Allure version:

```typescript
// form 1 — title token
it($TmsLink("B2CQA-604"), "exports swap history containing an ERC20 receive", async () => { … });

// form 2 — programmatic API
it("exports swap history containing an ERC20 receive", async () => {
  allure.tms("B2CQA-604", "https://ledgerhq.atlassian.net/browse/B2CQA-604");
  …
});
```

The title-token form is preferred because Xray's importer reads the Allure test name and automatically extracts `B2CQA-XXX`. It also keeps the spec top clean.

What `$TmsLink` emits in Allure:

- A clickable link labelled `B2CQA-604` in the test's sidebar.
- A `tms` entry in the JSON for the Xray custom reporter to consume.
- Nothing in the spec runtime — the wrapper is a no-op at runtime, it only annotates.

**Multiple Xray IDs per spec.** Wrap more than one:

```typescript
it($TmsLink("B2CQA-604", "B2CQA-605"), "exports swap history …", async () => { … });
```

### 39.10 PR Checklist

The PR template lives in `.github/PULL_REQUEST_TEMPLATE.md` and the `/create-pr` Claude Code skill fills most of it in. But the human-readable checklist below is what your reviewer actually looks for:

1. **Title** — `test(mobile): add swap history ERC20 export test (QAA-702)`. Conventional Commits, one line, references the Jira ticket in parens.
2. **Jira link** — the QAA ticket URL at the top of the description.
3. **Description** — acceptance criteria restated, a one-paragraph summary of the approach, a "testing notes" section listing the Detox configs you ran.
4. **Test evidence** — a screenshot or screencast of the test passing locally (Allure step list or Detox console output), plus the CI run link once green.
5. **CODEOWNERS** — GitHub auto-requests, but do a sanity check: swap-area touches should have `@ledgerhq/wallet-xp` and `@ledgerhq/ptx`.
6. **Reviewers** — your lead plus one peer familiar with the area.
7. **Semantic-release impact** — for a test-only PR, the `test(mobile): …` prefix means **no version bump**. For a `fix(mobile): …` inside the e2e folder only, still no bump — the publishable package is the app, not the e2e harness. For `fix(mobile-app): …` inside `apps/ledger-live-mobile/src/**`, a patch bump will trigger on `main`.
8. **Changeset** — the monorepo uses Changesets. If your diff only touches `apps/ledger-live-mobile/e2e/`, you do not need one. If you modify `apps/ledger-live-mobile/src/`, run `pnpm changeset` and pick "patch" or "none" as appropriate.

Open the PR as **draft** first. Mark it ready only after CI is green.

### 39.11 Review Etiquette

The golden rules, applied to mobile:

- **Small diffs per ticket.** A 1,500-line PR does not get a careful review — it gets a rubber stamp or a rejection. Split.
- **Rebase, do not merge.** `git pull --rebase origin dev` before pushing. A spaghetti merge graph makes `git bisect` useless.
- **Squash only for trivial branches.** `support/cleanup` branches squash cleanly. Feature branches with a thoughtful commit history should preserve that history on merge.
- **Address review comments in new commits, not amends.** The reviewer wants to see *what changed since I last looked* — a force-push nukes that. Only squash at the very end, and only if the reviewer agrees.
- **Block your own PR until CI is green.** A reviewer should never be the one who tells you CI is red.

Reviewer-facing signals that you did the work:

- The recon bullets from 39.4 in the PR description.
- The Allure link in the PR comments (or a screenshot if the report has rolled off).
- A note if any product change (testID addition, bridge extension) was required, with the linked product ticket.

<div class="chapter-outro">
<strong>Key takeaway.</strong> The workflow is: QAA ticket → B2CQA scenario → branch with <code>llm-</code> prefix → recon the repo → sketch the flow → write fixtures → write POM → write spec → dry-run loop → push → wait for CI → inspect Allure → open a small, well-scoped PR with CODEOWNERS auto-requested. The same dance, every ticket. Predictable is professional.
</div>

### 39.12 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — Daily Mobile Workflow</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> What is the difference between <code>QAA-702</code> and <code>B2CQA-604</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) They are aliases for the same ticket</button>
<button class="quiz-choice" data-value="B">B) <code>QAA-702</code> is the sprint work item; <code>B2CQA-604</code> is the Xray test scenario it is expected to automate</button>
<button class="quiz-choice" data-value="C">C) <code>QAA-702</code> is mobile-only; <code>B2CQA-604</code> is desktop-only</button>
<button class="quiz-choice" data-value="D">D) <code>B2CQA-604</code> is a bug report</button>
</div>
<p class="quiz-explanation">QAA tickets are sprint work items ("automate this"). B2CQA tickets live in Xray and describe a reusable test scenario that may be covered by LLD, LLM, or LLC.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> A testID you need is missing from a screen. What do you do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Select by visible text until the testID is added</button>
<button class="quiz-choice" data-value="B">B) Select by accessibility label</button>
<button class="quiz-choice" data-value="C">C) Stop, open a product ticket with <code>@ledgerhq/ptx</code>, and wait for the testID to land before writing the test</button>
<button class="quiz-choice" data-value="D">D) Ship the test with an XPath query</button>
</div>
<p class="quiz-explanation">Text and accessibility labels are i18n-fragile. The repo rule is: testIDs are the contract. If one is missing, product adds it first. Writing a test against a text selector is technical debt the next translator will silently break.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> Which branch name correctly follows the mobile conventions?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>QAA-702-swap-export</code></button>
<button class="quiz-choice" data-value="B">B) <code>swap-history-erc20</code></button>
<button class="quiz-choice" data-value="C">C) <code>feature/swap-history</code></button>
<button class="quiz-choice" data-value="D">D) <code>feat/llm-qaa-702-swap-history-erc20-export</code></button>
</div>
<p class="quiz-explanation">Prefix (<code>feat/</code>) + platform (<code>llm-</code>) + Jira ID (<code>qaa-702</code>) + short scope. The kebab-case format is required by <code>git-workflow.md</code>.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> What is the correct implementation order for a new mobile spec?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Userdata fixtures → POM methods → spec → dry-run → push</button>
<button class="quiz-choice" data-value="B">B) Spec → POM methods → userdata</button>
<button class="quiz-choice" data-value="C">C) POM methods → spec → userdata</button>
<button class="quiz-choice" data-value="D">D) Dry-run first, then write</button>
</div>
<p class="quiz-explanation">You cannot write a useful spec until the POM methods exist, and POM methods are meaningless without a known starting state (userdata). Bottom-up is the only sane order.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> What does <code>$TmsLink("B2CQA-604")</code> at the start of an <code>it()</code> do?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Causes the test to fail if Xray is unreachable</button>
<button class="quiz-choice" data-value="B">B) Annotates the Allure result with a clickable TMS link so Xray can match the spec to the scenario</button>
<button class="quiz-choice" data-value="C">C) Fetches the ticket description at runtime</button>
<button class="quiz-choice" data-value="D">D) Triggers a network request to Jira</button>
</div>
<p class="quiz-explanation">The wrapper is a no-op at runtime. It adds a <code>tms</code> annotation to the Allure result, which the Xray importer reads to update B2CQA-604's execution status.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q6.</strong> Why do you open the PR as draft first?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Draft PRs run faster in CI</button>
<button class="quiz-choice" data-value="B">B) GitHub requires it</button>
<button class="quiz-choice" data-value="C">C) It exempts the PR from CODEOWNERS</button>
<button class="quiz-choice" data-value="D">D) It lets you validate CI and Allure before burning reviewer attention — you mark ready only once the PR is actually reviewable</button>
</div>
<p class="quiz-explanation">A reviewer's time is the scarce resource. A red CI or a broken Allure report is something you find — not something your reviewer finds.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Real Ticket Walkthrough: QAA-702 — Swap History ERC20 Export

<div class="chapter-intro">
This is your mobile capstone. Everything from Chapters 32–39 converges here on a real Jira ticket: <strong>QAA-702 — [SWAP] [LLM] Improve 'History' test — add 'History export' step when there is a tx with ERC20 token in Receive field</strong>. The Xray test case is <strong>B2CQA-604</strong>. We will walk through the complete loop: decompose the ticket, recon the source, design the POM, write the userdata, wire the bridge, author the spec, run it, inspect Allure, and ship the PR. No shortcuts.
</div>

### 40.1 Ticket Decomposition

**Jira ticket (summarized):**

> **QAA-702 — [SWAP] [LLM] Improve 'History' test — add 'History export' step when there is a tx with ERC20 token in Receive field**
>
> **Description.** The base test `B2CQA-604` covers the Swap history screen. Today, it asserts a previous swap operation is visible. The improvement asks that when the swap's *Receive* account is an ERC20 token account (e.g., USDT on Ethereum), the test also exercises the **Export** button and verifies the exported CSV contains the correct ERC20 row.
>
> **Acceptance criteria.**
> 1. When a completed swap operation exists and its `toAccount.type === "TokenAccount"` (ERC20), the history tab shows an **Export** button (`testID=export-swap-operations-link`).
> 2. Tapping the button in DETOX mode triggers a `sendFile` bridge event with `fileName === "ledgerwallet-swap-history.csv"`.
> 3. The CSV file content includes a row whose "To account" column matches the parent Ethereum account name, and whose "To account address" column matches the parent account's fresh address.
> 4. The existing DEX swap flow is out of scope — the `dexSwap.spec.ts` suite is currently `describe.skip` due to an iOS new-architecture infinite re-render; do not re-enable it.

**Scope boundary.** You are writing **one new spec file** plus **one POM**, plus **one userdata fixture**. You are **not** touching product code. All testIDs and the DETOX `sendFile` branch already exist.

### 40.2 Understanding B2CQA-604

Open `https://ledgerhq.atlassian.net/browse/B2CQA-604` in a browser. You cannot do this task without it — the Xray record is the acceptance spec. The condensed version:

- **Precondition.** An account list that contains at least one Ethereum account with a USDT TokenAccount child, plus a completed swap in the history whose Receive side is that USDT account.
- **Steps.**
  1. Open the Swap tab.
  2. Navigate to "History".
  3. Observe the export button at the top of the list.
  4. Tap "Export".
  5. Observe a file dialog (native share or — in Detox mode — a bridge `sendFile`).
- **Expected result.** The CSV contains one line per swap, with columns including "To account" and "To account address". For ERC20 receives, these columns must be the **parent Ethereum account name** and **parent Ethereum address**, respectively — not the token account's name.

> When the B2CQA description ages, it ages. Ask your lead if the Xray record drifts from reality. Do not silently "fix" the test to match a stale scenario — update the scenario first.

### 40.3 Code Reconnaissance

Three files tell you everything you need.

**File 1 — `apps/ledger-live-mobile/src/screens/Swap/History/index.tsx`.**

The history screen. The relevant blocks:

```tsx
// lines 48-60
const ensureParentAccount = (
  item: MappedSwapOperation,
  accounts: AccountLike[],
): MappedSwapOperation => {
  if (item.toAccount.type === "TokenAccount" && !item.toParentAccount) {
    return {
      ...item,
      toParentAccount: getParentAccount(item.toAccount, accounts),
    };
  }
  return item;
};
```

This is the ERC20 detection rule: `toAccount.type === "TokenAccount"`. Your userdata fixture must produce a swap operation whose `toAccount` is a token account so this branch fires.

```tsx
// lines 145-174 — the export handler
const exportSwapHistory = async () => {
  const mapped = mappedSwapOperationsToCSV(sections);
  if (!getEnv("DETOX")) {
    // …native share sheet…
    await Share.open(options);
  } else {
    try {
      sendFile({ fileName: "ledgerwallet-swap-history.csv", fileContent: mapped });
    } catch (err) {
      logger.critical(err as Error);
    }
  }
};
```

The DETOX branch calls `sendFile` — the bridge message. The native `Share.open` path is the production path, but Detox cannot dismiss a native share sheet, so the app bypasses it entirely when `DETOX=true`. Your spec will subscribe to the `sendFile` bridge event on the Node side.

```tsx
// lines 199-206 — the button
<Button
  type="tertiary"
  title={t("transfer.swap.history.exportButton")}
  containerStyle={styles.button}
  IconLeft={DownloadFileIcon}
  onPress={exportSwapHistory}
  testID="export-swap-operations-link"
/>
```

`testID="export-swap-operations-link"` — this is your hook. The button only renders when `sections.length > 0`, so if the userdata produces zero operations you will never find it.

**File 2 — `libs/ledger-live-common/src/exchange/swap/csvExport.ts`.**

`mappedSwapOperationsToCSV()` serialises the operations:

```ts
{ title: "To account",         cell: ({ toAccount, toParentAccount }) =>
    getDefaultAccountName(getMainAccount(toAccount, toParentAccount)) },
{ title: "To account address", cell: ({ toAccount, toParentAccount }) => {
    const main = getMainAccount(toAccount, toParentAccount);
    return main.freshAddress;
} },
```

For an ERC20 receive, `toAccount` is the token account and `toParentAccount` is the Ethereum parent. `getMainAccount` returns the parent, so the CSV row's "To account" and "To account address" are the **Ethereum** ones — not the token's. Your assertion must match the parent's name and address.

Column list (verbatim from the file): `From account, From account address, From value, From ticker, From parent currency, To account, To account address, To value, To ticker, To parent currency, Operation date, Swap ID, Provider, Fees`.

**File 3 — `apps/ledger-live-mobile/src/screens/Swap/History/OperationRow.tsx`.**

Per-row testIDs:

- `swap-operation-row-${swapId}` — the whole row.
- `swap-history-toAccount-${swapId}` — the "To account" label.
- `swap-history-toAmount-${swapId}` — the "To amount" label. For an ERC20 receive, this contains the token ticker (e.g., `+ 42 USDT`).

These three are everything you need to assert the ERC20 row is visible before you tap Export.

### 40.4 Design Decisions

**Decision 1 — new POM or inline calls?**

Inline Detox matchers in the spec would read:

```typescript
await element(by.id("export-swap-operations-link")).tap();
```

Works for one assertion. But the spec will chain five to eight matcher calls, each of which deserves a readable step name. Inline code in the spec does not get an Allure `@Step`; a decorated POM method does.

Verdict — **new POM**, `e2e/page/trade/swapHistory.page.ts`. Extends nothing, registered alongside `swap.page.ts`.

**Decision 2 — fixture strategy.**

Two options:

- **Seed via CLI commands** like desktop does. Mobile does not have a live-common CLI hook inside the spec. Not applicable.
- **Ship a userdata JSON** with pre-populated accounts and a pre-recorded swap operation.

Verdict — **userdata JSON**, the mobile-idiomatic approach (Chapter 36). Filename `userdata/swapHistoryWithERC20.json`.

**Decision 3 — where does the CSV assertion run?**

Option A: decode the CSV in the Node-side bridge listener and assert there.
Option B: send the CSV back into the app via a secondary bridge call and assert inside the app.

Option B gives no advantage (the bridge already owns the CSV). **Option A** wins.

### 40.5 The Userdata Fixture

Create `apps/ledger-live-mobile/e2e/userdata/swapHistoryWithERC20.json`. Minimal shape (abbreviated — real fixtures carry a few more bookkeeping fields that `userdata-loader.ts` computes on import):

```json
{
  "user": { "theme": "light" },
  "accounts": [
    {
      "id": "js:2:ethereum:0xAA11...F7:bip39/metamask",
      "currencyId": "ethereum",
      "freshAddress": "0xAA11111111111111111111111111111111F7",
      "name": "Ethereum 1",
      "balance": "500000000000000000",
      "subAccounts": [
        {
          "id": "js:2:ethereum:0xAA11...F7:+erc20/usd_tether__erc20_",
          "parentId": "js:2:ethereum:0xAA11...F7:bip39/metamask",
          "type": "TokenAccount",
          "tokenId": "ethereum/erc20/usd_tether__erc20_",
          "balance": "42000000"
        }
      ]
    }
  ],
  "swap": {
    "history": [
      {
        "swapId": "e2e-swap-001",
        "provider": "changelly",
        "status": "finished",
        "fromAccountId": "js:2:bitcoin:…",
        "toAccountId":   "js:2:ethereum:0xAA11...F7:+erc20/usd_tether__erc20_",
        "fromAmount": "10000",
        "toAmount": "42000000",
        "operationDate": "2026-03-15T10:22:00Z"
      }
    ]
  }
}
```

The USDT subaccount's `type: "TokenAccount"` is what trips `ensureParentAccount()` in `History/index.tsx`. The swap's `toAccountId` must point at that token account. The operation date is recent enough that the section header renders ("Today" / "Yesterday" is irrelevant to the assertion).

Register the fixture in `e2e/userdata/index.ts` (or however `app.init({ userdata: "<name>" })` resolves names — this path is covered in Chapter 36.4). Reload Detox once after adding.

### 40.6 Bridge — Subscribing to `sendFile`

The bridge protocol lives at `apps/ledger-live-mobile/e2e/bridge/` (Chapter 35). Your spec needs to:

1. Register a listener for `sendFile` on the Node side before tapping Export.
2. Await the first message that matches `fileName === "ledgerwallet-swap-history.csv"`.
3. Return `{ fileName, fileContent }` to the caller.

Depending on how the mobile e2e codebase exposes the listener API, the helper looks like one of:

```typescript
// file: e2e/bridge/client/sendFileListener.ts
import { bridge } from "./bridge";

export function waitForSendFile(expectedFileName: string, timeoutMs = 10_000) {
  return new Promise<{ fileName: string; fileContent: string }>((resolve, reject) => {
    const timer = setTimeout(
      () => reject(new Error(`sendFile '${expectedFileName}' not received in ${timeoutMs}ms`)),
      timeoutMs,
    );
    const off = bridge.on("sendFile", (msg: { fileName: string; fileContent: string }) => {
      if (msg.fileName !== expectedFileName) return;
      clearTimeout(timer);
      off();
      resolve(msg);
    });
  });
}
```

Before you write this helper, confirm it does not already exist — `rg "sendFile" apps/ledger-live-mobile/e2e/bridge/`. If `bridge/types.ts` declares the message but no server-side consumer exists, you are adding ~20 lines to `bridge/server.ts` to forward the message to listeners. If the consumer exists, you are writing zero bridge code. Either way: **one small, reviewable change**, and it goes in its own commit (`feat(mobile): expose sendFile bridge listener for e2e`).

### 40.7 The POM — `swapHistory.page.ts`

Full file — drop it at `apps/ledger-live-mobile/e2e/page/trade/swapHistory.page.ts`:

```typescript
import { by, element, expect as detoxExpect } from "detox";
import { Step } from "jest-allure2-reporter/api";
import { waitForSendFile } from "e2e/bridge/client/sendFileListener";

export default class SwapHistoryPage {
  // ── Selectors ──
  private exportButton = () => element(by.id("export-swap-operations-link"));
  private operationRow = (swapId: string) => element(by.id(`swap-operation-row-${swapId}`));
  private toAccountName = (swapId: string) => element(by.id(`swap-history-toAccount-${swapId}`));
  private toAmount      = (swapId: string) => element(by.id(`swap-history-toAmount-${swapId}`));

  // ── Assertions ──

  @Step("Expect swap history row $0 to be visible")
  async expectOperationRowVisible(swapId: string) {
    await detoxExpect(this.operationRow(swapId)).toBeVisible();
  }

  @Step("Expect ERC20 receive $1 on swap $0")
  async expectERC20Received(swapId: string, tokenTicker: string) {
    await detoxExpect(this.toAmount(swapId)).toBeVisible();
    await detoxExpect(this.toAmount(swapId)).toHaveText(new RegExp(tokenTicker));
  }

  @Step("Expect 'To account' label of swap $0 to contain $1")
  async expectToAccountName(swapId: string, parentAccountName: string) {
    await detoxExpect(this.toAccountName(swapId)).toHaveText(new RegExp(parentAccountName));
  }

  // ── Actions ──

  @Step("Tap the 'Export history' button")
  async tapExportButton() {
    await detoxExpect(this.exportButton()).toBeVisible();
    await this.exportButton().tap();
  }

  // ── CSV receive ──

  @Step("Wait for the exported CSV and return its content")
  async awaitExportedCSV(): Promise<{ fileName: string; fileContent: string }> {
    return waitForSendFile("ledgerwallet-swap-history.csv");
  }
}
```

Wire it into the `Application` hub at `e2e/page/index.ts`:

```typescript
import SwapHistoryPage from "./trade/swapHistory.page";

export class Application {
  // …existing lazy-inits…
  swapHistory = lazyInit(SwapHistoryPage);
}
```

`lazyInit` is the singleton pattern from Chapter 34.5 — each POM is instantiated once per app session and reused across specs. If `lazyInit` does not exist in your codebase and the hub uses plain `new SwapHistoryPage()`, match the existing convention.

### 40.8 The Spec — `swapHistoryExportERC20.spec.ts`

Full spec — `apps/ledger-live-mobile/e2e/specs/swap/swapHistoryExportERC20.spec.ts`:

```typescript
import { Application } from "../../page";
import { $TmsLink } from "../../helpers/xray"; // whichever module exposes it — see Ch. 37.4

const app = new Application();

const FIXTURE = {
  userdata: "swapHistoryWithERC20",
  swapId: "e2e-swap-001",
  tokenTicker: "USDT",
  parentAccountName: "Ethereum 1",
  parentFreshAddress: "0xAA11111111111111111111111111111111F7",
};

describe("Swap History — Export", () => {
  beforeAll(async () => {
    await app.init({ userdata: FIXTURE.userdata, speculosApp: undefined });
    await app.portfolio.openSwapTab();
    await app.swap.openHistoryTab();
  });

  it(
    $TmsLink("B2CQA-604"),
    "exports swap history containing an ERC20 receive",
    async () => {
      // 1. Row is visible with ERC20 receive rendered
      await app.swapHistory.expectOperationRowVisible(FIXTURE.swapId);
      await app.swapHistory.expectERC20Received(FIXTURE.swapId, FIXTURE.tokenTicker);
      await app.swapHistory.expectToAccountName(FIXTURE.swapId, FIXTURE.parentAccountName);

      // 2. Register the listener BEFORE tapping (the event fires synchronously)
      const csvPromise = app.swapHistory.awaitExportedCSV();

      // 3. Tap export
      await app.swapHistory.tapExportButton();

      // 4. Await the bridge message
      const { fileName, fileContent } = await csvPromise;

      // 5. Structural assertions on the CSV
      expect(fileName).toBe("ledgerwallet-swap-history.csv");

      const [header, ...rows] = fileContent.split("\n");
      expect(header).toContain("To account");
      expect(header).toContain("To account address");

      const erc20Row = rows.find(r => r.includes(FIXTURE.swapId));
      expect(erc20Row).toBeDefined();

      const cells = erc20Row!.split(",");
      const toAccountIdx        = header.split(",").indexOf("To account");
      const toAccountAddressIdx = header.split(",").indexOf("To account address");
      const toTickerIdx         = header.split(",").indexOf("To ticker");

      // ERC20 receive rules: "To account" is the Ethereum parent name,
      // "To account address" is the Ethereum parent fresh address,
      // "To ticker" is the token ticker.
      expect(cells[toAccountIdx]).toBe(FIXTURE.parentAccountName);
      expect(cells[toAccountAddressIdx]).toBe(FIXTURE.parentFreshAddress);
      expect(cells[toTickerIdx]).toBe(FIXTURE.tokenTicker);
    },
  );
});
```

Why the listener is registered **before** `tap`: if you `await tap()` first and only then set up the listener, the bridge message may already have flown past. The Node-side `EventEmitter` emits synchronously; late listeners miss the event.

### 40.9 Running It

```bash
# 1. Build the app (once per session)
pnpm mobile e2e:build -c ios.sim.debug

# 2. Run only this spec
pnpm mobile e2e:test -c ios.sim.debug -t "exports swap history containing an ERC20 receive"
```

**Expected first-run failures** — in order of appearance:

1. `app.swapHistory is undefined`. Fix: you forgot to register `swapHistory` in `e2e/page/index.ts`.
2. `Cannot resolve userdata 'swapHistoryWithERC20'`. Fix: the userdata loader does not see your JSON; verify the filename and the loader's glob pattern in `e2e/userdata/index.ts`.
3. `sendFile 'ledgerwallet-swap-history.csv' not received in 10000ms`. Fix: either the bridge listener helper has the wrong event name, or the DETOX env var is not being passed to the app bundle. Check `process.env.DETOX` in the Detox runner and in `getEnv("DETOX")` inside the app. On iOS debug builds this is set by Detox itself; on release builds you sometimes need to pass it via `DETOX_CONFIGURATION` wiring.
4. `Expected "Ethereum 1", got "USDT 1"`. Fix: you asserted the token account's name. Re-read `csvExport.ts` — `getMainAccount` returns the **parent**.

After each fix: rerun. Green on the fourth try is normal.

Repeat the run three times. Mobile flakiness tends to show up on Metro cold starts — a test that is green three times back-to-back is trustworthy.

### 40.10 Allure Verification

```bash
# Produce the report
pnpm mobile e2e:report
# Serve it
allure serve apps/ledger-live-mobile/allure-results
```

Open the spec's entry. You should see:

```
Swap History — Export
  exports swap history containing an ERC20 receive            ✓ 9.8s
    Expect swap history row e2e-swap-001 to be visible         0.2s
    Expect ERC20 receive USDT on swap e2e-swap-001             0.3s
    Expect 'To account' label of swap e2e-swap-001 to contain… 0.2s
    Wait for the exported CSV and return its content           0.1s
    Tap the 'Export history' button                            0.4s
    Wait for the exported CSV and return its content            0.1s
```

Sidebar:

- **Links** → `B2CQA-604` (clickable, opens Jira).
- **Labels** → framework=Detox, host=<mac-or-CI-runner>, thread=e2e-sim-1.
- **Attachments** → device screenshot (only on failure), `sendFile` CSV (attached via an Allure `attach` call if you added it; optional but useful).

If the `B2CQA-604` link is missing, your `$TmsLink` wrapper is broken — likely the wrong import. Fix before opening the PR.

### 40.11 Opening the PR

```bash
git checkout -b feat/llm-qaa-702-swap-history-erc20-export
git add apps/ledger-live-mobile/e2e/userdata/swapHistoryWithERC20.json
git commit -m "test(mobile): add userdata fixture for swap history ERC20 (QAA-702)"

git add apps/ledger-live-mobile/e2e/page/trade/swapHistory.page.ts \
        apps/ledger-live-mobile/e2e/page/index.ts
git commit -m "feat(mobile): add SwapHistoryPage POM with @Step decorators"

git add apps/ledger-live-mobile/e2e/bridge/client/sendFileListener.ts
git commit -m "feat(mobile): expose sendFile bridge listener for e2e (QAA-702)"

git add apps/ledger-live-mobile/e2e/specs/swap/swapHistoryExportERC20.spec.ts
git commit -m "test(mobile): add swap history ERC20 export spec (QAA-702)"

git push -u origin feat/llm-qaa-702-swap-history-erc20-export
```

Four focused commits, each doing one thing. Open the PR:

```bash
/create-pr
```

Fill the template:

- **Jira** — `QAA-702`.
- **Scope** — `mobile`.
- **Description** — 4 lines: what, why, how, evidence.
- **Reviewers** — `@ledgerhq/wallet-xp` auto-requested via CODEOWNERS; add your lead explicitly.
- **Evidence** — link the CI run and paste the Allure step tree screenshot.

Mark as **draft**. Once CI turns green across iOS and Android, click "Ready for review". After approvals and merge, flip B2CQA-604 in Xray to `Automated` / `Automated In: LLM`.

### 40.12 Gotchas and Lessons

- **Why `sendFile` and not the native share sheet.** `react-native-share` renders an OS-level modal. Detox controls the app's JS process; it cannot dismiss OS modals. Instead, the app checks `getEnv("DETOX")` and dispatches the file payload over the bridge to the test runner. The production path is untouched — a user tapping Export still gets the native share sheet.
- **Why `ensureParentAccount` exists.** When a swap's `toAccount` is a `TokenAccount`, selectors for "main account" need its parent. The screen patches operations in memory before they reach `OperationRow`; without this helper, the row would show the token account name as "main" and the CSV would export the wrong address.
- **Why DEX swap is skipped.** `apps/ledger-live-mobile/e2e/specs/swap/dexSwap.spec.ts` begins with `describe.skip` because the DEX path triggers an infinite re-render under the iOS new architecture. This is tracked as a product bug. Do not re-enable it to "increase coverage" — you will just turn the pipeline red.
- **Why we could not reuse `dexSwap.spec.ts`.** Beyond the skip, its setup (`userdata`, provider injection) targets a live DEX API. The history export flow we are testing is data-seeded. They are two different tests.
- **Why you assert CSV columns by header-index lookup, not by fixed position.** `csvExport.ts` can reorder columns across live-common versions. Looking up `header.split(",").indexOf("To account")` keeps the assertion resilient.
- **Why register the bridge listener before tapping.** The bridge is a plain `EventEmitter`. Listeners added after an emission miss it. The `await` before `tap()` must be a no-op for the listener registration — which is why the helper returns a `Promise` created from a synchronous `bridge.on` call.

<div class="chapter-outro">
<strong>Key takeaway.</strong> QAA-702 was a pure test-authoring ticket: no product change, no testID addition, no framework upgrade. Four focused commits shipped it: userdata → POM → bridge helper → spec. The hardest part was not writing the assertions — it was <em>reading the three source files</em> carefully enough to know that an ERC20 swap's "To account" column carries the Ethereum parent's name and address, not the token account's. Source-first, spec-second.
</div>

### 40.13 Quiz

<div class="quiz-container" data-pass-threshold="80">
<h3>Quiz — QAA-702 Walkthrough</h3>
<p class="quiz-subtitle">6 questions · 80% to pass</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="B">
<p><strong>Q1.</strong> In the history screen, what triggers the DETOX branch of <code>exportSwapHistory</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A feature flag <code>detoxMode</code></button>
<button class="quiz-choice" data-value="B">B) <code>getEnv("DETOX")</code> returning truthy — set by the Detox runtime</button>
<button class="quiz-choice" data-value="C">C) The presence of a token account</button>
<button class="quiz-choice" data-value="D">D) A URL parameter</button>
</div>
<p class="quiz-explanation">The branch is <code>if (!getEnv("DETOX")) { Share.open(...) } else { sendFile(...) }</code>. DETOX is a live-common env flag set to truthy by the Detox harness.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q2.</strong> For an ERC20 receive, what does the CSV's "To account address" column contain?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The ERC20 token contract address</button>
<button class="quiz-choice" data-value="B">B) The token account's synthetic ID</button>
<button class="quiz-choice" data-value="C">C) The Ethereum parent account's fresh address — <code>getMainAccount(toAccount, toParentAccount).freshAddress</code></button>
<button class="quiz-choice" data-value="D">D) An empty string</button>
</div>
<p class="quiz-explanation"><code>csvExport.ts</code> calls <code>getMainAccount</code>, which unwraps a <code>TokenAccount</code> to its parent. The CSV address is the Ethereum address that received the tokens.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q3.</strong> Why is the bridge listener registered <em>before</em> tapping the Export button?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) To pre-warm the bridge connection</button>
<button class="quiz-choice" data-value="B">B) Detox requires listener registration before any tap</button>
<button class="quiz-choice" data-value="C">C) It is a readability preference only</button>
<button class="quiz-choice" data-value="D">D) The underlying <code>EventEmitter</code> emits synchronously — listeners added after emission miss the event</button>
</div>
<p class="quiz-explanation">If <code>sendFile</code> fires during the tap, a late-registered listener will never see it. Register first, then trigger, then await.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q4.</strong> Why can't you reuse <code>dexSwap.spec.ts</code> for QAA-702?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) It is <code>describe.skip</code> due to an iOS new-architecture infinite re-render, and it targets a live DEX provider</button>
<button class="quiz-choice" data-value="B">B) It lives in the wrong folder</button>
<button class="quiz-choice" data-value="C">C) It is written in JavaScript, not TypeScript</button>
<button class="quiz-choice" data-value="D">D) It does not import Detox</button>
</div>
<p class="quiz-explanation">The file is skipped for a real product bug, and even if you un-skipped it, its setup targets the DEX flow, not the CEX swap history we need.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q5.</strong> Which testID does the Export button expose?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>swap-export-button</code></button>
<button class="quiz-choice" data-value="B">B) <code>export-swap-operations-link</code></button>
<button class="quiz-choice" data-value="C">C) <code>history-export-csv</code></button>
<button class="quiz-choice" data-value="D">D) It has no testID</button>
</div>
<p class="quiz-explanation">Verbatim in <code>History/index.tsx</code> at line 205: <code>testID="export-swap-operations-link"</code>. Do not invent others.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q6.</strong> Which CODEOWNERS must review a PR that touches <code>apps/ledger-live-mobile/e2e/specs/swap/</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>@ledgerhq/mobile-core</code></button>
<button class="quiz-choice" data-value="B">B) <code>@ledgerhq/qa-infra</code></button>
<button class="quiz-choice" data-value="C">C) <code>@ledgerhq/wallet-xp</code> (and <code>@ledgerhq/ptx</code> if any product code under <code>src/screens/Swap/</code> is touched)</button>
<button class="quiz-choice" data-value="D">D) Nobody — e2e files are exempt</button>
</div>
<p class="quiz-explanation">The e2e folder is owned by wallet-xp. Swap product code is owned by ptx. A pure-e2e PR only needs wallet-xp; a cross-cutting PR needs both.</p>
</div>

<div class="quiz-score"></div>
</div>

---

## Mobile Exercises & Challenges

<div class="chapter-intro">
Reading and watching are not enough. The exercises below move from minor POM additions to CI-level changes and end with a simulator-warming wrapper that will speed up your daily loop. Do them in order — each one builds a skill you will need in the next. Every exercise has a verification step; if you cannot verify it yourself, grab a reviewer before moving on.
</div>

### 41.1 Exercise 1: Add a `@Step` for the Empty State (15 min)

**Objective.** Practice adding a decorated assertion to an existing POM.

**Instructions.**
1. Open `apps/ledger-live-mobile/e2e/page/trade/swapHistory.page.ts` (the POM you wrote in Chapter 40 — or a draft of it).
2. Add a selector for the history empty-state title: `swap-history-empty-title`.
3. Add a method `expectEmptyState()` decorated with `@Step("Expect swap history to be empty")` that asserts the empty-state title is visible.
4. Make sure the method is `public async`.

**Verification.** Run `pnpm mobile tsc --noEmit`. No type errors. Open the POM side by side with `swap.page.ts`; the signature should match the other `expect*` methods.

**Hints.**
- Copy the shape of `expectOperationRowVisible(swapId)`, drop the parameter.
- The empty-state subcomponent is `apps/ledger-live-mobile/src/screens/Swap/History/EmptyState.tsx`; open it to confirm the testID before adding it to the POM.

**Stretch goal.** Add a second `@Step("Expect export button to be hidden")` that uses `toBeNotVisible()` on `export-swap-operations-link`.

### 41.2 Exercise 2: A Modal Close Spec (30 min)

**Objective.** Write a minimal end-to-end spec that re-uses an existing POM.

**Instructions.**
1. Locate `apps/ledger-live-mobile/e2e/page/common/passwordEntry.page.ts` (or its equivalent — search with `rg "passwordEntry"`).
2. Write a spec `e2e/specs/common/passwordEntryClose.spec.ts` with a single `it()`:
   - `beforeAll` → `app.init({ userdata: "skipOnboardingWithPasswordLock" })` — if this fixture does not exist, use the closest one that boots the app at the lock screen.
   - Assert the password-entry modal is visible.
   - Tap the existing Close button via the POM's `tapCloseButton()`.
   - Assert the modal is gone.
3. Tag the spec with an arbitrary `$TmsLink("B2CQA-DUMMY")` — the point is the wiring, not the Xray link.

**Verification.** Run `pnpm mobile e2e:test -c ios.sim.debug -t "password entry close"`. Three runs, all green.

**Hints.**
- If `skipOnboardingWithPasswordLock` does not exist, check `userdata/` for a "locked" variant; if none, discuss with your lead before inventing one.
- The Close button may raise a confirmation. Use the POM's confirm method if so.

**Stretch goal.** Add a second `it()` that, instead of tapping Close, types the wrong password three times and asserts the lockout message appears.

### 41.3 Exercise 3: Android API-Level Matrix Entry (45 min)

**Objective.** Add a second Android configuration to `detox.config.js` and invoke it locally.

**Instructions.**
1. Open `apps/ledger-live-mobile/detox.config.js`.
2. Duplicate the existing `android.emu.debug` block into a new config `android.emu.api34.debug` that targets an AVD named `Pixel_7_API_34`.
3. Create the AVD if it does not exist (`avdmanager create avd …` or via Android Studio).
4. Build with `pnpm mobile e2e:build -c android.emu.api34.debug`.
5. Run one of the small specs you wrote in exercises 1–2 under the new config.

**Verification.** The spec boots on an API-34 emulator. Visible in `emulator -list-avds`.

**Hints.**
- The existing `android.emu.debug` block is your template; change only `device.avdName`.
- API 34 introduces stricter foreground-service policy — if the app fails to boot, you likely need to adjust `device.headless` or `device.utilBinaryPaths`.

**Stretch goal.** Add the new config to a nightly CI job by appending it to the matrix in `.github/workflows/test-mobile-e2e.yml`. Do not merge without your lead's sign-off — nightly minutes are not free.

### 41.4 Exercise 4: Multi-Account Userdata + Accounts List Spec (60 min)

**Objective.** Build a non-trivial userdata fixture and assert that the app renders it correctly.

**Instructions.**
1. Create `e2e/userdata/twoEthTwoUsdt.json` — two Ethereum parent accounts (`Ethereum 1`, `Ethereum 2`), each with a USDT TokenAccount child (balances: `100 USDT`, `250 USDT`).
2. Write `e2e/specs/accounts/listTwoEthTwoUsdt.spec.ts` that:
   - Boots with the new userdata.
   - Opens the Accounts tab via `app.accounts.openAccountsTab()`.
   - Asserts four rows are visible: `Ethereum 1`, `Ethereum 2`, and the two USDT child rows.
3. Tag with `$TmsLink("B2CQA-DUMMY2")`.

**Verification.** Three green runs. Open Allure and confirm each assertion is a separate `@Step`.

**Hints.**
- Use distinct, collision-free account IDs — copy the ID pattern from an existing userdata fixture.
- The Accounts list uses `accounts-account-row-$accountId` as its testID pattern (confirm by grepping `src/screens/Accounts/`).

**Stretch goal.** Extend with a third Ethereum account that has an ERC721 NFT sub-account, and assert the NFT row renders distinctly.

### 41.5 Exercise 5: Feature-Flag Override via Bridge (90 min)

**Objective.** Use the bridge's `setFeatureFlag` (or equivalent) to flip a flag in `beforeAll` and assert the UI changes.

**Instructions.**
1. Locate the bridge message responsible for setting a feature flag in `e2e/bridge/types.ts`.
2. In a new spec `swapLiveAppFlagOn.spec.ts`, in `beforeAll`, send `setFeatureFlag("ptxSwapLiveApp", { enabled: true })` to the app before it navigates.
3. In the `it()`, open the swap tab and assert the Live App route is mounted (look for a testID like `swap-live-app-root` — if it does not exist, add the product ticket and stop; do not hack around it).

**Verification.** Three green runs. Flip the flag to `false` and rerun; the test should fail as expected.

**Hints.**
- The bridge `setFeatureFlag` contract is documented in Chapter 35. If the payload shape is wrong, the app ignores the message silently — attach a `onerror` logger while iterating.
- Not every flag propagates at runtime without an app restart. Check `FeatureFlagsProvider` in `apps/ledger-live-mobile/src/components/FeatureFlags/`.

**Stretch goal.** Write a helper `withFeatureFlags(flags) { ... }` that accepts an object and sends one bridge message per flag. Use it in your `beforeAll`.

### 41.6 Exercise 6: Nightly Swap-Only CI Matrix (90 min)

**Objective.** Add a scheduled CI entry that runs only swap-tagged specs.

**Instructions.**
1. Open `.github/workflows/test-mobile-e2e-reusable.yml` and its caller (likely `test-mobile-e2e.yml`).
2. Add a `schedule` trigger firing at `0 2 * * *` (02:00 UTC).
3. Add a matrix entry that passes `-t "SWAP|Swap|swap"` to `pnpm mobile e2e:test` — this filters by test-name pattern.
4. Limit to one shard (swap-only run — no need to parallelize across 10 shards for a dozen tests).

**Verification.** Run `act` locally or push to a throwaway branch and wait for the scheduled run. Confirm only swap specs execute.

**Hints.**
- Use a `if: github.event_name == 'schedule'` guard so the matrix entry does not run on every PR.
- The sharding algorithm (Chapter 38) is irrelevant here — you are opting out of sharding by setting shard count to 1.

**Stretch goal.** Add a Slack notification step that posts the Allure link to `#qa-mobile` on failure.

### 41.7 Exercise 7: Extend QAA-702 with an Empty-State Path (120 min)

**Objective.** Add a second `it()` to `swapHistoryExportERC20.spec.ts` that covers the no-history case.

**Instructions.**
1. Create a userdata fixture `swapHistoryEmpty.json` with at least one Ethereum account but **no** entries in `swap.history`.
2. Add a new `describe("Swap History — Empty")` block with a single `it($TmsLink("B2CQA-604-empty"), "hides the export button when no swaps exist")`.
3. Use the `expectEmptyState()` method from Exercise 1.
4. Assert the export button is not visible.

**Verification.** Both specs — the ERC20 export spec and the empty-state spec — pass on three consecutive runs.

**Hints.**
- `beforeAll` in the new `describe` re-inits the app with the new userdata.
- Detox's `toBeNotVisible()` is different from `toBeHidden()`. Check which one exists in your Detox major version.

**Stretch goal.** Write a helper `seededSwapHistory(opts: { count: number; erc20: boolean })` that generates the JSON in-memory and hands it to `app.init()`. This removes two userdata files in favor of one generator.

### 41.8 Challenge: Warm-Simulator Detox Wrapper (120 min, stretch)

**Objective.** Cut local iteration time in half by keeping the simulator booted between runs.

**Task.** Write a small Node or shell wrapper — `scripts/e2e-warm.sh` or `scripts/e2e-warm.ts` — that:

1. Checks whether the iOS simulator UUID is already booted (`xcrun simctl list | rg Booted`).
2. If not, boots it.
3. Runs `detox test --reuse -c ios.sim.debug -t "$@"` — the `--reuse` flag reuses an already-installed app binary.
4. Does **not** shut down the simulator on exit.
5. Accepts the same `-t` pattern and `-c` config as `pnpm mobile e2e:test`.

The existing pnpm command rebuilds and rebooks every time; `--reuse` is up to 30 s faster per run.

**Verification.** Three consecutive runs of a small spec (Exercise 2's password-entry close) should each start the test in under 10 s of wall time after the first.

**Hints.**
- `detox test --reuse` is documented in the Detox CLI reference. If the binary isn't installed, `--reuse` falls back gracefully.
- `xcrun simctl boot <UDID>` is the lowest-level way to boot a sim; `applesimutils --booted` lets you query the currently booted sim without parsing.
- Print the elapsed time at the end of the wrapper so you can track your speedup.

**Stretch.** Add an Android branch that checks `adb devices` and only boots an emulator if none is running.

<div class="chapter-outro">
<strong>Key takeaway.</strong> The exercises ramp from touching a POM to rewriting your local loop. Exercises 1–2 are warm-ups; 3–4 build the fixture and config intuition; 5–6 move into infrastructure; 7 extends your capstone; 8 is the quality-of-life payoff. If you finish all eight without peeking at solutions, you are past the onboarding bar and into steady-state contribution territory.
</div>

---

## Part 6 Final Assessment

<div class="quiz-container" data-pass-threshold="80">
<h3>Part 6 Final Assessment</h3>
<p class="quiz-subtitle">10 questions · 80% to pass · Covers all Part 6 chapters (32–41)</p>
<div class="quiz-progress"><div class="quiz-progress-bar"></div></div>

<div class="quiz-question" data-correct="C">
<p><strong>Q1.</strong> In React Native, which primitive is the equivalent of an HTML <code>&lt;div&gt;</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>Text</code></button>
<button class="quiz-choice" data-value="B">B) <code>ScrollView</code></button>
<button class="quiz-choice" data-value="C">C) <code>View</code> — the layout container that maps to <code>UIView</code> on iOS and <code>ViewGroup</code> on Android</button>
<button class="quiz-choice" data-value="D">D) <code>Pressable</code></button>
</div>
<p class="quiz-explanation"><code>View</code> is the root layout primitive. Everything visible is either a <code>View</code>, a <code>Text</code>, an <code>Image</code>, or a native component derived from <code>View</code>.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q2.</strong> What does <code>pnpm mobile e2e:build</code> produce on iOS?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) An Xcode project file</button>
<button class="quiz-choice" data-value="B">B) A <code>.app</code> bundle — a release-configured iOS simulator binary packaged with the Metro JS bundle</button>
<button class="quiz-choice" data-value="C">C) A CocoaPods lockfile</button>
<button class="quiz-choice" data-value="D">D) A test report</button>
</div>
<p class="quiz-explanation">The e2e build pipeline compiles the iOS app into a <code>.app</code> directory (for sim targets) with the JS bundled in. Detox installs this bundle into the simulator before running the specs.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q3.</strong> Which Detox matcher is most idiomatic for selecting a Button by its <code>testID</code>?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>element(by.id("export-swap-operations-link"))</code></button>
<button class="quiz-choice" data-value="B">B) <code>element(by.text("Export"))</code></button>
<button class="quiz-choice" data-value="C">C) <code>element(by.label("Export"))</code></button>
<button class="quiz-choice" data-value="D">D) <code>element(by.type("Button"))</code></button>
</div>
<p class="quiz-explanation"><code>by.id</code> selects by testID — stable across i18n and accessibility changes. Text and label matchers are fragile; type matchers are rarely specific enough.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q4.</strong> What is the purpose of the Detox bridge in Ledger Live Mobile?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) To speed up Metro bundling</button>
<button class="quiz-choice" data-value="B">B) To replace the React Native <code>Bridge</code> at runtime</button>
<button class="quiz-choice" data-value="C">C) To connect to the Ledger device</button>
<button class="quiz-choice" data-value="D">D) To exchange typed messages between the test runner (Node) and the app — feature-flag overrides, userdata seeding, and events like <code>sendFile</code></button>
</div>
<p class="quiz-explanation">The e2e bridge is a thin typed message channel. It complements Detox (which drives UI) by giving the test state-level control — things Detox alone cannot do.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q5.</strong> Why is the Page Object <em>hub</em> (<code>Application</code>) initialized once per spec file and reused across <code>it()</code>s?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) To keep POM references stable and share cached selectors — each POM is a singleton created via <code>lazyInit</code> on first access</button>
<button class="quiz-choice" data-value="B">B) To reset the app between <code>it()</code>s</button>
<button class="quiz-choice" data-value="C">C) Detox requires exactly one <code>Application</code> per process</button>
<button class="quiz-choice" data-value="D">D) TypeScript decorators break when re-instantiated</button>
</div>
<p class="quiz-explanation">The singleton pattern avoids reconstructing selector factories on every step and keeps <code>this</code> references consistent across <code>@Step</code> calls inside a spec file.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q6.</strong> Which property does the sharding algorithm minimise when splitting specs across CI runners?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) The number of specs per shard</button>
<button class="quiz-choice" data-value="B">B) The maximum shard wall-clock — it uses historical durations to balance the slowest shard against the fastest</button>
<button class="quiz-choice" data-value="C">C) The total CPU used</button>
<button class="quiz-choice" data-value="D">D) The number of shards</button>
</div>
<p class="quiz-explanation">A naive round-robin leaves one shard finishing 4x slower than its peers. The algorithm bin-packs specs by recorded duration so the slowest shard drives the build time down.</p>
</div>

<div class="quiz-question" data-correct="C">
<p><strong>Q7.</strong> In QAA-702, why does the test register the <code>sendFile</code> listener <em>before</em> tapping the Export button?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Detox requires listener-before-action</button>
<button class="quiz-choice" data-value="B">B) To avoid opening the native share sheet</button>
<button class="quiz-choice" data-value="C">C) Because the underlying <code>EventEmitter</code> emits synchronously — a listener attached after emission would miss the event</button>
<button class="quiz-choice" data-value="D">D) To warm the bridge socket</button>
</div>
<p class="quiz-explanation">The bridge client uses a plain <code>EventEmitter</code>. Emissions happen synchronously during the tap; late listeners see nothing.</p>
</div>

<div class="quiz-question" data-correct="D">
<p><strong>Q8.</strong> Which environment variable must be truthy for the app to route swap-history exports through the bridge instead of the native share sheet?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) <code>MOCK</code></button>
<button class="quiz-choice" data-value="B">B) <code>CI</code></button>
<button class="quiz-choice" data-value="C">C) <code>PWDEBUG</code></button>
<button class="quiz-choice" data-value="D">D) <code>DETOX</code> — consumed inside the app via <code>getEnv("DETOX")</code></button>
</div>
<p class="quiz-explanation">The app's <code>exportSwapHistory</code> branches on <code>getEnv("DETOX")</code>. Detox sets this when launching the bundle so the app cooperates with the test harness.</p>
</div>

<div class="quiz-question" data-correct="A">
<p><strong>Q9.</strong> What does <code>$TmsLink("B2CQA-604")</code> ultimately produce for Xray?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) A <code>tms</code> annotation in the Allure result JSON, which the Xray importer reads to update B2CQA-604's execution status</button>
<button class="quiz-choice" data-value="B">B) A network call to Jira at test runtime</button>
<button class="quiz-choice" data-value="C">C) A clickable link in the Detox console</button>
<button class="quiz-choice" data-value="D">D) A GitHub pull-request comment</button>
</div>
<p class="quiz-explanation">The wrapper is a no-op at runtime; it tags the Allure output with a TMS link. Xray ingests Allure results post-run and maps them to B2CQA test cases.</p>
</div>

<div class="quiz-question" data-correct="B">
<p><strong>Q10.</strong> A PR modifies <code>apps/ledger-live-mobile/e2e/specs/swap/</code> and <code>apps/ledger-live-mobile/src/screens/Swap/History/index.tsx</code>. Which CODEOWNERS must review it?</p>
<div class="quiz-choices">
<button class="quiz-choice" data-value="A">A) Only <code>@ledgerhq/wallet-xp</code></button>
<button class="quiz-choice" data-value="B">B) Both <code>@ledgerhq/wallet-xp</code> (owns <code>e2e/</code>) and <code>@ledgerhq/ptx</code> (owns <code>src/screens/Swap/</code>)</button>
<button class="quiz-choice" data-value="C">C) Only <code>@ledgerhq/ptx</code></button>
<button class="quiz-choice" data-value="D">D) Neither — CODEOWNERS doesn't apply to mobile</button>
</div>
<p class="quiz-explanation">CODEOWNERS applies path-by-path. The e2e folder belongs to wallet-xp; the swap screens belong to ptx. A cross-cutting PR auto-requests both.</p>
</div>

<div class="quiz-score"></div>
</div>

<div class="chapter-outro">
<strong>Part 6 complete.</strong> You now own the mobile E2E story end-to-end: React Native primitives, the Detox toolchain, the matcher DSL, the bridge protocol, the POM singleton, the sharding algorithm, the Allure pipeline, the CI graph, the daily workflow, one real ticket shipped (QAA-702), and an exercise track you can work through at your own pace. <strong>Part 7</strong> zooms out of the apps and into the shared runtime — the <strong>Ledger Wallet Components</strong> library and the <strong>Swap Live App</strong> — the two pieces of code that power the swap flow you just tested. Bring what you learned here: the same ticket lifecycle, the same Xray traceability, the same PR etiquette. The terrain changes; the professionalism does not.
</div>
