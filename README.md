# mutility/ci: Centralized developer workflows

A small collection of robust, secure, reusable GitHub Actions workflows designed to streamline common testing pipelines and report diagnostics.

## Repository Structure

📂 `.github/workflows/`
- `go-test.yaml` core validation, testing, and coverage-calculating pipeline (read-only)
- `go-test-publish.yaml` securely comment on PRs and save coverage bases (needs write)

### Go Validation (go-test.yaml)

Executes formatting validation, lints code and modules, runs tests and benchmarks, optionally invokes golangci-lint, and reports coverage.

This workflow is recommended to run with default privileges (read-only) to maintain security against supply-chain attacks.

#### Inputs

| Name | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `runs-on` | `string` | `ubuntu-latest` | Specify your preferred runner, implying an OS. |
| `go-version` | `string` | `oldstable` | Target a Go toolchain via [actions/setup-go](https://github.com/actions/setup-go)'s go-version, supporting stable, oldstable, SemVer ranges, and more. |
| `go-packages`| `string` | `./...` | Packages to validate, test, etc. |
| `show-coverage` | `string` | `package` | Metric granularity: `all`, `module`, `root`, `package`, `file`, or `off`. |
| `run-golangci-lint` | `boolean` | `false` | Run golangci-lint; also implied by golangci-lint-config. |
| `golangci-lint-config` | `string` | - | Path to a golangci-lint configuration file. |

#### Artifacts

When `show-coverage` is not `off`, this workflow will generate an artifact including raw coverage information, summarized coverage information, a coverage link (markdown) and a coverge report (markdown).

The link includes just the total metric, where the coverage percentage is linked to the full report. The full report includes a table showing coverage by file, package, root (a toplevel package within a module), module (in case of multi-module repositories), or all rolled up into one row. If this table has multiple rows, a total is included at the bottom.

Note that the long SHAs you see here will render as 7 or so character links to your actual code when it's part of the same repository.

> [📊 **82.96%**](https://github.com/mutility/sandbox-parquetry/actions/runs/37180859437#summary-111372975395) for **dependabot/...** (c286675e43ae427f5075e7666ebcdd1be55444f9) using go1.27.1 on linux amd64

vs

> Test coverage for **dependabot/...** (c286675e43ae427f5075e7666ebcdd1be55444f9): **82.96%**
> 
> | File | Coverage | Statements |
> |:--|--:|--:|
> | github.com/mutility/parquetry/filter.go | 79.31% | 23 of 29 |
> | github.com/mutility/parquetry/main.go | 89.67% | 295 of 329 |
> | github.com/mutility/parquetry/reshape.go | 82.93% | 68 of 82 |
> | github.com/mutility/parquetry/types.go | 58.14% | 50 of 86 |
> | github.com/mutility/parquetry/write_csv.go | 86.21% | 25 of 29 |
> | github.com/mutility/parquetry/write_json.go | 80.00% | 16 of 20 |
> | **Total** | 82.96% | 477 of 575 |
>
> <small>go1.27.1 on linux amd64</small>

Or, when coverage information is available for the base version, the table includes the change and prior metrics.  This example comes from a dependabot update that doesn't touch coverage, so you can see +0.00 for all changes.

> [📊 **82.96%** (+0.00%)](https://github.com/mutility/sandbox-parquetry/actions/runs/37180859437#summary-111372975395) for **main** (e675419ed133e35759321eb7c68ff7868bba7115) to **dependabot/...** (c286675e43ae427f5075e7666ebcdd1be55444f9) using go1.27.1 on linux amd64

vs

> Test coverage change for **main** (e675419ed133e35759321eb7c68ff7868bba7115) to **dependabot/...** (c286675e43ae427f5075e7666ebcdd1be55444f9): **82.96%** (+0.00%)
> 
> | File | Coverage | Statements | Change | (Covered) | (Statements) |
> |:--|--:|--:|--:|--:|--:|
> | github.com/mutility/parquetry/filter.go | 79.31% | 23 of 29 | +0.00 | (79.31%) | (23 of 29)|
> | github.com/mutility/parquetry/main.go | 89.67% | 295 of 329 | +0.00 | (89.67%) | (295 of 329)|
> | github.com/mutility/parquetry/reshape.go | 82.93% | 68 of 82 | +0.00 | (82.93%) | (68 of 82)|
> | github.com/mutility/parquetry/types.go | 58.14% | 50 of 86 | +0.00 | (58.14%) | (50 of 86)|
> | github.com/mutility/parquetry/write_csv.go | 86.21% | 25 of 29 | +0.00 | (86.21%) | (25 of 29)|
> | github.com/mutility/parquetry/write_json.go | 80.00% | 16 of 20 | +0.00 | (80.00%) | (16 of 20)|
> | **Total** | 82.96% | 477 of 575 | +0.00 | (82.96%) | (477 of 575)|
>
> <small>go1.27.1 on linux amd64</small>

### Go Results Publisher (go-test-publish.yaml)

A decoupled, secure reporting mechanism designed to report testing metrics in carefully handled PR commenting patterns.

This workflow requires `contents` and `pull-requests` write permissions, so it is written with only `actions/*` dependencies.
For paranoid levels of security, review the code and reference it by SHA, and/or fork it to where you control it.

#### Inputs

| Name | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `runs-on` | `string` | `ubuntu-latest` | Specify your preferred runner, implying an OS. |
| `go-version` | `string` | `oldstable` | Target a Go toolchain via [actions/setup-go](https://github.com/actions/setup-go)'s go-version, supporting stable, oldstable, SemVer ranges, and more. |
| `pr-comment`| `string` | `outdate` | PR timeline management strategy: `always` post, `edit` existing, `replace` existing, `outdate` (minimize) existing, or `off`. |
| `pr-comment-content`| `string` | `report` | Visual layout format: full `report` or summarize and `link`. |

## Usage examples

Create your own `.github/workflows/tests.yaml` similar to one of the following.

### Simple Go testing without coverage tracking

<details>
<summary>This turns off coverage reporting because it chooses not to store coverage information for differential reports.</summary>
As shown this doesn't report coverage at all. But with show-coverage set to a grouping level, the test coverage would be shown in the step summary. Other examples show how to post coverage into a PR comment and/or track coverage changes.

```yaml
name: CI
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    uses: mutility/ci/.github/workflows/go-test.yaml@main
    with:
      show-coverage: off # or specify a grouping level to show coverage in the step summary
```
</details>

### Extended linting with coverage changes reported in PR comments

<details>
<summary>This uses a custom golangci-lint config, and saves and reports coverage changes in PR comments.</summary>

```yaml
name: CI
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    uses: mutility/ci/.github/workflows/go-test.yaml@main
    with:
      show-coverage: pkg # choose a granularity that fits the size of your project
      go-version: stable # specify the Go version
      golangci-lint-config: .github/golangci.yml

  publish:
    permissions:
      contents: write      # to store coverage info of future merge-bases 
      pull-requests: write # to make and update PR comments
    needs: test
    if: always() && (needs.test.result == 'success')
    uses: mutility/ci/.github/workflows/go-test-publish.yaml@main
    with:
      go-version: stable # specify the Go version
      pr-comment: 'edit'
```
</details>

### Test matrix with linked coverage

<details>
<summary>This uses a test matrix to test multiple versions on multiple OSs, and links via PR comments to the report.</summary>
As shown this calculates and reports coverage for each combination, but you can create a non-uniform matrix to only report some.
Note that your publish matrix must match your test matrix if you want all variants to be published.

```yaml
name: CI
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        go: ['stable', 'oldstable']                                 # test multiple Go versions
        runner: ['ubuntu-latest', 'windows-latest', 'macos-latest'] # across multiple OS
    uses: mutility/ci/.github/workflows/go-test.yaml@main
    with:
      show-coverage: pkg # choose a granularity that fits the size of your project
      go-version: ${{ matrix.go }} # specify the Go version
      runs-on: ${{ matrix.runner }} # specify the target runner OS

  publish:
    permissions:
      contents: write      # to store coverage info of future merge-bases 
      pull-requests: write # to make and update PR comments
    strategy:
      fail-fast: false
      matrix:
        go: ['stable', 'oldstable']                                 # report multiple Go versions
        runner: ['ubuntu-latest', 'windows-latest', 'macos-latest'] # across multiple operating systems
    needs: test
    if: always() && (needs.test.result == 'success')
    uses: mutility/ci/.github/workflows/go-test-publish.yaml@main
    with:
      go-version: ${{ matrix.go }} # specify the Go version
      runs-on: ${{ matrix.runner }} # specify the target runner OS
      pr-comment: 'outdate'
      pr-comment-content: 'link'
```
</details>

## Future opportunities

If these or other adjacent needs are important to you, we are receptive to chat about the ideas, or review PRs implementing them.
We just haven't needed them yet.

- Merge Go coverage reports from multiple runs using different tags (GOCOVERDIR + go tool covdata merge)
- Merge Go coverage reports from runs on multiple Operating Systems
- Similar workflows for other languages
