# TestSprite GitHub Action

Run [TestSprite](https://www.testsprite.com) tests in GitHub Actions by wrapping
the `testsprite` CLI. The action installs the CLI, runs a project's tests to a
terminal verdict, writes a **JUnit** report, emits **`::error::` annotations** +
a **job summary** for failures, and — by default — **fails the job when tests
are skipped**, so a partial run is never reported green.

> **Status: v1.** Pin `@v1` (moving major tag) or `@v1.0.0`. Targets projects on
> the V3 execution path via `--project`. A precise "run a named test list" input
> lands once the CLI's test list surface ships.

## Prerequisites

- A TestSprite API key, stored as a **repository/organization secret** (e.g.
  `TESTSPRITE_API_KEY`). Never inline the key in workflow YAML.
- A project id (or set `TESTSPRITE_PROJECT_ID`).

## Quick start

```yaml
name: TestSprite
on:
  push:
    branches: [main]
  pull_request:

jobs:
  e2e:
    runs-on: ubuntu-latest
    # Fork PRs run without repository secrets, so the API key would be empty and
    # the check permanently red. Skip forks (a maintainer's push still runs it).
    if: ${{ github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository }}
    steps:
      - uses: TestSprite/testsprite-action@v1
        with:
          api-key: ${{ secrets.TESTSPRITE_API_KEY }}
          project: "your-project-id"
```

That installs the CLI, runs `testsprite test run --all --wait --report junit`,
uploads the JUnit report as an artifact, and fails the job if any test did not
pass — or if tests were skipped (see [`allow-partial`](#partial-runs)).

## Inputs

| Input             | Default              | Description                                                                 |
| ----------------- | -------------------- | --------------------------------------------------------------------------- |
| `api-key`         | — (required)         | TestSprite API key. Pass a secret.                                          |
| `project`         | `""`                 | Project id. If empty, `TESTSPRITE_PROJECT_ID` must be set.                  |
| `test-id`         | `""`                 | Run a single test by id (whole-project run otherwise). `project` ignored; `filter` must NOT be set (mutually exclusive — fails fast if both given); no JUnit (batch-only). |
| `filter`          | `""`                 | Only run tests whose name contains this substring. Full-project run only; mutually exclusive with `test-id`. |
| `target-url`      | `""`                 | Target URL override. **Single-test (`test-id`) runs only** — the CLI rejects it on a full-project run, so setting it without `test-id` fails fast. |
| `report-file`     | `testsprite-junit.xml` | JUnit XML path.                                                           |
| `cli-version`     | `latest`             | npm version/dist-tag of `@testsprite/testsprite-cli`.                       |
| `auto-heal`       | `true`               | Let the agent re-author a test whose stored code no longer runs, instead of replaying a script that can only fail (see below). Set `false` for a strict regression gate. |
| `allow-partial`   | `false`              | If false, fail the job when tests are skipped (see below).                  |
| `timeout`         | `600`                | Max seconds to wait for a terminal verdict (1-3600).                        |
| `endpoint-url`    | `""`                 | API base URL override.                                                      |
| `node-version`    | `lts/*`              | Node version used to install/run the CLI.                                   |
| `upload-artifact` | `true`               | Upload the JUnit report as a workflow artifact.                             |
| `artifact-name`   | `testsprite-junit`   | Name of the uploaded JUnit artifact. Give each a **unique** name when the action runs more than once (matrix / multiple projects) in one workflow — `upload-artifact@v4` fails on duplicate names. |

## Outputs

| Output       | Description                                                                                                                              |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `junit-file` | Path to the JUnit report (empty in single-test mode — JUnit is batch-only).                                                              |
| `passed`     | Number of tests that passed.                                                                                                             |
| `failed`     | Number of tests that ran to a **failing verdict** (`failed`/`blocked`) — not deferred, conflicted, or skipped.                           |
| `total`      | Total tests in the summary.                                                                                                              |
| `skipped`    | Tests skipped and not run (full-project runs, e.g. frontend tests on the V2 path; `0` in single-test mode).                             |
| `timed-out`  | Tests that ran but did **not** reach a verdict within `timeout` (status `timeout`). Counted separately from `failed`; under `allow-partial: true` these still fail the job. |
| `error-code` | The CLI's API-layer error code when the run failed **before** a verdict (e.g. `AUTH_INVALID`, `NOT_FOUND`, `UNAVAILABLE`); empty otherwise. |

## CI-native output

Because the action runs under `GITHUB_ACTIONS=true`, the CLI auto-emits:

- one **`::error::` annotation** per non-passed test (shown on the PR **Checks** tab), and
- a **results table** in the job's **Summary**.

No configuration needed. The JUnit report is also uploaded as an artifact
(disable with `upload-artifact: false`) and consumed by any JUnit-aware tooling.

## Partial runs

The run unit is `test run --all` (all tests in a project). On the current **V2**
execution path a project's **frontend tests may be reported as skipped rather
than run**, and skipped tests do **not** affect the CLI exit code — so a partial
batch could otherwise exit green.

This action guards against that: if any test is skipped, it emits a warning, and
(unless `allow-partial: true`) **fails the job** with an actionable `::error::`.
For full frontend coverage, use a project on the **V3** path (or the CLI's test
list run unit once it ships).

## Example: gate a PR on TestSprite

```yaml
name: TestSprite
on:
  pull_request:

jobs:
  e2e:
    runs-on: ubuntu-latest
    # Skip fork PRs (no repository secrets ⇒ empty api-key ⇒ permanently red).
    if: ${{ github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository }}
    steps:
      - id: testsprite
        uses: TestSprite/testsprite-action@v1
        with:
          api-key: ${{ secrets.TESTSPRITE_API_KEY }}
          project: "your-project-id"
          filter: "checkout"          # optional
          allow-partial: false         # default — fail on skipped tests
      - if: always()
        run: echo "passed=${{ steps.testsprite.outputs.passed }} / total=${{ steps.testsprite.outputs.total }}"
```

## Auto-heal

By default a test whose stored code no longer executes is **re-authored by the
agent** and run against the new code. This matters most in CI, because CI is where
the same tests run over and over: the first run generates code, and every run after
it would otherwise replay that code unchanged. Replaying stored code passes far
less often than the agent path, and a suite left on replay decays into red builds
for changes that broke nothing.

Healing re-saves the test's code, so the stored version is replaced by the agent's.

Two things it deliberately does not do:

- **It never re-authors code you wrote** — uploaded through `test code put`, passed
  to `test create --code-file`, or hand-edited in the portal. That code is replayed
  as written, and a failure stays a failure.
- **It does not make a verdict deterministic.** A healed pass can hide a real
  regression. For a strict regression gate — or a reproduction you are bisecting —
  set `auto-heal: false`, which replays byte-identically and leaves a drifted test
  red.

```yaml
- uses: TestSprite/testsprite-action@v1
  with:
    api-key: ${{ secrets.TESTSPRITE_API_KEY }}
    project: "your-project-id"
    auto-heal: false # strict replay; a drifted test stays red
```

Older CLI versions have no `--no-auto-heal` on `test run`. Combining `auto-heal:
false` with such a pinned `cli-version` **fails the step up front**, naming the
installed version — rather than letting the run proceed and heal, which is the
opposite of what was asked for. Raise `cli-version`, or drop the pin to use
`latest`. Leaving `auto-heal` at its default works on any version.

## How it works

A thin **composite action** (no bundled JS): `actions/setup-node` → `npm install
-g @testsprite/testsprite-cli` → `testsprite test run --all --wait --report junit
--summary-file --output json` → parse counts + skipped tests → `actions/upload-artifact`.
All the CI reporting (annotations, job summary, JUnit) is produced by the CLI
itself; this action just wires it into a workflow.
