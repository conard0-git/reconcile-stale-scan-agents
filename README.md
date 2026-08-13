# Reconcile Stale Scan Agents

> Reconcile Nessus agents against a live AWS EC2 fleet and safely remove only the agents whose host is truly gone.

A Claude Code / Agent Skills skill that reconciles a **Nessus Manager** agent
inventory against a live **cloud fleet** (AWS EC2 in the reference
implementation) and safely removes stale, offline, or orphaned agents — deleting
an agent only when its host is genuinely gone, never when the host is merely
offline but still running. It can optionally scrub the removed hosts' findings
from Tenable Security Center.

## Why

Agent inventories drift from reality: instances get terminated but their agents
linger, offline records pile up, and IPs get reused by new hosts. Blindly
deleting offline agents is dangerous — an agent can be offline while its host is
perfectly alive. This skill reconciles against the cloud provider's own
inventory so that only truly-dead hosts are pruned.

**Canonical use case: high-churn Kubernetes / EKS environments.** When EC2
worker nodes come up and go down daily, every terminated node leaves an offline
Nessus agent record behind. Left alone, these pile up fast — not just cluttering
the Nessus Manager inventory, but keeping **stale vulnerability findings alive
in Tenable for hosts that are no longer in production**. Reconciling against
the live EC2 fleet keeps the agent inventory (and the vuln data downstream of
it) honest, without the risk of deleting an agent whose host is merely offline
but still alive.

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
- Optional: a Tenable Security Center target for findings cleanup, and
  `ssm:SendCommand` / `ssm:GetCommandInvocation` for the optional agent-health
  probe.

## Configuration

All configuration is environment-variable driven — no config files, no
hardcoded secrets, account IDs, network ranges, or repository IDs. Supply your
cloud credentials/profiles, region list, Nessus API keys, and (optionally) a
Tenable target through the environment. Keep every site-specific value out of
the code so the skill stays portable.

## Example usage

Once installed and your environment is configured, invoke the skill from
Claude Code by name. **All destructive modes require an explicit flag** —
running the bare command produces a dry-run report only.

Dry run (default — proposes deletions to a resumable JSON report, deletes nothing):

```
/reconcile-stale-scan-agents
```

Confirm and delete from a saved dry-run report:

```
/reconcile-stale-scan-agents --confirm-delete from dry_run_2026-08-12.json
```

Also delete offline agents whose host is *stopped* (safety override — opt-in):

```
/reconcile-stale-scan-agents --remove-stopped
```

Investigate the "offline in Nessus but running in EC2" bucket and, after
review, allow deletion of that ambiguous set:

```
/reconcile-stale-scan-agents --remove-offline-running
```

Standalone Security Center findings scrub for a caller-supplied IP list
(no Nessus or cloud-fleet access required — skip the reconciliation entirely):

```
/reconcile-stale-scan-agents scrub findings for ~/decommissioned-ips.txt
```

Turn on the opt-in coverage-trend snapshot + CSV so subsequent runs can show
gaps improving/worsening:

```
/reconcile-stale-scan-agents write coverage-trend snapshot
```

## Known limitations

Scope boundaries an operator should understand before running this in
production:

- **AWS EC2 is the only shipped "alive" source of truth.** The workflow is
  provider-agnostic in design, but the reference implementation only enumerates
  AWS EC2. Hosts in other clouds (Azure, GCP, OCI) or on-prem hardware that
  register Nessus agents will be treated as terminated/missing and become
  deletion candidates unless you extend the inventory source. **Make the AWS
  region list exhaustive** — a region omitted from configuration is invisible
  to the reconciliation, so an instance in that region will look terminated and
  its agent will be eligible to delete.
- **Running/stopped classification is primary-private-IP only.** The
  `running_ips` / `stopped_ips` sets are keyed on each instance's primary
  private IP. An offline agent that reports *only* a secondary NIC IP of a
  running host will not match the running set and will be treated as
  terminated. If agents in your environment can register under secondary IPs,
  extend the classification to the full NIC set (see
  `references/aws-ec2-reconciliation.md`).
- **IP-reuse detection runs *before* the running/stopped guards.** If any of an
  offline agent's IPs is now owned by an *online* agent, that offline agent is
  deleted even if that IP currently maps to a running instance. This is
  intentional — it cleans up a decommissioned host whose IP was reassigned —
  but it means the "running host is always protected" guarantee does not apply
  in the IP-reuse case.
- **Only offline agents are cleanup candidates.** Online agents are never
  touched, even if their host is gone. A phantom "still-online, host-terminated"
  record is out of scope; the skill treats an online agent as proof of a live
  host.
- **Findings scrub is Tenable Security Center only — not Tenable.io /
  Vulnerability Management.** The cleanup mechanism is scan-import
  reconciliation against a Security Center repository. Tenable.io uses a
  different asset/findings model that this path does not target.
- **Findings scrub is reconciliation, not a hard asset delete.** Importing a
  fresh scan result for decommissioned IPs ages out or overwrites stale
  findings, but does not purge the asset record itself. Whether findings fully
  clear depends on the repository's aging and authoritative-scan settings.
  Treat the result as "findings reconciled," not "asset guaranteed removed."
- **Findings scrub is best-effort.** Unreachable target, bad credentials, or
  unconfigured repository logs a warning and continues — it never aborts or
  rolls back the agent deletions that already happened.
- **Safety overrides require explicit opt-in and are never defaults.** Deleting
  an offline agent whose host is *running* requires `--remove-offline-running`;
  deleting one whose host is *stopped* requires `--remove-stopped`. These are
  the only supported way to weaken the safety guards.
- **SSM live-probe is optional and permission-gated.** The ambiguous
  "offline-in-Nessus but running-in-EC2" bucket can be checked live via
  `ssm:SendCommand` / `ssm:GetCommandInvocation`, but if the caller's role
  lacks those permissions the skill just prints copy-pasteable SSM commands
  instead. SSM permissions are never required for a normal cleanup run.
- **Dry-run reports are a point-in-time snapshot.** Resuming from a saved
  dry-run report re-uses the classification captured at the time of the dry
  run (by design, so the cloud/Nessus data does not have to be re-pulled). If
  the fleet has meaningfully changed between dry-run and confirm, re-run the
  dry run to get an up-to-date decision set.

## License

MIT — see `LICENSE`.
