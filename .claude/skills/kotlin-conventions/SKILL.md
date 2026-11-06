---
name: kotlin-conventions
description: "Kotlin style conventions for the Android app: naming, coroutines, and Jetpack/lifecycle, with severity tags. Load when reviewing or writing .kt files. Do not load for .swift or other non-Kotlin files."
---

## Rules

Apply these rules only to lines a PR adds or changes in `.kt` files. Cite the rule name in each
finding, for example `Rule: Kotlin naming`.

### Kotlin naming — `[LOW]`

- Classes, interfaces, and objects use `UpperCamelCase`.
- Functions and properties use `lowerCamelCase`.
- Constants use `UPPER_SNAKE_CASE` and live in a `companion object` or at the top level.

### Kotlin coroutines

- `[MEDIUM]` `GlobalScope` or `runBlocking` in production code. Use structured concurrency:
  `viewModelScope` or `lifecycleScope`.
- `[MEDIUM]` Blocking I/O (network, disk, database) on the main dispatcher. Run it with
  `withContext(Dispatchers.IO)`.

### Kotlin Jetpack/lifecycle

- `[MEDIUM]` A `ViewModel` holds an `Activity`, `Fragment`, `View`, or `Context` reference, which
  leaks memory. Use `AndroidViewModel` or inject what's needed instead.
- `[LOW]` State is exposed as a mutable public field (`var`, `MutableStateFlow`,
  `MutableLiveData`) instead of a read-only `StateFlow` or `LiveData` backed by a private mutable
  one.
- `[LOW]` Flows are collected without lifecycle awareness, for example in `lifecycleScope.launch`
  without `repeatOnLifecycle`, or without `collectAsStateWithLifecycle` in Compose.

## Template

Compliant:

```kotlin
class RideHistoryViewModel(
    private val rideRepository: RideRepository,
) : ViewModel() {

    private val _rides = MutableStateFlow<List<Ride>>(emptyList())
    val rides: StateFlow<List<Ride>> = _rides.asStateFlow()

    fun loadRides() {
        viewModelScope.launch {
            _rides.value = withContext(Dispatchers.IO) { rideRepository.recentRides() }
        }
    }

    companion object {
        const val MAX_RIDES = 50
    }
}
```

Non-compliant:

```kotlin
class ride_history_vm(                               // [LOW] Kotlin naming
    private val activity: Activity,                  // [MEDIUM] Context reference in ViewModel
) : ViewModel() {

    var Rides = MutableLiveData<List<Ride>>()        // [LOW] Kotlin naming; [LOW] mutable public state
    val maxRides = 50                                // [LOW] Kotlin naming: constant not UPPER_SNAKE_CASE

    fun LoadRides() {                                // [LOW] Kotlin naming
        GlobalScope.launch {                         // [MEDIUM] GlobalScope
            Rides.postValue(runBlocking { api.fetchRides() })   // [MEDIUM] runBlocking
        }
    }
}
```
