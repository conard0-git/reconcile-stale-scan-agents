# Findings Cleanup (optional, best-effort)

After agents are deleted, the hosts they represented may still have stale
findings in a Tenable Security Center (Tenable.sc) target. This optional step
scrubs those findings by importing a scan result that reflects the now-removed
IPs. The reconciliation mechanism described here (importing a `.nessus` scan into
a repository) is **Security Center-specific** — Tenable Vulnerability Management
(Tenable.io) uses a different asset/findings model and is not what this path
targets.

This step is **always best-effort**. A failure here — unreachable target, bad
credentials, unconfigured repository — must log a warning and continue. It must
never abort, roll back, or block the agent deletions that already happened.

## How the "removal" actually works

Security Center does not expose a simple "delete this host and its findings"
call that fits this workflow. Instead, cleanup is done by **scan import
reconciliation**: you import a fresh scan result for the decommissioned IPs, and
Security Center reconciles the repository's cumulative view for those IPs from
the imported result — aging out or overwriting the stale findings.

This is a reconciliation, not a hard delete of the asset. Whether findings fully
clear depends on the repository's aging and authoritative-scan settings, so
treat the result as "findings reconciled," not "asset guaranteed removed." (If
you truly need to purge an asset record, that is a separate Security Center asset
API operation outside this cleanup path.)

## Approach

1. **Collect the deleted IP set.** Gather every IP belonging to the hosts that
   were actually deleted (all NICs, not just the primary IP).

2. **Build an import artifact.** Start from a `.nessus` scan template and inject
   those IPs into the `TARGET` field — locate the `<name>TARGET</name>` element
   and replace the following `<value>` with the comma-joined IP list, then write
   out a modified `.nessus` file.

3. **Import to the target repository.** Upload the artifact as a scan result
   into the appropriate repository. With pyTenable's Security Center client this
   is `sc.scan_instances.import_scan(file_obj, repo_id)`. Re-open the file handle
   per repository (the import consumes it), and retry transient failures with
   backoff.

4. **Route by your own rules, externally.** If you maintain multiple
   repositories and need to route hosts to the right one, drive that mapping from
   configuration/environment — never hardcode repository IDs, network ranges, or
   environment names into the skill.

## Reference

- Client: [pyTenable](https://pytenable.readthedocs.io/) `TenableSC`.
- Call: `sc.scan_instances.import_scan(fobj, repository_id)`.
- A repository id may be a single value or a list, so one import can fan out to
  several repositories if your environment requires it.

## Guardrails

- Best-effort only: never make an upload failure fatal to the cleanup run.
- Require only the credentials the core cleanup needs (Nessus + cloud
  inventory). Treat findings-cleanup credentials as optional; when they are
  absent, skip the upload with a warning.
- Retry transient upload errors with backoff; give up gracefully after a
  reasonable number of attempts.

## Configuration surface (example)

- Tenable Security Center URL + API keys (optional).
- One or more target repository IDs, supplied via environment/config.
- A `.nessus` template file used to construct the import.

Keep all site-specific routing (which IPs go to which repository) in
configuration, not in code.

## Standalone use

The findings-scrub step can also run on its own, decoupled from any agent
reconciliation. Given a caller-supplied IP list (text file or CLI argument),
build the same scan-import artifact from the `.nessus` template and upload
to the configured repositories. This is useful for post-decommission cleanup
where the operator already knows the dead IPs and just wants their findings
reconciled — no Nessus or cloud-fleet access required for that mode. The
same guardrails apply: env-var-driven configuration, best-effort uploads,
retry with backoff.
