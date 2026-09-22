# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **Drop-in Android template.** Fill in "Project overrides" below, delete this blockquote, and delete any decision table once that decision is made and recorded.

## Project overrides

_None yet — defaults below apply._

<!-- Record deviations here so they win over everything downstream. Example:
- UI is XML Views, not Compose (legacy screens in :feature:checkout)
- Koin instead of Hilt (matches the wider org)
-->

---

## Non-negotiables

These hold regardless of which stack choices a project makes. Violating one is a bug, not a preference.

1. **The domain layer imports nothing from Android.** No `Context`, no `android.*`, no framework annotations. If it can't be tested by a plain JUnit run with no Robolectric, it isn't domain.
2. **Never call `Instant.now()`, `System.currentTimeMillis()`, or `LocalDate.now()` in business logic.** Inject a `Clock`. Time-dependent behaviour is otherwise untestable without sleeping.
3. **Derive state; don't store what you can compute.** A stored `isExpired`/`isAtRisk`/`isValid` flag needs a job to keep it true. A function of `(model, now)` does not.
4. **Unidirectional data flow.** State flows down, events flow up. A composable never mutates a model it was handed.
5. **One source of truth per piece of data.** If two layers both hold it, you have written a merge bug. Push ownership down to the lowest layer that can own it.
6. **Repository interfaces are owned by the consumer, implementations by the data layer.** The interface says what the feature needs, not what the API returns.
7. **No silent `catch`.** Either handle it, model it in a return type, or let it crash. `catch (e: Exception) {}` is never correct.
8. **Dispatchers are injected, never hardcoded** in anything you intend to test.

---

## Decisions to make per project

Pick deliberately, record the choice in "Project overrides," then delete the table.

### UI toolkit
| Choose | When |
|---|---|
| **Compose** (default) | New app or new feature surface. Anything greenfield. |
| XML Views | Existing XML codebase, or a team without Compose experience on a hard deadline. |
| Mixed | Migrating. Keep the boundary at whole screens, never inside one. |

### Presentation pattern
| Choose | When |
|---|---|
| **MVVM + repository** (default) | Most features. Lowest ceremony, well understood by any Android reviewer. |
| MVI / reducer | Complex state with many interleaved events, or you need time-travel/replay for debugging. Costs boilerplate. |
| Plain state holder | A screen with no async work. Don't reach for a ViewModel to hold two booleans. |

### Dependency injection
| Choose | When |
|---|---|
| **Hilt** (default) | Production app, team project, graph beyond a handful of bindings. |
| Hilt at the edges | Timed exercise or prototype. Annotations only on `Application`/`Activity`/`ViewModel`/one module; everything else plain constructors, so it's removable in minutes. |
| Manual container | Tiny app, or build-time/annotation-processing cost genuinely matters. Be ready to justify the absence of Hilt. |
| Koin | Org already standardised on it, or KMP shared DI. Runtime failures instead of compile-time. |

### Async & state
| Choose | When |
|---|---|
| **Coroutines + Flow** (default) | Everything. `StateFlow` for state, `Flow` for streams, `suspend` for one-shots. |
| RxJava | Existing Rx codebase only. Don't add it. |
| `LiveData` | Don't, in new code. `StateFlow` + `collectAsStateWithLifecycle()`. |

### Persistence
| Choose | When |
|---|---|
| In-memory `StateFlow` | Prototype, exercise, or genuinely ephemeral data. |
| **DataStore** | Preferences, small key-value, auth tokens. Never `SharedPreferences` in new code. |
| **Room** | Relational data, queries, or offline-first. |

### Navigation
| Choose | When |
|---|---|
| **Navigation Compose, type-safe routes** | Multi-screen Compose app. |
| Single composable + state | One or two screens. A nav graph for two screens is overhead. |

---

## Testing standards

- **Test the state machine and the transition rules first.** That's where real bugs live, and it needs no Android.
- **Virtual time, never real waiting.** `runTest` + `advanceTimeBy`. A test containing `Thread.sleep` or `delay` against a real clock is a flaky test.
- **Turbine for flow assertions.** Manual collection into a list is noisier and races.
- **Test behaviour, not implementation.** Assert on emitted state, not on which private method ran.
- **Name tests as sentences:** `` fun `rejects backward transition`() ``.
- What's worth testing: transition validity (including every invalid move), boundary conditions (exactly at the threshold, and one unit either side), empty/loading/error states, and that concurrent updates don't clobber each other.
- What isn't: getters, `data class` equality, framework behaviour, Compose layout details.

Baseline deps: `junit`, `kotlinx-coroutines-test`, `turbine`, `assertk` or `kotlin.test`. Add Robolectric only when a JVM test genuinely needs Android, and expect a slow first run.

---

## Compose rules

- **Hoist state.** Composables below the screen take data + lambdas, never a ViewModel.
- One `collectAsStateWithLifecycle()` per screen, at the top. Not `collectAsState()` — it keeps collecting in the background.
- Every reusable composable gets a `modifier: Modifier = Modifier` as its first optional parameter.
- Prefer a `@Preview` for each state (loading, empty, content, error). They're the fastest check that a state is actually handled.
- Don't read state higher in the tree than necessary — it widens recomposition scope.
- Lists: always supply a stable `key` to `items()`.

## Concurrency rules

- ViewModel work runs in `viewModelScope`; it's cancelled for you.
- Expose state with `.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), initial)`. The 5s window survives rotation without restarting upstream work.
- `WhileSubscribed` for UI state; `Eagerly` only when upstream must run regardless of observers.
- Repositories take an injected `CoroutineScope` for their own long-lived work — they do not create `GlobalScope`.

## Error handling

- Model expected failures in the return type (`Result<T>` or a sealed error hierarchy). Reserve exceptions for genuine bugs.
- Errors reaching the UI are a UI state, not a toast fired from a ViewModel.
- Invalid state transitions return a typed error naming *why* — not a bare `false`.

---

## Skeletons

The patterns that are easy to get subtly wrong.

### Injectable clock
```kotlin
// domain — no Android, no static time
fun interface Clock { fun now(): Instant }

object SystemClock : Clock { override fun now(): Instant = Instant.now() }

class FakeClock(var current: Instant) : Clock {
    override fun now(): Instant = current
    fun advanceBy(d: Duration) { current += d }
}
```

### UI state
```kotlin
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data object Empty : UiState<Nothing>
    data class Content<T>(val data: T) : UiState<T>
    data class Error(val message: String) : UiState<Nothing>
}
```
Empty is a distinct state, not `Content(emptyList())` — the UI almost always treats them differently.

### Repository as single source of truth, with compare-and-set writes
```kotlin
interface OrderRepository {
    val items: Flow<List<Order>>
    // 'expected' makes double-taps and lost races safe: a stale caller is rejected,
    // not silently applied on top of someone else's write.
    suspend fun advance(id: String, expected: Status): Result<Order>
}

class InMemoryOrderRepository(
    private val clock: Clock,
    private val scope: CoroutineScope,
) : OrderRepository {
    private val state = MutableStateFlow(emptyList<Order>())
    override val items: Flow<List<Order>> = state.asStateFlow()
    // Background producers write to the SAME flow, so incoming data can never
    // clobber a local edit. One list, therefore nothing to merge.
}
```

### Time-derived state that updates without new data
When state must change on a deadline (expiry, SLA breach, countdown), nothing pushes at the threshold — combine the data flow with a ticker.
```kotlin
private val ticker = flow {
    while (true) { emit(clock.now()); delay(1.seconds) }
}

val uiState: StateFlow<UiState<List<OrderUi>>> =
    combine(repository.items, ticker) { items, now -> items.toUiState(now) }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), UiState.Loading)
```

### Main dispatcher rule
```kotlin
class MainDispatcherRule(
    private val dispatcher: TestDispatcher = UnconfinedTestDispatcher(),
) : TestWatcher() {
    override fun starting(d: Description) = Dispatchers.setMain(dispatcher)
    override fun finished(d: Description) = Dispatchers.resetMain()
}
```

### Virtual-time flow test
```kotlin
@get:Rule val mainDispatcherRule = MainDispatcherRule()

@Test
fun `flags order once threshold elapses`() = runTest {
    viewModel.uiState.test {
        assertFalse(awaitItem().requireContent().first().isAtRisk)
        advanceTimeBy(5.minutes + 1.seconds)
        assertTrue(awaitItem().requireContent().first().isAtRisk)
        cancelAndIgnoreRemainingEvents()
    }
}
```

---

## Anti-patterns

| Don't | Do |
|---|---|
| `Instant.now()` in a ViewModel or use case | Inject `Clock` |
| Store a derived flag (`isExpired`) | Compute from `(model, now)` |
| ViewModel holds its own copy of repo data | Repository owns it; ViewModel derives |
| `setStatus(READY)` | `advance(id, expected = PREPARING)` |
| `catch (e: Exception) {}` | Typed error in the return, or let it crash |
| `GlobalScope.launch` | `viewModelScope`, or an injected scope |
| Pass ViewModel into child composables | Hoist state; pass data + lambdas |
| `Content(emptyList())` for empty | A distinct `Empty` state |
| `Thread.sleep` in tests | `runTest` + `advanceTimeBy` |
| `!!` | Model the absence, or fail with a message |

---

## Definition of done

A feature is done when: every state (loading, empty, content, error) is handled; transition/business rules have tests including the invalid cases; nothing reads wall-clock time directly; the domain layer still has no Android imports; and the architecture choice above can be justified in one sentence.
