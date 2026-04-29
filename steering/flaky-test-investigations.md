# Investigating flaky tests in OpenSearch

A flaky test is one whose result is not a function of the code under test
alone. The job of the investigator is to work out which other factors it
*is* a function of, and then either make the test depend only on the code
again or, failing that, label the test honestly so it isn't an ongoing
source of CI noise. These notes are specific to the OpenSearch repo and
to how its gradle-check CI works; the general discipline transfers but the
tooling references do not.

## Environmental context worth knowing up front

Around 2026-04-15, the gradle-check runners moved from `m5.8xlarge` to the
considerably faster `m7a.8xlarge`. Many tests that were latent-flaky on the
old hardware started reproducing more often on the new hardware. If you are
investigating a test whose failure rate climbed noticeably in or after
mid-April 2026, assume CPU-speed amplification before assuming a code
regression. Conversely, don't use "faster CPU" as a rubber stamp: confirm
it against the metrics-cluster history for that specific test.

## Before touching code

1. **Read the actual failure first.**
   - `curl -u "$(cat ~/.config/jenkins-cookies/github-credentials.txt)" \
     "https://build.ci.opensearch.org/job/gradle-check/<N>/testReport/api/json"`
     gives you the structured failing cases with error messages and stack
     traces. Start here. The console log is only necessary if `failCount=0`
     and the build itself failed, in which case grep `consoleText` for
     "What went wrong".
   - Note the exact assertion site and the seed (e.g.
     `[220320CBD3478DB4:F9CD0EA1C955AF00]`). These are the coordinates you
     will refer back to.

2. **Check whether the failure is new or chronic.**
   - The public metrics cluster indexes every gradle-check failure at
     `https://metrics.opensearch.org/_dashboards` (see the project
     documentation for the proxy path). Aggregate by month for the test
     name. A chronic test with a recent uptick and a brand-new failure are
     different problems.
   - Prefer `invoke_type:Timer` or `Post Merge Action` with
     `git_reference:main` for "against committed code" filtering; PR
     failures can be caused by the PR itself. The canonical filter is
     documented in the repo context.
   - Check the autocut issue tracker: search
     `repo:opensearch-project/OpenSearch <TestClass>` with the
     `autocut`/`flaky-test` labels. If `[AUTOCUT]` already exists, link
     your investigation to that issue.

3. **Read the test's commit history, not just the failure.**
   - `git log --oneline <test-file>` and `git log --oneline <subject-file>`
     for files the test exercises. If nothing has changed recently, treat
     "code regression" as a weaker hypothesis.
   - If the test was copied from Elasticsearch years ago (much of
     `internalClusterTest` was), the original ES issue tracker often has
     prior art. Search for the exact test method name in Elastic's GitHub.

4. **Compare the failure rate curve to known environmental changes.**
   - CI runner type changes (see the April 2026 note above).
   - JDK bumps (`.ci/java-versions.properties`).
   - Lucene upgrades.
   - Plugin or framework-wide test changes under `test/framework`.

## Reproducing

Reproduction is the single most important skill for flaky tests. Invest
heavily here before proposing a fix.

1. **Try the seed.**
   - `./gradlew :module:internalClusterTest --tests "ClassName.methodName" \
     -Dtests.seed=<SEED>`.
   - If it fails immediately, great — but don't assume the seed is
     deterministic.

2. **Confirm whether the seed is deterministic.**
   - Run the seed 10+ times in a row. If it passes most of them, the
     seed is *not* a reliable reproducer. `RandomizedRunner` seeds
     `Random` streams, but it does not (and cannot) control thread
     scheduling, network timing, GC pauses, disruption simulation
     timing, socket ordering, or JIT warmup. Anything that depends on
     wall-clock ordering will differ run to run.
   - Cluster-disruption tests (`AbstractDisruptionTestCase` subclasses,
     anything using `NetworkDisruption` or `ServiceDisruptionScheme`)
     are especially bad about this — the seed controls *which*
     disruption is chosen but not *when* packets arrive relative to
     application threads.
   - Write down whether the seed is deterministic. This will change
     what you can and cannot claim later.

3. **If the seed is not deterministic, try iteration.**
   - `-Dtests.iters=N` runs the test N times with fresh seeds. This is a
     blunt tool — it tells you a failure rate, not a root cause.
   - On a dev desktop that has more cores than the CI runner, flakes
     involving "more concurrent ops in a window" will reproduce *more*
     often than in CI. On a dev desktop that is slower, they will
     reproduce less. Keep this asymmetry in mind.

4. **If nothing reproduces, do not invent a root cause from logs alone.**
   This is the mistake that burned me on `ConcurrentSeqNoVersioningIT`:
   a single captured failure log is suggestive at best. Reproducing
   with targeted logging is what converts a hypothesis into evidence.

## Reading a concurrent-test log honestly

Concurrent-cluster tests produce a lot of log output and it is easy to
read more into a log line than it actually says. Some common pitfalls,
learned directly:

- **Coordinator vs. primary.** In an `IndexResponse` path, the coordinator
  is the node receiving the client request and re-routing it to the
  shard's primary. A log line that says "client X sent to node_t8" does
  NOT establish that node_t8 processed the write as primary. To know
  that, you need server-side TRACE logs or an explicit node identifier
  in the response path that you've instrumented.

- **`IndexResponse.primaryTerm` / `seqNo` is the *assigned* term/seqNo
  of the new write, not the *pre-write* doc state.** A successful CAS
  with `ifSeqNo=0 ifPrimaryTerm=1` returning `{primaryTerm=2, seqNo=12}`
  does mean the request's compare succeeded, but it does not directly
  prove "the accepting node's doc was at `{1,0}` immediately before."
  If you want the pre-write state, you have to log it server-side.

- **`ShardInfo.failed=0` does not mean "all replicas durably applied the
  write."** `ReplicationOperation.onFailure` intentionally filters
  exceptions that satisfy `TransportActions.isShardNotAvailableException`
  out of the returned failures array. You will often see `total=4,
  successful=3, failed=0`. That means "3 copies acknowledged, 1 copy was
  unavailable, no critical errors worth surfacing to the client." It
  does not, on its own, prove a silent divergence.

- **`total != successful + failed` is normal during allocation churn.**
  `total` includes copies the replication group knew about at the time
  the operation started, including copies that were promoted, demoted,
  or marked stale mid-flight.

- **"Primary-replica resync completed with N operations" is a strong
  signal.** A resync with N > 0 means a newly-promoted primary rolled
  back N operations that the old primary had applied locally. Search
  for this explicitly when investigating CAS or versioning tests.
  Absence of a resync line for a shard you are studying also means
  something — it probably didn't fail over, or the resync wasn't logged
  at INFO level.

- **Instrumentation perturbs timing.** Adding a synchronous
  `ClusterService.state()` lookup per request changed the interleaving
  enough in my investigation to hide the race entirely. Add only
  cheap, non-blocking logging — ideally things the test already has
  in hand (coordinator name, response fields). If you must read
  cluster state, do it async and accept that your repro rate will drop.

## Categorizing what you find

Once you have a reproducible (or at least repeatably-observable) failure,
classify the root cause. The category drives the fix.

1. **Test-logic bug** — the test races with itself, shares mutable state
   across threads, or relies on `Thread.sleep` for a condition.
   Fix: remove the race, use `assertBusy`/`waitUntil`, instrument with
   `CountDownLatch`. The repo's `AGENTS.md` already codifies this.

2. **Over-strict test invariant** — the test encodes a stronger
   guarantee than the system actually provides. Common in
   disruption-heavy tests translated from ES.
   Fix: relax the invariant, and *comment the relaxation clearly*,
   pointing at the spec of the weakened assumption. Do not "fix" by
   silently changing behavior.

3. **Environmental sensitivity (CPU speed, disk speed, GC)** — the test
   passes if operations complete within a window and fails if they don't.
   Fix: scale the window via `scaledRandomIntBetween` (the repo
   helper), increase timeouts where they actually matter, or reduce
   parallelism in the test itself. Do NOT add raw sleeps.

4. **Genuine production bug surfaced by the stress test.** The test
   *should* fail because the system it's exercising has a defect.
   Fix: report it as a bug, with the reproducer. Do not silence the test.

Getting category (2) and (4) confused is the single most consequential
mistake in this kind of work. If you can't distinguish them, say so out
loud in the investigation write-up and don't land a patch that silently
assumes one.

## Landing changes

The repo convention is documented in `AGENTS.md` and in the PR template.
Calling out the points most relevant to flaky-test work:

- **Never use `@Ignore`** to quiet a flaky test. The standard pattern is
  `@AwaitsFix(bugUrl = "<issue URL>")`. The annotation is inherited from
  `LuceneTestCase` via `OpenSearchTestCase`; no explicit import is
  required in most files.
- **Always link an issue.** Every `@AwaitsFix` should reference either a
  fresh GitHub issue you open, or the existing autocut tracking issue
  for the class (search `[AUTOCUT]` + the class name).
- **Muting is not a fix and the commit message should say so.** If you
  land an `@AwaitsFix`, the commit title should make it clear — e.g.
  "Mute flaky X test pending investigation", not "Fix X test".
- **Run the targeted gradle-check before pushing.**
  `./gradlew :server:precommit` catches formatting, license headers, and
  forbidden-api issues.
  `./gradlew :<module>:internalClusterTest --tests "<class>.<method>" \
   -Dtests.iters=N` for flaky tests; prefer N≥20 for anything you're
  claiming to have fixed.
- **Do not push to `opensearch-project/OpenSearch`.** Push to your fork
  and open a PR. If you don't have a fork yet, create one before you
  start.

## When to stop and ask

You will sometimes arrive at a state where:

- The seed isn't a reliable reproducer.
- The logs are suggestive but require inference to connect.
- Fixing the spec risks papering over a production bug.

When that happens, stop writing code and write down what you know, what
you don't, and what it would take to distinguish. Ask the domain owner
(usually a maintainer of the server area the test exercises) to review
the evidence before you push a fix. A wrong fix in this territory is
worse than no fix because it hides the problem for the next investigator.

## Sources of information

- `AGENTS.md` at the repo root: the authoritative build/test reference.
- `TESTING.md`: documents `tests.seed`, `tests.iters`, `tests.slow`,
  `tests.verbose`, and logger overrides for tests.
- `TestRunnerBuilder` / `OpenSearchTestCase` / `OpenSearchIntegTestCase`
  javadoc: base-class contracts for the test framework.
- Metrics dashboard: `https://metrics.opensearch.org/_dashboards` —
  monthly flake rates per test.
- Autocut tracker: search
  `repo:opensearch-project/OpenSearch label:autocut,flaky-test`.
- Elastic's issue tracker: still the source of truth for tests
  inherited from ES before the OpenSearch fork.

## A lesson from the `ConcurrentSeqNoVersioningIT` investigation

I spent a full session on that test. The useful output of the session
was a set of things to check next time; the useless output was two
speculative fixes that had to be retracted. The specific mistakes to
avoid repeating:

1. **Treating a single captured failure log as proof.** Without a
   deterministic reproducer, a suggestive log is one data point.
2. **Conflating coordinator identity with primary identity.**
3. **Conflating response-assigned term/seqNo with pre-request doc state.**
4. **Reading `successful < total` and `failed = 0` as evidence of silent
   divergence** without accounting for the deliberate filtering in
   `ReplicationOperation.onFailure`.
5. **Pushing a fix before the root cause was understood.** Twice.

These are the kinds of errors that don't show up in a passing test run
but will be visible to the next person who looks at your change. Be
willing to leave an investigation inconclusive.
