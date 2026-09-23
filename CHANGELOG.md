# Changelog

All notable changes to aws-find are recorded here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and versions follow
[Semantic Versioning](https://semver.org).

## [Unreleased]

## [0.5.0] - 2026-09-24

### Added

- `--regions all` (or `'*'`, `--regions=all`, `AWS_FIND_REGIONS=all`) scans
  every region each account has enabled, discovered per account so opt-in
  regions are covered. Global services still run once per account.

- `--state STATE[,..]` for `ec2`, with `--running` and `--stopped` shorthands,
  applied server-side. A `television/aws-find-ec2-running.toml` channel lists
  every running instance in the organization.

### Changed

- Region default now honours the standard `AWS_REGION` / `AWS_DEFAULT_REGION`
  when `AWS_FIND_REGIONS` is unset. Precedence: `--regions` flag,
  `AWS_FIND_REGIONS`, `AWS_REGION` / `AWS_DEFAULT_REGION`, built-in default.
- `--region` is accepted as an alias of `--regions`, and `--regions=VALUE` form
  is supported.

## [0.4.0] - 2026-09-24

### Added

- `aws-find ec2 <glob>`: instances across the organization by Name tag,
  instance id, public or private IP, or any tag value.
- `--json` rows now include `url` (the Identity Center console deep link) and
  `role` (the permission set used), so other tools can act on a row directly.
- `television/`: one channel per kind for the television fuzzy finder, giving
  an org-wide browser with preview and open-in-console on Enter.

## [0.3.0] - 2026-09-24

### Added

- `aws-find dns <dns-name>`: finds every hosted zone across the organization
  that could serve a name, shows the record (exact or wildcard, including
  aliases) that answers it, and says whether each zone is authoritative by
  comparing its delegation set with the public NS records. Delegated child
  zones, parent zones that delegate them, private zones and non-Route 53
  providers are each called out.
- `host` and `dns` accept a URL and reduce it to its hostname.

## [0.2.0] - 2026-09-24

### Added

- `--show-commands`: after a search, prints every `aws` CLI command that ran,
  shell-quoted and with the SSO token redacted, grouped across accounts and
  regions with a count, and a one-line explanation of each. Works with
  `--json` and `roles`.

## [0.1.0] - 2026-09-24

### Added

- `aws-find host <dns-name>`: finds the account and region serving a hostname
  by resolving it, classifying the IP against Amazon's published ranges, then
  matching network interfaces, Elastic IPs, Lightsail instances, load
  balancer targets, tag values and Route 53 records across every account.
- `aws-find s3|sg|pl|lambda <glob>`: case-insensitive glob search for S3
  buckets (optionally by bucket tags with `--tags`), security groups, managed
  prefix lists and Lambda functions. Hyphen, underscore and space in a pattern
  are optional separators.
- `aws-find roles`: lists the permission sets held in each account and the
  one that will be assumed.
- Parallel fan-out over account × region with `xargs -P`, per-account
  credential isolation, and per-account error reporting that names the role
  used.
- fzf picker with actions: open an Identity Center console deep link, print
  it, print credential export lines, or start an SSM session.
- `--json` output, `--debug` tracing, `--version`.
- Offline test suite (`tests/run`) against a fake AWS CLI.

[Unreleased]: https://github.com/GavinTomlins/aws-find/compare/v0.5.0...HEAD
[0.5.0]: https://github.com/GavinTomlins/aws-find/compare/v0.4.0...v0.5.0
[0.4.0]: https://github.com/GavinTomlins/aws-find/compare/v0.3.0...v0.4.0
[0.3.0]: https://github.com/GavinTomlins/aws-find/compare/v0.2.0...v0.3.0
[0.2.0]: https://github.com/GavinTomlins/aws-find/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/GavinTomlins/aws-find/releases/tag/v0.1.0
