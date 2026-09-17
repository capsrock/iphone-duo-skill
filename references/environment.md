# Toolchain, Simulator, and Release Timing

## Contents

- [Device and OS facts](#device-and-os-facts)
- [Xcode requirement](#xcode-requirement)
- [Simulator availability](#simulator-availability)
- [Xcode Cloud and CI](#xcode-cloud-and-ci)
- [App Store timing and featuring nominations](#app-store-timing-and-featuring-nominations)
- [Reading the primary sources](#reading-the-primary-sources)

## Device and OS facts

| | |
|---|---|
| Announced | 2026-09-09 |
| Pre-orders | 2026-10-16 |
| Availability | 2026-10-23 (70+ countries), 2026-10-30 (more regions) |
| Ships with | iOS 27.1 |
| Inner display | 7.6-inch, regular width |
| Outer display | 5.4-inch, compact width; wider and shorter than a normal iPhone |
| Model identifier | `iPhone19,4` |

Split View puts two apps side by side on iPhone for the first time, and two windows of the same app on the inner display. New windows can only be created on the inner display.

## Xcode requirement

**Build with Xcode 27.1 or later.** From Apple's own guidance: in earlier versions your app does not extend under the status bar and camera, so it cannot use the full screen on iPhone Duo. This is a build-time property of the binary — no runtime flag recovers it.

There are three steps, described in *Prepare your app for iPhone Duo* at 00:30–01:11:

| Built with | Result |
|---|---|
| Pre-iOS 27 SDK | The app runs. Closed, it uses the screen space to the left of the status bar and camera; open, it appears at a familiar size and aspect ratio |
| iOS 27 SDK | Extends to the left of the status bar area on the inner display — existing iPhone resizing work pays off here |
| iOS 27.1 SDK | Extends to the edge of the screen; standard navigation and toolbar buttons lay out vertically under the status bar |

You do not need every pose handled on day one — get resizing right first.

`UIRequiresFullScreen` continues to be honored, but the app still resizes as the device opens and closes, so it is not an escape hatch. It is also deprecated in Apple's documentation; treat "honored" as a transitional courtesy rather than a supported strategy.

Xcode 27.1 also renames the app modernization assistant to **App Resizability** and extends it to SwiftUI and iPhone Duo — worth running once the toolchain is available.

Check what a given Xcode is:

```bash
xcodebuild -version
DEVELOPER_DIR=/Applications/Xcode-27.1.0.app/Contents/Developer xcodebuild -version
```

Prefer `DEVELOPER_DIR` on individual commands over changing `xcode-select` globally, so other projects keep building with the toolchain they expect.

## Simulator availability

The iPhone Duo simulator in Device Hub requires Xcode 27.1. Older toolchains ship the device type but no runtime that supports it, and the failure is confusing if you have not seen it:

```
An error was encountered processing the command (domain=com.apple.CoreSimulator.SimError, code=403):
Incompatible device
```

The cause is visible in the profiles. The device type declares a minimum runtime:

```bash
plutil -p "/Library/Developer/CoreSimulator/Profiles/DeviceTypes/iPhone Duo.simdevicetype/Contents/Resources/profile.plist" \
  | grep -i "minRuntimeVersion\|modelIdentifier"
# minRuntimeVersion => 27.1
# modelIdentifier   => iPhone19,4
```

And a runtime can exclude it outright — observed on the iOS 27.2 beta runtime (24B5084k), which lists `iPhone19,4` under `unsupportedDeviceTypes`. A newer runtime version number does not imply Duo support.

Check which installed runtimes actually support the device before planning any fold testing:

```bash
xcrun simctl list runtimes -j | python3 -c "
import json,sys
for r in json.load(sys.stdin)['runtimes']:
    names=[t['name'] for t in r.get('supportedDeviceTypes',[])]
    print(r['name'], '| Duo:', any('Duo' in n for n in names))
"
```

Downloading a runtime (roughly 8 GB, and it takes a while):

```bash
DEVELOPER_DIR=/Applications/Xcode-27.1.0.app/Contents/Developer xcodebuild -downloadPlatform iOS
# specific version, if the asset exists for that Xcode:
xcodebuild -downloadPlatform iOS -buildVersion 27.1
```

If the asset is absent the command says so quickly (`iOS 27.1 is not available for download.`), which is a cheap way to find out whether a version is obtainable from the Xcode you have.

**Practical consequence:** when no Duo-capable runtime is installed, sequence the work so resizing comes first. Width and size-class adaptation is verifiable on any simulator and is most of the effort. Fold handling, pose testing, and anything gated on iOS 27.1 APIs waits for the toolchain.

### Testing before Xcode 27.1 exists

Use **iPhone Mirroring on a Mac** and resize the mirrored window. Apple draws the equivalence itself: with iOS 27 people can resize an app larger than ever through iPhone Mirroring, and "opening and closing iPhone Duo works the same way" — the app may react to different size class boundaries, "but it is still an iPhone app" (*Prepare your app for iPhone Duo*, 02:15–02:28).

The split is worth internalizing, because it tells you what you can and cannot sign off on before the simulator arrives:

| Found by resizing | Found only on the Duo simulator |
|---|---|
| Layout breaking near window corners, wrong layout guides, fixed widths, breakpoint assumptions | Asymmetric safe areas and margins caused by the vertical bar |

Portrait-locked apps carry the most risk in the second column: they have never exercised left or right safe area insets.

## Xcode Cloud and CI

A workflow pinned to an older Xcode silently produces a non-Duo-capable binary — it builds and passes, it just cannot use the full screen. Update the workflow's Xcode version as part of Duo work, not as an afterthought at submission time, and confirm the resulting build came from the intended toolchain.

## App Store timing and featuring nominations

From Apple's Getting Featured page:

> Featuring lead time varies — please give our team a minimum of two weeks notice.

> For wider featuring consideration, we recommend submitting a nomination up to three months in advance.

So for a launch-timed feature: **two weeks is the floor, not the target.** Counting back from 2026-10-23 availability, the hard deadline is 2026-10-09; earlier is materially better since editors plan ahead.

Nominations are filed before release and describe a planned update, so it is normal to submit while the work is still in progress. Fields include the nomination type (App Launch / App Enhancements / New Content — a Duo update to an existing app is App Enhancements), a publish date or range, a detailed description of what and why, platforms, countries, and up to five supplemental URLs. A public TestFlight link is a strong supplement because editors can see the thing working.

Editors weigh user experience, UI design, innovation, uniqueness, accessibility, localization, and the product page itself. There is no checklist that guarantees selection.

Sequence the release realistically: the Duo-optimized binary needs Xcode 27.1, then App Review, then release — all before the nominated publish date. If the toolchain slips, move the publish date rather than nominating against a release that cannot ship.

## Reading the primary sources

Apple documentation under `/documentation/` returns Markdown when you append `.md`, so no browser is needed:

```bash
curl -s "https://developer.apple.com/documentation/TechnologyOverviews/preparing-your-app-for-iphone-duo.md"
curl -s "https://developer.apple.com/documentation/SwiftUI/ArrangementView.md"
curl -s "https://developer.apple.com/documentation/SwiftUI/ReservedRegion.md"
```

Human Interface Guidelines pages return 404 for `.md`. Fetch them from the JSON endpoint and walk `primaryContentSections[].content`:

```bash
curl -s "https://developer.apple.com/tutorials/data/design/human-interface-guidelines/designing-for-iphone-duo.json"
```

Relevant Tech Talks: 111461 (prepare), 111462 (bars), 111463 (poses), 111464 (multiple displays and scenes), 111465 (camera), 111466 (design).

**Each session page embeds its full transcript and official code samples in the HTML**, so a session is readable — and quotable with timecodes — without watching it. The transcript lives in `<section id="transcript-content">` as `<span data-start="seconds">` sentences; the samples sit in the `supplement sample-code` block with their own timecodes. These samples have been the most reliable source for exact API spellings, since several symbols appeared there before reaching the public documentation or a shipped SDK.

```bash
curl -sL "https://developer.apple.com/videos/play/tech-talks/111463/" -o session.html
python3 - <<'PY'
import re, html
t = open('session.html', encoding='utf-8', errors='ignore').read()
m = re.search(r'<section id="transcript-content">(.*?)</section>', t, re.S)
for s in re.finditer(r'<span data-start="([\d.]+)">(.*?)</span>', m.group(1), re.S):
    sec = float(s.group(1))
    print(f"[{int(sec)//60:02d}:{int(sec)%60:02d}]", html.unescape(re.sub(r'<[^>]+>', '', s.group(2))).strip())
PY
```

Apple Design Resources provides the iOS and iPadOS 27 UI Kit (Figma and Sketch) and iPhone Duo bezels (Photoshop and PNG).

