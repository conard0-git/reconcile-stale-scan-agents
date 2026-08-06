# Findings Cleanup (optional, best-effort)

After agents are deleted, the hosts they represented may still have stale
findings in a Tenable Security Center (Tenable.sc) or Tenable Vulnerability
Management target. This optional step scrubs those findings by importing a scan
result that reflects the now-removed IPs.

This step is **always best-effort**. A failure here — unreachable target, bad
credentials, unconfigured repository — must log a warning and continue. It must
never abort, roll back, or block the agent deletions that already happened.

## Approach

1. **Collect the deleted IP set.** Gather every IP belonging to the hosts that
   were actually deleted (all NICs, not just the primary IP).

2. **Build an import artifact.** Inject those IPs into a `.nessus` scan template
   (the `TARGET` field) so the import represents the decommissioned hosts.

3. **Upload to the target repository.** Import the artifact into the appropriate
   repository so the target reconciles and ages out the stale findings. Retry
   transient failures with backoff.

4. **Route by your own rules, externally.** If you maintain multiple
   repositories and need to route hosts to the right one, drive that mapping from
   configuration/environment — never hardcode repository IDs, network ranges, or
   environment names into the skill.

## Guardrails

- Best-effort only: never make an upload failure fatal to the cleanup run.
- Require only the credentials the core cleanup needs (Nessus + cloud
  inventory). Treat findings-cleanup credentials as optional; when they are
  absent, skip the upload with a warning.
- Retry transient upload errors with backoff; give up gracefully after a
  reasonable number of attempts.

## Configuration surface (example)

- Tenable Security Center / Vulnerability Management URL + API keys (optional).
- One or more target repository IDs, supplied via environment/config.
- A `.nessus` template file used to construct the import.

Keep all site-specific routing (which IPs go to which repository) in
configuration, not in code.
