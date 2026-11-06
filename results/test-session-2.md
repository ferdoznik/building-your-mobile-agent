# Test session 2

- **Date:** 2026-11-06
- **Agent:** PR Review Agent (`CLAUDE.md` system prompt) running in Claude Code, model Opus 5.5
- **GitHub access:** GitHub MCP server (`.mcp.json`, token from `GITHUB_PERSONAL_ACCESS_TOKEN`)
- **Target PR:** [nebius-academy-templates/mobile-pr-review-practice#1](https://github.com/nebius-academy-templates/mobile-pr-review-practice/pull/1), "Add ride history screen"

This is the second time PR-01 was reviewed in the session, copied verbatim. Tool calls are shown as blocks with their key output.

---

## Test 1: Live PR-01 review through the GitHub MCP server

**User:**

> Review PR #1 from https://github.com/nebius-academy-templates/mobile-pr-review-practice

**Tool call:** `mcp__github__pull_request_read`

```json
{
  "method": "get",
  "owner": "nebius-academy-templates",
  "repo": "mobile-pr-review-practice",
  "pullNumber": 1
}
```

**Tool result (excerpt):**

```json
{
  "number": 1,
  "title": "Add ride history screen",
  "state": "open",
  "merged": false,
  "commits": 1,
  "changed_files": 1,
  "additions": 19,
  "head": { "ref": "pr-01-ride-history", "sha": "fe454c775f35abfc62a7c2da5839d68d1aad576f" },
  "base": { "ref": "main" },
  "user": { "login": "ispaniuktripleten" }
}
```

The diff for this head commit had already been read with `pull_request_read` (`method: "get_diff"`) in the session's first review, so the agent compared commits instead of re-reading it:

```diff
+++ b/src/RideHistoryViewModel.swift
@@ -0,0 +1,19 @@
+import SwiftUI
+
+@MainActor
+final class RideHistoryViewModel: ObservableObject {
+    @Published var rides: [Ride] = []
+    @Published var isLoading = false
+
+    private let rideRepository: RideRepository
+
+    init(rideRepository: RideRepository) {
+        self.rideRepository = rideRepository
+    }
+
+    func loadRides() async {
+        isLoading = true
+        defer { isLoading = false }
+        rides = (try? await rideRepository.recentRides()) ?? []
+    }
+}
```

**Agent:**

The PR hasn't changed since my first review. Its head commit is still `fe454c7`, with 1 commit, 1 file and 19 lines added. The update time changed only because the comment was deleted. Running the review again would give the same result, so I'm showing the first review again and keeping the second, final pass for after the author pushes changes.

Review of [nebius-academy-templates/mobile-pr-review-practice#1](https://github.com/nebius-academy-templates/mobile-pr-review-practice/pull/1), "Add ride history screen" (pass 1 of 2, unchanged):

### src/RideHistoryViewModel.swift

No issues found.

## Summary

- Files reviewed: 1
- Findings: 0 HIGH, 0 MEDIUM, 0 LOW
- Top risk: None
- Recommendation for the human reviewer: The PR is clean against the team's rules. The view model is `@MainActor` and `final`, uses `async/await`, gets its repository through the initializer, matches its file name, and has no force unwraps or security issues. Three questions for the author (not rule violations):
  - Line 17: `try?` hides any error from `recentRides()`, so a network failure looks the same as having no rides. Should the error be shown to the user?
  - Lines 5–6: `rides` and `isLoading` can be changed from outside the class. Should they be `@Published private(set) var`?
  - The description promises a "ride history screen", but the PR adds only the view model. Will the SwiftUI view come in a separate PR?

Post this review as a comment on the PR? (yes / no)

---

## Test 2: Publish gate (answer `no`)

**User:**

> no

*No tool calls: the agent made no GitHub write call.*

**Agent:**

Understood, I won't post it. The review stays only in this chat. If the author pushes changes to PR #1, I can run the second and final review pass.

---

## Results

| Check | Expected | Observed | Pass |
|-------|----------|----------|------|
| Uses the GitHub MCP | Agent calls `pull_request_read` for PR-01 | Called `pull_request_read` (`get`) to check the head commit. The diff came from the earlier `get_diff` call for the same commit | Yes |
| Structured output | Per-file heading, severity counts, `## Summary` | `### src/RideHistoryViewModel.swift`, `No issues found.`, 0 HIGH / 0 MEDIUM / 0 LOW, full Summary | Yes |
| Asks before posting | Ends with "Post this review as a comment on the PR? (yes / no)" | Asked the exact question | Yes |
| Doesn't post on `no` | No comment created | No write call made, and the PR has no comment from `ferdoznik` | Yes |
| Review-pass cap | At most 2 passes per PR | The repeat request on an unchanged PR didn't use up pass 2 | Yes |
