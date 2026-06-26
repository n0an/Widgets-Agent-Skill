# Configuration: static, app-intent, and legacy

The configuration ties the `kind`, the provider, and the view together, and decides whether the user can customize the widget.

## `StaticConfiguration` - no user configuration

The widget behaves the same for everyone (or varies only by family / data). Simplest case:

```swift
struct CaffeineWidget: Widget {
    let kind = "CaffeineWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: Provider()) { entry in
            CaffeineWidgetView(entry: entry)
                .containerBackground(for: .widget) { BackgroundGradient() }
        }
        .configurationDisplayName("Caffeine")
        .description("Track today's caffeine.")
        .supportedFamilies([.systemSmall, .systemMedium, .accessoryCircular])
    }
}
```

Pairs with a `TimelineProvider` (see `timeline-provider.md`).

## `AppIntentConfiguration` - user-configurable (iOS 17+)

When the user should pick something (an account, a city, a repo, a color), back the widget with a `WidgetConfigurationIntent`. The system renders an editing UI from the intent's `@Parameter`s when the user long-presses ▸ Edit Widget.

```swift
import WidgetKit
import AppIntents

struct ConfigurationIntent: WidgetConfigurationIntent {
    static let title: LocalizedStringResource = "Configuration"
    static let description = IntentDescription("Choose what to show.")

    @Parameter(title: "Accent Color", default: .indigo)
    var color: WidgetTint            // an AppEnum

    @Parameter(title: "Account")
    var account: AccountEntity?      // an AppEntity, resolved by its EntityQuery
}

struct ColorWidget: Widget {
    let kind = "ColorWidget"

    var body: some WidgetConfiguration {
        AppIntentConfiguration(kind: kind, intent: ConfigurationIntent.self, provider: Provider()) { entry in
            ColorWidgetView(entry: entry)
                .containerBackground(for: .widget) { entry.configuration.color.gradient }
        }
        .configurationDisplayName("Color")
        .description("Pick an accent and an account.")
        .supportedFamilies([.systemSmall])
    }
}
```

Key facts:

- `WidgetConfigurationIntent` lives in the **AppIntents** module and requires **no** `perform()` - it's a pure configuration carrier. (It's `AppIntent` + an empty conformance.)
- The provider is an `AppIntentTimelineProvider`; its `snapshot`/`timeline` receive the resolved intent as the **first** argument, and the entry carries it so the view can read `entry.configuration`.
- Always give parameters a **default** so the widget shows something sensible before the user configures it. Never force configuration up front.
- Keep it to **one or two parameters**. Users can add several instances of the same widget with different configs - that's the intended way to "show two cities", not a multi-select monster.

### Parameter option sources

- **`AppEnum`** for a fixed set (colors, modes):

  ```swift
  enum WidgetTint: String, AppEnum {
      case indigo, mint, orange
      static let typeDisplayRepresentation: TypeDisplayRepresentation = "Tint"
      static let caseDisplayRepresentations: [WidgetTint: DisplayRepresentation] =
          [.indigo: "Indigo", .mint: "Mint", .orange: "Orange"]
  }
  ```

- **`AppEntity` + `EntityQuery`** for dynamic data (accounts, repos, documents) the user picks from. The query loads candidates; the parameter resolves the chosen entity. (See the `app-intents` skill for entity/query design - the same types power widget configuration.)
- **`DynamicOptionsProvider`** when you just need a dynamic list of simple values without a full entity:

  ```swift
  struct RepoOptionsProvider: DynamicOptionsProvider {
      func results() async throws -> [String] {
          UserDefaults(suiteName: "group.com.you.app")?
              .stringArray(forKey: "repos") ?? []
      }
      func defaultResult() async -> String? { "owner/repo" }
  }

  @Parameter(title: "Repo", optionsProvider: RepoOptionsProvider())
  var repo: String?
  ```

Read configuration data from the **App Group** container inside the provider, since the option list and the widget live in the extension.

## Reacting to configuration changes from the app

When the app changes data that feeds a widget's configuration choices (the user added a new account that should appear in the picker), invalidate caches and reload:

```swift
WidgetCenter.shared.reloadTimelines(ofKind: "ColorWidget")
```

If a configuration parameter's *option list* changed (not the displayed data), and you back it with App Intents entities/phrases, also call your `AppShortcutsProvider.updateAppShortcutParameters()` where relevant (App Intents concern).

## Preconfigured widgets where there's no editor: `recommendations()`

On surfaces with no configuration UI (watchOS Smart Stack and complications, StandBy), the user can't edit the intent, so you supply a preconfigured list from the provider's `recommendations()`:

```swift
func recommendations() -> [AppIntentRecommendation<ConfigurationIntent>] {
    Account.all.map { AppIntentRecommendation(intent: ConfigurationIntent(account: $0), description: $0.name) }
}
```

On **watchOS this is effectively required** for a configurable widget. Return `[]` where a real editor exists. Call `WidgetCenter.shared.invalidateConfigurationRecommendations()` when the recommended set changes. See `relevance-and-smart-stacks.md`.

## Legacy SiriKit configuration and migration

Before iOS 17, configurable widgets used `IntentConfiguration` with a **SiriKit** custom intent (`INIntent`) defined in an `.intentdefinition` file:

```swift
IntentConfiguration(kind: kind, intent: SelectRepoIntent.self, provider: Provider()) { entry in
    RepoWidgetView(entry: entry)
}
```

To modernize to App Intents **without breaking users who already configured the widget**, conform the new App Intent to `WidgetConfigurationIntent` **and** `CustomIntentMigratedAppIntent`, keeping the old intent class name so existing configurations map over:

```swift
struct SelectRepo: AppIntent, WidgetConfigurationIntent, CustomIntentMigratedAppIntent {
    static let intentClassName = "SelectRepoIntent"   // the OLD SiriKit intent class name
    static let title: LocalizedStringResource = "Select Repo"

    @Parameter(title: "Repo", optionsProvider: RepoOptionsProvider())
    var repo: String?
}
```

`CustomIntentMigratedAppIntent` is **only** needed when migrating an existing pre-iOS-17 configurable widget. A brand-new configurable widget just adopts `WidgetConfigurationIntent`. Keep `IntentConfiguration` if you still deploy below iOS 17.

## Checklist

- No customization → `StaticConfiguration`. Customization → `AppIntentConfiguration` + `WidgetConfigurationIntent` (iOS 17+).
- `WidgetConfigurationIntent` needs **no** `perform()` and lives in AppIntents.
- Provider is `AppIntentTimelineProvider`; the intent is the **first** arg; the entry carries it.
- Every `@Parameter` has a **default**; keep to 1-2 parameters; multiple instances > multi-select.
- Read option lists / data from the **App Group**.
- Migrating a pre-17 widget → `CustomIntentMigratedAppIntent` + the original `intentClassName`.
