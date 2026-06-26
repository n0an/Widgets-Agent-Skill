# Live Activities and the Dynamic Island

A Live Activity (ActivityKit, iOS 16.1+) shows the live state of an **ongoing, time-bound event** - a delivery, a ride, a game, a workout, a timer - on the Lock Screen, in the Dynamic Island, in StandBy, and (from iOS 18) on the Apple Watch Smart Stack, CarPlay, and the macOS menu bar. It is **not** a fast-reloading widget; reach for it when there's a real event with a beginning and end the user wants to watch.

> Framework split: `ActivityAttributes`, `Activity.request/update/end`, and the push tokens are in **ActivityKit** (the app). `ActivityConfiguration`, `DynamicIsland`, and `ActivityFamily` are in **WidgetKit + SwiftUI** (the widget extension).

## The data model: attributes vs content state

Split the data in two. This split is what keeps updates cheap and pushable:

```swift
import ActivityKit

struct DeliveryAttributes: ActivityAttributes {
    // Immutable for the life of the activity:
    let orderNumber: String
    let restaurant: String

    // The only-updatable half. Keep the whole ContentState under 4 KB.
    public struct ContentState: Codable, Hashable {
        var status: DeliveryStatus
        var etaDate: Date
        var courierName: String?
    }
}
```

- `ActivityAttributes` = static identity, set once at start.
- nested `ContentState: Codable, Hashable` = everything that changes; this is what you `update(_:)` and what a push payload carries.

## The widget side: `ActivityConfiguration` + `DynamicIsland`

In the widget extension, declare the UI for both the Lock Screen banner and the three Dynamic Island presentations (compact, minimal, expanded). The API forces you to implement all of them:

```swift
import WidgetKit
import SwiftUI

struct DeliveryLiveActivity: Widget {
    var body: some WidgetConfiguration {
        ActivityConfiguration(for: DeliveryAttributes.self) { context in
            // Lock Screen / banner / StandBy
            DeliveryLockScreenView(context: context)
                .activityBackgroundTint(.black.opacity(0.4))            // background tint
                .activitySystemActionForegroundColor(.white)           // dismiss-button color
        } dynamicIsland: { context in
            DynamicIsland {
                DynamicIslandExpandedRegion(.leading)  { CourierView(context: context) }
                DynamicIslandExpandedRegion(.trailing) { EtaView(context: context) }
                DynamicIslandExpandedRegion(.center)   { StatusView(context: context) }
                DynamicIslandExpandedRegion(.bottom)   { ProgressBar(context: context) }
            } compactLeading: {
                Image(systemName: "bag.fill")
            } compactTrailing: {
                Text(context.state.etaDate, style: .timer).monospacedDigit()
            } minimal: {
                Image(systemName: "bag.fill")
            }
            .keylineTint(.orange)                                       // Dynamic Island border tint
            .widgetURL(URL(string: "delivery://order/\(context.attributes.orderNumber)"))
        }
        .supplementalActivityFamilies([.small])   // Apple Watch Smart Stack + CarPlay (iOS 18+)
    }
}
```

- `context` is an `ActivityViewContext<DeliveryAttributes>`: `context.attributes` (static), `context.state` (the current `ContentState`), `context.activityID`, and `context.isStale` (render a stale indicator when `true`).
- The four expanded regions are `.leading`, `.trailing`, `.center`, `.bottom` (with an optional `priority:`); plus `compactLeading`, `compactTrailing`, and `minimal`.
- The `minimal` view is shown when two activities share the Dynamic Island - it must still convey information, not just a logo. The compact view should stay as narrow as possible and be *informational*, not just a launch button.
- Use self-updating `Text(_:style:)` for countdowns/timers so you don't push every second.

## Starting, updating, ending (in the app)

```swift
import ActivityKit

// Start (from the foreground app, with activities enabled):
guard ActivityAuthorizationInfo().areActivitiesEnabled else { return }
let attributes = DeliveryAttributes(orderNumber: "1234", restaurant: "Café")
let initial = DeliveryAttributes.ContentState(status: .preparing, etaDate: .now + 1800, courierName: nil)
let activity = try Activity.request(
    attributes: attributes,
    content: ActivityContent(state: initial, staleDate: .now + 3600, relevanceScore: 0),
    pushType: nil                 // or .token (per device) / .channel(id) (broadcast)
)

// Update:
await activity.update(
    ActivityContent(state: .init(status: .onTheWay, etaDate: .now + 600, courierName: "Sam"),
                    staleDate: .now + 1200)
)

// Update with an alert (lights up the Dynamic Island; title/body are shown only on Apple Watch):
await activity.update(updatedContent, alertConfiguration:
    AlertConfiguration(title: "Order on the way", body: "Sam is heading over", sound: .default))

// End:
await activity.end(ActivityContent(state: finalState, staleDate: nil), dismissalPolicy: .after(.now + 60))
```

- `staleDate` and `relevanceScore` are set on **`ActivityContent.init`**, not on the configuration. Past `staleDate`, the system marks the content stale (`context.isStale == true`).
- `dismissalPolicy`: `.default` (keeps it on the Lock Screen briefly), `.immediate`, or `.after(date)`.
- `relevanceScore` orders multiple concurrent activities in the Dynamic Island.
- **API history:** the iOS 16.1 form was `request(attributes:contentState:pushType:)` with a raw `ContentState`; iOS 16.2 wrapped it in **`ActivityContent`** (adding `staleDate` + `relevanceScore`). Target the `ActivityContent` form.

### Observing state

```swift
let now = activity.activityState                       // .active / .ended / .dismissed / .stale
for await state in activity.activityStateUpdates {     // async sequence
    if state == .dismissed { cleanUp() }
}
```

## Push updates

Three flavors:

1. **Token push** - per-activity, per-device. After `request(..., pushType: .token)`, read the token from the **async sequence** (never synchronously - it's usually `nil` right after creation and rotates over the activity's life):

   ```swift
   Task {
       for await tokenData in activity.pushTokenUpdates {
           let token = tokenData.map { String(format: "%02x", $0) }.joined()
           await sendToServer(token, for: activity.id)   // invalidate the old token
       }
   }
   ```

2. **Channel / broadcast push** (iOS 18+) - `pushType: .channel(channelID)`; one APNs push fans out to **every** device subscribed to that base64 channel ID (a live game everyone follows), with no per-token storage. Manage channels via the APNs channel APIs / Push Notifications Console; channel lifetime is independent of any activity. Use channels for mass audiences, tokens for per-user.

3. **Push-to-start** (iOS 17.2+) - start an activity remotely without the app running. Read the **type-level** token and send an APNs payload with `event: "start"`:

   ```swift
   for await token in Activity<DeliveryAttributes>.pushToStartTokenUpdates {
       sendToServer(Data(token))            // Activity.pushToStartToken / pushToStartTokenUpdates
   }
   ```

   Push-to-start is **not** a `PushType` case - it's a separate token mechanism. `PushType` itself only has `.token` and `.channel(_:)`.

### The APNs wire format (token/channel updates)

- Connection: token-based (p8) APNs; add the **Push Notifications** capability to the **app** target (broadcast also needs the broadcast capability enabled in the portal).
- Headers: `apns-push-type: liveactivity`, `apns-topic: <bundleID>.push-type.liveactivity`, `apns-priority: 5` (low/opportunistic, unlimited) or `10` (high/immediate, budgeted).
- `aps` payload keys:
  - `timestamp` (epoch seconds - the system renders only the latest by timestamp),
  - `event`: `"update"` | `"start"` | `"end"`,
  - `content-state`: a JSON object of your `ContentState`. **Encode with a plain `JSONEncoder` (no custom key/date strategies)** - the device decodes with a default `JSONDecoder`; mismatched strategies fail silently.
  - optional `stale-date`, `dismissal-date` (with `event: end`), `relevance-score`,
  - optional `alert`: `{ title, body, sound }` (title/body may be `{ loc-key, loc-args }`).

## Entitlements and Info.plist

- The **app's** Info.plist must set `NSSupportsLiveActivities` = `YES` (without it ActivityKit is unavailable).
- For high-frequency push updates, also set `NSSupportsLiveActivitiesFrequentUpdates` = `YES`; check `ActivityAuthorizationInfo().frequentPushesEnabled` at runtime.

## StandBy, the small family, and the landscape Dynamic Island

- **StandBy** scales the Lock Screen view up ~200%. `@Environment(\.showsWidgetContainerBackground)` is `true` on the Lock Screen and `false` in StandBy - gate a Lock-Screen-only background with it, and use `.activityBackgroundTint(_:)` for an edge-to-edge fill.
- **`supplementalActivityFamilies([.small])`** (iOS 18+) opts the activity into the small family used by the Apple Watch Smart Stack and CarPlay. Branch on `@Environment(\.activityFamily)` (`.small` / `.medium`; there is no `.large`) to provide a compact view. If you don't, the watch shows your iOS Lock Screen content view (often too big). The iOS-26 news is that activities now surface on CarPlay and the macOS menu bar automatically - the API itself is iOS 18.
- **Apple Watch Always On:** the system forces dark scheme + reduced luminance on wrist-down. Read `@Environment(\.isLuminanceReduced)` and dim bright elements.
- **27 releases:** the Dynamic Island's compact and minimal views now render in **landscape too**, where width is constrained. Read `@Environment(\.isDynamicIslandLimitedInWidth)` and fall back to a narrower rendering (an icon instead of a timer) when it's `true`.

```swift
struct CompactTrailingView: View {
    @Environment(\.isDynamicIslandLimitedInWidth) var limited
    var context: ActivityViewContext<DeliveryAttributes>
    var body: some View {
        if limited { Image(systemName: "clock") }
        else { Text(context.state.etaDate, style: .timer).monospacedDigit() }
    }
}
```

## Interactive Live Activities

Buttons inside a Live Activity use `LiveActivityIntent` (see `interactive-widgets.md`). A common pattern: an "End" button in the expanded island or banner that ends the activity from its own `perform()` and can auto-dismiss:

```swift
struct RateIntent: LiveActivityIntent {
    static let title: LocalizedStringResource = "Rate"
    @Parameter(title: "Positive") var isPositive: Bool
    func perform() async throws -> some IntentResult {
        await store.record(rating: isPositive ? .up : .down)
        return .result()
    }
}
Button(intent: RateIntent(isPositive: true)) { Label("Good", systemImage: "hand.thumbsup") }
    .buttonStyle(.plain)
```

`symbolEffect` is **not** supported in Live Activities - it silently no-ops; use static SF Symbols and `Text` timers. See `animations-and-transitions.md` for `contentTransition`/`numericText` between content-state updates.

## Design rules

- Lock Screen uses **14-pt margins** aligned with notifications; build a graphically rich layout - don't mimic a notification. Supply background and foreground colors so the system can derive the swipe-to-dismiss button.
- Vary the presentation's **height** between moments to draw attention to important changes; alert only for genuinely attention-worthy updates.
- Dynamic Island content must be **concentric** with the pill and hug the sensor area (no "forehead"); apps must not draw UI pointing at the floating island.

## Previewing

```swift
#Preview("Delivery", as: .content, using: DeliveryAttributes.preview) {
    DeliveryLiveActivity()
} contentStates: {
    DeliveryAttributes.ContentState.preparing
    DeliveryAttributes.ContentState.onTheWay
    DeliveryAttributes.ContentState.delivered
}
```

## Checklist

- Split immutable `ActivityAttributes` from the updatable nested `ContentState` (< 4 KB, `Codable, Hashable`).
- `ActivityConfiguration` + `DynamicIsland` (four expanded regions + compact/minimal) in the widget extension; `keylineTint`/`widgetURL` on the island.
- `staleDate`/`relevanceScore` on `ActivityContent`; `request`/`update`/`end` are async; observe `activityStateUpdates`.
- Push: `.token` per device (read `pushTokenUpdates`, never sync), `.channel` for broadcast, `pushToStartToken` (iOS 17.2+) to start remotely; APNs `apns-push-type: liveactivity`, plain `JSONEncoder` for `content-state`.
- `NSSupportsLiveActivities` (+ `…FrequentUpdates`) in the **app** Info.plist; Push capability for push.
- `supplementalActivityFamilies([.small])` + `\.activityFamily` for Watch/CarPlay; `\.isLuminanceReduced` for Always On; `\.isDynamicIslandLimitedInWidth` for landscape (27 releases); `symbolEffect` doesn't work.
