# AWS EC2 Reconciliation (reference implementation)

This file documents how to use AWS EC2 as the "source of truth for what is
alive" when reconciling a Nessus agent inventory. The core skill workflow is
provider-agnostic; this is the concrete AWS implementation.

## Building the fleet inventory

Enumerate instances with `ec2:DescribeInstances` across every account/profile and
region in scope. For each instance capture:

- **Instance state** — `running`, `stopped`, `stopping`, `pending`, etc.
  Collapse to two working sets: `running_ips` and `stopped_ips`. Anything not
  found in either set is treated as terminated/missing.
- **All private IPs** — both the primary private IP and every secondary NIC
  private IP. Build an `ip -> instance` map from the full set so multi-NIC hosts
  resolve regardless of which IP the agent reports.
- **Instance ID** — needed for the optional SSM probe below.
- **Optional metadata for the coverage report** — instance Name and launch date,
  plus any ownership tags your org uses to group results (e.g. team/owner tags).

Iterate profiles × regions so cross-account fleets are fully covered. Use
whatever multi-account auth your environment provides (named profiles, SSO,
assumed roles); re-authenticate transparently when tokens expire rather than
failing the run.

## Matching agents to instances

1. Split Nessus agents into online and offline; only offline agents are
   candidates.
2. For each offline agent, expand its comma-separated IP list into a set.
3. Look each IP up in the running set, then the stopped set, then the
   online-agent IP set (for reuse detection).
4. Apply the classification table from `SKILL.md`.

## Instance-state → action mapping

| EC2 state of the agent's IP           | Default            | Override that changes it            |
| ------------------------------------- | ------------------ | ----------------------------------- |
| Running                               | Skip (protected)   | Opt-in "remove offline-but-running" |
| Stopped                               | Skip               | Opt-in "remove stopped"             |
| Terminated / not present in fleet     | Delete             | —                                   |
| IP now owned by an online agent       | Delete (IP reuse)  | —                                   |

The terminated/missing case is the safe default deletion: if none of an agent's
IPs appear anywhere in the live fleet, the host no longer exists.

## Optional: SSM probe for the ambiguous bucket

Offline-in-Nessus but running-in-EC2 is the ambiguous bucket worth
investigating before any opt-in deletion. For those hosts, resolve the instance
ID from the `ip -> instance` map and generate copy-pasteable AWS SSM commands to
check agent health on the box, for example:

- `aws ssm send-command` targeting the instance ID to run `nessuscli agent
  status` (provide both a Linux shell variant and a Windows variant).
- `aws ssm get-command-invocation` to read the result.

Treat live SSM as best-effort: attempt once, and if the caller's role lacks
`ssm:SendCommand`, stop retrying and just print the commands for a human to run.
Never require SSM permissions for a normal cleanup run.

## Required AWS permissions

- `ec2:DescribeInstances` (required) across the profiles/regions in scope.
- `ssm:SendCommand` + `ssm:GetCommandInvocation` (optional, probe only).

## Configuration surface (example)

Drive everything from environment variables — no config files, no baked-in
account IDs, secrets, or network ranges. A typical surface:

- One or more AWS profiles/credentials for the accounts to inventory.
- A list of regions to search.
- Nessus Manager URL + API keys.
- (Optional) a Tenable Security Center / Vulnerability Management target for
  findings cleanup.

Keep any site-specific values (account IDs, network CIDRs, repository IDs,
internal hostnames) out of the code and in the environment, so the skill stays
portable across organizations.
