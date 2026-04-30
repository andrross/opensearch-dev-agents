Create a report of OpenSearch-project repository maintainer inactivity and publish it as a public GitHub gist on the user's account.

## Data sources

- **Maintainer inactivity**: `maintainer-inactivity-*` indices on the OpenSearch metrics cluster at `metrics.opensearch.org`. Use the Dashboards console proxy. Fields relevant here: `repository`, `github_login`, `inactive` (bool, stored as 0/1), `event_type` (text; use `event_type.keyword` for aggregations), `current_date` (keyword timestamp).
- **Repo metadata (stars, forks, open issues, archived flag, default branch)**: GitHub REST API, `GET /repos/opensearch-project/{repo}`.
- **Open PR count**: GitHub search API, `GET /search/issues?q=repo:opensearch-project/{repo}+type:pr+state:open`. Do not use `open_issues_count` from the repo endpoint for this — it is issues plus PRs.
- **Latest commit on default branch**: GitHub REST API, `GET /repos/opensearch-project/{repo}/commits?sha={default_branch}&per_page=1`, read `commit.committer.date` from the first element. Do not try to filter to merge commits only — OpenSearch repos mostly squash-merge, so merge commits (≥2 parents) do not exist for most repos.

## Method

1. Find the most recent `current_date` snapshot in `maintainer-inactivity-*`:
   ```
   size: 0, aggs: { max_date: { max: { field: "current_date" } } }
   ```
2. Within that snapshot, restrict to `event_type.keyword = "All"` (the rollup row — one document per maintainer per repo). The other `event_type` values partition the same maintainers across event categories and would double-count.
3. Aggregate by `repository.keyword`, computing total maintainers (`doc_count`) and inactive count (`sum` of the `inactive` field) per repo.
4. For each repo, fetch its GitHub metadata and PR count and latest-commit timestamp.
5. Exclude repos where `archived = true`. Do not include them at all.
6. Sort by `% inactive` descending. Tie-break by inactive count desc, then maintainer count desc, then repo name ascending.

## Report format

Markdown. One table, sorted by % inactive descending. Columns:

| Column | Notes |
|---|---|
| Rank | 1-indexed |
| Repo | Linked to `https://github.com/opensearch-project/{repo}` |
| Maintainers | Total maintainers from the inactivity snapshot |
| Inactive | Count where `inactive = true` |
| % Inactive | `inactive / maintainers * 100`, two decimal places, trailing `%` |
| Stars | `stargazers_count` |
| Forks | `forks_count` |
| Open Issues | `open_issues_count - open_prs` (PRs subtracted) |
| Open PRs | From search API `total_count` |
| Latest commit (default branch) | `YYYY-MM-DD` from committer date |

Include a summary section above the table stating:
- Snapshot `current_date` used
- Total repos in the snapshot
- Count and names of archived repos that were excluded
- Count remaining in the report

Note in the summary that "Latest commit" means the HEAD of the default branch, not a git merge commit, because squash-merge is the norm.

## Publishing

Create a **public** gist on the user's GitHub account using `gh gist create --public`. The `gh` CLI is authenticated with `gist` scope. Filename: `opensearch-maintainer-inactivity.md`. Return the gist URL.

If a previous run created a gist and it should be updated rather than replaced, use `gh gist edit <gist-id> <file>` instead.

## Caveats to keep in the report

- "Inactive" is defined by the ingestion pipeline that populates `maintainer-inactivity-*`. The threshold is on the order of months; do not claim a specific number unless it has been verified against the pipeline source.
- A repo at 100% inactive is not necessarily abandoned — it may simply have a stale MAINTAINERS.md. Flag but do not draw conclusions.
