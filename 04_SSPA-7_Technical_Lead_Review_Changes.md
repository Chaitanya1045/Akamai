# SSPA-7 — Technical Lead Review Changes Explained

## 1. Purpose

This document maps the technical-lead feedback to the corrected SSPA-7 design and explains why each group of changes was required.

---

## 2. PR Scope Blockers

### Review concern

The PR included temporary storage together with ServiceNow validation and CTASK work.

### Fix

The SSPA-7 PR must contain only SSPA-7 temporary-storage changes.

Revert from this PR:

```text
ServiceNow playbook
CTASK playbook
ServiceNow tasks
CTASK tasks
unrelated collection version change
unrelated dispatcher imports
```

### Why

Each user story should have an independently reviewable implementation and PR.

---

## 3. Role Dispatcher Blocker

### Review concern

The default role entry point imported report storage and unrelated story tasks.

### Fix

SSPA-7 is called explicitly:

```yaml
ansible.builtin.include_role:
  name: akamai_version_push
  tasks_from: report_storage
```

### Why

A storage-only test should not unexpectedly execute ServiceNow or CTASK behavior.

---

## 4. Missing Task References

### Review concern

The dispatcher referenced task files that were absent from the PR.

### Fix

Remove unrelated references from the SSPA-7 PR.

### Why

A PR must be self-consistent and should not depend on files that are not part of that change.

---

## 5. Temporary Storage Lifecycle

### Review concern

`/tmp` behavior across AAP jobs was not documented.

### Fix

Documentation now explicitly states:

```text
temporary files are available only within the active EE/job lifecycle
```

### Why

Separate AAP jobs may use different Execution Environments.

---

## 6. Downstream Usage

### Review concern

Later user stories might be implemented as separate workflow jobs and incorrectly assume they can read the same `/tmp` file.

### Fix

Direct access is supported only for downstream tasks running in the same AAP job/EE.

Cross-job use requires a future approved transfer/persistent mechanism.

---

## 7. Published Metadata

### Review concern

Publishing a calculated `report_path` could be interpreted as proof that the final report already exists.

### Fix

SSPA-7 publishes truthful state metadata:

```yaml
configuration_validated: true
storage_verified: true/false
workspace_exists: true/false
report_exists: false
```

### Why

SSPA-7 validates storage; it does not generate the final report.

---

## 8. Planned vs Actual Report Path

### Review concern

`report_path` naming was ambiguous.

### Fix

SSPA-7 publishes the workspace path only.

The later report-generation story owns the actual final report path.

---

## 9. Logging Actor

### Review concern

The actor could resolve to the AAP launcher or generic automation.

### Fix

Use a fixed module/component identity for actor, for example:

```text
report_storage
```

The same module identity is used for success and error actor fields.

---

## 10. Triggered-By Identity

### Review concern

Human launcher and automation component were not clearly separated.

### Fix

`common_logging_triggered_by` represents the AAP launcher.

`common_logging_actor` represents the report-storage component.

---

## 11. Environment Fallback

### Review concern

Missing/invalid environment silently became `local`.

### Fix

For deployed AAP execution require:

```text
dev
rnd
qa
prod
```

Allow `test/local` only for explicit standalone/local test mode.

---

## 12. Protected Environment Source

### Review concern

Environment protection was not established.

### Fix

Environment must come from protected inventory or fixed Job Template configuration.

It should not be exposed through survey/Prompt on Launch.

---

## 13. Request ID Fallback

### Review concern

A missing shared request ID could become a generated `REPORT-<timestamp>` value even in integrated execution.

### Fix

Integrated execution requires the shared workflow request ID.

Generated fallback is limited to explicit standalone/local tests.

---

## 14. Execution ID

### Review concern

Execution ID could be independently generated.

### Fix

Use shared execution context, normally:

```text
awx_job_id
```

Fallback only for explicit local tests.

---

## 15. Logging Context Reuse

### Review concern

Logging context values were resolved through multiple independent expressions.

### Fix

Resolve context once and reuse it across all events and published metadata.

---

## 16. Dedicated Temporary Directory

### Review concern

Artifacts were placed directly under shared `/tmp`.

### Fix

Use an isolated execution workspace:

```text
/tmp/akamai-reports/<execution_id>/
```

---

## 17. Directory Permissions

### Review concern

A dedicated directory mode was not defined.

### Fix

Use:

```text
0700 private
0750 approved group access
```

---

## 18. Path Traversal

### Review concern

Path components were not fully protected against traversal.

### Fix

Validate identifiers before using them in paths and ensure resolved paths remain under the approved temporary root.

Reject path separators and traversal values.

---

## 19. Identifier Safety

### Review concern

Request/execution values may become path components.

### Fix

Restrict to:

```text
A-Z a-z 0-9 . _ -
```

---

## 20. Access Group

### Review concern

Any syntactically valid group could potentially be supplied.

### Fix

The group must come from approved/protected configuration.

Do not expose arbitrary OS group input through surveys.

---

## 21. Group Verification

### Review concern

Existence was checked, but approved ownership behavior was not sufficiently documented.

### Fix

Document approved group ownership and verify resulting directory/file group ownership and modes.

---

## 22. Final Report Cleanup

### Review concern

Probe cleanup existed, but cleanup of the final temporary workspace/report was not assigned.

### Fix

Provide:

```text
report_storage_cleanup.yml
```

The orchestration layer invokes it after delivery/consumption.

---

## 23. Cleanup Timing

### Review concern

The report could be deleted before downstream tasks finish.

### Fix

Cleanup runs only after all same-job consumers finish, preferably from an `always` block.

---

## 24. Check Mode Semantics

### Review concern

Check Mode validated configuration but did not clearly distinguish this from actual I/O verification.

### Fix

Publish:

```yaml
configuration_validated: true
storage_verified: false
```

in Check Mode.

---

## 25. Normal Run Semantics

### Review concern

Successful configuration and successful real I/O were not clearly separated.

### Fix

Set:

```yaml
storage_verified: true
```

only after write/read/list all succeed.

---

## 26. Audit Rendering

### Review concern

Standalone storage validation emitted events but did not render an audit when events existed.

### Fix

The standalone validation path can invoke the shared:

```text
common_logging/tasks/render_audit.yml
```

from its `always` section where appropriate.

SSPA-7 does not implement a second audit framework.

---

## 27. Audit Filename Ownership

### Review concern

SSPA-7 naming could overlap SSPA-8/common logging audit naming.

### Fix

`common_logging` owns audit-report naming.

SSPA-7 owns storage only.

---

## 28. Final Deployment Report Ownership

### Review concern

The final deployment report belongs to a later story.

### Fix

SSPA-7 does not own final report name/content.

It provides only the temporary workspace.

---

## 29. Common Logging Success/Failure

### Review concern

Existing success and failure integration was acceptable.

### Fix

Keep:

```text
tasks_from: log_event
tasks_from: fail_with_error
```

unchanged as the shared mechanisms.

---

## 30. Error Classification

### Review concern

Classification needed explicit test coverage.

### Fix

Use:

```text
validation     → configuration/path/input problems
infrastructure → filesystem/group/I/O problems
```

and add tests for these mappings.

---

## 31. Task File Size

### Review concern

The original `report_storage.yml` was very large.

### Fix

Split by responsibility:

```text
report_storage_context.yml
report_storage_validate.yml
report_storage_provision.yml
report_storage_verify.yml
report_storage_publish.yml
report_storage_cleanup.yml
```

`report_storage.yml` becomes orchestration.

---

## 32. Encoding

### Review concern

Corrupted/non-UTF-8 characters appeared in files.

### Fix

Save all files as UTF-8 and remove corrupted characters.

---

## 33. Documentation

### Review concern

No complete temporary-storage integration contract existed.

### Fix

Separate docs now cover:

```text
implementation
AAP QA testing
integration/lifecycle
lead-review change mapping
```

---

## 34. Automated Tests

### Review concern

No complete executable contract test existed.

### Fix

Add tests covering:

```text
normal filesystem contract
Check Mode contract
metadata
security/path behavior
common_logging integration
```

---

## 35. AAP Validation

### Review concern

Local Ansible tooling was unavailable during review.

### Fix

Run the corrected implementation in AAP QA and attach runtime output to the PR.

This remains a runtime validation activity and cannot be substituted by static YAML review alone.
