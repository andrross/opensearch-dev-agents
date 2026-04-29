---
inclusion: always
---
# Jenkins CI Access

When making HTTP requests to `build.ci.opensearch.org`, authenticate using a GitHub PAT via basic auth:

```bash
curl -u "$(cat ~/.config/jenkins-cookies/github-credentials.txt)" "https://build.ci.opensearch.org/..."
```

The credentials file (`~/.config/jenkins-cookies/github-credentials.txt`) contains `username:github_pat` (a GitHub Personal Access Token with `read:org` scope).

If a request returns 401/403, the PAT may have expired or been revoked. Ask the user to regenerate it at https://github.com/settings/tokens with `read:org` scope.

## Build references

When the user refers to a build by number (e.g. "build 72838"), it means a gradle-check build. Use the API to efficiently get failure info:

1. Get build overview and test failure count:
   ```
   /job/gradle-check/{number}/api/json?tree=result,description,duration,timestamp,actions[failCount,skipCount,totalCount]
   ```

2. Get failed tests:
   ```
   /job/gradle-check/{number}/testReport/api/json?tree=suites[cases[className,name,status,errorDetails,errorStackTrace]{status=FAILED,status=REGRESSION}]
   ```

3. If failCount is 0 but result is FAILURE, it's a build/infra failure. Search the console log:
   ```
   /job/gradle-check/{number}/consoleText
   ```
   Grep for "What went wrong" to find the root cause.
