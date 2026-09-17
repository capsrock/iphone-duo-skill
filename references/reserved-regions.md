# The Fold, Reserved Regions, and Arrangement Views

Three things are easy to confuse. **Displacement** is a design pattern — moving existing elements to fit the space available — and has no API of its own. **Arrangement** is an API for laying out two views. **Reserved regions** is an API for the areas to avoid, and it is what you build displacement on when you have to do it yourself.

## Contents

- [Poses](#poses)
- [Reserved regions](#reserved-regions)
- [Reading regions in SwiftUI](#reading-regions-in-swiftui)
- [Design rules for the fold](#design-rules-for-the-fold)
- [Split views](#split-views)
- [Arrangement views](#arrangement-views)
- [Hinge state and camera](#hinge-state-and-camera)

## Poses

People hold the device closed, fully open, partly folded like a book, standing on its edges, or laid down like a laptop. Poses are a continuum, not a set of modes.

Apple's guidance is unambiguous: **do not design a layout per pose.** Use size classes so the layout adapts naturally as the window changes size and shape. A hands-free variant for a laid-down device is acceptable only if it keeps the same controls and the same hierarchy — anything more and people have to relearn the app as they move it.

Avoid dramatic rearrangement as the device folds. Move only what must move to stay visible and tappable. Controls that vanish or jump are hard to track.

Guidance for moving elements:

- Elements that adapt independently move alone; elements that work together move together to keep the relationship legible.
- Do not move something so far that its connection to where it came from is lost.
- **Do not displace continuously scrolling content** — articles, feeds, documents, lists. Scrolling is already how they adapt, and moving them between regions breaks continuity.
- Choose the destination by the element's purpose and how the device is being held.

The system handles its own: action sheets, alerts, menus and popovers reposition around reserved regions, and split views adjust column widths — Reminders settles its two columns into an even 50/50 split when folded.

Apple's own worked examples are instructive about scope. Selecting a photo in an album moves the photo *and* its context menu together, aligned around the fold, rather than flinging the menu into the trailing region alone. A Fitness-style grid keeps every tile interactive by preserving the outer margins and widening the spacing around the hinge — the layout is not rearranged, only respaced. A focused search field keeps its position over the view it is searching, adapting width as the device folds. The common thread: move, resize, or respace what is already there.

## Reserved regions

A reserved region is an area of a view that something else claims. iOS models two kinds:

| Kind | Meaning | Examples |
|---|---|---|
| `occlusion` | Hardware covers content | Outer front camera (always present, expands into the Dynamic Island for Live Activities); inner front camera (present only while the camera is active); window controls on other platforms |
| `division` | Content splits into separate areas | The folding region when the device is partly open |

Each region carries a `frame`, `margins` (extra inset for interactive content), `isActive`, `kind`, and an `id`. Queries return every region intersecting the view **regardless of whether it is currently active** — a fold region exists but is inactive when the device is flat. Check `isActive` before treating one as real.

Framework-provided containers already adapt: alerts, context menus, sheets and split views move to avoid the fold, and split views adjust column widths and margins to match the inner display's symmetry. Custom layouts do the work themselves.

## Reading regions in SwiftUI

```swift
GeometryReader { proxy in
    RegionAvoidingLayout(
        regions: proxy.reservedRegions(kind: .occlusion)
    ) {
        ForEach(items) { item in
            ItemView(item)
        }
    }
}
```

The proxy can come from `GeometryReader` or from the `onGeometryChange` modifier — the latter is often the lighter option when you only need to react to a change rather than lay out inside a reader.

Signature:

```swift
func reservedRegions(
    kind: ReservedRegion.Kind,
    options: ReservedRegion.QueryOptions = [],
    layoutDirectionBehavior: LayoutDirectionBehavior = .mirrors
) -> [ReservedRegion]
```

Queries return only active regions unless you ask otherwise:

```swift
let regions = proxy.reservedRegions(kind: .division, options: .includeInactive)
```

Including inactive regions is how you make a decision that should hold regardless of the current fold state — keeping a grid at an even number of columns, for instance, so content always divides cleanly whenever the device does fold. A fold region that is inactive has zero width.

Reserved regions have a second use beyond avoidance: placing custom UI *outside* the safe area without colliding with system UI, which is how you build an edge-to-edge interface or your own bar. That is the sanctioned exception to "keep interactive elements inside the safe area."

By default the geometry is mirrored for right-to-left layouts, which is what you want when feeding a `Layout` — the layout already flips its subviews, so mirroring the regions keeps intersection tests correct. Pass `.fixed` only when you are positioning content manually against physical hardware locations.

UIKit: `UIView.reservedRegions(kind:options:)` returning `[UIView.ReservedRegion]`.

Note this is the *right* use of `GeometryReader` — reading the container you are in. The anti-pattern is reading the screen.

## Design rules for the fold

**Keep interactive elements out of the fold.** Non-interactive content may cross it. The failure mode to look for: a large, centered, tappable element. A hero image that acts as a button, a media surface that responds to taps, a big glyph that plays audio — these sit exactly where the fold lands in book pose, and the defect is invisible until someone tests a folded pose.

**Prefer containers that adapt on their own.** Notes' split view adjusts pane widths so both stay visible as the device folds. That behavior is free from standard components and expensive to rebuild.

**In grids, prefer an even number of columns** so content divides cleanly at the fold.

**Use `ReservedRegion` for anything the system cannot move for you** — keep important elements clear of the center.

## Split views

A split view expands on the inner display and collapses to a single pane on the outer display — the same adaptation it already performs between regular and compact environments elsewhere. Built from standard components, it also adapts to reserved regions automatically, adjusting column widths and margins around the fold.

If your hierarchy has a natural list/detail shape, `NavigationSplitView` is usually the correct answer for the inner display, and it costs nothing on existing iPhones because it collapses.

## Arrangement views

An arrangement view holds a primary and a secondary view and organizes them by available size, orientation, and reserved regions.

**Split arrangement** — divides the area. Side by side when wider than tall, stacked when taller than wide. Adapts placement around the fold. Use when the content already reads as an `HStack` or `VStack`.

```swift
ArrangementView {
    NowPlayingView()
} secondary: {
    LyricsView()
}
.arrangementViewStyle(.split)
```

**Overlay arrangement** — layers primary over secondary in z-order. When the device is closed or fully open the primary sits on top; when partly open the views separate to either side of the fold. Use when the content reads as a `ZStack`, such as playback controls over video.

```swift
ArrangementView {
    PlayerControls()
} secondary: {
    VideoPlayer()
}
.arrangementViewStyle(.overlay)
```

Constrain the axes when only one makes sense:

```swift
.arrangementViewStyle(.split.axes(.horizontal))
```

Two constraints worth respecting:

- **Keep navigation outside.** An arrangement view lays out content; it does not handle navigation. Put `NavigationSplitView` or `TabView` around it, never inside it.
- **Do not nest it in a navigation split view, list, or scroll view** — part of the content can become unreachable.

If an app has no layered or side-by-side content, an arrangement view has nothing to do. It is not a required adoption.

In an overlay arrangement, the secondary view can tell whether it is currently layered or side by side and change what it shows. Read the z-order: greater than zero means it is stacked, so a compact form is appropriate.

```swift
struct UpNextView: View {
    @Environment(\.overlayArrangementZIndex) private var zIndex: Int

    var body: some View {
        UpNextList(minimization: zIndex > 0 ? .collapsed : .expanded)
    }
}
```

UIKit: `UIArrangementViewController` with `setViewController(_:for:)` for `.primary` / `.secondary` and `updateArrangement(_:)`; read `state(for:)?.zIndex` for the same signal.

Constraining a split to one axis means the view shows a single pane when that axis does not match the current layout — `.split.axes(.horizontal)` shows one view when the container is taller than it is wide.

## Hinge state and camera

Hinge state is observable, but treat it as a signal for live effects — not for layout. Layout decisions belong to size classes and reserved regions, which describe the space you actually have.

```swift
GuitarView(pitchBend: pitchBend)
    .onHingeChange { _, context in
        // A nil hinge means a device without one
        if let hinge = context.hinge, hinge.status == .partiallyOpen {
            pitchBend = calculatePitchBend(angle: hinge.angle)
        } else {
            pitchBend = 0
        }
    }
```

The closure receives the previous and current context; `hinge.angle` is an `Angle`. UIKit exposes the same through `UIHingeInteraction`, with `UIHinge.Status` of `unknown` / `closed` / `partiallyOpen` / `fullyOpen`.

For camera apps: the display an app occupies and the direction a camera faces can change as the device opens, closes and rotates. Apple's guidance lives in *Choosing a camera by the direction it faces*. While the device is open and capturing with the rear camera, an app can show supplementary content on the outer display (subject preview, teleprompter) through a scene accessory — `CameraCaptureAccessory` in SwiftUI (iOS 27.1). Apps without a camera session can ignore all of this.

Note the conditions Apple states for that accessory: the app must be full screen on the inner display and have an active camera session. It is not a general "put something on the other display" capability.

The more broadly useful piece is `sceneAccessory(content:)` (iOS 27.0), which is not Duo-specific — it also drives an external display while the phone acts as a controller, for example. Availability is managed by the system and changes at runtime, so observe it (`onAvailabilityChange`) and disable the corresponding control rather than assuming it stays available.

## Multiple windows

Every app participates in multitasking on Duo. Two apps side by side and a video stacked above an app look the same from the app's side: a size change. Use size classes and scene geometry.

Duo is also the first iPhone that shows multiple instances of the same app's UI — the same capability as iPad, so apps that already support it get it here. **New windows can only be created on the inner display.** Because the ability comes and goes, handle the failure rather than assuming success: request with `UIApplication.activateSceneSession(for:errorHandler:)` and check `UISceneError.Code` for `.requestDenied` (cannot create one right now) and `.multipleScenesNotSupported`. `UIWindowScene.ActivationAction` hides itself automatically when creation is unavailable, which is usually the better choice for a UI affordance.

Shared storage such as `@AppStorage` is shared across instances, and changes propagate through the normal SwiftUI and UIKit update mechanisms.
