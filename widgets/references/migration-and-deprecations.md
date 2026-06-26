# Migration and deprecations: the old way vs the modern way

WidgetKit shipped in iOS 14 (2020) and has changed its configuration model, its complication model, and its preview model since. A large fraction of widget code online - and in LLM training data - is the **legacy** shape. This file is the old → new map so you don't generate or perpetuate the outdated approach.

> **The headline:** since **iOS 17**, configurable widgets use the **App Intents** framework, not SiriKit intent-definition files. If your deployment target is iOS 17+ / watchOS 10+ / macOS 14+, there is **no reason** to touch `IntentConfiguration`, `IntentTimelineProvider`, `INIntent`, `.intentdefinition`, or ClockKit.

## The migration table

| Legacy API | Modern replacement | Modern since | Status today | How to migrate |
|---|---|---|---|---|
| `.intentdefinition` SiriKit custom intent + generated `INIntent` subclass | App Intent struct conforming to `WidgetConfigurationIntent` (`@Parameter`, `AppEnum`/`AppEntity`, `EntityQuery`) | iOS 17 | legacy, supported | Xcode **Editor ▸ Convert to App Intent…** on the intent-definition file |
| `IntentConfiguration<INIntent, Content>` | `AppIntentConfiguration<WidgetConfigurationIntent, Content>` | iOS 17 | legacy, **not** deprecated | swap the type in `body`; pass the migrated intent + an `AppIntentTimelineProvider` |
| `IntentTimelineProvider` (completion-handler, `Intent: INIntent`) | `AppIntentTimelineProvider` (async, `Intent: WidgetConfigurationIntent`) | iOS 17 | legacy, **not** deprecated | same provider logic; the associated `Intent` type changes and methods become `async` |
| SiriKit dynamic options (`INObjectCollection`, Intents Extension `provide…OptionsCollection`) | `@Parameter` + `EntityQuery` / `DynamicOptionsProvider` (no extension, no plist intent) | iOS 17 | legacy | move the option logic into the query / provider |
| `INRelevantShortcut` + `INInteraction.donate()` for Smart Stack rotation | App Intents relevance (`RelevantContext`, `RelevantIntentManager`) + `TimelineEntryRelevance` | iOS 18 | donation half deprecated | replace donations; keep `TimelineEntryRelevance` |
| ClockKit (`CLKComplicationDataSource`, 12 `CLKComplicationFamily` cases) | WidgetKit accessory widgets (4 accessory `WidgetFamily` cases) | watchOS 9 | **effectively deprecated** | add a Widget Extension + `CLKComplicationWidgetMigrator` |
| `PreviewProvider` + `.previewContext(WidgetPreviewContext(family:))` | `#Preview(as:)` macros | iOS 17 / Xcode 15 | superseded (not formally deprecated) | replace the `PreviewProvider` struct with `#Preview` |
| `recommendations() -> [IntentRecommendation]` | `recommendations() -> [AppIntentRecommendation<Intent>]` | iOS 17 | legacy, not deprecated | return `AppIntentRecommendation`s for watchOS/StandBy preconfigured widgets |

## Configurable widget: before → after

### Before - SiriKit `IntentConfiguration` (iOS 14-16)

```swift
// 1. INIntent generated from a .intentdefinition file (no Swift source you own):
public class SelectCharacterIntent: INIntent {
    @NSManaged public var character: INObject?
}

// 2. IntentTimelineProvider - completion-handler, Intent: INIntent:
struct Provider: IntentTimelineProvider {
    func placeholder(in context: Context) -> Entry { .placeholder }
    func getSnapshot(for configuration: SelectCharacterIntent, in context: Context,
                     completion: @escaping (Entry) -> Void) { completion(.sample) }
    func getTimeline(for configuration: SelectCharacterIntent, in context: Context,
                     completion: @escaping (Timeline<Entry>) -> Void) {
        completion(Timeline(entries: [makeEntry(configuration.character)], policy: .atEnd))
    }
}

// 3. IntentConfiguration:
struct CharacterWidget: Widget {
    var body: some WidgetConfiguration {
        IntentConfiguration(kind: "Character", intent: SelectCharacterIntent.self, provider: Provider()) { entry in
            CharacterView(entry: entry)
        }
    }
}
```

### After - App Intents `AppIntentConfiguration` (iOS 17+)

```swift
import AppIntents

// 1. An App Intent struct replaces the .intentdefinition file entirely:
struct SelectCharacterIntent: WidgetConfigurationIntent {
    static let title: LocalizedStringResource = "Select Character"
    @Parameter(title: "Character") var character: CharacterEntity   // AppEntity + EntityQuery
    init() {}
}

// 2. AppIntentTimelineProvider - async, intent is the FIRST argument:
struct Provider: AppIntentTimelineProvider {
    func placeholder(in context: Context) -> Entry { .placeholder }
    func snapshot(for configuration: SelectCharacterIntent, in context: Context) async -> Entry { .sample }
    func timeline(for configuration: SelectCharacterIntent, in context: Context) async -> Timeline<Entry> {
        Timeline(entries: [makeEntry(configuration.character)], policy: .atEnd)
    }
}

// 3. AppIntentConfiguration:
struct CharacterWidget: Widget {
    var body: some WidgetConfiguration {
        AppIntentConfiguration(kind: "Character", intent: SelectCharacterIntent.self, provider: Provider()) { entry in
            CharacterView(entry: entry)
        }
    }
}
```

### Preserving existing user configurations

When you migrate, the system silently maps each user's stored `INIntent` configuration onto your new App Intent **by matching parameter name and type**. Conform the new intent to `CustomIntentMigratedAppIntent` and set `intentClassName` to the **old** SiriKit intent class name:

```swift
struct SelectCharacterIntent: AppIntent, WidgetConfigurationIntent, CustomIntentMigratedAppIntent {
    static let intentClassName = "SelectCharacterIntent"   // the original INIntent class name
    static let title: LocalizedStringResource = "Select Character"
    @Parameter(title: "Character") var character: CharacterEntity
    init() {}
}
```

- If a parameter's **name or type doesn't match**, that parameter is dropped and the user's choice is lost - keep them identical (`INPerson` → `IntentPerson`, custom enum → `AppEnum`, custom object → `TransientAppEntity`).
- `CustomIntentMigratedAppIntent` is **only** needed for migration. A brand-new configurable widget just adopts `WidgetConfigurationIntent`.

### Supporting iOS 16 and 17 in one widget

If you must deploy below iOS 17, keep both paths behind `#available`:

```swift
struct MyWidget: Widget {
    @WidgetConfigurationBuilder
    func config() -> some WidgetConfiguration {
        if #available(iOS 17.0, macOS 14.0, watchOS 10.0, *) {
            AppIntentConfiguration(kind: kind, intent: ModernIntent.self, provider: ModernProvider()) { Entry in EntryView(entry: Entry) }
        } else {
            IntentConfiguration(kind: kind, intent: LegacyIntent.self, provider: LegacyProvider()) { Entry in EntryView(entry: Entry) }
        }
    }
    var body: some WidgetConfiguration {
        config().configurationDisplayName("My Widget").description("…")
    }
}
```

## ClockKit → WidgetKit (watch complications)

ClockKit is legacy. Build watch complications as WidgetKit accessory widgets, and provide a migrator so complications already on users' watch faces upgrade automatically:

```swift
class ComplicationController: NSObject, CLKComplicationDataSource, CLKComplicationWidgetMigrator {
    var widgetMigrator: CLKComplicationWidgetMigrator { self }

    func widgetConfiguration(from descriptor: CLKComplicationDescriptor)
        async -> CLKComplicationWidgetMigrationConfiguration? {
        switch descriptor.identifier {
        case "caffeine":
            return CLKComplicationStaticWidgetMigrationConfiguration(
                kind: "CaffeineComplication",
                extensionBundleIdentifier: "com.example.app.Complications")
        default:
            return nil
        }
    }
}
```

- The moment a WidgetKit extension serves complications, the system **disables ClockKit** and stops calling your data source (except the migrator).
- Family mapping: `graphicRectangular` → `accessoryRectangular`; `graphicCorner` → `accessoryCorner`; `graphicCircular`/`graphicBezel`/`graphicExtraLarge` → `accessoryCircular`; `utilitarianSmallFlat`/`utilitarianLarge` → `accessoryInline`. (12 families collapse to 4.)
- Static config → `CLKComplicationStaticWidgetMigrationConfiguration`; intent config → `CLKComplicationIntentWidgetMigrationConfiguration` (legacy) or `CLKComplicationAppIntentWidgetMigrationConfiguration` (App Intents). See Apple TN3157.

## Previews: before → after

```swift
// Before - PreviewProvider + WidgetPreviewContext (iOS 14):
struct Widget_Previews: PreviewProvider {
    static var previews: some View {
        MyWidgetView(entry: .sample)
            .previewContext(WidgetPreviewContext(family: .systemSmall))
    }
}

// After - the #Preview macro (Xcode 15 / iOS 17):
#Preview(as: .systemSmall) { MyWidget() } timeline: { Entry.sample }
```

See `previews-and-testing.md` for all the macro overloads.

## Genuinely deprecated vs legacy-but-supported

**Stop using for new code (effectively deprecated):**
- **ClockKit** as the complication engine - migration-only.
- **SiriKit `.intentdefinition` custom intents** for widget configuration - replaced by App Intents in iOS 17; exists only for sub-iOS-17 deployment and to keep old donations resolving.
- **`INRelevantShortcut` / `INInteraction` donations** for Smart Stack - replaced by App Intents relevance in iOS 18.

**Legacy but fully supported (keep ONLY for sub-iOS-17 targets):**
- `IntentConfiguration`, `IntentTimelineProvider`, `IntentRecommendation` - not deprecated, identical behavior, just the `INIntent`-typed variant. Required below iOS 17.
- `WidgetPreviewContext` / `previewContext(_:)` - superseded by `#Preview`, not formally deprecated.

**Net rule:** with a deployment floor of iOS 17 / watchOS 10 / macOS 14, use **only** `AppIntentConfiguration` + `AppIntentTimelineProvider` + `WidgetConfigurationIntent` + `#Preview`. Below that, gate the legacy types behind `#available`.

## Checklist

- New configurable widget → App Intents (`WidgetConfigurationIntent` + `AppIntentConfiguration` + `AppIntentTimelineProvider`). Never a fresh `.intentdefinition`.
- Migrating an existing one → add `CustomIntentMigratedAppIntent` + matching `intentClassName`; keep parameter names/types identical.
- Watch complications → WidgetKit accessory widgets + `CLKComplicationWidgetMigrator`; ClockKit is migration-only.
- Previews → `#Preview(as:)`, not `PreviewProvider`.
- Smart Stack rotation → `TimelineEntryRelevance` / App Intents relevance, not `INRelevantShortcut` donations.
