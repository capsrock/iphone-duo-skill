---
name: iphone-duo-development
description: Adapt iOS apps for iPhone Duo, Apple's foldable iPhone (announced 2026-09-09, ships 2026-10-23 with iOS 27.1) — resizable layouts, the folding region, vertical toolbars and tab bars, split/arrangement views, and the Xcode 27.1 build requirement. Use this skill whenever the user mentions iPhone Duo, a foldable iPhone, the inner or outer display, the fold or hinge, ArrangementView, ReservedRegion, vertical bars, or asks why their app looks wrong on a foldable. Also use it when the user is making an existing iPhone app resizable, adopting size classes, adding NavigationSplitView for a wider screen, or preparing an App Store submission or featuring nomination timed to the Duo launch — those tasks are all the same work even when Duo is not named.
---

# iPhone Duo Development

iPhone Duo has two displays joined by a center hinge: a compact outer display used when closed, and a 7.6-inch inner display when open. It runs iOS 27.1. Content moves between displays as the device opens, closes, folds partway, and rotates.

The premise of everything here: **this is not a new platform.** It is an iPhone whose window changes size a lot. Apple says so directly — if your app already works on iPad and Mac, or resizes for iPhone Mirroring, you are most of the way there. Almost all of the work is making layouts resize, which is the same work as iPad adaptation.

## When to Activate

- Preparing an existing iOS app for iPhone Duo
- Diagnosing layouts that break on the inner display, in Split View, or when the device folds
- Adopting size classes, `NavigationSplitView`, or resizable layouts on iPhone
- Working with the fold: `ReservedRegion`, `ArrangementView`, hinge state
- Adapting toolbars and tab bars that the system moves to the side
- Planning a release or featuring nomination timed to the Duo launch

## Build Requirement

**Build with Xcode 27.1 or later, including CI.** Apple is explicit: in earlier versions your app does not extend under the status bar and camera, so it cannot use the full screen. An Xcode Cloud workflow pinned to an older Xcode builds and passes, and quietly produces a binary that is not Duo-capable. Xcode 27.1 shipped on 2026-10-15 (27A9275).

**The Duo simulator runs only on the iOS 27.1 runtime.** iOS 27.0 does not support it, and neither does the iOS 27.2 beta runtime, so a newer runtime is not a substitute. Where no installed runtime supports Duo, do the resizing work first — it is verifiable on any simulator — and stage the fold work behind it.

`references/environment.md` has the command that checks which runtimes support Duo, how to test under iPhone Mirroring before the toolchain is updated, the simulator's limits, Xcode Cloud, and release timing.

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

`NavigationStack`, `NavigationSplitView`, `TabView`, sheets, popovers, alerts and menus already handle resizing, fold avoidance, and camera occlusion. Reaching for a custom container means reimplementing all of that, and most Duo bugs are custom layout code doing what a standard container would have done correctly.

### Show more hierarchy on the inner display, not different features

The canonical pattern is Mail: closed, you see the message list *or* a message; open, you see both side by side. A `NavigationSplitView` expands on the inner display and collapses to one pane on the outer display automatically — the same regular/compact behavior it already has elsewhere.

Keep functionality and state identical across displays and poses. Someone mid-task who opens the device should land where they left off, with the same controls available.

## Bars Move to the Side

On the outer display, and for some views on the inner display, the system moves navigation bars, toolbars and tab bars to a vertical bar along one edge. The inner display in portrait is the exception — it keeps horizontal bars.

You get this for free **if** your bar content lives on a navigation container: `.toolbar { }` on a `NavigationStack` or `NavigationSplitView` in SwiftUI; toolbar items on a view controller inside a `UINavigationController` in UIKit. A hand-rolled `UIToolbar` or a custom bottom bar `HStack` will not participate.

**A toolbar item with a title but no icon is not shown in a vertical bar, and neither is one built from a custom view.** Give every item both a title and a symbol — the system picks the icon for the vertical bar and uses the title in the overflow menu.

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

The design rule: **keep interactive elements out of the fold.** Non-interactive content may cross it. A large tappable element centered on screen — a hero image that is also a button, a character that plays audio when tapped — lands exactly on the fold in book pose. The defect is invisible until you test a folded pose.

Standard components move themselves. Alerts, menus, sheets and split views already avoid the fold. Custom layouts query the regions:

```swift
GeometryReader { proxy in
    let folds = proxy.reservedRegions(kind: .division)
    // Inspect each region's frame and isActive, then lay out around it
}
```

All of it is iOS 27.1 — `ReservedRegion`, `ArrangementView`, and the hinge types alike, verified in the iOS 27.1 SDK. Below that deployment target, gate with `if #available(iOS 27.1, *)`; there is no fold on an older system to handle.

Read `references/reserved-regions.md` for the `ReservedRegion` API, `ArrangementView`, and the pose guidance.

## Adaptation Checklist

Work down this list in order. The first group is what breaks; the second makes the app better on the larger display; the last is how you confirm all of it.

**Fix what breaks**

1. Build with Xcode 27.1+, including CI.
2. Replace custom containers that reimplement a system one — `NavigationStack`, `NavigationSplitView`, `TabView`, sheets, popovers. Most Duo bugs are custom layout code doing what a standard container would have done correctly.
3. Remove screen-based layout math (`UIScreen.main`, hardcoded iPhone dimensions).
4. Make every screen survive a width change — check compact and regular width.
5. Handle each safe area edge separately; test the app on both the left and right side of Split View.
6. Give toolbar items both a title and a symbol; make sure bars come from navigation containers.
7. Keep interactive elements off the fold; adopt `ReservedRegion` in custom layouts.
8. Verify state and playback survive folding, unfolding, and rotation mid-task.
9. Turn on Reduce Transparency and re-check: the vertical bar draws an opaque background in that mode.

**Use the larger display**

1. Re-check sheets, popovers, and full-screen covers — a full-screen cover on a 7.6-inch display is often the wrong call.
2. Consider `NavigationSplitView` where your hierarchy has a natural list/detail split.
3. Check large Dynamic Type on the inner display — the extra room makes big text sizes far more usable, so people who need them will be there.

**Confirm**

1. Walk every screen across poses and both displays. On the simulator, [`duoctl`](https://github.com/skarol/duoctl) drives the poses and captures each one (`open` / `close` / `half` / `rotate`, then `screenshot`); open Device Hub only for what duoctl cannot do.

## Common Mistakes

**Expecting orientation lock to protect you.** It does not. The inner display honors the declared orientations by not rotating, and then scales the app — while still reporting regular/regular size classes. Staying portrait-only buys you a worse-looking result, not an exemption. Width adaptation is the real work.

**Designing for every pose.** When Apple was asked what design mistake they expect to see on Duo, this was the answer. Their own design team started by enumerating poses and building an experience for each, then found that getting the fundamentals right covered all of them. Go deeper on a specific pose only where it genuinely serves the app — a media player moving controls to the lower half when the device is laid down — and keep the same controls and hierarchy when you do.

**Assuming one way of holding it.** People stand the device up with the camera side down, or with the outer display down. Both should work the same way.

**Putting an `ArrangementView` inside a navigation container, list, or scroll view.** It lays out content and does not handle navigation; nesting it this way can make part of your view unreachable. Navigation goes outside.

**Overriding the system's bar placement.** Vertical bars are a core Duo pattern. `.toolbarVerticalBehavior(.disabled)` exists for genuinely full-bleed, non-scrolling interfaces — a video player, a calculator — not as a way to keep a familiar look.

**Assuming an app is fine because it launches.** A portrait-locked, non-resizing app still runs. It just gets scaled or letterboxed on a 7.6-inch display, which is exactly the outcome the work is meant to avoid.

## References

- `references/bars.md` — vertical bars, toolbar placement and overflow, sheet and split-view exceptions, API list
- `references/reserved-regions.md` — fold and camera regions, `ReservedRegion`, `ArrangementView`, poses
- `references/environment.md` — Xcode and simulator requirements, Xcode Cloud, App Store timing, featuring nominations, and how to read Apple's primary sources (documentation, HIG, Tech Talk transcripts) without a browser

The account of Apple's design team enumerating poses (under Common Mistakes) comes from the iPhone Duo Group Lab (Meet with Apple, 2026-09-16), which has no public transcript, by way of d_date's write-up: https://zenn.dev/d_date/articles/d874e248ac7851
