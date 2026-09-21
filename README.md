# brakeman-changed

Runs Brakeman and fails only on warnings **this branch introduced**.

Brakeman ships new checks in most releases. Any of them can flag code that has
been in the application for years, which turns every open pull request red for a
finding none of them caused — and the usual response is to stop reading the
check. This action keeps the signal on the branch that caused it.

Pair it with a scheduled workflow that scans the whole application — that is
what surfaces warnings nobody introduced. See [Companion workflow](#companion-workflow).

## Usage

```yaml
- uses: actions/checkout@v6
  with:
    fetch-depth: 0 # the base commit must be reachable

- uses: ruby/setup-ruby@v1
  with:
    bundler-cache: true

- uses: ten-thousand-hammers/brakeman-changed@v1
```

## How it works

1. Resolves the base commit: the merge base with the pull request's base branch,
   or `main`. On a push to the base branch itself it steps back to `HEAD~1`, so a
   merge that introduces a warning still fails.
2. Checks that commit out into a `git worktree` and scans it, producing a
   baseline. A worktree rather than a checkout, so the branch's installed bundle
   stays in place and **the same Brakeman build scans both trees** — scanning
   with two different versions is precisely what makes an upgrade look like a
   wave of new warnings.
3. Scans `HEAD` with `--compare <baseline>`, which reports `new`, `fixed` and
   `obsolete`.
4. Fails on `new`. `fixed` is reported as a courtesy.

If Brakeman cannot produce a trustworthy baseline the step fails with exit 2
rather than treating every pre-existing warning as new.

## Inputs

| Name | Default | Description |
| --- | --- | --- |
| `base-ref` | pull request base, else `main` | Branch to compare against. |
| `brakeman-command` | `bundle exec brakeman` | How to invoke Brakeman. |

## Outputs

| Name | Description |
| --- | --- |
| `new-count` | Warnings this branch introduced. |
| `fixed-count` | Warnings this branch removed. |

## Cost

Brakeman runs twice. On a small Rails application that is a few seconds; on a
large one, measure before adopting.

## Companion workflow

```yaml
name: Security Audit
on:
  schedule:
    - cron: "15 6 * * *"
  workflow_dispatch:
jobs:
  Audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - run: bundle exec brakeman --no-pager
```

## See also

[`bundler-audit-changed`](https://github.com/ten-thousand-hammers/bundler-audit-changed)
applies the same idea to gem advisories.
