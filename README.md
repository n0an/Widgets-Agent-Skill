<h1 align="center">Widgets Agent Skill</h1>

<p align="center">
    <img src="https://img.shields.io/badge/iOS-14+-2980b9.svg" alt="iOS 14+" />
    <img src="https://img.shields.io/badge/swift-5.9+-F05138.svg" alt="Swift 5.9+" />
    <img src="https://img.shields.io/badge/version-1.2.0-blueviolet.svg" alt="Version 1.2.0" />
    <img src="https://img.shields.io/badge/WWDC-2020--2026-FF2D55.svg" alt="Covers WWDC 2020-2026" />
    <img src="https://img.shields.io/badge/license-MIT-lightgrey.svg" alt="MIT License" />
    <a href="https://agentskills.io/home">
        <img src="https://img.shields.io/badge/Agent%20Skills-Compatible-purple.svg" alt="Agent Skills Compatible" />
    </a>
</p>

> **🍎 The full WidgetKit story, WWDC 2020 → 2026** - from the original timeline model and design principles through Lock Screen / accessory widgets (iOS 16), interactive widgets and StandBy (iOS 17), Controls and Live Activity broadcast (iOS 18), the tinted Home Screen / push-updated / relevance / spatial widgets (iOS 26), and the iOS 27 additions (landscape Dynamic Island, `systemExtraLargePortrait`, `allowedExecutionTargets`) - plus the old → new migration map (SiriKit `.intentdefinition` → App Intents, ClockKit → WidgetKit).

An agent skill that helps AI coding agents like Claude Code, Codex, Cursor, and Gemini write correct SwiftUI WidgetKit code - timeline providers, configurations, families, accented/tinted rendering, interactive widgets, Controls, Live Activities, Smart Stack relevance, and push-updated widgets - so widgets render correctly in every mode, update within their budget, and behave correctly across the extension/app process boundary.

It uses the [Agent Skills](https://agentskills.io/home) format, so it works smoothly with Claude Code, Codex, Gemini, Cursor, and more.


## What It Covers

- **Fundamentals** - the widget extension process, the App Group boundary, `WidgetBundle`, the `Widget` protocol, the timeline lifecycle, `containerBackground(for: .widget)`, content margins, deep linking (`widgetURL` / `Link`), and the "which surface" map
- **Timeline providers** - `TimelineProvider` vs `AppIntentTimelineProvider`, `TimelineEntry`, `Timeline`, reload policies (`.atEnd` / `.after(date)` / `.never`), the daily reload budget, `WidgetCenter` reloads, reload-on-background, self-updating `Text` timers, `TimelineEntryRelevance`
- **Configuration** - `StaticConfiguration`, `AppIntentConfiguration` + `WidgetConfigurationIntent`, dynamic options and entity-backed parameters, defaults, and migrating a pre-iOS-17 SiriKit widget with `CustomIntentMigratedAppIntent`
- **Families & layout** - `WidgetFamily` (including `systemExtraLarge` / `systemExtraLargePortrait` and the accessory families), `supportedFamilies`, the per-family switch, sizing without hardcoding, `ViewThatFits`, `ContainerRelativeShape`, content margins
- **Lock Screen & watch** - accessory widgets, watch complications, `widgetLabel`, `AccessoryWidgetBackground`, gauges, the empty-`containerBackground` transparency trick, and ClockKit → WidgetKit migration
- **Rendering modes & tinting** - `@Environment(\.widgetRenderingMode)` (`.fullColor` / `.accented` / `.vibrant`), `widgetAccentable()`, `widgetAccentedRenderingMode()`, designing for the tinted/clear Home Screen and the two accent groups, and the **luminance-to-alpha trick** for keeping a multicolor glyph crisp in tinted mode
- **Interactive widgets** - `Button(intent:)` / `Toggle(intent:)`, the app-process boundary, the App Group store, `reloadTimelines`, `invalidatableContent`, `isDiscoverable`, and `allowedExecutionTargets` (the 27 releases)
- **Controls** - `ControlWidget`, `StaticControlConfiguration` / `AppIntentControlConfiguration`, `ControlWidgetToggle` (`SetValueIntent`) vs `ControlWidgetButton` (`AppIntent` / `OpenIntent`), value providers, `ControlCenter` reloads, status/action-hint modifiers, and the macOS/watchOS 26 availability
- **Live Activities** - `ActivityAttributes` / `ContentState`, `ActivityConfiguration`, the `DynamicIsland`, `Activity.request` / `update` / `end`, `ActivityContent` staleness, push (token / channel / push-to-start), `supplementalActivityFamilies` / `activityFamily`, the StandBy and landscape-Dynamic-Island behaviors, `LiveActivityIntent`, and the Info.plist keys
- **Relevance & Smart Stacks** - `TimelineEntryRelevance`, `relevance()` + `WidgetRelevance` / `WidgetRelevanceAttribute` / `RelevantContext`, the watchOS `RelevanceConfiguration` + `RelevanceEntriesProvider`, and **push-updated widgets** (`WidgetPushHandler`, iOS 26)
- **Previews & testing** - the `#Preview(as:)` macro variants, `WidgetPreviewContext`, testing every family and rendering mode, WidgetKit developer mode, and a debugging checklist for "won't update / stale / blank"
- **Migration & deprecations** - the old → new map: SiriKit `.intentdefinition` / `IntentConfiguration` → `AppIntentConfiguration` + `WidgetConfigurationIntent`, `IntentTimelineProvider` → `AppIntentTimelineProvider`, ClockKit → WidgetKit (`CLKComplicationWidgetMigrator`), `PreviewProvider` → `#Preview`, `CustomIntentMigratedAppIntent`, dual-path `#available`, and what's genuinely deprecated vs legacy-but-supported
- **Data & networking** - reading App-Group snapshots, `async` and background `URLSession` fetches from the provider, image downsampling, SwiftData/Core Data in widgets and intents, `NSWidgetUsesLocation`, and Lock Screen privacy
- **Design & gallery** - glanceable/relevant/personal, designing each size, tap styles, layout margins and type, the explicit don'ts (no app icon/name, no "last updated", no spinners), the gallery snapshot, and placeholder design
- **Deep linking & navigation** - `widgetURL` vs `Link`, per-family tap rules, URL-scheme design, handling on the app side (`onOpenURL` / scene delegate), Live Activity `widgetURL` / `keylineTint`, and control `OpenIntent`s
- **Animations & transitions** - the entry-diffing model, `contentTransition(.numericText)`, `.transition` + `.id` identity, the optimistic `Toggle`, `invalidatableContent`, and what does not animate (`symbolEffect`, loops, video)
- **Platforms** - visionOS spatial widgets (`supportedMountingStyles`, `widgetTexture`, `levelOfDetail`), CarPlay widgets and Live Activities, the macOS menu bar, watchOS, and a cross-platform availability map
- **Worked example** - one widget built end to end (a `systemMedium` Quick Actions widget): entry + provider, multiple deep links, full-color-vs-tinted handling, an entry-driven `MeshGradient` animation, `contentMarginsDisabled`, and the preview, with pointers into the deeper references
- **Anti-patterns** - ~40 catches including `UserDefaults.standard` in the extension, gesture-based "interactivity", polling reload policies, missing `containerBackground`, designing only for full color, struct-vs-enum rendering modes, the wrong control intent type, `staleDate` on the wrong type, and the WWDC-2026-is-iOS-27 naming


## Installing

You can install this skill into Claude Code, Codex, Gemini, Cursor, and more by using `npx`:

```bash
npx skills add https://github.com/n0an/Widgets-Agent-Skill --skill widgets
```

If you get the error `npx: command not found`, it means you don't currently have Node installed. You need to run this command to install Node through Homebrew:

```bash
brew install node
```

And if *that* fails it usually means you need to [install Homebrew](https://brew.sh) first.

When using `npx`, you can select exactly which agents you want to use during the installation. You can also select whether the skill should be installed just for one project, or whether it should be made available for all your projects.

### Alternative install methods

**Claude Code:**

```bash
/plugin install n0an/Widgets-Agent-Skill
```

**Gemini:**

```bash
gemini extensions install https://github.com/n0an/Widgets-Agent-Skill.git --consent
```

Alternatively, you can clone this whole repository and install it however you want.


## Using Widgets

The skill is called Widgets, and can be triggered in various ways. For example, in Claude Code you would use this:

> /widgets

And in Codex you would use this:

> $widgets

In both cases you can provide specific instructions if you want only a partial review. For example, `/widgets Build a Lock Screen accessory widget for SyncStatus.swift` on Claude, or `$widgets Fix the timeline reload policy in CaffeineProvider.swift` in Codex.

You can also trigger the skill using natural language:

> Use the Widgets skill to figure out why my widget looks wrong in tinted mode.


## Why Use an Agent Skill for Widgets?

WidgetKit has grown across many releases - interactive widgets in iOS 17, Controls in iOS 18, the tinted/clear Home Screen and push-updated widgets in iOS 26, and the landscape Dynamic Island in iOS 27 - and most LLM training data reflects the older shape. So agents routinely generate widget code that:

- Reads `UserDefaults.standard` in the extension (a different sandbox) instead of an App Group, so the widget shows nothing or stale data
- Wires up taps with `onTapGesture` / closures, which silently do nothing - the only interactivity is an `AppIntent` button or toggle
- Mutates data in an intent but never calls `WidgetCenter` to reload, so the widget never updates
- Polls with a tight `.after(60s)` reload policy that exhausts the daily reload budget by mid-morning
- Omits `containerBackground(for: .widget)`, or adds an opaque background to an accessory widget that should be transparent
- Designs only for full color, so the widget turns into a white blob on the Lock Screen or tinted Home Screen
- Confuses the API surface: treats the rendering modes as enums, uses `.after(timeInterval)`, reaches for `controlWidgetStatusText`, puts `staleDate` on the configuration, or calls the WWDC 2026 cohort "iOS 19"

This skill:

- **Catches anti-patterns** LLMs default to, with before/after fixes
- **Routes work to the right surface** with explicit decision trees (widget vs Live Activity vs Control, which configuration, which reload policy, which interactive element)
- **Covers newer APIs** like Controls, push-updated widgets, relevance widgets, spatial widgets, and the landscape Dynamic Island
- **Enforces the system contract** - the process boundary, the synchronous placeholder, the reload budget, the accent groups, and the tinted-mode design rules


## Contributing

Contributions are welcome - whether adding new checks, improving existing examples, or fixing typos.

- Keep Markdown concise. There is a token cost to using skills, so respect the token budgets of users.
- Do not repeat things LLMs already know. Focus on edge cases, surprises, and common mistakes.
- All work must be licensed under the MIT license.


## License

Available under the [MIT License](LICENSE), which permits commercial use, modification, distribution, and private use.
