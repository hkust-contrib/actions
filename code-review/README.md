# Code Review GitHub Action

Automated pull request code review powered by the [OpenCode](https://opencode.ai) agent.

The action analyzes the diff of a pull request against its base branch, identifies real defects (bugs, security issues, data-loss, concurrency problems, and doc-drift), and posts inline review comments directly on the PR via GitHub's review API.

```yml
- name: Automated Code Review
  uses: hkust-contrib/actions/code-review@latest
  with:
    model: anthropic/claude-sonnet-4-5
```

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `model` | yes | The model identifier passed to the OpenCode agent (e.g. `anthropic/claude-sonnet-4-5`) |

## What it reviews

The action only reports findings where **all** of the following hold:

- **Severity** is one of: `bug`, `security`, `data-loss`, `concurrency`, or `doc-drift`
- Confidence is ≥80% — a concrete failure scenario (inputs + steps) must be describable
- The cited line appears in the PR diff (added or modified lines only)
- At most **8 comments** per review (most severe kept if more are found)

It will **never** report style, naming, formatting, import order, comment wording, test naming, refactor suggestions, or subjective improvements.

**Skipped files:** `orm/**`, `*.pb.go`, `*_gen.go`, `go.sum`, `docs/swagger.*`, `vendor/**`

## How it works

1. **Build review context** — fetches the base branch and builds a whitelist of added/modified `path:line` pairs from the diff.
2. **Run OpenCode agent** — the agent reads the diff, reads relevant file context around each suspected defect, then writes a structured JSON review to `/tmp/opencode-review.json`. Before finishing it self-verifies the JSON is valid, all cited lines are in the diff whitelist, and the comment count is ≤8.
3. **Post inline review** — a GitHub script reads the JSON and calls `pulls.createReview` to post the summary and inline comments on the PR.

## Permissions

The workflow calling this action must have write access to pull requests:

```yml
permissions:
  pull-requests: write
```

## Example workflow

```yml
name: Code Review

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  pull-requests: write

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Automated Code Review
        uses: hkust-contrib/actions/code-review@latest
        with:
          model: anthropic/claude-sonnet-4-5
```
