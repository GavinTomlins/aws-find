# Changelog

All notable changes to aws-find are recorded here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and versions follow
[Semantic Versioning](https://semver.org).

## [Unreleased]

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

[Unreleased]: https://github.com/GavinTomlins/aws-find/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/GavinTomlins/aws-find/releases/tag/v0.1.0
