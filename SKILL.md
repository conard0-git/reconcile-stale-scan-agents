---
name: reconcile-stale-scan-agents
description: >-
  Reconcile Nessus Manager agent inventory against a live cloud fleet (AWS EC2)
  and safely remove stale, offline, or orphaned agents — deleting an agent only
  when its host is genuinely gone, never when the host is merely offline but
  still running. Optionally scrubs the removed hosts' findings from Tenable
  Security Center / Tenable Vulnerability Management. Use when asked to "clean up
  Nessus agents," "remove stale/offline agents," "reconcile agents against EC2,"
  "find orphaned Nessus agents," "remove agents for terminated instances," or to
  keep an agent inventory in sync with a cloud environment.
license: MIT
---

# Reconcile Stale Scan Agents

Keep a Nessus Manager agent inventory clean by reconciling it against a live
cloud fleet and removing only the agents whose underlying host no longer exists.
The guiding principle is **safety-first deletion**: an offline agent is not proof
that a host is gone, so the skill deletes an agent only after confirming, against
the cloud provider's inventory, that the host is truly terminated or missing.

This skill is provider-agnostic in design; the reference implementation uses
**AWS EC2** as the source of truth for what is alive. See
`references/aws-ec2-reconciliation.md` for the EC2 details and
`references/findings-cleanup.md` for the optional post-deletion findings scrub.

## When to use this

Use this workflow when a Nessus (or similar) agent inventory has drifted from
the real fleet — terminated instances still showing agents, offline records
piling up, IPs reused by new hosts — and you want to prune the dead records
without risking deletion of agents for hosts that are still in service.

## Inputs and prerequisites

- Read access to the Nessus Manager API (URL + API access/secret keys).
- Read access to the cloud fleet inventory (for AWS EC2:
  `ec2:DescribeInstances` across the relevant profiles/regions).
- Optional, for findings cleanup: credentials to a Tenable Security Center /
  Tenable Vulnerability Management target.
- Configuration is supplied via environment variables — no config files, no
  hardcoded secrets. See the README for the variable list.

## The workflow

1. **Build the cloud inventory.** Enumerate live instances across the configured
   accounts/regions and split them into `running` and `stopped` private-IP sets,
   retaining instance IDs and metadata. Map both primary and secondary NIC
   private IPs so multi-NIC hosts resolve correctly.

2. **Fetch the agent inventory.** Pull the full agent list from Nessus Manager.
   Only **offline** agents are cleanup candidates; online agents are never
   touched.

3. **Match each offline agent by IP.** Agents may carry multiple comma-separated
   IPs — match on the full set, not just the first IP.

4. **Classify and decide** (this is the safety core):

   | Host state (from cloud inventory)        | Default action                    |
   | ---------------------------------------- | --------------------------------- |
   | IP belongs to a **running** instance     | Skip (protected) — host is alive  |
   | IP belongs to a **stopped** instance     | Skip unless an explicit override  |
   | IP is **terminated / not in the fleet**  | Eligible to delete                |
   | IP now belongs to an **online** agent    | Delete stale record (IP reuse)    |

   Never weaken these guards without an explicit instruction from the user.

5. **Dry run by default.** Produce a report of what *would* be deleted and save
   it to a resumable file. Delete nothing until the user explicitly confirms
   (e.g. a `--confirm-delete` flag or an equivalent menu choice). Support
   resuming a delete from a saved dry-run file so the cloud/Nessus data does not
   have to be re-pulled.

6. **Coverage guardrail (report only).** On every run, surface running instances
   that have **no** agent record at all. These are coverage gaps to investigate —
   they are alerted, never deleted.

7. **Optional findings cleanup.** For hosts that were deleted, optionally push
   their IPs to a Tenable Security Center / Tenable Vulnerability Management
   target to scrub stale findings. Treat this as **best-effort** — a findings
   upload failure must never abort or roll back the agent deletions. See
   `references/findings-cleanup.md`.

## Safety rules (do not violate)

- Dry run is the default; nothing is deleted without explicit confirmation.
- An offline agent whose host is still running is **kept**, always, unless the
  user explicitly opts in to removing it.
- Findings-cleanup uploads are best-effort and never fatal to the core cleanup.
- Preserve multi-NIC handling — match and clean up on every IP a host owns.
- The "skip unless overridden" cases are unlocked only by explicit opt-in
  flags (e.g. `--remove-offline-running` for the ambiguous bucket,
  `--remove-stopped` for stopped instances). These flags are the only way to
  weaken the safety guards and must never be defaults.

## Output

- A dry-run report (JSON) of proposed deletions, resumable for a later confirmed
  run.
- A per-run CSV audit log of deleted agents with summary counts.
- A coverage report of running hosts missing an agent.

## Distinct entry points

The core workflow above is one entry point, but the underlying capabilities
support two others worth exposing to end users:

- **Standalone findings-scrub for a known IP list.** Skip the reconciliation
  entirely and go straight to the Security Center / VM scan-import step for a
  caller-supplied set of IPs (from a text file or command-line list). Useful
  after any bulk decommission where the operator already knows which hosts
  are gone and just wants their findings reconciled — no Nessus or cloud-fleet
  access required for that mode. See `references/findings-cleanup.md` for the
  import mechanics.
- **Coverage-trend tracking.** On each run, snapshot the "running hosts with
  no agent" list to a timestamped CSV and diff it against the prior snapshot,
  so the operator can see whether coverage is improving over time. This is a
  report-only feature; it never drives deletions.
