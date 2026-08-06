# Reconcile Stale Scan Agents

> Reconcile Nessus agents against a live AWS EC2 fleet and safely remove only the agents whose host is truly gone.

A Claude Code / Agent Skills skill that reconciles a **Nessus Manager** agent
inventory against a live **cloud fleet** (AWS EC2 in the reference
implementation) and safely removes stale, offline, or orphaned agents — deleting
an agent only when its host is genuinely gone, never when the host is merely
offline but still running. It can optionally scrub the removed hosts' findings
from Tenable Security Center / Tenable Vulnerability Management.

## Why

Agent inventories drift from reality: instances get terminated but their agents
linger, offline records pile up, and IPs get reused by new hosts. Blindly
deleting offline agents is dangerous — an agent can be offline while its host is
perfectly alive. This skill reconciles against the cloud provider's own
inventory so that only truly-dead hosts are pruned.

## What it does

1. Builds a live inventory of the cloud fleet (running vs. stopped, all NIC IPs).
2. Pulls the Nessus agent list and takes only offline agents as candidates.
3. Matches each candidate by IP and classifies its host as running, stopped,
   terminated/missing, or reused.
4. Proposes deletions in a **dry run by default**, and deletes only the
   truly-gone hosts after explicit confirmation.
5. Reports running hosts that have no agent at all (a coverage gap, never
   deleted).
6. Optionally scrubs the deleted hosts' findings from a Tenable target
   (best-effort).

See `SKILL.md` for the full workflow, `references/aws-ec2-reconciliation.md` for
the AWS EC2 details, and `references/findings-cleanup.md` for the optional
findings scrub.

## Safety model

| Host state of the agent's IP          | Default action                    |
| ------------------------------------- | --------------------------------- |
| Running instance                      | Skip — protected                  |
| Stopped instance                      | Skip unless explicitly overridden |
| Terminated / not in the fleet         | Eligible to delete                |
| IP now owned by an online agent       | Delete stale record (IP reuse)    |

Dry run is the default. Nothing is deleted without explicit confirmation, and
findings-cleanup uploads are best-effort — they never abort the core cleanup.

## Installation

Clone into your skills directory:

```bash
# Global (available in all projects)
git clone <this-repo-url> ~/.claude/skills/reconcile-stale-scan-agents

# Or per-project
git clone <this-repo-url> .claude/skills/reconcile-stale-scan-agents
```

The folder must contain `SKILL.md` at its root.

## Prerequisites

- Nessus Manager API access (URL + access/secret keys).
- Cloud fleet read access — for AWS EC2, `ec2:DescribeInstances` across the
  accounts/regions in scope.
- Optional: a Tenable Security Center / Vulnerability Management target for
  findings cleanup, and `ssm:SendCommand` / `ssm:GetCommandInvocation` for the
  optional agent-health probe.

## Configuration

All configuration is environment-variable driven — no config files, no
hardcoded secrets, account IDs, network ranges, or repository IDs. Supply your
cloud credentials/profiles, region list, Nessus API keys, and (optionally) a
Tenable target through the environment. Keep every site-specific value out of
the code so the skill stays portable.

## License

MIT — see `LICENSE`.
