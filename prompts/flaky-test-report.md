Look at the builds in the past 24 hours in the opensearch metrics cluster. Find the tests that failed against committed code (timer or post-merge runs). Do not attempt to root cause the failures. Do not make any changes other than to write log files and create the report as a new GitHub issue.

For each failing test, look at the historical failure pattern across all build types (including PR builds) to understand the true flake rate. Use monthly aggregations with unique build counts.

For each distinct failing test (up to a maximum of 10), attempt to reproduce the failure locally in the current repository using the specific seed from the failing build. Extract the seed from the Jenkins test report error details or stack trace (look for "tests.seed=" or "-Dtests.seed="). Run the appropriate gradle test command with that seed, e.g.:
  ./gradlew <module>:test --tests "<fully.qualified.TestClass.testMethod>" -Dtests.seed=<SEED>
Run each test one at a time. Record whether the failure reproduced deterministically with that seed.

Produce a report with:
- Each distinct failing test, with a link to the recent build that failed (https://build.ci.opensearch.org/job/gradle-check/{build_number}/)
- Whether the failure reproduced locally with the original seed
- When the test first started failing
- Total unique builds affected historically
- A description of the failure pattern and whether it's stable, worsening, or improving
- A summary table sorted by number of builds affected

Then create the report as a GitHub issue in https://github.com/andrross/OpenSearch with the title "Flaky test report: committed-code failures on YYYY-MM-DD".
