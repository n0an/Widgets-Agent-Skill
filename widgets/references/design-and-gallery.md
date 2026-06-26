# Designing widgets and presenting them in the gallery

WidgetKit's design rules are unusually specific, and getting them wrong is what makes a technically-correct widget feel off. Apple's framing across the "Design great widgets" / "Principles of great widgets" sessions: a great widget is **glanceable, relevant, and personal** - and it is **not a mini-app**. You're projecting a piece of your app's content onto the Home Screen, not shrinking your UI into a tile.

## The three qualities

- **Glanceable** - readable in a moment. The Home Screen is hit ~90 times a day for seconds at a time; the worst thing a user can see is a loading spinner or a wall of controls. Show content, not chrome.
- **Relevant** - shows what matters right now (the next event, today's total, the current status), and surfaces in the Smart Stack at the right moment.
- **Personal** - the user chose to place it; let them configure it to their context (which city, which account) rather than guessing.

A widget is **not** a mini-app: no scrolling, no dense control panels, no "open this, then tap that". Interactivity is one or two purposeful buttons/toggles (see `interactive-widgets.md`), not a UI.

## Sizes: design each one, don't scale

Support the families that genuinely suit the content, and design each distinctly:

- **systemSmall** - one focused piece of content; a **single tap target** (the whole widget) that deep-links to exactly what's shown. Aim for **~4 pieces of information max**.
- **systemMedium / systemLarge** - room for multiple items and multiple tap targets via `Link`, each deep-linking to its own content.
- **systemExtraLarge** (iPad) - a dashboard; don't just stretch the large layout.
- **accessory families** - monochrome-friendly, gauge/glyph-first (see `lock-screen-and-watch.md` and `rendering-modes-and-tinting.md`).

Never scale a small widget up to fill a larger family - the larger sizes should show *more*, not *bigger*. When the primary content runs out (no more events today), **show what's next** rather than an empty widget.

### Tap styles

Three content-presentation styles to keep layouts consistent:
- **fill** - a single piece of content fills the widget (all small widgets).
- **cell** - content sits in its own contained shape.
- **content** - uncontained content directly on the background.

## Layout and typography

- **Layout margins:** default to **~16 pt**; use tighter **~11 pt** margins for graphical/inset shapes (circles, platters). Don't hardcode corner radii - use `ContainerRelativeShape` so nested shapes stay concentric with the widget's corners. On modern OSes, read `@Environment(\.widgetContentMargins)` and apply them yourself when you've disabled default margins.
- **Type:** prefer the system fonts (SF Pro / SF Mono / SF Pro Rounded) with standard text styles, sizes, and padding. Custom fonts only when on-brand and still legible at a glance.
- **Dynamic Type & Dark Mode:** support both automatically. Use asset-catalog colors that adapt; you don't have to invert your palette (Music and Stocks keep their dark look in light mode - that's fine).
- **Use `ViewThatFits`** to gracefully pick a denser or sparser layout when space is tight, especially for accessory/inline content.

## The don'ts (these are explicit Apple rules)

- **No app icon** inside the widget, and **no app name** - the Home Screen already labels it. No word marks.
- A **logo only for aggregators** (a widget showing content from many sources), pinned **top-right**, small.
- **No instructional / "talking to the user" text** - communicate graphically. No "Tap to refresh", no empty-state paragraphs.
- **No "Last updated" / "Last checked" timestamps** for chronological data - it reads as stale and apologetic.
- **No raw spinners or "Loading…"** - use a content-shaped placeholder instead (below).

## The gallery: configurationDisplayName, description, snapshot

When the user browses to add a widget, three things represent yours:

```swift
.configurationDisplayName("Caffeine")          // the title in the gallery
.description("Track today's caffeine intake.")  // one-line subtitle
```

- The **snapshot** the gallery shows is your **real widget rendered live** (via `getSnapshot` / `snapshot(for:in:)`), not a screenshot. Return **realistic, populated, inviting** data - never an empty or loading state, or users won't add it. This is the single highest-leverage thing for adoption.
- List widgets in the `WidgetBundle` with your **hero widget first** - bundle order is gallery order.
- You **cannot** publish or retract a widget dynamically after install; the set is fixed by the bundle.

## The placeholder: shape, not spinner

The placeholder is the first-paint and redacted skeleton. It should mirror the real layout with content **blocked out** - text as rounded rectangles, images as filled shapes - so there's no layout or color shift when real data arrives:

```swift
func placeholder(in context: Context) -> Entry { .sample }   // a realistic entry...
// ...rendered redacted by the system, or forced:
EntryView(entry: .sample).redacted(reason: .placeholder)
```

Match the placeholder's structure to the loaded state exactly - same number of lines, same image sizes - so the transition is seamless.

## Personalization: prefer many simple widgets

- Offer **multiple single-purpose widgets** the user can place and configure independently (two Weather widgets for two cities) rather than one complex combined widget with a busy editor.
- Keep configuration to **one or two parameters** with sensible defaults; never force configuration before the widget shows anything (see `configuration-and-intents.md`).

## Smart Stack etiquette

- Drive rotation with **timely, obviously-valuable** information - surface on a thunderstorm, not a one-degree temperature change.
- Don't camp the top slot; over-surfacing trains users to remove your widget.
- Use `TimelineEntryRelevance` / App Intents relevance to say *when* you matter (see `relevance-and-smart-stacks.md`).

## Removable backgrounds and many placements

One widget now appears on the Home Screen, the Lock Screen, StandBy, the Mac desktop, and the watch Smart Stack. Design it so the system can **remove its background** in the contexts that need it:

- Put the background inside `.containerBackground(for: .widget) { ... }` so it can be removed.
- Read `@Environment(\.showsWidgetContainerBackground)` and restructure when the background is gone (push content edge-to-edge, enlarge the key element) - the Weather widget does this between Home Screen and Lock Screen.
- Design for `.accented` / `.vibrant` rendering, not just full color (see `rendering-modes-and-tinting.md`).

## Checklist

- Glanceable, relevant, personal; content over chrome; not a mini-app.
- Design each family distinctly (small = 1 tap target, ~4 facts); show "what's next" instead of empty.
- ~16 pt margins (11 pt for graphical), system fonts, Dynamic Type + Dark Mode, `ContainerRelativeShape`.
- No app icon/name/word marks, no instructional text, no "last updated", no spinners.
- Gallery snapshot = realistic populated data; hero widget first in the bundle.
- Placeholder mirrors the real layout (blocked-out shapes), not a spinner.
- Many simple configurable widgets > one complex one; Smart Stack only when genuinely relevant.
