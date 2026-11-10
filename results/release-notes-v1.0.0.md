# v1.0.0

## Features

- Switch between light and dark themes from the settings menu, and your choice is remembered across sessions.
- Set invoices to repeat on a fixed schedule, such as weekly or monthly, so they're generated automatically.
- Filter invoices by client name with the new search on the dashboard.

## Fixes

- Exported CSV invoice totals now match the total shown on screen instead of being off by a few cents.
- Creating an invoice with no line items no longer crashes the app; the form now blocks submission and shows a validation message.
- Invoice due dates no longer shift by a day for users outside UTC, because they're now calculated in your local time zone.

## Improvements

- The dashboard loads faster on first open, roughly halving load time on large accounts.
- Secondary buttons are easier to read, and all buttons now meet WCAG AA contrast guidelines.

---

## Notes on this draft

- **Source:** [nebius-academy-templates/release-notes-practice](https://github.com/nebius-academy-templates/release-notes-practice), generated with `/draft-release-notes v1.0.0` on 2026-11-10.
- **Pipeline:** PRs fetched via the GitHub MCP → labels applied with `pr-label-format` → descriptions cleaned by the `ticket-fetcher` subagent → sorted by the `categorizer` subagent → each entry drafted as one user-facing sentence.
- **Unmerged PRs:** the repository has no merged PRs and no earlier release, so this draft uses the open PRs into `main` (#1–8).
- **Excluded:** #9 and #10 (`chore`), #11 and #12 (`internal`), per `pr-label-format`.
- **Label mapping:** the repo labels `feature` (#1–3) and `improvement` (#7–8) don't exactly match the `pr-label-format` rules (`feat` / `type: feature`, `perf` / `type: improvement`); they were mapped by their evident meaning, and `categorizer` sorted them the same way. `fix` (#4–6) matches.
- **Linked issues** (#12, #18, #21, #25, #27, #30) were not fetched; entries use only the PR titles and descriptions.
- **Not published:** the release was approved but not published, because `ferdoznik` has no push access to the source repository.
