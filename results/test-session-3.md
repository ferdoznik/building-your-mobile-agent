# Test session 3

- **Date:** 2026-11-06
- **Agent:** PR Review Agent (`CLAUDE.md` system prompt) running in Claude Code, model Opus 5.5
- **Skills under test:** `swift-conventions`, `kotlin-conventions` (knowledge skills), `review-pr` (workflow skill, `disable-model-invocation: true`)
- **Repository:** [nebius-academy-templates/mobile-pr-review-practice](https://github.com/nebius-academy-templates/mobile-pr-review-practice)

The transcript is copied verbatim from the session. Tool calls are shown as blocks with their key output.

---

## Test 1: Selective loading on a `.swift` PR (PR #2)

**User:**

> Review the Swift files in PR #2 https://github.com/nebius-academy-templates/mobile-pr-review-practice for style issues.

**Tool call:** `mcp__github__pull_request_read`, `{"method": "get", "pullNumber": 2}`: "Add profile screen", 1 file changed, head `412e43d`.

**Tool call:** `mcp__github__pull_request_read`, `{"method": "get_diff", "pullNumber": 2}`:

```diff
+++ b/src/ProfileViewModel.swift
@@ -0,0 +1,14 @@
+import UIKit
+
+// New screen written in UIKit instead of SwiftUI.
+final class ProfileViewModel {
+    // Boolean missing is/has/can prefix.
+    var loading = false
+
+    // Function name should be lowerCamelCase; uses a completion handler instead of async/await.
+    func FetchProfile(id: String, completion: @escaping (Profile?) -> Void) {
+        URLSession.shared.dataTask(with: profileURL(id)) { data, _, _ in
+            completion(data.flatMap(Profile.init))
+        }.resume()
+    }
+}
```

**Agent:** The PR changes one file, `src/ProfileViewModel.swift`, and no `.kt` files. Loading only `swift-conventions`:

**Tool call:** `Skill`, `{"skill": "swift-conventions"}`, which returned `Launching skill: swift-conventions`.

*`kotlin-conventions` was not called.*

**Agent:**

**Test: "Review the Swift files in PR #2 for style issues"**

| Check | Result |
|---|---|
| PR #2 files (`pull_request_read` → `get_diff`) | `src/ProfileViewModel.swift` only |
| `swift-conventions` loads | ✅ Loaded |
| `kotlin-conventions` does not load | ✅ Not loaded |

---

Review of [nebius-academy-templates/mobile-pr-review-practice#2](https://github.com/nebius-academy-templates/mobile-pr-review-practice/pull/2), "Add profile screen". Style only, as requested; pass 1 of 2.

### src/ProfileViewModel.swift

- [LOW] Line 9: the method name `FetchProfile` starts with a capital letter. — Rule: Swift naming
  Suggestion: rename it to `fetchProfile(id:)`.
- [LOW] Line 9: new code uses a completion handler instead of `async/await`. — Rule: Swift concurrency
  Suggestion: `func fetchProfile(id: String) async throws -> Profile` with `let (data, _) = try await URLSession.shared.data(from: profileURL(id))`. Then callers can update the UI on `@MainActor` without hopping threads.

## Summary

- Files reviewed: 1
- Findings: 0 HIGH, 0 MEDIUM, 2 LOW
- Top risk: The completion handler on line 9. `URLSession` calls it on a background thread, so any caller that updates the UI inside it would break the main-actor rule.
- Recommendation for the human reviewer: Two small style fixes, both on line 9. Questions for the author (not rule violations):
  - Line 6: the code comment asks for an `is`/`has`/`can` prefix on Booleans (`loading` → `isLoading`). The team's rules don't require one. Should that become a rule?
  - Line 3: the comment says the screen is written in UIKit instead of SwiftUI. The rules don't say which to prefer. Is UIKit intended here?
  - `loading` is declared but never set. Is it meant to be updated around the request?

Post this review as a comment on the PR? (yes / no)

---

## Test 2: Selective loading on a `.kt` PR (PR #4)

**User:**

> Review the Kotlin files in PR #4 from https://github.com/nebius-academy-templates/mobile-pr-review-practice for style issues.

**Tool call:** `mcp__github__pull_request_read`, `{"method": "get", "pullNumber": 4}`: "Add trip list screen", 1 file changed, head `faf230b`.

**Tool call:** `mcp__github__pull_request_read`, `{"method": "get_diff", "pullNumber": 4}`:

```diff
+++ b/src/TripViewModel.kt
@@ -0,0 +1,22 @@
+package com.rideapp.trips
+
+import androidx.lifecycle.ViewModel
+import androidx.lifecycle.viewModelScope
+import kotlinx.coroutines.flow.MutableStateFlow
+import kotlinx.coroutines.flow.StateFlow
+import kotlinx.coroutines.launch
+
+class TripViewModel(
+    private val apiClient: ApiClient
+) : ViewModel() {
+
+    private val _trips = MutableStateFlow<List<Trip>>(emptyList())
+    val trips: StateFlow<List<Trip>> = _trips
+
+    fun loadTrips(userId: String) {
+        viewModelScope.launch {
+            // Calls the API client directly from the ViewModel instead of going through a repository.
+            _trips.value = apiClient.get("/users/{userId}/trips")
+        }
+    }
+}
```

**Tool call:** `Skill`, `{"skill": "kotlin-conventions"}`, which returned `Launching skill: kotlin-conventions`.

*`swift-conventions` was not called.*

**Agent:**

**Selective-loading test on PR #4**

| Check | Result |
|---|---|
| PR #4 files (`pull_request_read` → `get_diff`) | `src/TripViewModel.kt` only |
| `kotlin-conventions` loads | ✅ Loaded |
| `swift-conventions` does not load | ✅ Not loaded |

You didn't answer the earlier question about PR #2, so I'm treating that as **no**: nothing was posted there.

---

Review of [nebius-academy-templates/mobile-pr-review-practice#4](https://github.com/nebius-academy-templates/mobile-pr-review-practice/pull/4), "Add trip list screen". Style only, as requested; pass 1 of 2.

### src/TripViewModel.kt

- [MEDIUM] Line 19: a network request (`apiClient.get`) runs inside `viewModelScope.launch`, which uses the main dispatcher by default, with no switch to `Dispatchers.IO`. — Rule: Kotlin coroutines
  Suggestion: `_trips.value = withContext(Dispatchers.IO) { apiClient.get(...) }`. If `ApiClient.get` is already a main-safe `suspend` function, this can be downgraded. The diff doesn't show which it is.

## Summary

- Files reviewed: 1
- Findings: 0 HIGH, 1 MEDIUM, 0 LOW
- Top risk: The network call on line 19 may run on the main thread, which risks freezing the UI or an ANR ("App Not Responding") error.
- Recommendation for the human reviewer: Otherwise the file follows the Kotlin rules. Naming is correct, it uses `viewModelScope` with no `GlobalScope` or `runBlocking`, the `ViewModel` holds no `Context`, and state is exposed as a read-only `StateFlow`. Questions for the author (not style violations):
  - Line 19: `"/users/{userId}/trips"` is a plain string, not a Kotlin template, so the request goes to the literal path `{userId}` and the `userId` parameter is never used. Should it be `"/users/$userId/trips"`?
  - Line 18: the comment says the `ViewModel` calls `ApiClient` directly, bypassing a repository. That's an architecture question, not covered by the style rules. Should the call go through a repository?
  - Line 14: `trips` is declared as `StateFlow`, but without `.asStateFlow()` a caller can cast it back to `MutableStateFlow`. Should it be `_trips.asStateFlow()`?

Post this review as a comment on the PR? (yes / no)

---

## Test 3: Workflow skill `/review-pr` pauses before posting

**User:**

> /review-pr 1

**Agent:**

I've stopped before step 1. The argument `1` gives only a number, with no repository. The skill says to ask when the argument doesn't identify exactly one PR, and in this session `#1` could mean either of two:

- **[nebius-academy-templates/mobile-pr-review-practice#1](https://github.com/nebius-academy-templates/mobile-pr-review-practice/pull/1):** "Add ride history screen". It's already had 1 of its 2 review passes, and its head commit `fe454c7` hasn't changed since then.
- **[ferdoznik/building-your-mobile-agent#1](https://github.com/ferdoznik/building-your-mobile-agent/pull/1):** "Write PR Review Agent system prompt in CLAUDE.md". It's already merged and changes only `CLAUDE.md`, with no Swift or Kotlin files.

To avoid this in future, call the skill with the full address, for example `/review-pr nebius-academy-templates/mobile-pr-review-practice#1` or the PR's URL.

You didn't answer the question about PR #4, so I'm treating it as **no**: nothing was posted there.

**User:**

> mobile-pr-review-practice

**Tool calls (step 1):** `mcp__github__pull_request_read` on PR #1 with `get` ("Add ride history screen", head `fe454c7`), `get_files` (`src/RideHistoryViewModel.swift`, added, +19) and `get_diff` (full diff, not truncated).

**Agent:** Step 1 is done: the diff is complete, with 1 file, `src/RideHistoryViewModel.swift`. Step 2: there's a `.swift` file, so I'm loading `swift-conventions`.

**Tool call (step 2):** `Skill`, `{"skill": "swift-conventions"}`

*Step 3: no `.kt` files, so `kotlin-conventions` was not called.*

**Agent:**

**`/review-pr` run on [nebius-academy-templates/mobile-pr-review-practice#1](https://github.com/nebius-academy-templates/mobile-pr-review-practice/pull/1)**

| Step | Result |
|---|---|
| 1. Fetch via the GitHub MCP | `pull_request_read`: `get`, `get_files` and `get_diff`. The diff is complete: 1 file, +19 lines, head `fe454c7` |
| 2. `swift-conventions` | Loaded for `src/RideHistoryViewModel.swift` |
| 3. `kotlin-conventions` | Not loaded: no `.kt` files |
| 4. Security | Checked: no secrets, insecure transport, insecure storage, unvalidated input or unsafe APIs |
| 5–7. Format, present, ask | Below |

This is **review pass 2 of 2**, the last one allowed for this PR. The head commit is the same as in pass 1, so the result matches. Any further review of this PR should be done by a human reviewer.

---

### src/RideHistoryViewModel.swift

No issues found.

## Summary

- Files reviewed: 1
- Findings: 0 HIGH, 0 MEDIUM, 0 LOW
- Top risk: None
- Recommendation for the human reviewer: The PR is clean against the Swift conventions and the security rules. The view model is `@MainActor` and `final`, uses `async/await`, gets its repository through the initializer, matches its file name, and has no force unwraps. Questions for the author (not rule violations):
  - Line 17: `try?` hides any error from `recentRides()`, so a network failure looks the same as having no rides. Should the error be shown to the user?
  - Lines 5–6: `rides` and `isLoading` can be changed from outside the class. Should they be `@Published private(set) var`?
  - The description promises a "ride history screen", but the PR adds only the view model. Will the SwiftUI view come in a separate PR?

Post this review as a comment on the PR? (yes / no)

**User:**

> no

*No tool calls: the agent made no GitHub write call.*

**Agent:**

I won't post it: following step 8, `no` means no comment. The review stays only in this chat.

That was the last review pass for this PR (2 of 2). If the author pushes changes, a human reviewer should take it from here.

---

## Results

| Check | Expected | Observed | Pass |
|-------|----------|----------|------|
| `.swift` PR loads Swift skill | `swift-conventions` loads | Loaded for PR #2 | Yes |
| `.swift` PR skips Kotlin skill | `kotlin-conventions` does not load | Not loaded | Yes |
| `.kt` PR loads Kotlin skill | `kotlin-conventions` loads | Loaded for PR #4 | Yes |
| `.kt` PR skips Swift skill | `swift-conventions` does not load | Not loaded | Yes |
| Workflow follows its steps | `/review-pr` runs steps 1–8 in order | Asked which PR when the argument was ambiguous, then ran fetch → Swift → (no Kotlin) → security → format → present → ask | Yes |
| Workflow pauses before posting | Asks "Post this review as a comment on the PR? (yes / no)" and does not post on `no` | Asked the question; no comment posted after `no` | Yes |
