# SSPA-7 — AAP QA Validation Guide

## 1. Purpose

This document describes how to validate the corrected SSPA-7 temporary report-storage implementation in AAP QA.

The objective is to prove:

- configuration validation works;
- normal execution performs real filesystem verification;
- Check Mode does not claim I/O verification;
- permissions are correct;
- storage metadata is accurate;
- path isolation works;
- cleanup behavior is correct;
- `common_logging` success/failure handling works.

---

## 2. Prerequisites

Before testing:

1. Push the corrected SSPA-7 branch.
2. Ensure the AAP Project syncs the correct branch/commit.
3. Use the approved Execution Environment.
4. Confirm `common_logging` is available in the repository/role path.
5. Ensure the Job Template has the environment supplied from protected configuration.
6. Do not include unrelated ServiceNow or CTASK tasks in the test Job Template.
7. Use the standalone SSPA-7 playbook:

```text
playbooks/akamai_version_push/report_storage.yml
```

---

## 3. PR Scope Check Before Runtime Testing

Confirm the PR does not include:

```text
ServiceNow validation playbook
CTASK update playbook
ServiceNow task implementation
CTASK task implementation
unrelated collection version changes
unrelated tasks/main.yml imports
```

If these appear, revert them to the `dev` baseline before continuing.

---

## 4. Test 1 — Normal Private Workspace

### Goal

Prove that a standard run creates and verifies isolated temporary storage.

### Configuration

Use an approved QA environment and no shared access group.

Expected security:

```text
directory mode = 0700
file mode      = 0600
```

### Run

Launch the Job Template in normal Run mode.

### Verify job output

Confirm the sequence includes:

```text
context resolution
configuration validation
workspace provisioning
probe write
probe stat/permission check
probe read
probe content validation
probe list
probe cleanup
metadata publication
success logging
```

### Expected metadata

```yaml
configuration_validated: true
storage_verified: true
workspace_exists: true
report_exists: false
ephemeral: true
```

The exact workspace should be isolated beneath:

```text
/tmp/akamai-reports/
```

---

## 5. Test 2 — Check Mode

### Goal

Prove that Check Mode validates configuration without claiming filesystem verification.

### Run

Set the AAP Job Type to:

```text
Check
```

Do not emulate Check Mode using an Extra Var.

### Expected

```yaml
configuration_validated: true
storage_verified: false
workspace_exists: false
report_exists: false
```

There should be no real probe write/read/list operation.

The job should clearly state that storage I/O was not verified.

---

## 6. Test 3 — Invalid Environment

### Goal

Prove a deployed AAP job does not silently fall back to `local`.

### Setup

Supply or force an invalid environment in the protected test configuration.

### Expected

The job fails through:

```text
common_logging/fail_with_error.yml
```

Expected category:

```text
validation
```

For deployed execution, valid environments are:

```text
dev
rnd
qa
prod
```

`test` and `local` are reserved for explicit standalone/local test mode.

---

## 7. Test 4 — Unsafe Execution Identifier

### Goal

Prove path traversal is blocked.

### Examples to reject

```text
../123
abc/def
abc\def
../../tmp
```

### Expected

The task fails before directory creation.

Expected category:

```text
validation
```

Only safe characters should be accepted for path components:

```text
A-Z a-z 0-9 . _ -
```

---

## 8. Test 5 — Invalid File Mode

### Goal

Prove unsafe permission configuration is blocked.

### Example

Set an unsafe mode such as:

```text
0666
```

or any value that grants permissions to `others`.

### Expected

Fail before probe creation.

Expected category:

```text
validation
```

---

## 9. Test 6 — Shared Group Access

### Goal

Prove approved group-readable access works.

### Setup

Configure an approved OS group that exists in the EE.

Expected permissions:

```text
directory = 0750
file      = 0640
```

### Expected

- group lookup succeeds;
- directory group ownership matches configured group;
- probe group ownership matches configured group;
- write/read/list passes;
- `storage_verified: true`.

---

## 10. Test 7 — Invalid/Unavailable Group

### Goal

Prove a missing group is rejected.

### Setup

Configure a group that does not exist inside the EE.

### Expected

Fail through `common_logging`.

Expected category:

```text
infrastructure
```

The failure should recommend using an approved group available inside the EE.

---

## 11. Test 8 — Write Failure

### Goal

Prove write failures are classified correctly.

### Setup

Use a controlled QA method to make the target location unwritable.

### Expected

```text
write verification fails
probe cleanup attempted
common_logging failure emitted
job stops
```

Expected category:

```text
infrastructure
```

---

## 12. Test 9 — Read Failure

### Goal

Prove read errors do not get reported as success.

### Expected

The job fails with the read-related SSPA-7 error key and `infrastructure` category.

---

## 13. Test 10 — Hidden Probe List Verification

### Goal

Prove the production fix for hidden files.

The probe begins with:

```text
.
```

The `find` task must contain:

```yaml
hidden: true
```

### Expected

```text
matched = 1
```

Without this setting the list test can fail even after successful write/read.

---

## 14. Test 11 — Probe Cleanup

### Goal

Prove the verification file is not left behind.

After successful validation:

```text
verification probe → deleted
workspace          → retained for downstream consumers
```

The final workspace should not contain the temporary probe.

---

## 15. Test 12 — Metadata Semantics

### Goal

Prove the task does not claim the final report exists.

After SSPA-7 only:

```yaml
storage_verified: true
report_exists: false
```

There should be no metadata field implying that a generated report already exists unless a later report-generation task has actually created one.

---

## 16. Test 13 — Same-Job Downstream Usage

### Goal

Prove a later task in the same AAP job can use the workspace.

Run:

```text
SSPA-7 storage preparation
      ↓
same play / same job downstream dummy consumer
```

The dummy consumer should be able to access the published workspace path.

---

## 17. Test 14 — Cleanup After Consumers

### Goal

Prove cleanup is correctly sequenced.

Expected orchestration:

```yaml
block:
  - prepare storage
  - generate/consume test artifact

always:
  - include cleanup task
```

After cleanup, the execution-specific workspace should no longer exist.

Do not run cleanup immediately after SSPA-7 validation if downstream tasks still require the directory.

---

## 18. Test 15 — common_logging Success

### Goal

Confirm successful events use:

```text
tasks_from: log_event
```

Review the emitted event and verify:

```text
request ID
execution ID
environment
actor/module
triggered_by launcher identity
stage
result
message
```

are consistent.

---

## 19. Test 16 — common_logging Failure

### Goal

Trigger a controlled validation failure.

Confirm the terminal failure uses:

```text
tasks_from: fail_with_error
```

and publishes the expected error category/key before stopping the play.

---

## 20. Evidence to Attach to PR

Capture AAP output showing:

1. successful QA normal run;
2. `storage_verified: true`;
3. correct workspace path;
4. correct directory/file modes;
5. successful write/read/list;
6. `hidden: true` list result;
7. probe cleanup;
8. `report_exists: false`;
9. successful Check Mode;
10. Check Mode `storage_verified: false`;
11. at least one validation failure;
12. at least one infrastructure failure;
13. common-logging success event;
14. common-logging terminal failure event.

---

## 21. Minimum Approval Set

At minimum, provide evidence for:

```text
Normal private run
Check Mode
Invalid environment
Unsafe identifier/path
Private permission verification
Group-access verification if group mode is supported
Hidden probe list validation
Probe cleanup
Metadata semantics
Final workspace cleanup
common_logging success/failure
```

---

## 22. Pass Criteria

SSPA-7 can be considered QA-validated only when:

```text
configuration validation passes
AND
normal run performs successful real I/O verification
AND
Check Mode does not claim I/O verification
AND
permissions are correct
AND
metadata is truthful
AND
common_logging works
AND
cleanup is correctly sequenced
```
