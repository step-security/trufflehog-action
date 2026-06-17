[![StepSecurity Maintained Action](https://raw.githubusercontent.com/step-security/maintained-actions-assets/main/assets/maintained-action-banner.png)](https://docs.stepsecurity.io/actions/stepsecurity-maintained-actions)

<p align="center">
  <img alt="TruffleHog Logo" src="assets/pixel_pig.png" height="140" />
  <h2 align="center">TruffleHog</h2>
  <p align="center">Find leaked credentials in your source code.</p>
</p>

---

<div align="center">

[![License](https://img.shields.io/badge/license-AGPL--3.0-brightgreen)](/LICENSE)

</div>

---

## What is TruffleHog?

TruffleHog scans your Git history for leaked secrets and credentials. It classifies over 800 secret types and can verify whether found credentials are live by testing them against their respective APIs.

## :octocat: GitHub Action Usage

### General Usage

```yaml
on:
  push:
    branches:
      - main
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v6
        with:
          fetch-depth: 0
      - name: Secret Scanning
        uses: step-security/trufflehog-action@v6
        with:
          extra_args: --results=verified,unknown
```

In the example above, only code changes in the referenced commits are scanned. For scanning an entire branch, see the Advanced Usage section below.

### Inputs

| Input | Description | Required | Default |
|---|---|---|---|
| `path` | Repository path | No | `./` |
| `base` | Start scanning from here (usually main branch) | No | `""` |
| `head` | Scan commits until here (usually dev branch) | No | |
| `extra_args` | Extra args to pass to the trufflehog CLI | No | `""` |
| `version` | Scan with this trufflehog CLI version | No | `latest` |

### Shallow Cloning

If you're incorporating TruffleHog into a standalone workflow and aren't running any other CI/CD tooling alongside TruffleHog, then we recommend using [Shallow Cloning](https://git-scm.com/docs/git-clone#Documentation/git-clone.txt---depthltdepthgt) to speed up your workflow. Here's an example of how to do it:

```yaml
      - shell: bash
        run: |
          if [ "${{ github.event_name }}" == "push" ]; then
            echo "depth=$(($(jq length <<< '${{ toJson(github.event.commits) }}') + 2))" >> $GITHUB_ENV
            echo "branch=${{ github.ref_name }}" >> $GITHUB_ENV
          fi
          if [ "${{ github.event_name }}" == "pull_request" ]; then
            echo "depth=$((${{ github.event.pull_request.commits }}+2))" >> $GITHUB_ENV
            echo "branch=${{ github.event.pull_request.head.ref }}" >> $GITHUB_ENV
          fi
      - uses: actions/checkout@v3
        with:
          ref: ${{env.branch}}
          fetch-depth: ${{env.depth}}
      - uses: step-security/trufflehog-action@v3
        with:
          extra_args: --results=verified,unknown
```

Depending on the event type (push or PR), we calculate the number of commits present. Then we add 2, so that we can reference a base commit before our code changes. We pass that integer value to the `fetch-depth` flag in the checkout action in addition to the relevant branch. Now our checkout process should be much shorter.

### Advanced Usage

```yaml
- name: TruffleHog
  uses: step-security/trufflehog-action@v3
  with:
    # Repository path
    path:
    # Start scanning from here (usually main branch).
    base:
    # Scan commits until here (usually dev branch).
    head: # optional
    # Extra args to be passed to the trufflehog cli.
    extra_args: --log-level=2 --results=verified,unknown
```

If you'd like to specify specific `base` and `head` refs, you can use the `base` argument (`--since-commit` flag in TruffleHog CLI) and the `head` argument (`--branch` flag in the TruffleHog CLI). We only recommend using these arguments for very specific use cases, where the default behavior does not work.

#### Advanced Usage: Scan entire branch

```
- name: scan-push
        uses: step-security/trufflehog-action@v3
        with:
          base: ""
          head: ${{ github.ref_name }}
          extra_args: --results=verified,unknown
```

## TruffleHog GitLab CI

### Example Usage

```yaml
stages:
  - security

security-secrets:
  stage: security
  allow_failure: false
  image: alpine:latest
  variables:
    SCAN_PATH: "." # Set the relative path in the repo to scan
  before_script:
    - apk add --no-cache git curl jq
    - curl -sSfL https://raw.githubusercontent.com/step-security/trufflehog-action/main/scripts/install.sh | sh -s -- -b /usr/local/bin
  script:
    - trufflehog filesystem "$SCAN_PATH" --results=verified,unknown --fail --json | jq
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

In the example pipeline above, we're scanning for live secrets in all repository directories and files. This job runs only when the pipeline source is a merge request event, meaning it's triggered when a new merge request is created.
