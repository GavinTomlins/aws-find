# aws-find — search every account in an AWS Organization

**Problem.** An AWS Organization with a dozen member accounts under IAM
Identity Center. Finding which account holds a resource (the EC2 instance
behind `app.example.com`, a bucket named `*backup*`, a security group, a prefix
list, a Lambda) means switching accounts in the console one by one.
`aws-find` does the lookup across all of them from the terminal and hands back
a one-click console deep link into the right account and role.

```
aws-find host   <dns-name>      which account/region hosts this hostname?
aws-find s3     <glob>          S3 buckets whose name matches
aws-find sg     <glob>          security groups by id, name, description or tag
aws-find pl     <glob>          managed prefix lists by id, name or tag
aws-find lambda <glob>          Lambda functions by name
aws-find roles                  list the permission sets you hold in each account
```

Globs are case-insensitive and support `*` and `?`. A pattern with no wildcard
is treated as `*pattern*`. Hyphen, underscore and space are optional
separators, so `digital-asset` also matches `DigitalAssetStack`. `aws-find-host <dns>` is kept as a shortcut for
`aws-find host <dns>`.

## Examples

```bash
aws-find host app.example.com
aws-find host --name payroll app.example.com             # also match tag values *payroll*
aws-find s3 backup                                       # any bucket containing "backup"
aws-find s3 'acme-*-logs'                                # anchored glob
aws-find s3 digital-asset --tags                         # also match bucket tags (CDK/CFN stack name, logical id)
aws-find sg 'sg-0a1b*'                                   # by group id prefix
aws-find sg payroll --regions "ap-southeast-2"           # by name/description/tag, one region
aws-find pl office                                       # prefix lists named *office*
aws-find lambda 'billing-*' --accounts pick              # fzf-select which accounts to scan
aws-find lambda thumbnail --json | jq .                  # machine-readable, no picker
aws-find host app.example.com --role ReadOnlyAccess --debug
```

`aws-find --help` prints the same list.

## How it works

```
 pattern / dns name
        │
        ▼
 aws sso list-accounts  ──▶  per account: sso get-role-credentials
                                └▶ per region (parallel, xargs -P): one scan function per kind
                                     host    ENI by IP, EIP, Lightsail, ELB→targets, tag-value, Route 53
                                     s3      list-buckets (+ get-bucket-location for the region)
                                     sg      describe-security-groups, matched client-side
                                     pl      describe-managed-prefix-lists, matched client-side
                                     lambda  list-functions, matched client-side
        │
        ▼
 summary lines + table  ▶  fzf: pick a hit  ▶  pick an action
                            open console │ print URL │ export creds │ ssm start-session (ec2)
```

All calls are `Describe`/`List`. The minimum permission set is `ReadOnlyAccess`
(or `ViewOnlyAccess` plus `sso:ListAccounts`, `sso:ListAccountRoles`,
`sso:GetRoleCredentials`). Accounts where the role cannot make a call are
reported by name after the scan rather than silently skipped.

### host: why match on ENI rather than describe-instances

`describe-network-interfaces` filtered on `association.public-ip` or
`addresses.private-ip-address` finds the interface that owns the IP whatever it
is attached to: an EC2 instance, an ALB/NLB node, a NAT gateway, RDS, or a
Lambda. When an instance is attached the tool follows it to the instance and
pulls the `Name` tag, state, type, and AZ. Unattached Elastic IPs and Lightsail
instances (which never appear as ENIs) are checked separately, and any tag
value containing the first DNS label (`app`) is reported as a name/tag match so
hosts behind CloudFront or a proxy still surface.

### host: why Route 53 is queried too

The account that owns the hosted zone is often not the account that runs the
workload. Showing both answers "where is the box" and "where do I change the
DNS".

### host: fast path via Control Tower's Config aggregator

If the organization was set up with Control Tower, an org-wide Config
aggregator named `aws-controltower-GuardrailsComplianceAggregator` exists in
the Audit account. One query answers for every account and region:

```bash
AWS_FIND_AGG_PROFILE=audit aws-find host --fast app.example.com
```

Needs a profile into the Audit account with `config:SelectAggregateResourceConfig`.
Falls through to the fan-out automatically if nothing comes back. Untested so far.

## Install

Dependencies: `aws` CLI v2, `jq`, `dig` (host only), `fzf` (optional, enables
the picker), `xargs`.

```bash
cp aws-find aws-find-host ~/bin/
```

## Flags and environment

| Option | Default | Meaning |
|---|---|---|
| `--regions "r1 r2"` / `AWS_FIND_REGIONS` | `ap-southeast-2 us-east-1` | Regions scanned per account |
| `--accounts pick` | scan all | fzf multi-select which accounts to scan |
| `--role NAME` / `AWS_FIND_ROLE` | most capable held | Permission set to assume. Default picks the first of AWSAdministratorAccess, AdministratorAccess, AWSPowerUserAccess, PowerUserAccess, AWSReadOnlyAccess, ReadOnlyAccess, ViewOnlyAccess you hold in each account. `aws-find roles` shows the choice per account. |
| `--parallel N` / `AWS_FIND_PARALLEL` | `8` | Concurrent account×region workers |
| `--json` | off | Print result rows as JSON lines to stdout, no picker |
| `--debug` | off | Print every captured AWS error and write a full xtrace file |
| `--tags` | off | s3 only: also match bucket tag values and show the CloudFormation stack name as the label. One extra call per bucket, so slower. |
| `--name PAT` / `--no-name` | first DNS label | host only: tag/name pattern |
| `--fast` + `AWS_FIND_AGG_PROFILE` | off | host only: use the Config aggregator |
| `AWS_FIND_SSO_SESSION` | the only one in `~/.aws/config` | `sso-session` block to use; required when the file has several |

The SSO token is read from `~/.aws/sso/cache`. If none is valid the tool runs
`aws sso login --sso-session <name>` and continues. Set up a session once with
`aws configure sso-session` if you have none.

## What you see

```
» Resolving app.example.com
»   Tag/name    : *app*
»   A records   : 203.0.113.10
»   203.0.113.10 is AWS EC2 in region ap-southeast-2
» Scanning 12 account(s) x regions [ap-southeast-2 us-east-1] with 8 workers
   ...
! Sandbox (as S3OnlyRole) ap-southeast-2: An error occurred (UnauthorizedOperation) ...

✔ app-web-01 (i-0123456789abcdef0) is in account Web Production [123456789012] region ap-southeast-2  (IP match)
  DNS record owned by account Shared Services [210987654321] (zone:example.com.)

ec2      Web Production   123456789012  ap-southeast-2  i-0123456789abcdef0  app-web-01  running t3.large 203.0.113.10  eni:association.public-ip
route53  Shared Services  210987654321  global          -                    A -> 203.0.113.10                          zone:example.com.
```

Selecting the ec2 row and "open console in browser" opens

```
https://<your-portal>.awsapps.com/start/#/console?account_id=123456789012&role_name=<role>&destination=<ec2 InstanceDetails page>
```

Every kind has its own console destination: bucket objects tab, security group
detail, prefix list detail, Lambda function page.

## Cost and runtime

Fan-out is 12 accounts × 2 regions × 1–6 read-only calls, so under ~150 calls
and 10–25 seconds with 8 workers. S3 and Route 53 are global and are only
queried once per account.

## Extending it

- **Another kind.** Add a `scan_<kind>` function that emits rows with the
  common keys (`kind acct name region id label detail via`), register it in
  the `__scan` dispatcher and the kind list, and add a console `DEST`.
- **Resource Explorer.** If an org-level Resource Explorer view exists,
  `aws resource-explorer-2 search --query-string "k2"` gives tag/name hits
  across accounts in one call. It cannot search by IP.
- **Richer TUI.** The worker output is JSON lines, so a Python `textual` front
  end can stream rows into a live table via `subprocess` without touching the
  scan logic.
