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
