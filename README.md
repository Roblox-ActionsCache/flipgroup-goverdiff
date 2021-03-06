# goverdiff

[![Test](https://github.com/flipgroup/goverdiff/actions/workflows/test.yml/badge.svg)](https://github.com/flipgroup/goverdiff/actions/workflows/test.yml)

This tool will compare two cover profiles produced by `go test` and reports deltas back into a GitHub pull request.

Example GitHub Actions workflow to integrate:

```yaml
name: Coverage report
concurrency:
  cancel-in-progress: true
  group: cover-pr-${{ github.event.number }}

on:
  pull_request:

jobs:
  main:
    name: Coverage
    runs-on: ubuntu-latest
    if: github.actor != 'dependabot[bot]'
    steps:
      - name: Setup Golang
        uses: actions/setup-go@v2
        with:
          go-version: ~1.16
      - name: Checkout base
        uses: actions/checkout@v2
        with:
          ref: ${{ github.event.pull_request.base.ref }}
      - name: Setup Golang caches
        uses: actions/cache@v2
        with:
          path: |
            ~/.cache/go-build
            ~/go/pkg/mod
          key: ${{ runner.os }}-golang-${{ hashFiles('go.sum') }}
          restore-keys: |
            ${{ runner.os }}-golang-
      - name: Cache base test coverage data
        id: cache-base
        uses: actions/cache@v2
        with:
          path: base.profile
          key: coverage-${{ hashFiles('**/*.go') }}
      - name: Extract base test coverage
        if: steps.cache-base.outputs.cache-hit != 'true'
        run: go test -cover -coverprofile=base.profile ./...
      - name: Checkout head
        uses: actions/checkout@v2
        with:
          clean: false
      - name: Cache head test coverage data
        id: cache-head
        uses: actions/cache@v2
        with:
          path: head.profile
          key: coverage-${{ hashFiles('**/*.go') }}
      - name: Extract head test coverage
        if: steps.cache-head.outputs.cache-hit != 'true'
        run: go test -cover -coverprofile=head.profile ./...
      - name: Diff test coverage
        env:
          GITHUB_PULL_REQUEST_ID: ${{ github.event.number }}
          GITHUB_TOKEN: ${{ github.token }}
        run: |
          go install github.com/flipgroup/goverdiff@main && \
            goverdiff base.profile head.profile
```

**Note:** This only works for a `pull_request` workflow event type. A `push` event type doesn't provide a `base.ref` to check against.
