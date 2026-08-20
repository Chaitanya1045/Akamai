# SSPA-7 — Updated Test Cases and Test Matrix

## 1. Purpose

This document contains the regression and QA test cases for the **updated SSPA-7 temporary report-storage implementation** after the technical-lead review changes.

The test cases are based on the current code structure:

```text
roles/akamai_version_push/tasks/
├── report_storage.yml
├── report_storage_context.yml
├── report_storage_validate.yml
├── report_storage_provision.yml
├── report_storage_verify.yml
├── report_storage_publish.yml
└── report_storage_cleanup.yml
```

The updated design uses:

```text
/tmp/akamai-reports/<execution_id>/
```

and distinguishes:

```text
configuration_validated
storage_verified
workspace_exists
report_exists
```

SSPA-7 validates temporary storage only. It does **not** create the final deployment report.

---

# 2. Important Runtime Rules

## Deployed AAP execution

For a deployed AAP run:

```text
report_storage_test_mode = false
```

The code requires:

```text
common_logging_request_id
execution ID from common_logging_execution_id or awx_job_id
AAP launcher identity from awx_user_email or awx_user_name
environment = dev / rnd / qa / prod
```

The environment should come from protected inventory or fixed Job Template configuration.

## Standalone/local test execution

For dedicated test execution:

```yaml
report_storage_test_mode: true
```

The code permits:

```text
environment = test / local
```

and allows local request/execution/launcher fallbacks.

---

# 3. Core Test Matrix

| TC ID | Priority | Scenario | Execution Type | Expected Result |
|---|---|---|---|---|
| TC00 | Blocker | PR contains only SSPA-7 changes | Static review | PASS |
| TC01 | Blocker | Normal private storage validation | Normal | PASS |
| TC02 | Blocker | AAP QA deployed-context validation | Normal AAP | PASS |
| TC03 | Blocker | Check Mode semantics | Check Mode | PASS |
| TC04 | High | I/O verification explicitly disabled | Normal | PASS with `storage_verified=false` |
| TC05 | High | Missing request ID in deployed context | Normal | FAIL |
| TC06 | High | Missing execution ID in deployed context | Normal | FAIL |
| TC07 | High | Missing launcher identity in deployed context | Normal | FAIL |
| TC08 | High | Invalid deployed environment | Normal | FAIL |
| TC09 | High | `local/test` used while test mode is false | Normal | FAIL |
| TC10 | High | Explicit local test fallback | Local test | PASS |
| TC11 | High | Unsafe execution ID containing `/` | Local test | FAIL |
| TC12 | High | Unsafe execution ID containing `..` | Local test | FAIL |
| TC13 | High | Unsafe execution ID containing `\` | Local test | FAIL |
| TC14 | High | Execution ID longer than 128 chars | Local test | FAIL |
| TC15 | High | Request ID longer than 256 chars | Local test | FAIL |
| TC16 | High | Tamper approved storage root | Local test | FAIL |
| TC17 | High | Tamper secure file-mode contract | Local test | FAIL |
| TC18 | High | Tamper secure directory-mode contract | Local test | FAIL |
| TC19 | High | Invalid access-group syntax | Local test | FAIL |
| TC20 | High | Access group does not exist in EE | Local/AAP | FAIL |
| TC21 | High | Valid protected group access | Normal | PASS |
| TC22 | High | Hidden probe can be listed | Normal | PASS |
| TC23 | High | Verification probe is removed | Normal | PASS |
| TC24 | High | Published metadata is truthful | Normal | PASS |
| TC25 | High | Same-job downstream task can use workspace | Normal | PASS |
| TC26 | High | Explicit cleanup removes workspace | Normal | PASS |
| TC27 | High | Unsafe cleanup target is refused | Normal | FAIL |
| TC28 | High | Actor and triggered-by are separated | Normal AAP | PASS |
| TC29 | Medium | common_logging success events exist | Normal | PASS |
| TC30 | Medium | Standalone audit rendering | Standalone | PASS |
| TC31 | Medium | Controlled write failure | Failure injection | FAIL |
| TC32 | Medium | Controlled read failure | Failure injection | FAIL |
| TC33 | Medium | Controlled list failure | Failure injection | FAIL |
| TC34 | Medium | Controlled workspace provisioning failure | Failure injection | FAIL |
| TC35 | Medium | Controlled cleanup failure | Failure injection | FAIL |
| TC36 | Medium | UTF-8 / no corrupted characters | Static review | PASS |
| TC37 | Medium | Existing automated normal-run contract | Automated test | PASS |
| TC38 | Medium | Existing automated Check Mode contract | Automated test | PASS |

---

# 4. TC00 — PR Scope Validation

## Goal

Confirm that the SSPA-7 PR contains only temporary-storage work.

## Verify that the PR includes only relevant files such as

```text
roles/akamai_version_push/defaults/main.yml
roles/akamai_version_push/vars/main.yml
roles/akamai_version_push/tasks/report_storage*.yml
playbooks/akamai_version_push/report_storage.yml
docs/
tests/
```

## Verify that the PR does NOT introduce unrelated changes

```text
ServiceNow validation playbook
CTASK update playbook
ServiceNow task implementation
CTASK task implementation
unrelated ServiceNow collection version changes
unrelated tasks/main.yml imports
```

## Expected

```text
PASS
```

This is a mandatory PR-level test before runtime validation.

---

# 5. TC01 — Normal Private Storage Validation

## Goal

Validate the complete private-storage happy path.

## Recommended standalone test input

```yaml
report_storage_test_mode: true

common_logging_environment: "local"

common_logging_request_id: "REPORT-STORAGE-TC01"

common_logging_execution_id: "TEST-EXEC-TC01"

report_storage_verify_io: true

report_storage_access_group: ""
```

Run using the standalone playbook or the executable contract test.

## Expected runtime flow

```text
Resolve context
      ↓
Validate context
      ↓
Validate storage/path contract
      ↓
Validate /tmp
      ↓
Create /tmp/akamai-reports
      ↓
Create /tmp/akamai-reports/TEST-EXEC-TC01
      ↓
Directory mode = 0700
      ↓
Create hidden probe
      ↓
File mode = 0600
      ↓
WRITE passes
      ↓
READ/content passes
      ↓
LIST passes with hidden=true
      ↓
Probe removed
      ↓
Metadata published
```

## Expected metadata before explicit workspace cleanup

```yaml
configuration_validated: true
storage_verified: true
workspace_exists: true
report_exists: false
file_mode: "0600"
directory_mode: "0700"
ephemeral: true
same_job_only: true
separate_job_transfer_required: true
cleanup_required: true
check_mode: false
```

## Expected workspace

```text
/tmp/akamai-reports/TEST-EXEC-TC01
```

---

# 6. TC02 — AAP QA Deployed Context

## Goal

Validate the real deployed-context rules.

## AAP setup

Keep:

```yaml
report_storage_test_mode: false
```

Provide the environment through protected inventory or fixed Job Template configuration:

```yaml
akamai:
  environment:
    name: qa
```

or:

```yaml
common_logging_environment: "qa"
```

The integrated workflow must provide:

```text
common_logging_request_id
```

AAP should provide:

```text
awx_job_id
awx_user_email or awx_user_name
```

## Expected context

```text
request_id    = shared workflow request ID
execution_id  = shared execution ID or awx_job_id
actor         = akamai_version_push.report_storage
triggered_by  = AAP launcher
environment   = qa
test_mode     = false
```

## Expected

```text
PASS
```

Do not manually fake AAP launcher fields for the final QA evidence if AAP already provides them.

---

# 7. TC03 — Check Mode Semantics

## Goal

Ensure Check Mode validates configuration without falsely claiming filesystem I/O verification.

## Input

```yaml
report_storage_test_mode: true

common_logging_environment: "local"

common_logging_request_id: "REPORT-STORAGE-CHECK-001"

common_logging_execution_id: "TEST-CHECK-001"

report_storage_verify_io: true

report_storage_access_group: ""
```

## Run

Use AAP Job Type:

```text
Check
```

or locally:

```text
ansible-playbook tests/report_storage_check_mode.yml --check
```

## Expected metadata

```yaml
configuration_validated: true
storage_verified: false
workspace_exists: false
report_exists: false
check_mode: true
```

## Expected behavior

The following must **not** happen:

```text
probe creation
probe write
probe read
probe list
workspace creation
```

A `common_logging` Check Mode success event should state that filesystem I/O was intentionally not performed.

---

# 8. TC04 — I/O Verification Disabled

## Goal

Validate the explicit `report_storage_verify_io: false` behavior.

## Input

```yaml
report_storage_test_mode: true
common_logging_environment: "local"
common_logging_request_id: "REPORT-STORAGE-TC04"
common_logging_execution_id: "TEST-EXEC-TC04"

report_storage_verify_io: false
report_storage_access_group: ""
```

## Expected

The workspace is still provisioned in normal mode, but probe I/O is skipped.

Expected metadata:

```yaml
configuration_validated: true
storage_verified: false
workspace_exists: true
report_exists: false
check_mode: false
```

Expected log result for skipped I/O:

```text
result = skipped
```

This confirms that `storage_verified` means **real I/O was successfully tested**, not merely that configuration validation succeeded.

---

# 9. TC05 — Missing Request ID in Deployed Context

## Goal

Confirm that deployed execution does not silently generate a local request ID.

## Setup

```yaml
report_storage_test_mode: false
common_logging_environment: "qa"
common_logging_execution_id: "AAP-TEST-001"
```

Do not provide:

```text
common_logging_request_id
```

## Expected

Bootstrap context validation fails.

Expected assertion reason includes the requirement for:

```text
shared common_logging_request_id
```

## Important

This failure occurs during bootstrap context validation **before `common_logging` is invoked**, because the logger itself requires valid correlation context.

---

# 10. TC06 — Missing Execution ID in Deployed Context

## Goal

Confirm a deployed execution requires a shared/AAP execution identifier.

## Controlled standalone setup

```yaml
report_storage_test_mode: false
common_logging_environment: "qa"
common_logging_request_id: "REPORT-STORAGE-TC06"
```

Ensure neither of these is available:

```text
common_logging_execution_id
awx_job_id
```

## Expected

```text
FAIL during context bootstrap validation
```

The code must not generate:

```text
LOCAL-<timestamp>
```

while `report_storage_test_mode=false`.

---

# 11. TC07 — Missing Launcher Identity in Deployed Context

## Goal

Ensure `triggered_by` does not silently become generic automation.

## Controlled setup

Use a non-AAP/local harness with:

```yaml
report_storage_test_mode: false
common_logging_environment: "qa"
common_logging_request_id: "REPORT-STORAGE-TC07"
common_logging_execution_id: "EXEC-TC07"
```

Ensure these are unavailable:

```text
awx_user_email
awx_user_name
```

## Expected

```text
FAIL during context bootstrap validation
```

In deployed AAP QA, verify that `triggered_by` resolves to the actual AAP launcher.

---

# 12. TC08 — Invalid Deployed Environment

## Goal

Ensure invalid deployed environments are rejected instead of converted to `local`.

## Input

```yaml
report_storage_test_mode: false
common_logging_request_id: "REPORT-STORAGE-TC08"
common_logging_execution_id: "EXEC-TC08"
common_logging_environment: "invalid"
```

Provide a launcher identity in the test harness.

## Expected

```text
FAIL during context validation
```

Allowed deployed values are:

```text
dev
rnd
qa
prod
```

---

# 13. TC09 — `local` or `test` Not Allowed in Deployed Mode

## Goal

Confirm test environments are permitted only when explicit test mode is enabled.

## Example

```yaml
report_storage_test_mode: false
common_logging_environment: "local"
common_logging_request_id: "REPORT-STORAGE-TC09"
common_logging_execution_id: "EXEC-TC09"
```

## Expected

```text
FAIL
```

Repeat with:

```yaml
common_logging_environment: "test"
```

Expected:

```text
FAIL
```

---

# 14. TC10 — Explicit Local Test Fallback

## Goal

Confirm fallback behavior remains available for dedicated local/standalone tests only.

## Input

```yaml
report_storage_test_mode: true
```

Do not provide request ID, execution ID, environment or launcher identity.

## Expected context

Conceptually:

```text
request_id   = REPORT-<timestamp>
execution_id = LOCAL-<timestamp>
triggered_by = local-test
environment  = local
actor        = akamai_version_push.report_storage
```

## Expected

```text
PASS
```

This fallback must never occur in deployed mode.

---

# 15. TC11 — Execution ID Contains `/`

## Input

```yaml
report_storage_test_mode: true
common_logging_environment: "local"
common_logging_request_id: "REPORT-STORAGE-TC11"
common_logging_execution_id: "TEST/EXEC"
```

## Expected

```text
FAIL during context validation
```

The execution ID does not satisfy:

```text
^[A-Za-z0-9._-]{1,128}$
```

No workspace should be created.

---

# 16. TC12 — Execution ID Contains `..`

## Input

```yaml
common_logging_execution_id: "../TEST"
```

with otherwise valid local-test inputs.

## Expected

```text
FAIL
```

This proves traversal-like execution identifiers are blocked.

---

# 17. TC13 — Execution ID Contains Backslash

## Input

```yaml
common_logging_execution_id: "TEST\\EXEC"
```

## Expected

```text
FAIL
```

No storage path should be created outside the approved contract.

---

# 18. TC14 — Execution ID Too Long

## Goal

Validate the 128-character execution path-component limit.

## Setup

Use an execution ID longer than 128 characters.

## Expected

```text
FAIL during context validation
```

---

# 19. TC15 — Request ID Too Long

## Goal

Validate the request ID maximum length.

## Setup

Use:

```text
common_logging_request_id
```

with more than 256 characters.

## Expected

```text
FAIL during context bootstrap validation
```

---

# 20. TC16 — Approved Root Tampering

## Goal

Confirm storage cannot be redirected outside the approved application root.

The current internal contract requires:

```yaml
report_storage_base_root: "/tmp"
report_storage_root: "/tmp/akamai-reports"
```

## Controlled negative test

Override/tamper one of these values in a dedicated test harness.

Example:

```yaml
report_storage_root: "/var/tmp/akamai-reports"
```

## Expected

Terminal failure through:

```text
common_logging/fail_with_error.yml
```

Expected:

```text
category = validation
error key = REPORT_STORAGE_CONFIG_INVALID
```

No execution workspace should be provisioned.

---

# 21. TC17 — Unsafe File Permission Contract

## Goal

Confirm file permissions cannot be relaxed.

Required values:

```text
private = 0600
group   = 0640
```

## Controlled negative example

```yaml
report_storage_private_file_mode: "0666"
```

## Expected

```text
FAIL
category = validation
error key = REPORT_STORAGE_CONFIG_INVALID
```

---

# 22. TC18 — Unsafe Directory Permission Contract

## Goal

Confirm directory permissions cannot be relaxed.

Required values:

```text
private = 0700
group   = 0750
```

## Controlled negative example

```yaml
report_storage_private_directory_mode: "0777"
```

## Expected

```text
FAIL
category = validation
error key = REPORT_STORAGE_CONFIG_INVALID
```

---

# 23. TC19 — Invalid Access-Group Syntax

## Input

Example:

```yaml
report_storage_access_group: "../../admins"
```

or:

```yaml
report_storage_access_group: "group/name"
```

## Expected

```text
FAIL
category = validation
error key = REPORT_STORAGE_CONFIG_INVALID
```

The syntax must match:

```text
^[A-Za-z_][A-Za-z0-9_.-]{0,63}$
```

---

# 24. TC20 — Access Group Does Not Exist

## Goal

Ensure syntactically valid but unavailable groups are rejected.

## Input

```yaml
report_storage_access_group: "akamai_group_that_does_not_exist"
```

Use otherwise valid context.

## Expected

The `getent` lookup fails.

Expected terminal result:

```text
category = validation
error key = REPORT_ACCESS_GROUP_INVALID
```

This expectation reflects the current implementation.

---

# 25. TC21 — Valid Protected Group Access

## Goal

Validate group-readable storage.

## Preconditions

Use an approved OS group that actually exists inside the EE.

Example:

```yaml
report_storage_access_group: "<APPROVED_EE_GROUP>"
```

## Expected directory mode

```text
0750
```

## Expected probe file mode

```text
0640
```

## Expected

Directory and probe `gr_name` must equal the configured group.

Metadata:

```yaml
file_mode: "0640"
directory_mode: "0750"
access_group: "<APPROVED_EE_GROUP>"
storage_verified: true
```

The access group should come from protected inventory/fixed Job Template configuration rather than an unrestricted survey.

---

# 26. TC22 — Hidden Probe List Verification

## Goal

Revalidate the known hidden-file issue.

The probe is named like:

```text
.akamai_report_probe_20260820T140000Z.tmp
```

The current code uses:

```yaml
ansible.builtin.find:
  hidden: true
```

## Expected

```text
report_storage_probe_list.matched = 1
```

This confirms the previous `matched: 0` failure is fixed.

---

# 27. TC23 — Probe Cleanup

## Goal

Ensure the temporary verification probe is removed after I/O validation.

## After successful normal validation

Run:

```yaml
ansible.builtin.find:
  paths: "{{ report_storage_execution_dir }}"
  patterns:
    - ".akamai_report_probe_*"
  file_type: file
  recurse: false
  hidden: true
```

## Expected

```text
matched = 0
```

The execution workspace should still exist until explicit downstream cleanup.

---

# 28. TC24 — Published Metadata Contract

## Goal

Validate the exact SSPA-7 metadata semantics.

## Expected normal private run

```yaml
base_root: "/tmp"
root: "/tmp/akamai-reports"
workspace_path: "/tmp/akamai-reports/<execution_id>"
workspace_exists: true
configuration_validated: true
storage_verified: true
report_exists: false
file_mode: "0600"
directory_mode: "0700"
access_group: ""
ephemeral: true
same_job_only: true
separate_job_transfer_required: true
cleanup_required: true
cleanup_task: "report_storage_cleanup"
final_report_naming_owner: "downstream_report_generation"
audit_filename_owner: "common_logging"
request_id: "<request-id>"
execution_id: "<execution-id>"
environment: "<environment>"
check_mode: false
```

## Critical assertion

This must remain:

```yaml
report_exists: false
```

SSPA-7 does not create the final deployment report.

---

# 29. TC25 — Same-Job Downstream Usage

## Goal

Validate the lifecycle contract.

Run a dummy downstream task in the **same play/job** after `tasks_from: report_storage`.

Example concept:

```yaml
- name: "Create dummy downstream file"
  ansible.builtin.copy:
    dest: "{{ report_storage_metadata.workspace_path }}/downstream_test.txt"
    content: "same-job consumer test"
    mode: "0600"
```

## Expected

```text
PASS
```

The downstream task can use the workspace during the same AAP execution.

## Important

Do not treat a separate AAP workflow node/job as equivalent. A new job may receive a different EE and must not assume the original `/tmp` content exists.

---

# 30. TC26 — Explicit Workspace Cleanup

## Goal

Ensure cleanup occurs only after same-job consumers complete.

## Flow

```text
report_storage
    ↓
dummy downstream consumer
    ↓
report_storage_cleanup
```

## Expected before cleanup

```text
/tmp/akamai-reports/<execution_id> exists
```

## Expected after cleanup

```text
/tmp/akamai-reports/<execution_id> does not exist
```

A cleanup success event should be emitted through `common_logging/log_event.yml`.

---

# 31. TC27 — Unsafe Cleanup Target

## Goal

Ensure cleanup refuses paths outside the approved per-execution directory.

## Controlled test

After valid context resolution, tamper the cleanup target in a test harness.

Example unsafe target:

```text
/tmp
```

or:

```text
/tmp/akamai-reports
```

## Expected

Cleanup validation fails and is routed through:

```text
common_logging/fail_with_error.yml
```

Expected:

```text
category = infrastructure
error key = REPORT_STORAGE_CLEANUP_FAILED
```

The unsafe target must not be deleted.

---

# 32. TC28 — Actor vs Triggered-By Separation

## Goal

Verify the technical-lead logging identity requirement.

## Expected actor

```text
akamai_version_push.report_storage
```

This comes from:

```text
report_storage_module
```

## Expected triggered_by

In AAP:

```text
actual AAP launcher email/name
```

## Expected

```text
actor != triggered_by
```

unless by coincidence the launcher text exactly matches the module string.

The actor must never be replaced by generic `automation` in deployed execution.

---

# 33. TC29 — common_logging Success Events

## Goal

Confirm the role reuses the shared logging framework.

A successful normal run should generate events for stages including:

```text
report_storage_context
report_storage_validation
report_storage_provisioning
report_storage_write
report_storage_read
report_storage_list
report_storage_publish
```

If cleanup is executed:

```text
report_storage_cleanup
```

should also be present.

## Expected

```text
common_logging_history is defined
common_logging_history length > 0
```

and events use:

```text
tasks_from: log_event
```

---

# 34. TC30 — Standalone Audit Rendering

## Goal

Verify the standalone playbook uses the existing `common_logging` audit renderer.

Run:

```text
playbooks/akamai_version_push/report_storage.yml
```

The `always` section should:

1. clean the standalone workspace;
2. call `common_logging/tasks/render_audit.yml` when logging history exists.

## Expected

```text
common_logging_history length > 0
render_audit is invoked
```

Audit filename/content remains owned by `common_logging`, not SSPA-7.

---

# 35. TC31 — Controlled Write Failure

## Goal

Validate write failure classification.

This requires a controlled QA/test-harness failure because the normal path under `/tmp` is intentionally writable.

## Failure injection requirement

Cause the probe creation step to fail **after configuration/provisioning has succeeded**.

Do not change the production contract merely to make the test fail.

## Expected

```text
FAIL
stage = report_storage_write
category = infrastructure
error key = REPORT_STORAGE_WRITE_FAILED
```

The verification artifact cleanup in the `always` section should still be attempted.

---

# 36. TC32 — Controlled Read Failure

## Goal

Validate read failure classification.

Use a controlled test harness to make the probe unreadable/remove it after write verification and before `slurp`.

## Expected

```text
FAIL
stage = report_storage_read
category = infrastructure
error key = REPORT_STORAGE_READ_FAILED
```

---

# 37. TC33 — Controlled List Failure

## Goal

Validate list failure classification.

Use a controlled harness to cause the probe list assertion to fail after successful write/read.

## Expected

```text
FAIL
stage = report_storage_list
category = infrastructure
error key = REPORT_STORAGE_LIST_FAILED
```

This is different from the normal hidden-probe test, which must pass because production code has:

```yaml
hidden: true
```

---

# 38. TC34 — Controlled Workspace Provisioning Failure

## Goal

Validate provisioning failure classification.

Cause directory creation/ownership validation to fail in a controlled EE test.

## Expected

```text
FAIL
stage = report_storage_provisioning
category = infrastructure
error key = REPORT_STORAGE_UNAVAILABLE
```

---

# 39. TC35 — Controlled Cleanup Failure

## Goal

Validate cleanup failure classification.

Use a controlled test harness where the execution-specific workspace cannot be removed.

## Expected

```text
FAIL
stage = report_storage_cleanup
category = infrastructure
error key = REPORT_STORAGE_CLEANUP_FAILED
```

---

# 40. TC36 — UTF-8 / Encoding Validation

## Goal

Address the technical-lead encoding comment.

Validate all changed files are UTF-8 and contain no corrupted characters.

Recommended repository checks:

```text
open files in VS Code with UTF-8 encoding
review git diff for replacement/corrupted characters
ensure YAML parser can read all changed .yml files
```

## Expected

```text
PASS
```

No corrupted characters should appear in task names, comments, documentation or YAML values.

---

# 41. TC37 — Automated Normal-Run Contract Test

The updated code already contains:

```text
tests/report_storage_contract.yml
```

## Input inside test

```yaml
report_storage_test_mode: true
common_logging_environment: "local"
common_logging_request_id: "REPORT-STORAGE-TEST-001"
common_logging_execution_id: "TEST-EXEC-001"
report_storage_verify_io: true
report_storage_access_group: ""
```

## Assertions already covered

```text
configuration_validated = true
storage_verified = true
report_exists = false
workspace_exists = true
ephemeral = true
same_job_only = true
separate_job_transfer_required = true
file_mode = 0600
directory_mode = 0700
workspace_path = /tmp/akamai-reports/TEST-EXEC-001
probe matched after verification = 0
common_logging_history exists
workspace removed after explicit cleanup
```

## Expected

```text
PASS
```

This is the first automated regression test to run.

---

# 42. TC38 — Automated Check Mode Contract Test

The updated code contains:

```text
tests/report_storage_check_mode.yml
```

## Run

```text
ansible-playbook tests/report_storage_check_mode.yml --check
```

or configure the equivalent AAP Job Template in Check Mode.

## Assertions

```text
configuration_validated = true
storage_verified = false
report_exists = false
workspace_exists = false
check_mode = true
```

## Expected

```text
PASS
```

---

# 43. Error-Key Matrix Based on Current Code

| Failure | Expected Category | Expected Error Key |
|---|---|---|
| Invalid storage root/modes/path contract | `validation` | `REPORT_STORAGE_CONFIG_INVALID` |
| Protected access group unavailable | `validation` | `REPORT_ACCESS_GROUP_INVALID` |
| `/tmp` unavailable | `infrastructure` | `REPORT_STORAGE_UNAVAILABLE` |
| Workspace provisioning/mode/ownership failure | `infrastructure` | `REPORT_STORAGE_UNAVAILABLE` |
| Probe write failure | `infrastructure` | `REPORT_STORAGE_WRITE_FAILED` |
| Probe read/content failure | `infrastructure` | `REPORT_STORAGE_READ_FAILED` |
| Probe list failure | `infrastructure` | `REPORT_STORAGE_LIST_FAILED` |
| Workspace cleanup failure/unsafe cleanup target | `infrastructure` | `REPORT_STORAGE_CLEANUP_FAILED` |

## Bootstrap-context note

The current code validates the initial request/execution/environment/launcher context with `ansible.builtin.assert` **before** invoking `common_logging`.

Therefore invalid bootstrap context cases such as:

```text
missing request ID in deployed mode
missing execution ID in deployed mode
invalid execution-ID characters
missing launcher
invalid deployed environment
```

are expected to fail directly at:

```text
Report Storage | Validate shared execution context
```

rather than producing a `common_logging/fail_with_error` event.

This expectation is based on the current implementation and should be used when reviewing the test output.

---

# 44. Recommended Test Execution Order

For the next regression cycle, run in this order:

| Order | Test | Why |
|---:|---|---|
| 1 | TC00 | Ensure PR scope is correct before testing |
| 2 | TC37 | Automated normal contract |
| 3 | TC38 | Automated Check Mode contract |
| 4 | TC01 | Manual normal happy path |
| 5 | TC24 | Verify new metadata semantics |
| 6 | TC22 | Verify hidden-file listing fix |
| 7 | TC23 | Verify probe cleanup |
| 8 | TC26 | Verify final workspace cleanup |
| 9 | TC08/TC09 | Verify environment protection |
| 10 | TC11–TC14 | Verify identifier/path protection |
| 11 | TC17/TC18 | Verify permission protections |
| 12 | TC20 | Verify invalid group handling |
| 13 | TC21 | Verify valid shared-group mode |
| 14 | TC28/TC29 | Verify common_logging identity/events |
| 15 | TC02 | Capture real AAP QA deployed evidence |
| 16 | TC31–TC35 | Controlled failure-classification tests |

---

# 45. Minimum Evidence for PR Re-Review

For technical-lead re-review, capture evidence for at least:

```text
1. PR contains SSPA-7 only
2. Normal AAP QA run succeeds
3. Workspace uses /tmp/akamai-reports/<execution_id>
4. Private directory mode is 0700
5. Private probe mode is 0600
6. Write/read/list all pass
7. Hidden probe is found using hidden=true
8. Probe is removed after verification
9. storage_verified=true only after real I/O
10. report_exists=false
11. Check Mode has storage_verified=false
12. Invalid deployed environment is rejected
13. Unsafe execution/path identifier is rejected
14. Actor is akamai_version_push.report_storage
15. triggered_by is the AAP launcher
16. common_logging events are generated
17. Explicit cleanup removes execution workspace
18. At least one validation failure mapping is demonstrated
19. At least one infrastructure failure mapping is demonstrated
20. Files are UTF-8 and YAML parses successfully
```

---

# 46. Final Acceptance Summary

The updated SSPA-7 implementation should be considered successfully tested when:

```text
Correct PR scope
AND
protected execution context
AND
safe isolated workspace
AND
secure permissions
AND
normal write/read/list verification
AND
truthful metadata
AND
correct Check Mode semantics
AND
same-job lifecycle documented/tested
AND
explicit cleanup succeeds
AND
common_logging behavior is verified
AND
AAP QA runtime evidence is captured
```
