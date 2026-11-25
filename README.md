# AIDP audit alerts

Installable Python wheel and sample notebook for configuration-driven AIDP audit
alerts delivered through Resend. Run the notebook from an externally scheduled
AIDP job; the schedule interval can change without rebuilding the wheel.

## Files

- [Wheel 0.4.1](wheels/aidp_audit_alerts-0.4.1-py3-none-any.whl)
- [Sample notebook](notebooks/AIDP_Critical_Alerts_Sample.ipynb)
- [SHA-256 checksum](SHA256SUMS)

## Setup

1. Download the wheel and install it on the monitoring cluster. Python 3.10+ is
   required. Restart the notebook kernel after replacing an older package.
2. Import the sample notebook. Configure recipient and Resend API key in Cell 1;
   a managed `RESEND_API_KEY` environment variable is also supported.
3. Configure the AIDP API endpoint/version, platform OCID, OCI region, existing
   API-key profile path, and persistent workspace checkpoint path in Cell 2.
   `/20240831` uses `dataLakes`; `/20260430` uses `aiDataPlatforms`.
4. Review the alert policy in Cell 3 against your deployment's audit records.
5. Run once and verify real audit visibility, checkpoint writes and email delivery.
   Run again to confirm accepted events are skipped. Verify the checkpoint folder
   survives cluster replacement before relying on scheduled execution.
6. Add the notebook to an AIDP job running at your desired interval, with one
   active run and independent failure/staleness notifications.

For notebook installation, use your environment's supported cluster-library
installation mechanism or a kernel-readable wheel path:

```python
%pip install "/path/to/aidp_audit_alerts-0.4.1-py3-none-any.whl"
```

The default Resend test sender only permits the account owner's recipient email.
Use a sender on your verified domain for team distribution. Avoid committing
filled-in notebooks, private keys, environment files or operational checkpoints.

## Default critical-event policy

| Event | Object type | Operations |
|---|---|---|
| Cluster deletion | COMPUTE | DELETE, TERMINATE |
| Cluster edits | COMPUTE | UPDATE, ATTACH, DETACH, RENAME |
| Job deletion | WORKFLOW | DELETE |
| Job activity | WORKFLOW | START, STOP, EXECUTE |

All starter rules are CRITICAL and match all recorded statuses, including failed
attempts. A FAILED deletion is not evidence that deletion succeeded. The workflow
mapping was confirmed in the originating deployment; validate labels and coverage
in your own deployment. Rules can be edited without a wheel change.

## Scheduling, checkpoints and duplicate handling

The first invocation reads the previous 60 minutes. Later invocations resume from
their saved read timestamp through a fixed current-run cutoff. Missed schedules
are caught up in bounded one-hour windows, up to the configured 72-hour gap.
Change `initial_lookback_minutes` for a different initial history; changing the
external schedule does not require a wheel rebuild.

Workspace file state contains the source cursor, pending outbox and per-recipient
acceptance receipts. Ten minutes of overlap catch ordinary late arrivals, while
saved event IDs prevent repeated email requests for accepted events. A no-event
run still saves its cursor. The queued outbox is persisted before sending, so
pending messages are retained even when the read cursor has already advanced.

Same-ID payload/status revisions are skipped after acceptance. Records without
IDs fall back to full-record fingerprints. Receipts are retained for 90 days.
Late events outside the overlap and no-ID revisions need operational review.
First initialization with a new state file considers its initial history afresh.
Preserve the state file and its `.lock` companion; do not reset them to fix errors.

The folder must be kernel-readable and persistent, with working locks and atomic
replacement. The runtime probes basic file operations and same-host lock exclusion;
that does not certify persistence across cluster replacement or distributed locks.
Use a single monitoring worker. Checkpoints are operational state, not a complete
audit-log archive. Original audit records remain in AIDP.

Known email rejections fail the run and remain pending. Uncertain or interrupted
sends stop for reconciliation against provider logs; the sample includes recovery
instructions. Provider idempotency expires after 24 hours. Acceptance is not inbox
delivery, and the implementation does not promise exactly-once delivery.

## Validation

Release 0.4.1 passed 304 offline tests, including hourly restart/catch-up,
event-ID deduplication, outbox persistence, delivery failures and notebook
execution with mocked source/HTTP. An isolated Python process imported the
built wheel and exercised the real audit reader with mocked HTTP.

The deployed audit API returned a row 1.714749 seconds beyond the requested
end time. The reader now keeps only rows inside the exact inclusive UTC window,
reads every continuation page, and warns with excluded/scanned counts. Every
returned row still counts against the event bound. Invalid timestamps, malformed
pages, failed pagination and exceeded bounds remain errors. The runner retains
its independent window validation. A newer excluded row can be collected in a
later invocation when it falls within the requested window.

When upgrading, install wheel 0.4.1, restart the notebook Python session, and
rerun the same code with the same configuration and checkpoint. No checkpoint
reset is required. Environment-specific deployment checks remain necessary;
offline tests cannot certify source completeness, workspace persistence, or
actual provider delivery. Compare known events against the AIDP Audit Logs UI.

## References

- [Oracle audit-search API](https://docs.oracle.com/en/cloud/paas/ai-data-platform/aiwap/op-aidataplatforms-aidataplatformid-actions-searchauditlogs-post.html)
- [Resend idempotency](https://resend.com/docs/dashboard/emails/idempotency-keys)
- [Resend test sender restrictions](https://resend.com/docs/knowledge-base/403-error-resend-dev-domain)
