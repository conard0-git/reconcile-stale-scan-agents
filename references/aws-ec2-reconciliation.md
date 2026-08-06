# AWS EC2 Reconciliation (reference implementation)

This file documents how to use AWS EC2 as the "source of truth for what is
alive" when reconciling a Nessus agent inventory. The core skill workflow is
provider-agnostic; this is the concrete AWS implementation.

## Building the fleet inventory

Enumerate instances with `ec2:DescribeInstances` across every account/profile and
region in scope. For each instance capture:

- **Instance state** — `running`, `stopped`, `stopping`, `pending`, etc.
  Collapse to two working sets: `running_ips` and `stopped_ips`, keyed on each
  instance's **primary private IP**. Anything not found in either set is treated
  as terminated/missing.
- **All private IPs** — also build an `ip -> instance` map that includes every
  secondary NIC private IP. Note the scope of this map: it feeds instance-metadata
  lookups, the SSM probe, and the online-agent reuse index — **not** the
  `running_ips` / `stopped_ips` classification, which is primary-IP-only. So an
  offline agent that reports only a *secondary* NIC IP of a running host will not
  match the running set and will be treated as terminated. Extend `running_ips` /
  `stopped_ips` to the full NIC set if that matters in your environment.
- **Instance ID** — needed for the optional SSM probe below.
- **Optional metadata for the coverage report** — instance Name and launch date,
  plus any ownership tags your org uses to group results (e.g. team/owner tags).

Iterate profiles × regions so cross-account fleets are fully covered. Use
whatever multi-account auth your environment provides (named profiles, SSO,
assumed roles); re-authenticate transparently when tokens expire rather than
failing the run.

## Matching agents to instances

1. Split Nessus agents into online and offline; only offline agents are
   candidates. Build the online-agent IP index over every online agent's full
   IP set (this is what reuse detection tests against).
2. For each offline agent, expand its comma-separated IP list into a set.
3. Evaluate in this precedence — **reuse first**, then running, then stopped:
   a. If any IP is now owned by an **online** agent → delete (IP reuse). This is
      checked *before* the running/stopped guards, so a decommissioned host whose
      IP has been reassigned is cleaned up even if that IP is currently running.
   b. Else if any IP is in the **running** set → skip (protected) unless opted in.
   c. Else if any IP is in the **stopped** set → skip unless opted in.
   d. Else → terminated/missing → eligible to delete.
4. See the classification table in `SKILL.md` (which lists the same outcomes).

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
  status` — emit the variant matching the instance's detected OS (a Windows
  document for Windows instances, a Linux shell document otherwise).
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
- (Optional) a Tenable Security Center target for findings cleanup.

Keep any site-specific values (account IDs, network CIDRs, repository IDs,
internal hostnames) out of the code and in the environment, so the skill stays
portable across organizations.
