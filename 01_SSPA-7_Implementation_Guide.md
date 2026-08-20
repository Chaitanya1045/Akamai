# SSPA-7 Temporary Report Storage — Implementation Guide

## 1. Purpose

SSPA-7 provides a **temporary report-storage workspace** inside the active AAP Execution Environment (EE).

It does not generate the final deployment report, send email, or provide persistent retention.

The implementation is responsible for:

- resolving a safe execution context;
- creating a temporary workspace under `/tmp`;
- applying secure directory and file permissions;
- verifying actual write/read/list access in normal execution;
- supporting AAP Check Mode without performing filesystem writes;
- publishing storage metadata for downstream tasks;
- using `common_logging` for success/failure events;
- providing a cleanup task that downstream orchestration can call after the report has been consumed.

---

## 2. Scope

### In scope

```text
Temporary workspace creation
Filesystem accessibility validation
Directory/file permission validation
Write verification
Read verification
List verification
Safe path construction
AAP metadata publication
common_logging integration
Cleanup task
```

### Out of scope

```text
ServiceNow validation
CTASK updates
Akamai deployment
Final report-content generation
Email delivery
Persistent report retention
Cross-job /tmp persistence
```

---

## 3. Repository Structure

```text
roles/
└── akamai_version_push/
    ├── defaults/
    │   └── main.yml
    ├── vars/
    │   └── main.yml
    └── tasks/
        ├── report_storage.yml
        ├── report_storage_context.yml
        ├── report_storage_validate.yml
        ├── report_storage_provision.yml
        ├── report_storage_verify.yml
        ├── report_storage_publish.yml
        └── report_storage_cleanup.yml

playbooks/
└── akamai_version_push/
    └── report_storage.yml

docs/
└── ...

tests/
├── report_storage_contract.yml
└── report_storage_check_mode.yml
```

The SSPA-7 PR should contain only SSPA-7 temporary-storage work. ServiceNow, CTASK, unrelated collection changes, and unrelated role-dispatcher changes should remain in their own PRs.

---

## 4. Runtime Flow

The execution flow is:

```text
Start
  ↓
Resolve common context
  ↓
Validate configuration
  ↓
Build safe workspace path
  ↓
Provision workspace
  ↓
Check Mode?
  ├── Yes
  │    ↓
  │  No filesystem I/O verification
  │    ↓
  │  Publish configuration_validated=true
  │  Publish storage_verified=false
  │
  └── No
       ↓
     Create hidden probe
       ↓
     Verify WRITE
       ↓
     Verify file permissions
       ↓
     Verify READ/content
       ↓
     Verify LIST
       ↓
     Delete probe
       ↓
     Publish storage_verified=true
  ↓
Expose workspace metadata
  ↓
Downstream same-job tasks use workspace
  ↓
Downstream consumer finishes
  ↓
Call report_storage_cleanup.yml
```

---

## 5. Step 1 — Resolve Execution Context

File:

```text
roles/akamai_version_push/tasks/report_storage_context.yml
```

This task resolves the logging and path context once and reuses it across the remaining SSPA-7 tasks.

The context includes:

```text
request ID
execution ID
environment
module actor
AAP launcher / triggered-by identity
timestamp
safe execution identifier
```

### Why this is important

Previously, request ID, execution ID, actor, environment, and launcher values could be resolved independently in several places.

The updated implementation resolves the context once so all log events and storage metadata use the same values.

---

## 6. Step 2 — Environment Rules

For deployed AAP execution, the supported environments are:

```text
dev
rnd
qa
prod
```

`test` and `local` are intended only for explicit standalone/local testing.

A deployed job must not silently convert a missing or invalid environment into `local`.

The environment should come from protected inventory or fixed Job Template configuration and should not be exposed through Prompt on Launch or an unrestricted survey field.

---

## 7. Step 3 — Request and Execution IDs

Integrated execution should use the shared workflow/request identifiers.

Typical execution ID source:

```text
awx_job_id
```

Fallback IDs are only appropriate for explicit standalone/local testing.

The execution ID is also used to build a safe directory name, so it is normalized/restricted before becoming part of a filesystem path.

Allowed path-safe characters:

```text
A-Z
a-z
0-9
.
_
-
```

This prevents unsafe path components such as:

```text
../
/
\
```

from being used inside the temporary workspace.

---

## 8. Step 4 — Temporary Workspace

The underlying temporary root remains:

```text
/tmp
```

The implementation uses an isolated directory similar to:

```text
/tmp/akamai-reports/<execution_id>/
```

This provides separation between concurrent executions and avoids placing all artifacts directly under the shared `/tmp` root.

### Directory permissions

Private access:

```text
0700
```

Approved shared-group access:

```text
0750
```

### File permissions

Private report/probe file:

```text
0600
```

Approved group-readable report/probe file:

```text
0640
```

No permissions should be granted to `others`.

---

## 9. Step 5 — Optional Access Group

If shared access is enabled, the group must:

1. be configured from protected automation configuration;
2. not be supplied through an unrestricted survey;
3. exist inside the Execution Environment;
4. become the actual group owner of the created directory/file;
5. be validated after creation.

If no approved group is configured, execution-user-only access is used.

---

## 10. Step 6 — Validate Configuration

File:

```text
roles/akamai_version_push/tasks/report_storage_validate.yml
```

Validation occurs before filesystem changes.

The task verifies:

- temporary root is the approved root;
- execution/path identifiers are safe;
- requested permission modes are valid;
- no access is granted to `others`;
- optional group configuration is valid;
- environment is allowed for the current execution type;
- the final resolved path remains below the approved temporary root.

Failures are sent through:

```yaml
ansible.builtin.include_role:
  name: common_logging
  tasks_from: fail_with_error
```

Configuration failures use the `validation` category.

---

## 11. Step 7 — Provision Workspace

File:

```text
roles/akamai_version_push/tasks/report_storage_provision.yml
```

In a normal run, the task creates the isolated directory.

Example:

```text
/tmp/akamai-reports/123456/
```

The implementation then validates the resulting ownership and mode.

Provisioning/filesystem failures are classified as infrastructure failures.

---

## 12. Step 8 — Verify Actual I/O

File:

```text
roles/akamai_version_push/tasks/report_storage_verify.yml
```

In a normal run, SSPA-7 performs real I/O verification.

### 12.1 Write

A unique hidden probe is created.

Example:

```text
/tmp/akamai-reports/123456/.akamai_report_probe_123456_20260820T140000Z.tmp
```

The task confirms:

- the file exists;
- it is a regular file;
- the mode is correct;
- group ownership is correct when group access is enabled.

### 12.2 Read

The probe is read back using `slurp`.

The decoded content must match what was written.

### 12.3 List

The probe is searched using `ansible.builtin.find`.

Because the probe starts with `.`, the task must use:

```yaml
hidden: true
```

Without `hidden: true`, write/read can succeed while the list test incorrectly returns zero matches.

### 12.4 Probe cleanup

The verification probe is removed after verification.

The probe is not the final report.

---

## 13. Step 9 — Check Mode Semantics

AAP Check Mode must not perform actual write/read/list verification.

Check Mode validates configuration and planned storage behavior only.

Published metadata should clearly distinguish this state:

```yaml
configuration_validated: true
storage_verified: false
```

This avoids claiming that filesystem I/O succeeded when no filesystem I/O was performed.

---

## 14. Step 10 — Publish Storage Metadata

File:

```text
roles/akamai_version_push/tasks/report_storage_publish.yml
```

SSPA-7 publishes metadata for downstream same-job consumers.

The important distinction is:

```text
workspace/path planned or provisioned
≠
final report exists
```

Recommended published fields include:

```yaml
report_storage:
  workspace_root: /tmp/akamai-reports
  workspace_path: /tmp/akamai-reports/<execution_id>
  configuration_validated: true
  storage_verified: true
  workspace_exists: true
  report_exists: false
  ephemeral: true
```

In Check Mode:

```yaml
configuration_validated: true
storage_verified: false
workspace_exists: false
report_exists: false
```

`report_exists` becomes `true` only after a later report-generation task actually creates the final report.

---

## 15. Report Naming Ownership

SSPA-7 does not own the final deployment-report filename.

Its responsibility is storage only.

The final report-generation story owns:

```text
final report filename
final report content
actual report path
```

`common_logging` owns its own audit-report naming.

This avoids two different stories defining overlapping audit/report naming rules.

---

## 16. Same-Job Lifecycle Limitation

The workspace exists inside the current AAP Execution Environment.

This means:

```text
same AAP job / same EE
    → downstream task can read it

different AAP job / new EE
    → /tmp file is not guaranteed to exist
```

Therefore a later workflow node running as a separate AAP job cannot assume that `/tmp/akamai-reports/...` from an earlier node still exists.

If a future design requires cross-job usage, an approved transfer/persistent-storage mechanism is required.

---

## 17. Cleanup

File:

```text
roles/akamai_version_push/tasks/report_storage_cleanup.yml
```

SSPA-7 should not delete the workspace immediately after validation because downstream report-generation/email tasks may still need it.

The intended lifecycle is:

```text
prepare workspace
      ↓
generate report
      ↓
consume/send report
      ↓
all same-job consumers finish
      ↓
cleanup workspace
```

The orchestrating playbook or workflow should invoke cleanup from an `always` section after downstream consumers complete.

---

## 18. common_logging Integration

Successful events use:

```yaml
ansible.builtin.include_role:
  name: common_logging
  tasks_from: log_event
```

Terminal failures use:

```yaml
ansible.builtin.include_role:
  name: common_logging
  tasks_from: fail_with_error
```

The module actor should identify the automation component, for example:

```text
report_storage
```

The `triggered_by` identity should represent the AAP launcher/user rather than the module itself.

---

## 19. Error Classification

Configuration/path/input problems:

```text
category = validation
```

Filesystem, permission, group, write/read/list problems:

```text
category = infrastructure
```

Stable SSPA-7 error keys remain owned by the Akamai role while `common_logging` maps the category to the canonical numeric code.

---

## 20. How to Call SSPA-7

SSPA-7 should be called explicitly.

Example:

```yaml
- name: "Prepare temporary report storage"
  ansible.builtin.include_role:
    name: akamai_version_push
    tasks_from: report_storage
```

Do not automatically import unrelated ServiceNow, CTASK, or report-storage tasks from the role's default `tasks/main.yml` simply to support this story.

---

## 21. Production Checklist

Before merge, verify:

- PR contains only SSPA-7 changes.
- No ServiceNow playbook is included.
- No CTASK playbook is included.
- No unrelated collection version is changed.
- No unrelated role dispatcher imports are added.
- Deployed environment is sourced from protected configuration.
- AAP job uses an approved environment.
- Normal run publishes `storage_verified: true`.
- Check Mode publishes `storage_verified: false`.
- Workspace is isolated per execution.
- Directory mode is `0700` or approved `0750`.
- File mode is `0600` or approved `0640`.
- Probe list check uses `hidden: true`.
- Final report is not falsely reported as existing.
- Cleanup is called only after downstream same-job consumers finish.
- AAP QA evidence is captured before final approval.
