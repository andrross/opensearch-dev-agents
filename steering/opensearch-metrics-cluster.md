---
inclusion: always
---
# OpenSearch Metrics Cluster

Public OpenSearch cluster at `metrics.opensearch.org` that indexes CI test failures and other project metrics.

This cluster logs failures for checks that run against pull requests. Some failures are caused by problems in the PR code, but many are due to flaky tests, so historical failure patterns are still very useful for identifying flakiness. Some checks also run against committed code (post-merge).

## Access

The cluster sits behind OpenSearch Dashboards. Direct OpenSearch API endpoints (e.g. `/_cat/indices`, `/_mapping`) are not exposed. Use the Dashboards console proxy instead:

```bash
curl -s -X POST "https://metrics.opensearch.org/_dashboards/api/console/proxy?path=<url-encoded-path>&method=<METHOD>" \
  -H "osd-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '<request-body>'
```

The path parameter must be URL-encoded (e.g. `gradle-check-*%2F_search`).

Access is read-only (`OpenSearchReadonlyUserAccess` role). Admin APIs like `_mapping` and `_cat/indices` return 403.

## Test Failure Index

Index pattern: `gradle-check-*` (monthly indices like `gradle-check-03-2026`)

Time field: `build_start_time` (epoch millis)

### Key Fields (all keyword type)

| Field | Description | Example |
|---|---|---|
| `build_number` | Jenkins gradle-check build number | `73223` |
| `test_class` | Short test class name | `MixedClusterClientYamlTestSuiteIT` |
| `test_name` | Fully qualified test name with params | `org.opensearch.backwards.MixedClusterClientYamlTestSuiteIT.test {p0=...}` |
| `test_status` | `FAILED` for test failures, `BUILD_SUMMARY` for build summary records (since Mar 5, 2026) | `FAILED` |
| `build_result` | Overall build result | `FAILURE` |
| `invoke_type` | What triggered the build | `Timer`, `Pull Request`, `Post Merge Action` |
| `git_reference` | Branch name or commit SHA | `main`, `e67f291a3ec...` |
| `pull_request` | PR number or `"null"` | `20526`, `null` |
| `pull_request_owner` | PR author or `"null"` | |
| `pull_request_title` | PR title or `"null"` | |
| `test_fail_count` | Total failures in the build | `8` |
| `test_passed_count` | Total passed in the build | `38229` |
| `test_skipped_count` | Total skipped in the build | `637` |
| `build_duration` | Build duration in millis | `3995033` |

### Query Tips

- Since all text fields are `keyword` type, use `term` for exact matches and `wildcard` for partial matches. Do not use `match` or `match_phrase`.
- Only failed tests are indexed — there is no need to filter on `test_status`. However, since Mar 5, 2026, `BUILD_SUMMARY` records are also written; exclude them with a `term` filter on `test_status` if needed.
- Use `build_number` as a numeric term to find all failures in a specific build.
- Use `wildcard` on `test_name` to search for test name fragments (e.g. `*310_match_bool_prefix*`).
- Use `cardinality` aggregation on `build_number` to count unique builds affected by a failure.
- To select only tests that ran against committed code (not PR code), use this query filter:
  ```json
  {
    "bool": {
      "minimum_should_match": 1,
      "should": [
        {
          "bool": {
            "must": [
              { "term": { "invoke_type.keyword": "Timer" } },
              { "term": { "git_reference.keyword": "main" } }
            ]
          }
        },
        {
          "bool": {
            "must": [
              { "term": { "invoke_type.keyword": "Post Merge Action" } },
              { "prefix": { "pull_request_title.keyword": "push trigger main" } }
            ]
          }
        }
      ]
    }
  }
  ```

### Example: Historical flakiness check

```bash
curl -s -X POST "https://metrics.opensearch.org/_dashboards/api/console/proxy?path=gradle-check-*%2F_search&method=POST" \
  -H "osd-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "query": {
      "wildcard": {"test_name": "*SomeTestName*"}
    },
    "aggs": {
      "by_month": {
        "date_histogram": {
          "field": "build_start_time",
          "calendar_interval": "month"
        },
        "aggs": {
          "unique_builds": {"cardinality": {"field": "build_number"}}
        }
      }
    }
  }'
```
