---
name: iphone-duo-development
description: Adapt iOS apps for iPhone Duo, Apple's foldable iPhone (announced 2026-09-09, ships 2026-10-23 with iOS 27.1) — resizable layouts, the folding region, vertical toolbars and tab bars, split/arrangement views, and the Xcode 27.1 build requirement. Use this skill whenever the user mentions iPhone Duo, a foldable iPhone, the inner or outer display, the fold or hinge, ArrangementView, ReservedRegion, vertical bars, or asks why their app looks wrong on a foldable. Also use it when the user is making an existing iPhone app resizable, adopting size classes, adding NavigationSplitView for a wider screen, or preparing an App Store submission or featuring nomination timed to the Duo launch — those tasks are all the same work even when Duo is not named.
---

# iPhone Duo Development

iPhone Duo has two displays joined by a center hinge: a compact outer display used when closed, and a 7.6-inch inner display when open. It runs iOS 27.1. Content moves between displays as the device opens, closes, folds partway, and rotates.

The single most useful thing to understand: **this is not a new platform.** It is an iPhone whose window changes size a lot. Apple says so directly — if your app already works on iPad and Mac, or resizes for iPhone Mirroring, you are most of the way there. Almost all of the work is making layouts resize, which is the same work as iPad adaptation.

## When to Activate

- Preparing an existing iOS app for iPhone Duo
- Diagnosing layouts that break on the inner display, in Split View, or when the device folds
- Adopting size classes, `NavigationSplitView`, or resizable layouts on iPhone
- Working with the fold: `ReservedRegion`, `ArrangementView`, hinge state
- Adapting toolbars and tab bars that the system moves to the side
- Planning a release or featuring nomination timed to the Duo launch

## Environment Requirements — Check This First

**Build with Xcode 27.1 or later.** Apple is explicit: in earlier versions your app does not extend under the status bar and camera, so it cannot use the full screen. This applies to CI too — an Xcode Cloud workflow pinned to an older Xcode produces a non-Duo-capable binary.

**The Duo simulator needs Xcode 27.1.** Device Hub's iPhone Duo requires it. Verified behavior on older toolchains: the `iPhone Duo` device type (`iPhone19,4`) declares `minRuntimeVersion = 27.1`, and the iOS 27.2 beta runtime lists `iPhone19,4` in `unsupportedDeviceTypes`, so `simctl create` fails with `Incompatible device` (403). Check before assuming a simulator is available:

```bash
xcrun simctl list runtimes -j | python3 -c "
import json,sys
for r in json.load(sys.stdin)['runtimes']:
    names=[t['name'] for t in r.get('supportedDeviceTypes',[])]
    print(r['name'], any('Duo' in n for n in names))
"
```

If no runtime supports Duo, the fold-specific work (`ReservedRegion`, pose testing) cannot be verified yet. Do the resizing work first — it is verifiable on any simulator — and stage the fold work behind that.

**Before Xcode 27.1 exists, resize the app under iPhone Mirroring on a Mac.** Apple establishes the equivalence directly: with iOS 27 people can resize an app larger than ever through iPhone Mirroring, and "opening and closing iPhone Duo works the same way" — the app may cross size class boundaries, "but it is still an iPhone app" (*Prepare your app for iPhone Duo*, 02:15–02:28). Mirroring therefore exercises the same resizing path without the Duo toolchain. What it will *not* surface is the asymmetric safe areas and margins caused by the vertical bar; that needs the simulator.

See `references/environment.md` for release timing, Xcode Cloud, and featuring nomination deadlines.

## What Your Build Actually Gets

How the app appears depends on which SDK it was linked against (*Prepare your app for iPhone Duo*, 00:30–01:11):

| Built with | Result |
|---|---|
| Pre-iOS 27 SDK | Runs. Closed, it uses the screen space to the left of the status bar and camera; open, it appears at a familiar size and aspect ratio |
| iOS 27 SDK | Extends to the left of the status bar area on the inner display — this is where existing iPhone resizing work pays off |
| iOS 27.1 SDK | Extends to the edge of the screen, and standard navigation and toolbar buttons lay out vertically under the status bar |

`UIRequiresFullScreen` is still honored, but the app resizes anyway when the device opens or closes (04:37), so it is not an exemption — and it is deprecated in Apple's documentation besides.

## The Layout Rules That Matter

### Size classes, not idiom or orientation

| State | Horizontal | Vertical |
|---|---|---|
| Outer display, portrait | compact | regular |
| Outer display, landscape | compact | compact |
| Inner display | regular | regular |

Two layouts cover every pose: **compact width for the outer display, regular width for the inner display.** Do not build a layout per pose — poses are a continuum, and per-pose layouts make controls jump around.

Never branch on `UIDevice.userInterfaceIdiom` or `UIInterfaceOrientation` for layout. Size classes and the container's own bounds are the only reliable signals. Prefer deciding on *available width* rather than orientation even when you want a landscape-specific look — a window can be landscape and narrow.

Apple is unusually blunt about orientation (*Prepare your app for iPhone Duo*, 03:42–04:51):

> The inner display doesn't honor your supported interface orientations.

> iPhone Duo respects your supported interface orientations, but your app will scale on the inner display, including in Split View multitasking.

So a portrait-only app is not exempt — it just gets scaled, while the size classes are regular/regular either way. The outer display behaves like a normal iPhone, and Apple calls Duo "a great opportunity to support landscape orientation" because people set the device down like a tent.

Avoid `UIScreen.main`: on a two-display device it is ambiguous and "will be deprecated in a future release" (03:57). Prefer the environment, trait collection, or the scene's bounds; read the screen dynamically from the window scene if you truly need it, and use `traitCollection.displayScale` in place of `UIScreen.main.scale`.

To fit the screen's corners, use the iOS 26 Concentricity APIs, updated for Duo's screen shapes: `ConcentricRectangle` in SwiftUI, `UICornerConfiguration` in UIKit (04:18).

### Safe areas are asymmetric

This is the failure mode the simulator finds and window-resizing does not. Because the bar sits along one edge, opposing insets differ, and so do layout margins and content insets.

```swift
// Wrong — assumes left and right insets match
let width = view.bounds.width - view.safeAreaInsets.left * 2

// Right — each edge handled on its own
let width = view.bounds.inset(by: view.safeAreaInsets).width
```

Portrait-locked apps are the usual offenders: they have only ever exercised top and bottom insets, so nothing in the codebase was written with a left or right inset in mind.

Keep interactive elements and foreground content inside the safe area; extend backgrounds beyond it and behind the bars. SwiftUI already places content inside the safe area, so the thing to handle deliberately is the background (`ignoresSafeArea()` on the background layer only).

### Size relative to the container, never the screen

```swift
// Wrong — screen dimensions are meaningless when the window resizes
let width = UIScreen.main.bounds.width * 0.4

// Right — the container proposes a size; use it
GeometryReader { proxy in
    Rectangle().frame(width: proxy.size.width * 0.4)
}
```

`GeometryReader` is not the problem — reading the *screen* is. A proportional spacer driven by the container is legal, though on a much larger display a percentage of height becomes a large dead gap; prefer fixed grid spacing plus flexible spacers when you can accept the visual change. `containerRelativeFrame(_:_:)` measures the nearest container (a `NavigationStack`, a tab of a `TabView`, a scroll view, or the window) minus its safe area insets, which makes it a direct replacement for a `GeometryReader` that existed only to compute a ratio.

### Prefer system containers

`NavigationStack`, `NavigationSplitView`, `TabView`, sheets, popovers, alerts and menus already handle resizing, fold avoidance, and camera occlusion. Reaching for a custom container means reimplementing all of that. This is the highest-leverage rule in the whole skill: most Duo bugs are custom layout code doing what a standard container would have done correctly.

### Show more hierarchy on the inner display, not different features

The canonical pattern is Mail: closed, you see the message list *or* a message; open, you see both side by side. A `NavigationSplitView` expands on the inner display and collapses to one pane on the outer display automatically — the same regular/compact behavior it already has elsewhere.

Keep functionality and state identical across displays and poses. Someone mid-task who opens the device should land where they left off, with the same controls available.

## Bars Move to the Side

On the outer display, and for some views on the inner display, the system moves navigation bars, toolbars and tab bars to a vertical bar along one edge. The inner display in portrait is the exception — it keeps horizontal bars.

You get this for free **if** your bar content lives on a navigation container: `.toolbar { }` on a `NavigationStack` or `NavigationSplitView` in SwiftUI; toolbar items on a view controller inside a `UINavigationController` in UIKit. A hand-rolled `UIToolbar` or a custom bottom bar `HStack` will not participate.

The rule that bites most often: **a toolbar item with a title but no icon is not shown in a vertical bar, and neither is one built from a custom view.** Give every item both a title and a symbol — the system picks the icon for the vertical bar and uses the title in the overflow menu.

```swift
ToolbarItem(placement: .topBarTrailing) {
    Button { showFilter = true } label: {
        Label("Filter", systemImage: "line.3.horizontal.decrease")
    }
}
```

Read `references/bars.md` before tuning bar behavior — it covers placement order, overflow priority, the sheet and split-view exceptions, and the full API list with availability.

## The Fold

Fold-related APIs come in layers. Reach for the highest one that solves your problem — computing a layout from the hinge angle yourself is almost always the wrong level.

| Layer | What it gives you |
|---|---|
| System components | Alerts, sheets, menus, split views — adapt with no code |
| Arrangement view | An `HStack`/`ZStack`-shaped container that adapts to the fold |
| Reserved regions | Rectangles of the areas to avoid |
| Hinge | The raw fold angle — for live effects, not layout |

When the device is partly open, the folding region divides the inner display. iOS models this and the cameras as **reserved regions**: `division` (the fold splits content) and `occlusion` (hardware covers content).

What matters for design: **keep interactive elements out of the fold.** Non-interactive content may cross it. A large tappable element centered on screen — a hero image that is also a button, a character that plays audio when tapped — lands exactly on the fold in book pose. This is the most common app-specific Duo defect and it is invisible until you test a folded pose.

Standard components move themselves. Alerts, menus, sheets and split views already avoid the fold. Custom layouts query the regions:

```swift
GeometryReader { proxy in
    let folds = proxy.reservedRegions(kind: .division)
    // Inspect each region's frame and isActive, then lay out around it
}
```

Read `references/reserved-regions.md` for the `ReservedRegion` API, `ArrangementView`, and the pose guidance.

## Adaptation Checklist

Work down this list; it is roughly ordered by how much breakage each item causes.

1. Build with Xcode 27.1+, including CI.
2. Remove screen-based layout math (`UIScreen.main`, hardcoded iPhone dimensions).
3. Make every screen survive a width change — check compact and regular width.
4. Give toolbar items both a title and a symbol; make sure bars come from navigation containers.
5. Keep interactive elements off the fold; adopt `ReservedRegion` in custom layouts.
6. Consider `NavigationSplitView` where your hierarchy has a natural list/detail split.
7. Re-check sheets, popovers, and full-screen covers — a full-screen cover on a 7.6-inch display is often the wrong call.
8. Handle each safe area edge separately; test the app on both the left and right side of Split View.
9. Verify state and playback survive folding, unfolding, and rotation mid-task.
10. Check large Dynamic Type on the inner display — the extra room makes big text sizes far more usable, so people who need them will be there.
11. Turn on Reduce Transparency and re-check: the vertical bar draws an opaque background in that mode.
12. Walk every screen in Device Hub across poses and both displays.

## Common Mistakes

**Expecting orientation lock to protect you.** It does not. The inner display honors the declared orientations by not rotating, and then scales the app — while still reporting regular/regular size classes. Staying portrait-only buys you a worse-looking result, not an exemption. Width adaptation is the real work.

**Designing for every pose.** Apple's instruction is to target two size classes instead — compact width outside, regular width inside — and to design the app to be freely resizable. The one optional exception they name is a seated, hands-free layout with media at the top and tappable controls on the stable base at the bottom, and even then it has to carry the same controls and general hierarchy as every other pose. Functionality must never be tied to a pose.

**Putting an `ArrangementView` inside a navigation container, list, or scroll view.** It lays out content and does not handle navigation; nesting it this way can make part of your view unreachable. Navigation goes outside.

**Overriding the system's bar placement.** Vertical bars are a core Duo pattern. `.toolbarVerticalBehavior(.disabled)` exists for genuinely full-bleed, non-scrolling interfaces — a video player, a calculator — not as a way to keep a familiar look.

**Assuming an app is fine because it launches.** A portrait-locked, non-resizing app still runs. It just gets scaled or letterboxed on a 7.6-inch display, which is exactly the outcome the work is meant to avoid.

## References

- `references/bars.md` — vertical bars, toolbar placement and overflow, sheet and split-view exceptions, API list
- `references/reserved-regions.md` — fold and camera regions, `ReservedRegion`, `ArrangementView`, poses
- `references/environment.md` — Xcode and simulator requirements, Xcode Cloud, App Store timing, featuring nominations

Primary sources, both readable without a browser:

```bash
curl -s "https://developer.apple.com/documentation/TechnologyOverviews/preparing-your-app-for-iphone-duo.md"
curl -s "https://developer.apple.com/tutorials/data/design/human-interface-guidelines/designing-for-iphone-duo.json"
```

Apple documentation pages under `/documentation/` return Markdown when you append `.md`. Human Interface Guidelines pages do not — fetch those from the `tutorials/data` JSON endpoint above.

Tech Talks 111461–111466 cover preparation, bars, poses, multiple displays, camera, and design. Each session page carries a full transcript and official code samples in its HTML, so they are readable without watching:

```bash
curl -sL "https://developer.apple.com/videos/play/tech-talks/111461/" | \
  python3 -c "import re,sys,html; t=sys.stdin.read(); m=re.search(r'<section id=\"transcript-content\">(.*?)</section>',t,re.S); print(html.unescape(re.sub(r'<[^>]+>',' ',m.group(1))))"
```

Apple Design Resources ships the iOS/iPadOS 27 UI Kit and iPhone Duo bezels.

