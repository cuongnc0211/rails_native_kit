# Bridge Components — Android

The JS and HTML sides are **identical** to iOS (`bridge-components-ios.md`). Only the Kotlin
implementation differs. Content is docs-sourced (hotwire-native-android 1.2.8).

## What's different from iOS

| Aspect | iOS (Swift) | Android (Kotlin) |
|---|---|---|
| Base class | `BridgeComponent` | `BridgeComponent<NavDestination>` |
| Native button | `UIBarButtonItem` in code | `MenuItem` via `MenuProvider` XML |
| Registration | `Hotwire.registerBridgeComponents([Class.self])` | `BridgeComponentFactory("name", ::Class)` |
| Reply to JS | `reply(to: "connect")` | `reply("connect")` |
| Lines of code | ~40 | ~100 (Android menu verbosity) |

Since Android 1.2.5: bridge components can receive destination Fragment view lifecycle events —
useful for components that need to show/hide with the screen lifecycle.

## Minimal working example

```kotlin
// BridgeComponents/ButtonComponent.kt
class ButtonComponent(
    name: String,
    private val delegate: BridgeDelegate<NavDestination>
) : BridgeComponent<NavDestination>(name, delegate), MenuProvider {

    private var menuItem: MenuItem? = null

    override fun onReceive(message: Message) {
        if (message.event == "connect") {
            requireActivity().addMenuProvider(this, viewLifecycleOwner)
        }
    }

    override fun onCreateMenu(menu: Menu, menuInflater: MenuInflater) {
        menuInflater.inflate(R.menu.bridge_button, menu)
        menuItem = menu.findItem(R.id.bridge_button)
        // set title from message if needed: menuItem?.title = lastTitle
    }

    override fun onMenuItemSelected(item: MenuItem): Boolean {
        if (item.itemId == R.id.bridge_button) {
            reply("connect")   // triggers JS callback
            return true
        }
        return false
    }
}
```

Menu XML resource — one file per component:

```xml
<!-- res/menu/bridge_button.xml -->
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:app="http://schemas.android.com/apk/res-auto">
    <item
        android:id="@+id/bridge_button"
        android:title="Action"
        app:showAsAction="always" />
</menu>
```

## Registration

```kotlin
// DemoApplication.kt — in onCreate()
Hotwire.registerBridgeComponents(
    BridgeComponentFactory("button", ::ButtonComponent)
)
```

`"button"` must match `static component = "button"` in the Stimulus JS controller.

## Payload access

Older SDK preview docs showed `message.metadata?.get("title")` but the Android 1.2.x SDK
may use a different mechanism. Before writing production code that reads payload data, confirm
the actual `Message` class API from the resolved Kotlin source:

```bash
# In Android Studio terminal or via SDK sources
grep -rn "class Message\|val metadata\|val jsonData" ~/.gradle/caches/
```

If `jsonData: String` exists (mirroring the iOS API), decode it with Kotlinx Serialization or Gson
rather than the `metadata` map accessor.

## HTML side (shared with iOS)

```erb
<button
  data-controller="bridge--button"
  data-bridge-title="Action">
  Action  <%# fallback for web / older app versions %>
</button>
```

The `data-controller` Stimulus identifier and the `static component` name must match the string
passed to `BridgeComponentFactory`. They do **not** need a `bridge--` folder prefix.
