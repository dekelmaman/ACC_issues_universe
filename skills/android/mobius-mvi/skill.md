# Mobius MVI Expert Skill

Deep reference for implementing features using the Mobius MVI architecture pattern with `MobiusViewModel`. Use this skill when creating new features, modifying existing Mobius state machines, writing Mobius tests, or debugging state/effect issues.

## When to Use
- Creating a new feature (scaffold the full Mobius structure)
- Adding new Actions, Effects, or Events to an existing feature
- Writing or modifying Updater, Processor, or StateMapper
- Building test infrastructure for a Mobius feature
- Debugging state transitions or effect handling
- Reviewing Mobius code for correctness

---

## The Mobius Loop

```
┌─────────────────────────────────────────────────────┐
│                     UI (Compose)                     │
│  Renders RenderData, dispatches Action.View          │
└──────────┬──────────────────────────────▲────────────┘
           │ Action.View                  │ RenderData
           ▼                              │
┌──────────────────────┐     ┌────────────────────────┐
│       Updater        │     │     StateMapper         │
│  Pure: (State,Action)│     │  Pure: State→RenderData │
│  → (State,Effects,   │     │  Phone/tablet aware     │
│     Events)          │     │  NO string resolution   │
└──────┬───────┬───────┘     └────────────▲────────────┘
       │       │                          │
Effects│  Events│                    State │
       │       │                          │
       ▼       ▼                          │
┌──────────────────────┐                  │
│     Processor        │──────────────────┘
│  Async: RxJava       │  Action.Internal
│  Repos, analytics,   │
│  network, prefs      │
└──────────────────────┘
```

### Data Flow
1. UI dispatches `Action.View` (user tapped something)
2. Updater receives `(currentState, action)` → returns `(newState, effects, events)`
3. Events are consumed by UI immediately (navigation, toasts, dialogs) — one-shot
4. Effects are sent to Processor for async execution
5. Processor does the work (API calls, DB queries, analytics) → emits `Action.Internal`
6. `Action.Internal` goes back to Updater → cycle continues
7. StateMapper transforms `State → RenderData` for the UI on every state change

---

## File Structure

Every Mobius feature requires exactly these files:

```
mobius/
├── <Feature>Mobius.kt        # All types: State, Action, Effect, Event, RenderData, Factory
├── <Feature>Updater.kt       # Pure state transitions
├── <Feature>Processor.kt     # Async effect handlers
└── <Feature>StateMapper.kt   # State → RenderData transformation
```

---

## 1. <Feature>Mobius.kt — The Type Hub

All Mobius types live in ONE file. This is the single source of truth for the feature's state machine.

```kotlin
class <Feature>Mobius(
    updater: <Feature>Updater,
    processor: <Feature>Processor,
    stateMapper: <Feature>StateMapper,
    initialState: State,
) : MobiusViewModel<Action, Effect, State, RenderData, Event>(
    updater = updater,
    processor = processor,
    stateMapper = stateMapper,
    initialState = initialState,
) {

    // ─── State ───────────────────────────────────────────
    // The complete, serializable state of the feature.
    // Every field has a sensible default.
    data class State(
        val isLoading: Boolean = true,
        val isSaving: Boolean = false,
        val items: List<ItemModel> = emptyList(),
        val selectedId: String? = null,
        val errorMessage: String? = null,
    )

    // ─── Actions ─────────────────────────────────────────
    // View = from UI, Internal = from Processor
    sealed class Action {
        sealed class View : Action() {
            data object ScreenOpened : View()
            data object BackTapped : View()
            data class ItemSelected(val id: String) : View()
            data object SubmitTapped : View()
            data object RetryTapped : View()
        }
        sealed class Internal : Action() {
            data class DataLoaded(val items: List<ItemModel>) : Internal()
            data class ItemSaved(val id: String) : Internal()
            data class Error(val message: String) : Internal()
        }
    }

    // ─── Effects ─────────────────────────────────────────
    // Commands sent to Processor. Each maps to one async operation.
    sealed class Effect {
        data object LoadData : Effect()
        data class SaveItem(val item: ItemModel) : Effect()
        data class DeleteItem(val id: String) : Effect()
        data class TrackEvent(val name: String, val params: Map<String, Any>) : Effect()
    }

    // ─── Events ──────────────────────────────────────────
    // One-shot UI events. Consumed once, never replayed.
    sealed class Event {
        data object NavigateBack : Event()
        data class NavigateToDetail(val id: String) : Event()
        data class ShowError(val message: String) : Event()
        data object ShowSuccessToast : Event()
    }

    // ─── RenderData ──────────────────────────────────────
    // What Compose sees. Derived from State by StateMapper.
    // NO business logic. NO string resources (those resolve in Compose).
    data class RenderData(
        val isLoading: Boolean,
        val showEmptyState: Boolean,
        val items: List<ItemRenderModel>,
        val isSubmitEnabled: Boolean,
        val submitLabel: SubmitLabel,
    )

    // ─── Factory ─────────────────────────────────────────
    class Factory @Inject constructor(
        private val updaterFactory: <Feature>Updater.Factory,
        private val processorFactory: <Feature>Processor.Factory,
        private val stateMapperFactory: <Feature>StateMapper.Factory,
    ) : ViewModelProvider.Factory {
        fun create(initialArgs: InitArgs): <Feature>Mobius {
            return <Feature>Mobius(
                updater = updaterFactory.create(),
                processor = processorFactory.create(),
                stateMapper = stateMapperFactory.create(),
                initialState = State(/* from initialArgs */),
            )
        }
    }
}
```

### Design Decisions
- **Enums for UI labels** (e.g., `SubmitLabel.CREATE`, `SubmitLabel.UPDATE`) instead of strings — StateMapper outputs enums, Compose resolves to `stringResource()`
- **State fields have defaults** — enables easy test builder construction
- **Action.View vs Action.Internal** — enforced separation prevents UI from faking system responses
- **Effects are commands, not events** — they describe what TO DO, not what HAPPENED

---

## 2. <Feature>Updater.kt — Pure State Machine

The Updater is a PURE FUNCTION. Given `(State, Action)`, it returns `(State, Set<Effect>, Set<Event>)`. Nothing else.

```kotlin
class <Feature>Updater : Updater<State, Action, Effect, Event> {

    override fun update(state: State, action: Action): UpdateResult<State, Effect, Event> {
        return when (action) {
            // ─── View Actions ────────────────────────
            is Action.View.ScreenOpened -> UpdateResult(
                state = state.copy(isLoading = true),
                effects = setOf(Effect.LoadData),
            )

            is Action.View.ItemSelected -> UpdateResult(
                state = state.copy(selectedId = action.id),
            )

            is Action.View.SubmitTapped -> {
                val selectedItem = state.items.find { it.id == state.selectedId }
                    ?: return UpdateResult(state) // No-op if nothing selected
                UpdateResult(
                    state = state.copy(isSaving = true),
                    effects = setOf(
                        Effect.SaveItem(selectedItem),
                        Effect.TrackEvent("submit_tapped", mapOf("itemId" to selectedItem.id)),
                    ),
                )
            }

            is Action.View.BackTapped -> UpdateResult(
                state = state,
                events = setOf(Event.NavigateBack),
            )

            is Action.View.RetryTapped -> UpdateResult(
                state = state.copy(isLoading = true, errorMessage = null),
                effects = setOf(Effect.LoadData),
            )

            // ─── Internal Actions ────────────────────
            is Action.Internal.DataLoaded -> UpdateResult(
                state = state.copy(
                    isLoading = false,
                    items = action.items,
                ),
            )

            is Action.Internal.ItemSaved -> UpdateResult(
                state = state.copy(isSaving = false),
                events = setOf(Event.ShowSuccessToast, Event.NavigateBack),
            )

            is Action.Internal.Error -> UpdateResult(
                state = state.copy(
                    isLoading = false,
                    isSaving = false,
                    errorMessage = action.message,
                ),
                events = setOf(Event.ShowError(action.message)),
            )
        }
    }

    @ActivityScope
    class Factory @Inject constructor() {
        fun create(): <Feature>Updater = <Feature>Updater()
    }
}
```

### ABSOLUTE RULES
- **NO IO** — no repository calls, no SharedPreferences, no analytics, no logging with side effects
- **NO suspend functions** — Updater is synchronous
- **NO randomness or time** — if you need a timestamp or UUID, pass it via Action.Internal from Processor
- **Handle every Action** — exhaustive `when` with no `else` branch (compiler enforces new actions are handled)
- **Return empty sets** — `UpdateResult(state)` when no effects/events needed, not null

### Patterns

**Guard clauses for no-op:**
```kotlin
is Action.View.SubmitTapped -> {
    if (state.isSaving) return UpdateResult(state) // Already saving, ignore
    // ... proceed
}
```

**Loading/saving locks:**
```kotlin
// isSaving = true disables all interactive elements via RenderData
is Action.View.SubmitTapped -> UpdateResult(
    state = state.copy(isSaving = true), // UI disables buttons
    effects = setOf(Effect.SaveItem(...)),
)
```

**Batch effects:**
```kotlin
// Multiple effects from one action
is Action.View.ScreenOpened -> UpdateResult(
    state = state.copy(isLoading = true),
    effects = setOf(
        Effect.LoadData,
        Effect.TrackEvent("screen_opened", emptyMap()),
        Effect.ResolveStickyDefaults,
    ),
)
```

---

## 3. <Feature>Processor.kt — Async Effect Handler

The Processor handles ALL async work. It receives Effects, does the work, and emits Action.Internal.

```kotlin
class <Feature>Processor(
    private val repository: <Feature>Repository,
    private val analytics: <Feature>Analytics,
    private val stickyDefaults: <Feature>StickyDefaults,
) : Processor<Effect, Action> {

    override fun process(effects: Observable<Effect>): Observable<Action> {
        return effects.flatMap { effect ->
            when (effect) {
                is Effect.LoadData -> repository.getItems()
                    .map<Action> { items -> Action.Internal.DataLoaded(items) }
                    .onErrorReturn { e -> Action.Internal.Error(e.message ?: "Load failed") }

                is Effect.SaveItem -> repository.saveItem(effect.item)
                    .toSingleDefault<Action>(Action.Internal.ItemSaved(effect.item.id))
                    .onErrorReturn { e -> Action.Internal.Error(e.message ?: "Save failed") }
                    .toObservable()

                is Effect.DeleteItem -> repository.deleteItem(effect.id)
                    .toSingleDefault<Action>(Action.Internal.ItemDeleted(effect.id))
                    .onErrorReturn { e -> Action.Internal.Error(e.message ?: "Delete failed") }
                    .toObservable()

                is Effect.TrackEvent -> {
                    analytics.track(effect.name, effect.params)
                    Observable.empty() // Analytics is fire-and-forget
                }
            }
        }
    }

    @ActivityScope
    class Factory @Inject constructor(
        private val repository: <Feature>Repository,
        private val analytics: <Feature>Analytics,
        private val stickyDefaults: <Feature>StickyDefaults,
    ) {
        fun create(): <Feature>Processor = <Feature>Processor(repository, analytics, stickyDefaults)
    }
}
```

### ABSOLUTE RULES
- **ALL async work lives here** — repository calls, analytics, SharedPreferences reads/writes, network
- **Always handle errors** — every Observable chain must have `onErrorReturn` emitting `Action.Internal.Error`
- **Analytics = fire-and-forget** — return `Observable.empty()` after tracking
- **Use RxJava operators** — `flatMap`, `switchMap`, `concatMap` depending on concurrency needs
- **Never emit Action.View** — Processor only emits Action.Internal

### Concurrency Patterns

**flatMap** — parallel execution (default for independent effects):
```kotlin
effects.flatMap { effect -> handleEffect(effect) }
```

**switchMap** — cancel previous on new (search/autocomplete):
```kotlin
effects.ofType<Effect.Search>().switchMap { effect ->
    repository.search(effect.query).map { Action.Internal.SearchResults(it) }
}
```

**concatMap** — sequential (ordered operations):
```kotlin
effects.ofType<Effect.SaveAll>().concatMap { effect ->
    Observable.fromIterable(effect.items).concatMap { item ->
        repository.save(item).toObservable()
    }
}
```

---

## 4. <Feature>StateMapper.kt — State to RenderData

Pure transformation from internal State to what Compose renders.

```kotlin
class <Feature>StateMapper(
    private val isPhone: Boolean,
) : StateMapper<State, RenderData> {

    override fun invoke(currentState: State): RenderData = with(currentState) {
        RenderData(
            isLoading = isLoading,
            showEmptyState = !isLoading && items.isEmpty(),
            items = items.map { item ->
                ItemRenderModel(
                    id = item.id,
                    title = item.name,
                    isSelected = item.id == selectedId,
                    thumbnailSize = if (isPhone) ThumbnailSize.PHONE else ThumbnailSize.TABLET,
                )
            },
            isSubmitEnabled = !isSaving && selectedId != null,
            submitLabel = if (selectedId == null) SubmitLabel.SELECT_FIRST else SubmitLabel.SUBMIT,
        )
    }

    @ActivityScope
    class Factory @Inject constructor(
        private val deviceTypeProvider: DeviceTypeProvider,
    ) {
        fun create(): <Feature>StateMapper = <Feature>StateMapper(
            isPhone = deviceTypeProvider.isPhone,
        )
    }
}
```

### ABSOLUTE RULES
- **Pure function** — no IO, no side effects, no repository calls
- **NO string resolution** — output enums/sealed classes for labels, Compose resolves via `stringResource()`
- **Phone/tablet awareness** — receives `isPhone` from `DeviceTypeProvider` at construction
- **Derive everything** — `isSubmitEnabled`, `showEmptyState`, etc. are computed from State, never stored in State

### Anti-patterns
```kotlin
// WRONG — string resolution in StateMapper
val submitText = if (isPhone) "Save" else "Save and Continue"

// CORRECT — enum that Compose resolves
val submitLabel = if (isPhone) SubmitLabel.SHORT else SubmitLabel.LONG

// In Compose:
Text(stringResource(when (renderData.submitLabel) {
    SubmitLabel.SHORT -> R.string.feature_save
    SubmitLabel.LONG -> R.string.feature_save_and_continue
}))
```

---

## 5. Fragment Entry Point

```kotlin
@FragmentKey(<Feature>Fragment::class)
@ContributesMultibinding(ActivityScope::class, Fragment::class)
class <Feature>Fragment @Inject constructor(
    private val mobiusFactory: <Feature>Mobius.Factory,
) : Fragment() {

    private val viewModel: <Feature>Mobius by viewModels {
        mobiusFactory.create(InitArgs(/* from arguments */))
    }

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View {
        return ComposeView(requireContext()).apply {
            setContent {
                val renderData by viewModel.renderData.collectAsState()
                val events by viewModel.events.collectAsState(initial = null)

                HandleEvents(events)

                <Feature>ScreenHost(
                    renderData = renderData,
                    onAction = viewModel::dispatch,
                )
            }
        }
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        // CRITICAL: Handle system back press = same as close button
        requireActivity().onBackPressedDispatcher.addCallback(viewLifecycleOwner) {
            viewModel.dispatch(Action.View.BackTapped)
        }
    }

    @Composable
    private fun HandleEvents(event: Event?) {
        LaunchedEffect(event) {
            when (event) {
                is Event.NavigateBack -> findNavController().popBackStack()
                is Event.NavigateToDetail -> findNavController().navigate(...)
                is Event.ShowError -> Toast.makeText(context, event.message, Toast.LENGTH_SHORT).show()
                is Event.ShowSuccessToast -> Toast.makeText(context, R.string.success, Toast.LENGTH_SHORT).show()
                null -> Unit
            }
        }
    }
}
```

---

## 6. Testing

### Test Infrastructure Files

```
test/
├── fixtures/
│   ├── <Feature>TestFixture.kt      # testMobius { } DSL
│   ├── <Feature>TestMocks.kt        # Stateful mock implementations
│   └── <Feature>TestBuilders.kt     # Test data factories with defaults
├── helpers/
│   └── <Feature>RenderDataAccessors.kt  # Type-safe field access
├── <Feature>UpdaterTest.kt
├── <Feature>ProcessorTest.kt
├── <Feature>StateMapperTest.kt
└── <Feature>MobiusTest.kt           # Integration tests
```

### Updater Tests (Pure — No Mocks)

```kotlin
class <Feature>UpdaterTest {
    private val updater = <Feature>Updater()

    @Test
    fun `ScreenOpened sets loading true and emits LoadData effect`() {
        val state = TestBuilders.States.buildDefault()
        val result = updater.update(state, Action.View.ScreenOpened)

        assertThat(result.state.isLoading).isTrue()
        assertThat(result.effects).containsExactly(Effect.LoadData)
        assertThat(result.events).isEmpty()
    }

    @Test
    fun `SubmitTapped when already saving is no-op`() {
        val state = TestBuilders.States.buildDefault(isSaving = true)
        val result = updater.update(state, Action.View.SubmitTapped)

        assertThat(result.state).isEqualTo(state) // Unchanged
        assertThat(result.effects).isEmpty()
    }

    @Test
    fun `Error clears loading and saving, emits ShowError event`() {
        val state = TestBuilders.States.buildDefault(isLoading = true)
        val result = updater.update(state, Action.Internal.Error("Network error"))

        assertThat(result.state.isLoading).isFalse()
        assertThat(result.state.isSaving).isFalse()
        assertThat(result.events).containsExactly(Event.ShowError("Network error"))
    }
}
```

### Processor Tests (With SchedulerRules)

```kotlin
class <Feature>ProcessorTest {
    @get:Rule val schedulerRules = SchedulerRules()

    private val mocks = <Feature>TestMocks()
    private val processor = <Feature>Processor(
        repository = mocks.repository,
        analytics = mocks.analytics,
        stickyDefaults = mocks.stickyDefaults,
    )

    @Test
    fun `LoadData calls repository and emits DataLoaded`() {
        val items = listOf(TestBuilders.Items.buildItem())
        mocks.repository.setItems(items)

        val observer = processor.process(Observable.just(Effect.LoadData)).test()
        schedulerRules.advanceSchedulerUntil { observer.valueCount() >= 1 }

        observer.assertValue(Action.Internal.DataLoaded(items))
    }

    @Test
    fun `LoadData on error emits Error action`() {
        mocks.repository.setError(IOException("Network error"))

        val observer = processor.process(Observable.just(Effect.LoadData)).test()
        schedulerRules.advanceSchedulerUntil { observer.valueCount() >= 1 }

        assertThat(observer.values().first()).isInstanceOf(Action.Internal.Error::class.java)
    }
}
```

### StateMapper Tests (Pure)

```kotlin
class <Feature>StateMapperTest {
    @Test
    fun `phone layout uses short submit label`() {
        val mapper = <Feature>StateMapper(isPhone = true)
        val state = TestBuilders.States.buildDefault(selectedId = "123")
        val renderData = mapper(state)

        assertThat(renderData.submitLabel).isEqualTo(SubmitLabel.SHORT)
    }

    @Test
    fun `tablet layout uses long submit label`() {
        val mapper = <Feature>StateMapper(isPhone = false)
        val state = TestBuilders.States.buildDefault(selectedId = "123")
        val renderData = mapper(state)

        assertThat(renderData.submitLabel).isEqualTo(SubmitLabel.LONG)
    }

    @Test
    fun `empty items with loading false shows empty state`() {
        val mapper = <Feature>StateMapper(isPhone = true)
        val state = TestBuilders.States.buildDefault(isLoading = false, items = emptyList())
        val renderData = mapper(state)

        assertThat(renderData.showEmptyState).isTrue()
    }
}
```

### Test Builders

```kotlin
object <Feature>TestBuilders {
    object States {
        fun buildDefault(
            isLoading: Boolean = false,
            isSaving: Boolean = false,
            items: List<ItemModel> = listOf(buildItem()),
            selectedId: String? = null,
            errorMessage: String? = null,
        ) = State(
            isLoading = isLoading,
            isSaving = isSaving,
            items = items,
            selectedId = selectedId,
            errorMessage = errorMessage,
        )
    }

    object Items {
        fun buildItem(
            id: String = "test-item-id",
            name: String = "Test Item",
        ) = ItemModel(id = id, name = name)
    }
}
```

### Integration Tests (TestFixture DSL)

```kotlin
class <Feature>MobiusTest {
    @get:Rule val schedulerRules = SchedulerRules()

    private val mocks = <Feature>TestMocks()

    private fun testMobius(
        initialState: State = TestBuilders.States.buildDefault(),
        block: TestFixture.() -> Unit,
    ) {
        TestFixture(
            updater = <Feature>Updater(),
            processor = <Feature>Processor(mocks.repository, mocks.analytics, mocks.stickyDefaults),
            stateMapper = <Feature>StateMapper(isPhone = true),
            initialState = initialState,
            schedulerRules = schedulerRules,
        ).apply(block)
    }

    @Test
    fun `full flow - screen opens, data loads, user selects and submits`() {
        val items = listOf(TestBuilders.Items.buildItem(id = "item-1"))
        mocks.repository.setItems(items)

        testMobius {
            dispatch(Action.View.ScreenOpened)
            advanceUntil { !latestState.isLoading }

            assertThat(latestState.items).hasSize(1)

            dispatch(Action.View.ItemSelected("item-1"))
            assertThat(latestState.selectedId).isEqualTo("item-1")

            dispatch(Action.View.SubmitTapped)
            advanceUntil { !latestState.isSaving }

            assertThat(latestEvents).contains(Event.ShowSuccessToast)
        }
    }
}
```

### Test Naming Convention
Use backtick-style descriptive names that describe the scenario:
```kotlin
fun `ScreenOpened sets photos and triggers ResolveDefaultType effect`()
fun `PhotoDeleted on last item selects previous photo`()
fun `TypeUpdateCompleted with required fields transitions Open to Draft`()
fun `SubmitTapped when isSaving is true does nothing`()
```

---

## 7. Error Handling Patterns

```kotlin
// In Updater — set loading/saving flags to disable UI
is Action.View.SubmitTapped -> UpdateResult(
    state = state.copy(isSaving = true), // Disables all buttons via RenderData
    effects = setOf(Effect.SaveItem(item)),
)

// In Processor — always catch errors
is Effect.SaveItem -> repository.save(effect.item)
    .toSingleDefault<Action>(Action.Internal.ItemSaved(effect.item.id))
    .onErrorReturn { e -> Action.Internal.Error(e.message ?: "Save failed") }
    .toObservable()

// In Updater — handle error action
is Action.Internal.Error -> UpdateResult(
    state = state.copy(isLoading = false, isSaving = false),
    events = setOf(Event.ShowError(action.message)),
)
```

### Error Scenario Table Template
| Scenario | Effect | Processor Handling | Updater Handling | User Impact |
|----------|--------|-------------------|------------------|-------------|
| Network error | LoadData | `onErrorReturn(Error)` | Clear loading, show error | Toast + retry option |
| Save failed | SaveItem | `onErrorReturn(Error)` | Clear saving, show error | Toast, can retry |
| Not found | LoadDetail | `onErrorReturn(Error)` | Navigate back | Screen closes |

---

## Quick Reference: What Goes Where

| Logic Type | Where | Example |
|-----------|-------|---------|
| State transitions | Updater | `isSaving = true`, `selectedId = action.id` |
| API/DB calls | Processor | `repository.getItems()` |
| Analytics tracking | Processor | `analytics.trackScreenOpened()` |
| SharedPreferences | Processor | `stickyDefaults.saveLastType(type)` |
| String resolution | Compose UI | `stringResource(R.string.title)` |
| Phone/tablet labels | StateMapper | `if (isPhone) Label.SHORT else Label.LONG` |
| Phone/tablet layout | Compose UI | `if (isPhoneLayout) PhoneLayout() else TabletLayout()` |
| Navigation | Fragment (via Event) | `Event.NavigateBack` → `findNavController().popBackStack()` |
| Back press | Fragment | `onBackPressedDispatcher.addCallback { dispatch(BackTapped) }` |
| Button enabled/disabled | StateMapper → RenderData | `isSubmitEnabled = !isSaving && selectedId != null` |
| Loading spinners | StateMapper → RenderData | `isLoading = state.isLoading` |
