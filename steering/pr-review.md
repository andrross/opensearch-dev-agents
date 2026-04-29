---
inclusion: always
---
# Reviewing GitHub PRs

When asked to review a PR, follow this process.

## Fetching PR Content

Use the `gh` CLI (already authenticated) to fetch PR details:

```bash
# PR metadata (title, body, author, labels, base branch, CI status)
gh pr view {PR_NUMBER} --repo {OWNER/REPO}

# Full diff
gh pr diff {PR_NUMBER} --repo {OWNER/REPO}

# List changed files only
gh pr diff {PR_NUMBER} --repo {OWNER/REPO} --name-only
```

For large diffs, start with `--name-only` to understand scope, then fetch the full diff or read specific files from the PR branch.

## Review Checklist

Evaluate each PR against:

1. **Problem & Approach** — Is the problem clearly articulated and worth solving? Are there simpler alternatives (existing mechanisms, configuration, smaller change)? Does this change belong where it's being made (e.g., server core vs. plugin/module)?
2. **Correctness** — Logic errors, off-by-one bugs, missing edge cases
2. **Backwards compatibility** — `Version.onOrAfter`/`Version.before` for format changes, API annotations (`@PublicApi`, `@InternalApi`, `@ExperimentalApi`, `@DeprecatedApi`), `>breaking` label if needed
3. **Thread safety** — Proper synchronization of shared mutable state, race conditions in async/listener code
4. **Error handling** — Exceptions handled appropriately, resources closed via try-with-resources
5. **Testing** — Adequate coverage, naming conventions (`*Tests` for unit, `*IT` for integration), `assertBusy`/`waitUntil` instead of `Thread.sleep`
6. **Commit message** — Title focused on user impact, not implementation details

## Review Output Format

- **Summary**: One-paragraph overview of what the PR does
- **Key concerns**: Numbered list ordered by severity (blockers first)
- **Suggestions**: Optional non-blocking improvements
- **Verdict**: APPROVE, REQUEST_CHANGES, or COMMENT with brief justification

## Checking CI Status

If CI failed, check the metrics cluster for whether failures are known flaky tests vs. regressions introduced by the PR.
