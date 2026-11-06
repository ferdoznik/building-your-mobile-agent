---
name: review-pr
description: "Run the full PR review on a single command. Triggered explicitly with /review-pr <PR URL or owner/repo#number>."
disable-model-invocation: true
---

Follow these steps exactly as written, in order. Do not skip, merge, or reorder steps. The PR to
review is `$ARGUMENTS`. If it is empty or doesn't identify one PR, ask for it and stop.

## Steps

1. **Fetch the PR via the GitHub MCP server.** Call `pull_request_read` with `get` for the title,
   description, and head commit, `get_files` for the changed files, and `get_diff` for the diff. If
   the diff is missing, truncated, or too large to review reliably, say so and stop.
2. **Apply `swift-conventions` to Swift files.** If any changed file ends in `.swift`, load the
   `swift-conventions` skill and check only the added or changed lines in those files.
3. **Apply `kotlin-conventions` to Kotlin files.** If any changed file ends in `.kt`, load the
   `kotlin-conventions` skill and check only the added or changed lines in those files.
4. **Flag security issues.** Check every changed line, on both platforms, against the Security
   rules in `CLAUDE.md`: hardcoded secrets, insecure transport, insecure storage, missing input
   validation, and unsafe API usage. Mask any secret you quote.
5. **Compile findings into the output format** from `CLAUDE.md`: one `### <path>` heading per file
   in diff order, each finding as `[SEVERITY] Line <n>: … — Rule: …` with a `Suggestion:`,
   `No issues found.` for clean files, and the closing `## Summary`. Do not invent findings.
6. **Present the review** in full in the chat.
7. **Ask for confirmation before posting:** "Post this review as a comment on the PR? (yes / no)"
8. **Post only on `yes` or `post`.** On any other answer, or no answer, do not post. Post at most
   once per confirmation, as a comment. Never approve, request changes, merge, or edit the PR.
