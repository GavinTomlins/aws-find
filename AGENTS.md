# Working on aws-find

Guidance for coding agents (and humans) changing this repository.

## What this is

A single bash script, `aws-find`, that searches every account of an AWS
Organization through IAM Identity Center for a resource and offers a console
deep link. Read `README.md` first; the "How it works" and "Parallel workers" sections describe
the architecture you must preserve.

## Setup

- Requires `bash` 4+, `aws` CLI v2, `jq`, `dig`, `xargs`, and optionally
  `fzf`. `shellcheck` for linting.
- Live use needs an `[sso-session NAME]` block in `~/.aws/config`. With
  several blocks, set `AWS_FIND_SSO_SESSION=NAME`.
- Tests need no AWS access: `tests/run`. The README screenshot comes from
  the same harness via `assets/make-screenshot`.

## Before you claim a change works

1. `bash -n aws-find && shellcheck -S warning aws-find tests/run tests/fake-aws`
2. `tests/run` must report 0 failed. Add a `check` for any new behaviour.
3. Never run the real tool against an organization as part of a test; the
   fake CLI in `tests/fake-aws` is the harness. Extend it rather than
   bypassing it.

## Conventions that matter

- **Errexit discipline.** The main process runs `set -euo pipefail` with an
  ERR trap. Any command that may legitimately fail (fzf cancel, an optional
  API) must be guarded with `|| true` or an `if`. Workers (`__scan`) run with
  `set +e` on purpose so one account cannot abort the scan. Do not change that.
- **Worker contract.** A scan function emits JSON lines with the common keys
  `kind acct name region id label detail via`. Presentation, dedupe, `--json`
  and the console link all rely on those names. Kind-specific extras are fine.
- **Adding a kind.** Write `scan_<kind>`, register it in the `__scan`
  dispatcher and the kind list in the CLI parser, add a `DEST` console URL,
  document it in the help header and README, and add tests.
- **Global services** (S3, Route 53) must guard on
  `[[ "$W_REGION" == "$AWS_FIND_FIRST_REGION" ]]` so they run once per account.
- **Argument passing to xargs** is NUL-delimited. Keep it that way; account
  names contain spaces.
- **No org-specific data.** Examples use `example.com`, the documentation
  account IDs `123456789012` / `210987654321`, and generic names. Never commit
  real account IDs, portal URLs, hostnames, instance IDs, IPs or credentials.
  Credentials only ever live in the per-run temp directory that is deleted on
  exit.
- **Every AWS call goes through the `aws()` wrapper function** so that
  `--show-commands` can log it. Never call `command aws` directly from a scan
  function, and when adding a new call add a matching line to
  `explain_cmd` so the report can describe it. Anything secret must travel in
  the environment, not in arguments; the only argument-borne secret is
  `--access-token`, which the wrapper redacts.
- **Environment variables** are prefixed `AWS_FIND_`.
- **Help text is the spec.** The header comment of `aws-find` is what
  `--help` prints (lines 2–45). Keep the examples there and in the README in
  sync.

## Releasing

1. Bump `VERSION` at the top of `aws-find`.
2. Move the `[Unreleased]` entries in `CHANGELOG.md` under a new
   `[x.y.z] - YYYY-MM-DD` heading and update the compare links.
3. Regenerate the screenshot if the output changed: `assets/make-screenshot`
   (offline, uses the fake CLI; never run it against a real organization).
4. Commit, then `git tag v<x.y.z>`.

## Commit messages

```
<type>: <summary>

<what changed and why>

Changelog: <added|fixed|changed|deprecated|removed|security|performance|other>
```

Types: feat, fix, docs, test, refactor, perf, style, build, ci, changed,
removed, security, deprecated, other. `feat` pairs with `Changelog: added`.
No AI attribution trailers.
