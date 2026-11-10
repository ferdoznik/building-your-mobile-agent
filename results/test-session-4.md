# Test session 4

- **Date:** 2026-11-10
- **Agent:** PR Review Agent (`CLAUDE.md` system prompt) running in Claude Code, model Opus 5.5
- **Under test:** workflow skill `review-pr` delegating to three subagents in sequence: `style-reviewer` (Sonnet, preloads `swift-conventions` and `kotlin-conventions`), `security-reviewer` (Sonnet), and `architecture-reviewer` (Opus, preloads `architecture-guidelines`, which is built from `docs/adr/`)
- **Repository:** [nebius-academy-templates/mobile-pr-review-practice](https://github.com/nebius-academy-templates/mobile-pr-review-practice)

The transcript is copied from the session. Tool calls are shown as blocks with their key output. The secret in the diff is masked (`sk-live-9f8a…`) as the `CLAUDE.md` guardrail requires.

---

## Run: `/review-pr 3`

**User:**

> /review-pr 3

**Agent:**

I stopped before step 1. The argument `3` is just a number with no repository, and the skill says to ask when it doesn't point to exactly one PR. In this session, #3 could be either of these:

- **nebius-academy-templates/mobile-pr-review-practice#3:** the repo we've been using for PR reviews (#1, #2 and #4 so far). I haven't looked at #3 yet.
- **nebius-academy-templates/release-notes-practice#3:** "Add client search on the dashboard". This is the release-notes practice repo, not mobile code.

To skip this question, call it with the full name, for example `/review-pr nebius-academy-templates/mobile-pr-review-practice#3`, or with the PR's URL.

**User:**

> mobile-pr-review-practice

### Step 1: Fetch the PR via the GitHub MCP

**Tool call:** `mcp__github__pull_request_read`, `{"method": "get", "pullNumber": 3}`: "Add payment service" by `ispaniuktripleten`, 1 file changed, +18, head `38b3e72`. Description: "Adds the payment service that charges a rider's saved card via the payments API."

**Tool call:** `mcp__github__pull_request_read`, `{"method": "get_files", "pullNumber": 3}`: `src/PaymentService.kt`, added, +18.

**Tool call:** `mcp__github__pull_request_read`, `{"method": "get_diff", "pullNumber": 3}`:

```diff
diff --git a/src/PaymentService.kt b/src/PaymentService.kt
new file mode 100644
--- /dev/null
+++ b/src/PaymentService.kt
@@ -0,0 +1,18 @@
+package com.rideapp.payments
+
+import retrofit2.Retrofit
+import retrofit2.converter.gson.GsonConverterFactory
+
+class PaymentService {
+
+    val API_KEY = "sk-live-9f8a…"
+
+    private val retrofit = Retrofit.Builder()
+        .baseUrl("https://api.rideapp.com/payments/")
+        .addConverterFactory(GsonConverterFactory.create())
+        .build()
+
+    fun charge(amount: Int, token: String) {
+        // Sends the charge request with API_KEY in the Authorization header.
+    }
+}
\ No newline at end of file
```

**Agent:**

`/review-pr` on [mobile-pr-review-practice#3](https://github.com/nebius-academy-templates/mobile-pr-review-practice/pull/3) ("Add payment service") is underway.

- **Step 1 is done.** The diff is complete: one Kotlin file, `src/PaymentService.kt`, with 18 added lines.
- **Step 2 is running.** `style-reviewer` is reviewing the diff.

After it finishes, `security-reviewer` and then `architecture-reviewer` will review the same diff, one at a time. Then I'll combine their findings into a single review.

### Step 2: `style-reviewer` subagent

**Tool call:** `Agent`, `{"subagent_type": "style-reviewer", "description": "Style review PR #3"}`. The prompt contains the PR, the changed-file list, and the numbered diff.

**Subagent report (`style-reviewer`):**

> ### src/PaymentService.kt
>
> - [LOW] Line 8: `val API_KEY` is an UPPER_SNAKE_CASE name on an instance property inside the class body. Constants must live in a `companion object` or at top level, and instance properties must use `lowerCamelCase`. — Rule: Kotlin naming
>   Suggestion: If it is a constant, move it into `companion object { const val API_KEY = ... }`. If it is not a constant, rename it to `apiKey`. Either way, the value should not be hardcoded (see the question below).
>
> Style findings: 0 MEDIUM, 1 LOW
> Questions for the author:
> - Line 8 appears to hold a hardcoded live-looking API key (masked: `sk-live-9f8a…`). No style rule covers this, so I am not reporting it as a finding. It is in security reviewer scope.
> - Line 18: the file has no trailing newline. No rule covers this.
> - Line 15: `charge` has an empty body, and the parameters `amount` and `token` are unused. Is this intentionally a stub?

**Agent:**

Step 2 is done. `style-reviewer` found one `[LOW]` naming issue and left three questions for the author. Step 3 is running now, with `security-reviewer` reviewing the same diff.

### Step 3: `security-reviewer` subagent

**Tool call:** `Agent`, `{"subagent_type": "security-reviewer", "description": "Security review PR #3"}`. The prompt contains the same PR data and diff, with no style findings.

**Subagent report (`security-reviewer`):**

> ### src/PaymentService.kt
>
> - [HIGH] Line 8: Hardcoded live payments API key in source (`sk-live-9f8a…`). It is a production secret committed to the repo, and it is also a public `val` on the class, so any caller can read it. — Rule: Hardcoded secrets
>   Suggestion: Remove the key from source and treat it as compromised, so revoke and rotate it. Have the backend issue short-lived tokens, or inject the key at build time from CI secrets or a secure config. Never ship a live secret in the client. Make any remaining reference `private`.
>
> Notes (not findings, since the diff shows no violation):
> - Line 11 uses `https://`, so transport is fine.
> - Line 16 is a comment only. `charge()` has no body, so nothing about logging, storage, or authorization can be assessed yet.
>
> Security findings: 1 HIGH, 0 MEDIUM
> Questions for the author:
> - `charge(amount, token)` is empty. When it is implemented, how will the caller be authorized to charge a rider's saved card, and how will `amount` and `token` be validated? Please confirm that the key and the token will not be logged.
> - Is `sk-live-` a real production key? If so, it is already exposed in git history and must be rotated even if it is removed from the file.

**Agent:**

Step 3 is done. `security-reviewer` found one `[HIGH]` issue: a hardcoded live API key on line 8. Step 4 is running now, with `architecture-reviewer` (Opus) checking the diff against the ADR guidelines.

### Step 4: `architecture-reviewer` subagent

**Tool call:** `Agent`, `{"subagent_type": "architecture-reviewer", "description": "Architecture review PR #3"}`. The prompt contains the same PR data and diff, plus the path to `docs/adr/`. The subagent used Glob and Grep to read the ADRs (5 tool calls).

**Subagent report (`architecture-reviewer`):**

> ### src/PaymentService.kt
>
> - [HIGH] Line 10: `PaymentService` builds its own Retrofit HTTP client (lines 10-13, base URL `https://api.rideapp.com/payments/`). By the PR's own description, `charge()` uses it to call the payments API. That makes `PaymentService` a second type doing HTTP outside the app's `ApiClient`. ADR-002 says `APIClient` is "the only type permitted to perform URL requests" and that higher layers must not build requests or call the HTTP stack themselves. — Guideline: Networking stack (ADR-002)
>   Suggestion: Delete the private `Retrofit` instance. Add the payments endpoint to the shared `ApiClient` as a `suspend` function, and reach it through a payments repository (for example a `PaymentRepository` interface with an implementation that wraps `ApiClient`), as ADR-003 says.
>
> Architecture findings: 1 HIGH, 0 LOW
> Questions for the author:
> 1. What layer is `PaymentService` meant to be in? Under ADR-003, only a repository may depend on `ApiClient`. If this class is the data-layer entry point for payments, should it be a `PaymentRepository` interface plus an implementation that wraps `ApiClient`? If not, which repository will own the payments endpoint and its model?
> 2. Line 15: `charge(amount: Int, token: String)` is a plain blocking function with an empty body. It returns no result and reports no errors. ADR-002 expects `async throws` request APIs (Kotlin: `suspend`). It isn't a callback API, so the LOW rule doesn't strictly apply. Will the finished version be a `suspend` function that returns or throws a result?
> 3. The Retrofit instance is built inside the class instead of being passed in through the constructor. The dependency-injection guideline (ADR-003) only covers ViewModels and singletons, so this isn't a finding. Will the class's dependencies be injected once it's reworked to sit on `ApiClient`?
>
> Outside architecture scope (for the security and style reviewers, not counted above):
> - Line 8 hardcodes a live secret key (`sk-live-9f8a…`) in source as a public `val`. This is a hardcoded-secrets issue for the security reviewer, and the key should be treated as exposed.
> - Line 8's `API_KEY` is an instance property named in `UPPER_SNAKE_CASE` rather than a constant in a `companion object` or at top level. That's a Kotlin naming issue for the style reviewer.
>
> Checked against docs/adr/ADR-002-networking-stack.md and docs/adr/ADR-003-repository-pattern.md. There are no Kotlin or Swift files in the local repo, so I couldn't check what the existing `ApiClient` looks like.

### Steps 5–7: Consolidate, present, ask

**Agent:**

Here's the review of [nebius-academy-templates/mobile-pr-review-practice#3](https://github.com/nebius-academy-templates/mobile-pr-review-practice/pull/3), "Add payment service". This is review pass 1 of 2.

All three subagents ran, one after another:

| Step | Subagent | Result |
|---|---|---|
| 2 | `style-reviewer` (Sonnet) | 1 LOW |
| 3 | `security-reviewer` (Sonnet) | 1 HIGH |
| 4 | `architecture-reviewer` (Opus) | 1 HIGH, checked against ADR-002 and ADR-003 |

---

### src/PaymentService.kt

- [HIGH] Line 8: a live payments API key (`sk-live-9f8a…`) is hardcoded in source, and it's a public `val`, so any caller can read it. — Rule: Hardcoded secrets
  Suggestion: Remove the key from the code and treat it as compromised: revoke and rotate it. Have the backend issue short-lived tokens, or inject the key from CI secrets or secure config. Never ship a live secret in the client.
- [HIGH] Line 10: `PaymentService` builds its own Retrofit HTTP client (lines 10–13). That makes it a second type sending HTTP requests outside the shared `ApiClient`. — Rule: Networking stack (ADR-002)
  Suggestion: Remove the private `Retrofit` instance. Add the payments endpoint to `ApiClient` as a `suspend` function, and call it through a `PaymentRepository` interface whose implementation wraps `ApiClient` (ADR-003).
- [LOW] Line 8: `API_KEY` is an instance property named in `UPPER_SNAKE_CASE`. Constants belong in a `companion object` or at the top level, and properties use `lowerCamelCase`. — Rule: Kotlin naming
  Suggestion: Rename it to `apiKey` or move it to `companion object { const val … }`. The HIGH finding above should remove the hardcoded value anyway.

## Summary

- Files reviewed: 1
- Findings: 2 HIGH, 0 MEDIUM, 1 LOW
- Top risk: The live payments key hardcoded on line 8. It's already in the PR's git history, so it has to be rotated even if this PR is never merged.
- Recommendation for the human reviewer: Don't merge until the key is removed and rotated and the networking goes through `ApiClient` and a repository. Questions for the author (not rule violations):
  - `charge(amount, token)` on line 15 has an empty body. When it's implemented, how will callers be authorized to charge a saved card, and how will `amount` and `token` be validated? Please make sure neither the key nor the token is logged.
  - Which layer does `PaymentService` belong to? Should it become a `PaymentRepository` interface plus an implementation that wraps `ApiClient`?
  - Line 15: `charge` is a plain blocking function that returns nothing. Will the finished version be a `suspend` function that returns a result or throws?
  - The file has no trailing newline (line 18).

---

How I combined the findings:
- **Two findings on line 8, kept separate.** The style and security findings are different problems: naming versus a secret in source. I kept both and listed the HIGH one first.
- **Out-of-scope notes dropped.** `architecture-reviewer` also mentioned the key and the naming as outside its scope. I didn't count those again.
- **Nothing added.** I didn't add any findings of my own.

Post this review as a comment on the PR? (yes / no)

*The agent stopped here. It made no GitHub write calls until the user answered.*

### Step 8: Post only on `yes`

**User:**

> yes

**Tool call:** `mcp__github__add_issue_comment`, `{"owner": "nebius-academy-templates", "repo": "mobile-pr-review-practice", "issue_number": 3, "body": "<the consolidated review above, secret masked>"}` returned this error:

```
failed to create comment: POST https://api.github.com/repos/nebius-academy-templates/mobile-pr-review-practice/issues/3/comments: 403 Resource not accessible by personal access token []
```

**Agent:**

The comment wasn't posted. GitHub returned a 403 error:

> Resource not accessible by personal access token

This is the same thing that happened with PR #1. The fine-grained token in the GitHub MCP has no write access to the `nebius-academy-templates` organization. That was the one posting attempt for your `yes`, so nothing is on the PR, and the review is only here in the chat.

The `gh` CLI is logged in as `ferdoznik` and posted to this repo before, so it would probably work. I haven't switched to it on my own, because it means posting outside the GitHub MCP, and last time you asked me to remove a comment posted that way.

*No retry and no fallback to another tool: one posting attempt per confirmation.*

---

## Results

| Check | Expected | Observed | Pass |
|-------|----------|----------|------|
| Subagents fire in sequence | `style-reviewer` → `security-reviewer` → `architecture-reviewer`, one at a time | Each started only after the previous one reported back, in that order | Yes |
| Each subagent gets only PR data | No subagent sees another's findings | Every prompt held only the PR metadata and diff | Yes |
| Subagents stay in their lane | Each reports only its own rule set | Style flagged naming; security flagged the secret; architecture flagged the HTTP client and pointed the other issues to the right reviewers | Yes |
| HIGH security finding in consolidated output | Hardcoded credential reported as `[HIGH]` with `Rule: Hardcoded secrets` | `[HIGH] Line 8 … — Rule: Hardcoded secrets`, top risk in Summary | Yes |
| Secret masked | The key never appears in full | Shown as `sk-live-9f8a…` in all output | Yes |
| Architecture finding cites an ADR | Finding names a guideline and ADR | `Networking stack (ADR-002)`, checked against `docs/adr/` | Yes |
| Agent pauses before posting | Asks "Post this review as a comment on the PR? (yes / no)" and makes no write call before the answer | Asked; no write call until `yes` | Yes |
| Posts at most once per confirmation | One attempt after `yes` | One `add_issue_comment` call; it failed with 403, and the agent didn't retry or switch tools without asking | Yes (not posted: token lacks org write access) |

<!--
TODO: /review-pr 4 run (architecture finding citing ADR-003) — not yet run.
-->
