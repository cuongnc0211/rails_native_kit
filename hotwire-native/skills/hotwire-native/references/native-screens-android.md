# Native Screens — Android

When to build a fully native Jetpack Compose screen and how to wire it up via path config.
Content is docs-sourced (hotwire-native-android 1.2.8) — not yet verified in a real build.

See `native-screens-ios.md` for the decision framework (same threshold applies on Android).

## Step 1 — Add a path config rule

Android screens are routed via a `uri` property — a custom URI scheme that maps to a registered
Fragment class:

```json
{ "patterns": ["/dashboard"], "properties": { "uri": "hotwire://fragment/native/dashboard" } }
```

The wildcard catch-all (`".*"` → `hotwire://fragment/web`) must remain the **first** rule.

## Step 2 — Register the Fragment

In your `Application` subclass:

```kotlin
// DemoApplication.kt
class DemoApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        Hotwire.loadPathConfiguration(...)
        Hotwire.registerFragmentDestinations(
            "hotwire://fragment/native/dashboard" to DashboardFragment::class
        )
    }
}
```

Unlike iOS (where `handle(proposal:)` returns the class at runtime), Android requires upfront
registration. A URI that appears in path config without a registered Fragment causes a crash.

## Step 3 — Build the Fragment

```kotlin
// DashboardFragment.kt
class DashboardFragment : HotwireFragment() {
    private val viewModel: DashboardViewModel by viewModels()

    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?
    ) = ComposeView(requireContext()).apply {
        setContent {
            val state by viewModel.uiState.collectAsState()
            DashboardScreen(state)
        }
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        viewModel.load(location.url)   // location.url = URL from path config
    }
}
```

`location.url` is the URL that triggered navigation to this Fragment — use it to fetch data.

## Step 4 — Serve JSON from Rails

```ruby
# app/controllers/dashboards_controller.rb
def show
  respond_to do |format|
    format.html
    format.json { render json: { streak: current_user.streak } }
  end
end
```

```kotlin
// DashboardViewModel.kt
class DashboardViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<DashboardState>(DashboardState.Loading)
    val uiState: StateFlow<DashboardState> = _uiState

    fun load(url: String) {
        viewModelScope.launch {
            val result = runCatching { fetchJson("$url.json") }
            _uiState.value = result.fold(
                onSuccess  = { DashboardState.Success(it) },
                onFailure  = { DashboardState.Error(it.message) }
            )
        }
    }
}
```

## iOS comparison

| Concept | iOS | Android |
|---|---|---|
| Routing key | `view_controller: "dashboard"` | `uri: "hotwire://fragment/native/dashboard"` |
| Registration | None — class returned at runtime in `handle(proposal:)` | `registerFragmentDestinations()` in `Application` |
| Native UI layer | `UIHostingController(rootView:)` wrapping SwiftUI | `ComposeView { setContent {} }` hosting Composable |
| Data layer | `@StateObject` + `ObservableObject` | `ViewModel` + `StateFlow` |
| URL access | `proposal.url` passed at init | `location.url` on `HotwireFragment` |

## Key constraint

**Register before you navigate.** A `uri` in path config that has no matching registration crashes
the app. If you're deploying a path config update that adds a new native screen, ship the app binary
(with the Fragment registered) before or simultaneously with the path config change.
