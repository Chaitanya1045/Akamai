# SSPA-7 — Integration, Lifecycle and Downstream Usage Guide

## 1. Purpose

This document explains how SSPA-7 temporary report storage is intended to be consumed by later Akamai automation tasks.

The key rule is:

> The temporary workspace is an execution-local workspace, not persistent storage.

---

## 2. Storage Lifecycle

The expected lifecycle is:

```text
AAP job starts
    ↓
SSPA-7 validates and prepares workspace
    ↓
Later task generates final report
    ↓
Later task consumes/sends report
    ↓
All same-job consumers finish
    ↓
Cleanup task removes workspace
    ↓
AAP job ends
```

---

## 3. Same Job Requirement

Files under `/tmp` exist inside the active Execution Environment.

A later task can directly access the file only when it runs in the same job/EE context.

Supported pattern:

```text
Job A
 ├─ SSPA-7 prepare storage
 ├─ generate report
 ├─ send/consume report
 └─ cleanup
```

Unsafe assumption:

```text
Job A creates /tmp file
      ↓
Job A ends
      ↓
Job B expects same /tmp file
```

That second pattern is not guaranteed because Job B may run in a different Execution Environment.

---

## 4. Cross-Job Usage

If a future workflow needs one AAP job to produce a report and another AAP job to consume it, `/tmp` is not sufficient.

An approved transfer mechanism is required, for example a future persistent artifact mechanism approved by the platform/security team.

SSPA-7 does not implement such a mechanism.

---

## 5. Published Contract

Downstream tasks should consume the published `report_storage` object instead of rebuilding paths independently.

Conceptual example:

```yaml
report_storage:
  workspace_root: "/tmp/akamai-reports"
  workspace_path: "/tmp/akamai-reports/123456"
  configuration_validated: true
  storage_verified: true
  workspace_exists: true
  report_exists: false
  ephemeral: true
```

Downstream code should use:

```text
report_storage.workspace_path
```

as the base directory.

---

## 6. `storage_verified`

`storage_verified` answers:

> Did this execution successfully perform real write/read/list verification?

Normal successful run:

```yaml
storage_verified: true
```

Check Mode:

```yaml
storage_verified: false
```

A downstream task that requires actual filesystem assurance should require `storage_verified: true`.

---

## 7. `report_exists`

After SSPA-7 only:

```yaml
report_exists: false
```

This is correct because SSPA-7 prepares and verifies storage but does not generate the final report.

The later report-generation task should set or publish the final report existence/path after successful creation.

---

## 8. Final Report Path Ownership

SSPA-7 should not publish a path that looks like an already-existing final report.

The later report-generation story owns:

```text
final filename
final report path
final content
report_exists=true
```

SSPA-7 owns:

```text
workspace root
workspace path
storage verification
permission/security contract
cleanup capability
```

---

## 9. Example Integrated Playbook Pattern

Conceptual orchestration:

```yaml
- name: "Run Akamai deployment workflow"
  hosts: localhost
  gather_facts: false

  tasks:

    - name: "Prepare temporary report storage"
      ansible.builtin.include_role:
        name: akamai_version_push
        tasks_from: report_storage

    - name: "Generate final deployment report"
      ansible.builtin.include_role:
        name: akamai_version_push
        tasks_from: report_generation

    - name: "Consume or deliver report"
      ansible.builtin.include_role:
        name: akamai_version_push
        tasks_from: report_delivery
```

Cleanup should preferably be placed in an `always` section:

```yaml
    - name: "Run report workflow"
      block:

        - name: "Prepare storage"
          ansible.builtin.include_role:
            name: akamai_version_push
            tasks_from: report_storage

        - name: "Generate report"
          ansible.builtin.include_role:
            name: akamai_version_push
            tasks_from: report_generation

        - name: "Consume report"
          ansible.builtin.include_role:
            name: akamai_version_push
            tasks_from: report_delivery

      always:

        - name: "Clean temporary report workspace"
          ansible.builtin.include_role:
            name: akamai_version_push
            tasks_from: report_storage_cleanup
```

This ensures cleanup occurs even when a later consumer fails.

---

## 10. Why SSPA-7 Is Not in Default Role Dispatcher

The technical-lead review requested that unrelated work not be automatically executed through the role entry point.

SSPA-7 is therefore explicitly callable:

```yaml
tasks_from: report_storage
```

This prevents a standalone storage test from unexpectedly running:

```text
ServiceNow validation
CTASK update
deployment logic
other unrelated story tasks
```

---

## 11. Environment Source

The environment should be supplied from protected inventory or fixed AAP Job Template configuration.

Recommended deployed values:

```text
dev
rnd
qa
prod
```

Do not expose the environment as an unrestricted Prompt on Launch value.

---

## 12. Access Group Source

If group-readable report access is required, the group should also come from protected configuration.

Do not let arbitrary users supply any OS group through a survey.

The configured group must be approved and available inside the EE.

---

## 13. Actor vs Triggered By

The two logging identities have different meanings.

### Actor

The automation component performing the action:

```text
report_storage
```

### Triggered by

The human/AAP launcher responsible for starting the execution.

The implementation keeps these identities separate so audit logs can distinguish the component from the launcher.

---

## 14. Request ID / Execution ID

For integrated workflows:

- reuse the shared workflow request ID;
- reuse the shared execution ID;
- normally use `awx_job_id` for AAP execution identity.

Do not generate a new unrelated request ID for each storage log event.

Standalone/local test fallback is acceptable only when explicitly running in test mode.

---

## 15. Cleanup Ownership

SSPA-7 provides the cleanup capability, but the orchestration layer determines the correct time to call it.

Why:

```text
SSPA-7 cannot know when all future report consumers have finished.
```

Therefore cleanup must happen after:

```text
report generation
email/report delivery
any same-job audit/consumer
```

have completed.

---

## 16. Failure Handling

A failure in:

```text
configuration/path validation
```

uses the validation category.

A failure in:

```text
directory creation
group ownership
write/read/list
filesystem access
```

uses the infrastructure category.

All terminal failures must pass through `common_logging/fail_with_error.yml`.

---

## 17. Integration Checklist

Before integrating SSPA-7 with later stories:

- Call it explicitly with `tasks_from: report_storage`.
- Confirm environment is protected.
- Confirm request/execution IDs come from shared workflow context.
- Verify `storage_verified: true` before relying on actual I/O.
- Use `workspace_path` rather than reconstructing `/tmp` paths.
- Do not interpret workspace path as final report path.
- Let report-generation task publish actual report path.
- Keep all direct `/tmp` consumers in the same AAP job.
- Run cleanup only after all consumers finish.
- Do not introduce persistent-retention behavior into SSPA-7.
