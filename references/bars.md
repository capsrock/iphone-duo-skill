# Vertical Bars on iPhone Duo

On iPhone Duo the system moves navigation bars, toolbars and tab bars from the top and bottom to a vertical bar along one edge. This preserves vertical space on a display that is wider and shorter than a normal iPhone, and keeps controls in reach.

## Contents

- [When bars go vertical](#when-bars-go-vertical)
- [Getting the behavior at all](#getting-the-behavior-at-all)
- [How items are represented](#how-items-are-represented)
- [Placement order and overflow](#placement-order-and-overflow)
- [Compression: toolbar vs tab bar](#compression-toolbar-vs-tab-bar)
- [Opting out](#opting-out)
- [Laying out custom UI around the bar](#laying-out-custom-ui-around-the-bar)
- [API summary](#api-summary)

## When bars go vertical

| Context | Bar orientation |
|---|---|
| Outer display (device closed) | Vertical |
| Inner display, portrait | **Horizontal** — enough vertical space |
| Inner display, landscape | Vertical |
| Split view: sidebar / content column | Horizontal |
| Split view: detail column | Vertical |
| Inspector | Always horizontal |
| Sheet on outer display | Vertical by default |
| Sheet on inner display | Horizontal for centered or leading placement; vertical for trailing |

Two apps sharing the inner display in Split View each place controls along their own outer edge — the left app's bar is on the left.

Bar placement stays fixed relative to the hardware, so it does not flip in right-to-left languages. **It does not stay on one side either:** the bar follows the camera, so rotating the closed device puts it on the left. Any code that assumes a trailing-edge bar is wrong half the time.

A useful mental image: the horizontal bar rotated 90° into a vertical stack. What lands in it depends on the screen — Notes contributes a toolbar, Clock a tab bar, Fitness both.

## Getting the behavior at all

The system only adapts bars that come from a navigation container.

- SwiftUI: `.toolbar { }` attached to a `NavigationStack` or `NavigationSplitView`
- UIKit: toolbar items set on a view controller inside a `UINavigationController`

A custom `UIToolbar`, `UINavigationBar`, `UITabBar`, or a hand-built bottom `HStack` will not become vertical. If bars are not adapting, this is the first thing to check.

A `UITabBar` added as a subview is the common case, and it does not migrate. Moving to `UITabBarController` or SwiftUI's `TabView` gets the behavior with no other work — Maps and Find My did exactly this migration for screens that had embedded custom bars. Staying custom means reproducing everything yourself, including details you may not have noticed: the system tab bar reveals each tab's label once you press and start dragging. If you must stay custom, size your bar against the reserved-region geometry rather than guessing.

Also keep a keyboard accessory bar attached to the keyboard rather than promoting it into the vertical bar.

## How items are represented

The system chooses a representation per context:

| Context | Uses |
|---|---|
| Vertical bar | Icon |
| Horizontal bar | Icon or title, preferring the icon |
| Overflow menu | Icon **and** title |

Two consequences worth internalizing:

- **A title-only item never appears in a vertical bar.**
- **An item built from a custom view never appears in a vertical bar.**

So give every non-text-only item both, and prefer symbols over text buttons — text labels keep an item pinned to a horizontal bar.

```swift
.toolbar {
    ToolbarItem(placement: .topBarTrailing) {
        Button { compose() } label: {
            Label("Compose", systemImage: "square.and.pencil")
        }
    }
}
```

An `accessibilityLabel` does not substitute: it feeds VoiceOver, not the system's choice of representation.

The system infers placement from the item's content. Its own Edit button stays horizontal, and UIKit custom or complex views default to horizontal.

For a custom view that carries text, ask whether the text merely reinforces the symbol or carries information of its own. "Share" next to a share glyph is reinforcement — drop it and the symbol suffices. A cart button showing a running total is information — leave that control in the horizontal bar.

Replace count text with a badge rather than a text label, which keeps the item eligible for the vertical bar:

```swift
// SwiftUI
Button { showInbox() } label: {
    Label("Inbox", systemImage: "tray")
}
.badge(unreadCount)
```

```swift
// UIKit (iOS 26+)
item.badge = .count(unreadCount)
item.badge = .string("New")
item.badge = .indicator()   // dot only, no number
```

## Placement order and overflow

Reserve the top of the vertical axis for primary navigation — Back or Close — followed by prominent actions such as Done. A navigation controller adds Back automatically.

- Custom Back/Close: `ToolbarItem(placement: .cancellationAction)`; in UIKit set `navigationItem.leftItemsSupplementBackButton = false` (the default) and use `leadingItemGroups`
- Prominent trailing action (Done): `.topBarPinnedTrailing` — pinned items move to overflow only when search is active and space runs out
- Group related items with `ToolbarItemGroup` rather than manual spacing; groups space themselves and adapt as space changes

Items overflow from bottom to top by default. Change the order with `visibilityPriority(_:)` — lower priority moves into the overflow menu first. Set priority on whole groups first (`ToolbarItemGroup` / `UIBarButtonItemGroup`), then individual items if you need finer control. Beyond `.high` and `.low` there are relative initializers — `init(higherThan:)` and `init(lowerThan:)` — when two items need a defined order rather than a coarse tier. UIKit's default is `.standard`; SwiftUI's is `.automatic`.

Overflow is not only a Duo situation: items also spill when the outer display is landscape or the keyboard appears.

Keep frequently used actions and status-carrying items (badges) visible longest. Put always-overflow actions in `ToolbarOverflowMenu`, and if the app has its own ellipsis menu, fold it into the system one so there is a single place to look.

Restrict an item to one axis with `axisBehavior(_:)`: `.horizontalOnly` for something that only makes sense wide (a segmented control), `.verticalPreferred` to bias an item toward the vertical bar when both exist.

## Compression: toolbar vs tab bar

When a vertical bar cannot hold both toolbar items and tabs, `toolbarVerticalCompressionBehavior(_:)` decides which survives.

- `.automatic` — system default: navigation-focused, keeps the tab bar and pushes toolbar items to overflow
- `.prefersTabBar` — same, stated explicitly
- `.prefersToolbarItems` — task-focused screens where the toolbar actions are the point; minimizes the tab bar instead

Pick based on what the view is for: browsing (keep tabs) versus completing a task (keep actions).

UIKit uses `UINavigationItem.verticalBarCompressionBehavior`, and its value for keeping actions is spelled `.prefersBarItems`:

```swift
navigationItem.verticalBarCompressionBehavior = .prefersBarItems
```

## Opting out

```swift
TabView { … }
    .toolbarVerticalBehavior(.disabled)
```

This falls back to standard horizontal top and bottom bars. Reserve it for interfaces genuinely better served horizontally — a full-screen video player with transport controls, or a non-scrolling layout like a calculator where horizontal space is precious. Two other cases Apple names: a single-page app whose content sits at the bottom, and a sheet whose only bar item is a close button, where a whole vertical bar is a poor trade for the width it costs.

Immersive full-screen apps (AR and similar) do not need vertical controls at all — build them as you would anywhere and hide the status bar.

Treat it as a stable property of a screen. Toggling it as the user navigates, or driving it from view state, produces a bar that animates in and out and costs people their sense of where controls live. To hide bars on one screen, use `toolbarVisibility(_:for:)` instead — that hides without changing the layout model.

The effective behavior is resolved by the container: a `NavigationStack` uses its topmost view, a `TabView` its selected view, a `NavigationSplitView` its trailing-most column.

## Laying out custom UI around the bar

`@Environment(\.toolbarVerticalEdge)` reports the system's preferred edge for the vertical bar as a `HorizontalEdge?`, whether or not a bar is currently visible. It is `nil` where the system never places one. Use it to align floating palettes or custom overlays with the system's choice.

```swift
struct ContentView: View {
    @Environment(\.toolbarVerticalEdge) var toolbarVerticalEdge

    var body: some View {
        FloatingToolPalette()
            .frame(maxWidth: .infinity,
                   alignment: toolbarVerticalEdge == .trailing ? .trailing : .leading)
    }
}
```

Because the bar sits on one edge, the content area is asymmetric. Rely on safe areas rather than assuming symmetric margins — and remember the opposite edge can also be inset when two apps share the inner display.

For a hero or background image that should run under the bar, use `.backgroundExtensionEffect()` (SwiftUI) or `UIBackgroundExtensionView` (UIKit). It mirrors and blurs the view into the surrounding safe area so the visual reads as full-bleed while content stays inset. Use it sparingly — typically one background element per screen.

Some layouts are better full width: immersive, non-scrolling interfaces where nothing collides with the Dynamic Island or status bar. A hybrid also works — background spanning the full width, scrollable content inset.

## API summary

SwiftUI, with availability:

| API | Availability | Purpose |
|---|---|---|
| `ToolbarItemVisibilityPriority` (`.low` / `.automatic` / `.high`) | iOS 27.0 | Overflow order |
| `ToolbarContent.visibilityPriority(_:)` | iOS 27.0 | Apply the priority |
| `ToolbarItemPlacement.topBarPinnedTrailing` | iOS 27.0 | Pin a prominent action |
| `ToolbarOverflowMenu` | iOS 27.0 | Always-overflow actions |
| `ToolbarItemAxisBehavior` (`.automatic` / `.horizontalOnly` / `.verticalPreferred`) | iOS 27.1 | Restrict an item's axis |
| `ToolbarVerticalCompressionBehavior` (`.automatic` / `.prefersTabBar` / `.prefersToolbarItems`) | iOS 27.1 | Toolbar vs tab bar |
| `View.toolbarVerticalBehavior(_:)` with `ToolbarVerticalBehavior.disabled` | iOS 27.1 | Opt out |
| `EnvironmentValues.toolbarVerticalEdge` | iOS 27.1 | Which edge the bar uses |
| `View.presentationPlacement(_:)`, `PresentationPlacement` (`.automatic` / `.center` / `.leading` / `.trailing`) | iOS 27.0 | Sheet placement; sheets only |
| `View.backgroundExtensionEffect()` | iOS 26 | Extend a background under the bar |
| `View.defaultTabBarPlacement(_:)` with `.sidebar` / `.tabBar` | iOS 27.0 | Sidebar vs tab bar where the bar cannot morph; pairs with `.tabViewStyle(.sidebarAdaptable)` |

SwiftUI also offers `toolbarOverflowMenu(content:)` as a modifier alongside the `ToolbarOverflowMenu` type, and `UITabBarController.sidebar.preferredPlacement` is the UIKit counterpart to `defaultTabBarPlacement(_:)` (`.automatic` / `.sidebar` / `.tabBar`, iOS 27.0+).

UIKit equivalents: `UIBarButtonItemVisibilityPriority` (default `.standard`), `UIBarButtonItem.axisBehavior`, `UIBarButtonItem.badge`, `UINavigationItem.pinnedTrailingGroup`, `UINavigationItem.leadingItemGroups`, `UINavigationItem.leftItemsSupplementBackButton`, `UINavigationItem.additionalOverflowItems`, `UINavigationItem.verticalBarCompressionBehavior` (`.prefersBarItems`), `UIViewController.preferredVerticalBarBehavior`, `UISheetPresentationController.preferredPlacement`, `UITraitCollection.verticalBarEdge`, `UIBackgroundExtensionView`.

These versions were read out of the iOS 27.1 SDK's `.swiftinterface` files (Xcode 27.1 beta, 27A9269, on 2026-09-19), not out of the documentation, which lags. The SDK spells Duo-era availability `@available(anyAppleOS 27.1, *)`.

The split is worth holding onto when you pick a deployment target. **Organizing a toolbar is 27.0; inspecting or steering the vertical bar is 27.1.** `visibilityPriority`, `topBarPinnedTrailing`, `ToolbarOverflowMenu` and `defaultTabBarPlacement` are 27.0, while `toolbarVerticalEdge`, `axisBehavior`, `ToolbarVerticalCompressionBehavior` and `toolbarVerticalBehavior` are 27.1. The vertical bar itself is not an API you adopt — the system does it, given a binary built with the 27.1 SDK — so an app with a lower deployment target still gets the behavior and simply gates the tuning APIs behind `if #available(iOS 27.1, *)`.

UIKit's Objective-C surface was not swept the same way; check the headers before relying on a UIKit spelling.

## Spacing and backgrounds

A vertical bar has no scroll-edge effect by default, the same as a horizontal one. It *does* draw a background when the Reduce Transparency accessibility setting is on, so custom view content has to stay legible either way (*Raise the bar with iPhone Duo*, 10:52–11:03). This is easy to ship without ever having seen it — turn the setting on once and walk the app.

Flexible spacers collapse to zero size on the vertical axis; fixed spacers keep their minimum. Either way, your app should not be adding spacing of its own, in horizontal or vertical bars — groups already handle it (11:07–11:19).

## Sheets

On the outer display, a sheet's controls move to the side like everything else. Disabling the vertical bar suits a sheet whose bar holds a single button; when you do, the sheet stops just short of the front-facing camera and the status bar repositions itself (*Design for iPhone Duo*).

On the inner display, sheets use standard horizontal bars in both portrait and landscape. The finer rule from the developer documentation — horizontal for centered or leading placement, vertical for trailing — is what `presentationPlacement(_:)` lets you steer.

When the device is partially folded, sheets slide aside rather than resting in the fold, along with alerts, menus, toolbar buttons and more. Apple's reason is worth remembering as a design principle: buttons that land in the fold are hard to tap.
