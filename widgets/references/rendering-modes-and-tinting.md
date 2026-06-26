# Rendering modes, tinting, and the accent group

A widget must look right in more than full color. On the Lock Screen, in StandBy, on a tinted/clear Home Screen, and in watch complications, the system **re-renders your view in a tint mode** - flattening color and applying the user's accent. Code that only considers full color produces white blobs, invisible text, and crushed icons. This is the area where generated widget code most often looks broken.

## The three rendering modes

Read the current mode from the environment:

```swift
@Environment(\.widgetRenderingMode) private var renderingMode
```

`WidgetRenderingMode` is a **struct with type properties** (not an enum). Its values:

- **`.fullColor`** - Home Screen default (and StandBy night mode on some devices). Your colors render as authored.
- **`.accented`** - Lock Screen, tinted Home Screen, watch complications. The system splits your view into **two groups** and recolors each with a system/user tint. This is the mode that needs the most care.
- **`.vibrant`** - the iOS Lock Screen, StandBy Night mode, and the iPad Lock Screen. The system desaturates your content and maps it into an adaptive material over the wallpaper. **Light/dark maps to legibility:** darker colors (toward black) read as *less prominent* (low brightness + heavy blur), lighter colors as more prominent. So in vibrant mode use brightness - not hue - to express hierarchy, avoid translucent colors (they wash out), and drop background platters that hurt contrast.

A short history worth knowing: `.accented` and `.widgetAccentable()` started as **Lock Screen / watch-only** in iOS 16. iOS 18 introduced the **tinted (and dark) Home Screen**, which renders ordinary Home Screen widgets in `.accented` too - so accent-group design is no longer just a Lock Screen concern, it's every widget's concern. iOS 26's Liquid Glass redesign reworked the accented material further (see Apple's "Optimizing your widget for accented rendering mode and Liquid Glass").

Branch on it for materially different layouts:

```swift
private var isFullColor: Bool { renderingMode == .fullColor }

@ViewBuilder var icon: some View {
    if isFullColor {
        Image("Logo").resizable().scaledToFit()         // rich, multicolor
    } else {
        Image("Logo").resizable().scaledToFit()
            .luminanceToAlpha()                          // clean glyph for tinting
    }
}
```

## The accent group: `widgetAccentable`

In `.accented` mode the system divides your widget into exactly **two tint groups**:

- the **default group** (the rest), and
- the **accent group** - everything you mark with `.widgetAccentable()`.

The system colors the two groups with two related tints (e.g. a brighter accent and a dimmer default), keyed off the **alpha channel** of your views - it treats them like template images. So:

```swift
VStack {
    Image(systemName: "drop.fill").widgetAccentable()   // accent group → brighter tint
    Text("Hydration").widgetAccentable()                // accent group
    Text("2.1 L")                                       // default group → dimmer tint
}
```

Put your primary, must-read content in the accent group. Wrap conditional accenting in a tiny modifier so the call sites stay clean:

```swift
struct AccentWhen: ViewModifier {
    let on: Bool
    func body(content: Content) -> some View { on ? AnyView(content.widgetAccentable()) : AnyView(content) }
}
extension View { func accentable(_ on: Bool) -> some View { modifier(AccentWhen(on: on)) } }
```

Because tinting works off **alpha**, opaque shapes become solid tinted blocks and detail inside a full-color bitmap is lost - which is exactly what `widgetAccentedRenderingMode` and `luminanceToAlpha` solve.

## `widgetAccentedRenderingMode` - controlling how images flatten (iOS 18+)

For an **`Image`**, this modifier (iOS 18+, re-spotlighted at WWDC 2025 when the tinted/clear Home Screen made it ubiquitous) chooses how the image behaves in accented mode. `WidgetAccentedRenderingMode` is a **struct** with these type properties:

- **`.accented`** - include the image in the accent group (tinted to the accent color).
- **`.desaturated`** - map the image's **luminance to alpha** and recolor with the **default** group color. Bright pixels stay, dark pixels fade.
- **`.accentedDesaturated`** - luminance→alpha, recolored with the **accent** group color (desaturate *and* accent).
- **`.fullColor`** - render the image **unmodified**, keeping its real colors even in accented contexts. (Only applies on iOS; ignored on watchOS.) Use this for a photo, app icon, or brand mark that must stay true-to-color so it doesn't become a white rectangle in clear/tinted mode.

```swift
// Keep a product/cover image in real color even on a tinted Home Screen:
Image("AlbumArt")
    .resizable().scaledToFit()
    .widgetAccentedRenderingMode(.fullColor)

// Let a glyph desaturate into the default tint group:
Image("Latte")
    .widgetAccentedRenderingMode(.desaturated)
```

Don't combine `.widgetAccentable()` on a parent group with a conflicting `widgetAccentedRenderingMode` on a child image - the two can fight; pick one intent per image.

## The luminance-to-alpha trick (multicolor glyph in tint mode)

The lesser-known move that makes a **multicolor logo or icon** survive tinting as a recognizable shape instead of a flat silhouette:

```swift
Image("AppIconFrameless")
    .resizable().scaledToFit()
    .luminanceToAlpha()
```

`luminanceToAlpha()` is a SwiftUI filter that converts each pixel's **brightness into its alpha**: bright pixels become opaque, dark pixels become transparent, midtones become semi-transparent. The result is a clean monochrome mask of your artwork's internal structure. When the system then tints that mask in `.accented` mode, you get a crisp glyph that keeps the icon's detail - not the solid blob you'd get from tinting the opaque original.

### When to use this vs `widgetAccentedRenderingMode`

These two are the same effect by two routes, split by OS version:

- On **iOS 18+**, prefer the declarative modifier: `Image(...).widgetAccentedRenderingMode(.desaturated)` (default-group tint) or `.accentedDesaturated` (accent-group tint). It maps luminance to alpha and recolors *for you* - no manual filter.
- On **iOS 17 and below**, `widgetAccentedRenderingMode` **does not exist** (it's iOS 18+), so the manual `.luminanceToAlpha()` filter is the equivalent - it's how you got a desaturated, tint-friendly glyph before the modifier shipped.

`.luminanceToAlpha()` still works on iOS 18+, so it's also the right tool when you want **hands-on control** - to combine the mask with a gradient lift, a custom blend mode, or your own `widgetRenderingMode` branch (as in the recipe below). Gate by availability when you want the proper modifier where it exists and the manual fallback below it:

```swift
@ViewBuilder
var glyph: some View {
    let base = Image("AppIconFrameless").resizable().scaledToFit()
    if #available(iOS 18, *) {
        base.widgetAccentedRenderingMode(.desaturated)   // proper modifier (iOS 18+)
    } else {
        base.luminanceToAlpha()                          // manual equivalent (iOS 17 and below)
    }
}
```

The full recipe (from a shipping complication/icon widget):

```swift
@ViewBuilder
private var iconImage: some View {
    if renderingMode == .fullColor {
        Image("AppIconFrameless").resizable().scaledToFit()        // rich color on Home Screen
    } else {
        Image("AppIconFrameless").resizable().scaledToFit()
            .luminanceToAlpha()                                    // → monochrome mask for tint
    }
}

// In the accessory/complication view, pair it with a soft lift + accent group:
content
    .luminanceToAlpha()
    .background {
        LinearGradient(colors: [.white.opacity(0.4), .clear],
                       startPoint: UnitPoint(x: 0.5, y: 1.5), endPoint: .top)
    }
    .widgetAccentable()
    .containerBackground(for: .widget) { }     // transparent so the wallpaper/face shows through
```

Why each piece:
- `luminanceToAlpha()` turns the multicolor art into a tintable mask.
- the faint gradient gives the glyph a soft base so it reads on busy wallpapers.
- `widgetAccentable()` puts it in the accent group so it takes the brighter tint.
- the **empty** `containerBackground` keeps the background transparent (see `lock-screen-and-watch.md`).

This is the difference between an icon that looks intentional in tinted mode and one that vanishes or turns into a white circle.

## Designing color-mode-aware backgrounds

In accented/tinted contexts a heavy colored background is wrong - the system wants to tint content over the wallpaper. Make backgrounds conditional:

```swift
private var background: some View {
    Group {
        if renderingMode == .fullColor {
            LinearGradient(colors: [.indigo, .purple], startPoint: .top, endPoint: .bottom)
        } else {
            Color.clear                    // let the system tint/blur take over
        }
    }
}
// ...
.containerBackground(for: .widget) { background }
```

Also respect `@Environment(\.colorScheme)` for light/dark full-color variants, and (Live Activities) `@Environment(\.showsWidgetContainerBackground)` which is `false` in StandBy so you can drop a Lock-Screen-only background.

## Checklist

- `WidgetRenderingMode` and `WidgetAccentedRenderingMode` are **structs**, not enums. Branch on `\.widgetRenderingMode`.
- Put primary content in the **accent group** with `.widgetAccentable()`; tinting works off **alpha**.
- `widgetAccentedRenderingMode(.fullColor)` (iOS 18+) keeps a photo/brand image true-to-color so it isn't a white block; `.desaturated` / `.accentedDesaturated` flatten via luminance.
- Use `.luminanceToAlpha()` to turn a multicolor glyph into a clean tintable mask; pair with a soft gradient + `widgetAccentable()` + empty `containerBackground`.
- Make heavy backgrounds **full-color only**; let `.accented`/`.vibrant` fall back to clear.
