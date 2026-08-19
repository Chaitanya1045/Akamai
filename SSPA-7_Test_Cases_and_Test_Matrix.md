# SSPA-7 — Temporary Report Storage Test Cases & Test Matrix

## Purpose

This test set is intentionally limited to the **core behavior implemented for SSPA-7**.

The current design uses `/tmp` inside the AAP Execution Environment as **temporary report storage**. The story does not currently provide persistent retention because the Execution Environment filesystem is ephemeral.

The tests therefore focus on:

- `/tmp` availability
- Write/read/list verification
- Probe cleanup
- Secure file permissions
- Optional group-access behavior
- Report filename generation
- Invalid report metadata handling
- AAP Check Mode behavior
- Standardized failure handling through `common_logging`

The tests do **not** cover email delivery, report content generation, or long-term retention because those are outside the current SSPA-7 implementation scope.

---

# 1. Recommended Test Matrix

| TC ID | Scenario | Main Input / Setup | Expected Result | Expected Error Key / Output | Priority |
|---|---|---|---|---|---|
| TC01 | Valid temporary storage with report naming | `/tmp`, verify I/O enabled, version `252`, status `success` | PASS | `report_storage.report_path` populated | P1 |
| TC02 | Storage-only validation without report metadata | version/status empty | PASS | write/read/list pass; filename/path empty | P2 |
| TC03 | I/O verification disabled | `report_storage_verify_io: false` | PASS | `/tmp` validation passes; write/read/list skipped | P2 |
| TC04 | Invalid report version | version `abc` | FAIL | `REPORT_NAME_INVALID` | P1 |
| TC05 | Invalid file permission configuration | unsafe mode such as `0644` | FAIL | `REPORT_STORAGE_CONFIG_INVALID` | P1 |
| TC06 | Invalid/nonexistent access group | group does not exist inside EE | FAIL | `REPORT_ACCESS_GROUP_INVALID` | P1 |
| TC07 | Write verification failure | controlled non-writable storage condition | FAIL | `REPORT_STORAGE_WRITE_FAILED` | P1 |
| TC08 | Read verification failure | controlled unreadable/corrupted probe condition | FAIL | `REPORT_STORAGE_READ_FAILED` | P1 |
| TC09 | List verification failure | controlled list/find failure | FAIL | `REPORT_STORAGE_LIST_FAILED` | P1 |
| TC10 | Hidden probe file listing | hidden probe + `hidden: true` | PASS | probe found successfully | P1 |
| TC11 | Check Mode / dry-run validation | AAP Job Type = Check | PASS | no probe write; dry-run validation succeeds | P1 |
| TC12 | Group-readable mode | valid OS group configured | PASS | effective mode `0640` | P2 |
| TC13 | Default private mode | access group empty | PASS | effective mode `0600` | P1 |
| TC14 | Final probe cleanup | normal successful run | PASS | temporary probe removed from `/tmp` | P1 |

> **P1** = required for functional acceptance / demonstration  
> **P2** = useful secondary coverage, but not necessary to spend equal demo time on

---

# 2. Baseline Test Input

Use this as the standard positive input:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "REPORT-TEST-001"

report_storage_verify_io: true

report_storage_access_group: ""

report_storage_version: "252"

report_storage_status: "success"
```

Expected naming pattern:

```text
{timestamp}_{version}_{status}.txt
```

Example:

```text
/tmp/20260819T090000Z_252_success.txt
```

---

# 3. Detailed Test Cases

## TC01 — Valid Temporary Storage with Report Naming

### Objective

Verify the complete successful SSPA-7 path.

### Inputs

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "REPORT-TEST-001"

report_storage_verify_io: true

report_storage_access_group: ""

report_storage_version: "252"

report_storage_status: "success"
```

### Expected Flow

```text
Validate configuration
        ↓
Verify /tmp exists
        ↓
Create hidden probe file
        ↓
Verify write
        ↓
Verify file mode
        ↓
Read probe
        ↓
Verify probe content
        ↓
List/find probe
        ↓
Delete probe
        ↓
Validate version/status
        ↓
Build final report filename/path
        ↓
PASS
```

### Expected Result

```text
Job Status: SUCCESS

report_storage:
  root: /tmp
  filename_format: "{timestamp}_{version}_{status}.txt"
  report_filename: "<timestamp>_252_success.txt"
  report_path: "/tmp/<timestamp>_252_success.txt"
  file_mode: "0600"
  ephemeral: true
  dry_run: false
```

### Why this test is required

This is the primary happy-path test and proves that the temporary report workspace can be used successfully by the AAP Execution Environment.

---

## TC02 — Storage-Only Validation

### Objective

Verify that `/tmp` validation can run independently of final report naming.

### Inputs

```yaml
report_storage_verify_io: true

report_storage_access_group: ""

report_storage_version: ""

report_storage_status: ""
```

### Expected Result

```text
Job Status: SUCCESS

Write validation: PASS
Read validation: PASS
List validation: PASS

report_filename: ""
report_path: ""
```

### Why this test is useful

SSPA-7 owns the temporary storage capability. Final report metadata may be provided by downstream workflow logic.

---

## TC03 — I/O Verification Disabled

### Objective

Verify the feature flag that disables active write/read/list verification.

### Inputs

```yaml
report_storage_verify_io: false

report_storage_access_group: ""

report_storage_version: "252"

report_storage_status: "success"
```

### Expected Result

```text
Job Status: SUCCESS

/tmp validation: PASS
Write verification: SKIPPED
Read verification: SKIPPED
List verification: SKIPPED
Filename validation: PASS
```

### Why this test is useful

It proves that active I/O testing can be disabled without changing the implementation.

---

## TC04 — Invalid Report Version

### Objective

Verify that the report version must be numeric.

### Inputs

```yaml
report_storage_version: "abc"

report_storage_status: "success"
```

### Expected Result

```text
Job Status: FAILED
Error Key: REPORT_NAME_INVALID
```

### Why this test is required

The naming contract expects a numeric Akamai version.

Valid:

```text
252
```

Invalid:

```text
abc
252a
v252
```

---

## TC05 — Unsafe File Permission Configuration

### Objective

Verify that insecure file permissions are rejected.

### Example Input

```yaml
report_storage_private_file_mode: "0644"
```

### Expected Result

```text
Job Status: FAILED
Error Key: REPORT_STORAGE_CONFIG_INVALID
Category: validation
```

### Why this test is required

`0644` allows other users to read the report.

The default private mode must remain:

```text
0600
```

and the optional group-readable mode:

```text
0640
```

---

## TC06 — Invalid or Missing Access Group

### Objective

Verify handling when a configured OS group does not exist inside the Execution Environment.

### Inputs

```yaml
report_storage_access_group: "group_that_does_not_exist"
```

### Expected Result

```text
Job Status: FAILED
Error Key: REPORT_ACCESS_GROUP_INVALID
Category: infrastructure
```

### Why this test is required

The automation must not apply invalid group ownership to a report artifact.

---

## TC07 — Write Verification Failure

### Objective

Verify standardized handling when the Execution Environment cannot create the probe file.

### Controlled Setup

Use a test condition where the configured temporary location is not writable.

The production configuration should still remain:

```text
/tmp
```

### Expected Result

```text
Job Status: FAILED
Error Key: REPORT_STORAGE_WRITE_FAILED
Category: infrastructure
```

### Expected Behavior

The failure must be routed through `common_logging`.

### Why this test is required

Writing is the first real filesystem capability required for report generation.

---

## TC08 — Read Verification Failure

### Objective

Verify standardized handling when a probe file is written but cannot be correctly read back.

### Controlled Setup

Force a controlled read/content mismatch condition.

### Expected Result

```text
Job Status: FAILED
Error Key: REPORT_STORAGE_READ_FAILED
Category: infrastructure
```

### Why this test is required

A successful write alone is insufficient. The downstream report flow must also be able to read the generated artifact.

---

## TC09 — List Verification Failure

### Objective

Verify standardized handling when the probe artifact cannot be discovered by the list/find operation.

### Expected Result

```text
Job Status: FAILED
Error Key: REPORT_STORAGE_LIST_FAILED
Category: infrastructure
```

### Why this test is required

This verifies the exact list capability required by the story and also proves the terminal error path.

---

## TC10 — Hidden Probe File Listing

### Objective

Verify the fix identified during real AAP testing.

### Background

The verification probe is intentionally hidden:

```text
/tmp/.akamai_report_probe_...
```

`ansible.builtin.find` ignores hidden files unless explicitly enabled.

The final task therefore requires:

```yaml
hidden: true
```

### Expected Result

```text
Job Status: SUCCESS

Probe write: PASS
Probe read: PASS
Probe list/find: PASS
```

### Why this test is required

This test proves the production fix for the exact issue found during AAP runtime testing.

---

## TC11 — AAP Check Mode / Dry Run

### Objective

Verify that the storage task supports dry-run behavior without performing real probe I/O.

### AAP Setup

Run the Job Template using:

```text
Job Type = Check
```

Do not simulate Check Mode using an Extra Var.

### Inputs

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "REPORT-CHECK-001"

report_storage_verify_io: true

report_storage_version: "252"

report_storage_status: "success"
```

### Expected Result

```text
Job Status: SUCCESS

dry_run: true
```

Expected behavior:

```text
Configuration validation: PASS
/tmp availability validation: PASS
Dry-run naming validation: PASS
Probe write/read/list: SKIPPED
```

### Why this test is required

It proves that the SSPA-7 implementation can be safely exercised in AAP Check Mode.

---

## TC12 — Valid Group-Readable Mode

### Objective

Verify the optional access-group flow.

### Preconditions

A valid Linux group must exist inside the Execution Environment.

### Inputs

```yaml
report_storage_access_group: "<VALID_EE_GROUP>"
```

### Expected Result

```text
Job Status: SUCCESS
Effective file mode: 0640
```

Expected permissions:

```text
Owner  → read/write
Group  → read
Others → none
```

### Priority

```text
P2
```

### Why this test is useful

It verifies the optional shared-read access model without weakening permissions for everyone else.

---

## TC13 — Default Private File Mode

### Objective

Verify the default secure file mode when no group is supplied.

### Inputs

```yaml
report_storage_access_group: ""
```

### Expected Result

```text
Job Status: SUCCESS
Effective file mode: 0600
```

Expected permissions:

```text
Owner  → read/write
Group  → none
Others → none
```

### Why this test is required

This is the default production behavior.

---

## TC14 — Probe Cleanup

### Objective

Verify that the temporary verification artifact is removed after successful validation.

### Inputs

Use the normal TC01 input.

### Expected Result

After successful execution:

```text
/tmp/.akamai_report_probe_<execution>_<timestamp>
```

must no longer exist.

### Expected Job Result

```text
SUCCESS
```

### Why this test is required

The validation task must not leave unnecessary hidden probe artifacts behind inside the Execution Environment.

---

# 4. Minimum Test Set for Sprint / Showcase

If time is limited, these are the most valuable cases to demonstrate:

| Demo Order | Test | What It Proves |
|---|---|---|
| 1 | TC01 | Complete happy path |
| 2 | TC13 | Default secure `0600` permission |
| 3 | TC10 | Hidden probe listing fix works |
| 4 | TC11 | AAP Check Mode works without writes |
| 5 | TC04 | Invalid version is rejected |
| 6 | TC05 | Unsafe permissions are rejected |
| 7 | TC06 | Invalid OS group is rejected |
| 8 | TC09 | Standardized list failure handling |
| 9 | TC14 | Probe cleanup works |

TC02, TC03, TC07, TC08, and TC12 are useful secondary tests, but they do not need to dominate the showcase.

---

# 5. Functional Coverage Mapping

| SSPA-7 Behavior | Covered By |
|---|---|
| `/tmp` storage available | TC01, TC11 |
| Write capability | TC01, TC07 |
| Read capability | TC01, TC08 |
| List capability | TC01, TC09, TC10 |
| Hidden probe discoverability | TC10 |
| Probe cleanup | TC14 |
| Secure default permissions | TC13 |
| Optional group access | TC06, TC12 |
| Report naming convention | TC01, TC04 |
| Invalid configuration handling | TC05 |
| Dry-run / Check Mode | TC11 |
| Standardized common logging failure path | TC04–TC09 |
| AAP artifact publication | TC01 |
| Temporary/ephemeral storage behavior | TC01, TC11 |

---

# 6. Expected Error Matrix

| Failure Type | Error Category | Error Key |
|---|---|---|
| Invalid storage configuration | `validation` | `REPORT_STORAGE_CONFIG_INVALID` |
| Invalid access group | `infrastructure` | `REPORT_ACCESS_GROUP_INVALID` |
| `/tmp` unavailable | `infrastructure` | `REPORT_STORAGE_UNAVAILABLE` |
| Probe write failure | `infrastructure` | `REPORT_STORAGE_WRITE_FAILED` |
| Probe read failure | `infrastructure` | `REPORT_STORAGE_READ_FAILED` |
| Probe list failure | `infrastructure` | `REPORT_STORAGE_LIST_FAILED` |
| Dry-run validation failure | appropriate validation/infrastructure classification | `REPORT_STORAGE_DRY_RUN_FAILED` |
| Invalid report filename/version/status | `validation` | `REPORT_NAME_INVALID` |

---

# 7. Test Execution Evidence to Capture

For each test, capture only the evidence needed to prove expected behavior:

1. AAP Job ID
2. Test Case ID
3. Extra Vars used
4. Job Type (`Run` or `Check`)
5. Final AAP Job Status
6. Expected error key for negative tests
7. `report_storage` artifact for successful tests
8. Effective file mode
9. Relevant `common_logging` message
10. Confirmation that the probe was removed after successful execution

Do not capture credentials, tokens, passwords, or unrelated AAP environment data.

---

# 8. Expected Successful AAP Artifact

For the normal happy path:

```yaml
report_storage:
  root: /tmp
  filename_format: "{timestamp}_{version}_{status}.txt"
  report_filename: "<timestamp>_252_success.txt"
  report_path: "/tmp/<timestamp>_252_success.txt"
  file_mode: "0600"
  ephemeral: true
  dry_run: false
```

For Check Mode:

```yaml
report_storage:
  root: /tmp
  filename_format: "{timestamp}_{version}_{status}.txt"
  report_filename: "<timestamp>_252_success.txt"
  report_path: "/tmp/<timestamp>_252_success.txt"
  file_mode: "0600"
  ephemeral: true
  dry_run: true
```

---

# 9. Important Scope Note

The current SSPA-7 implementation validates **temporary report storage inside the AAP Execution Environment**.

It does not currently implement:

```text
Persistent storage
15-day retention
Email delivery
Deployment report content generation
Cross-job file persistence
```

The expected runtime behavior is:

```text
AAP Job starts
    ↓
Temporary report workspace available under /tmp
    ↓
Report generated/consumed during same execution
    ↓
Downstream delivery logic uses the file
    ↓
Execution Environment terminates
    ↓
Temporary file is not treated as persistent storage
```

Therefore, tests should validate the temporary workspace behavior rather than attempting to prove long-term retention inside the Execution Environment.

---

# 10. Final Recommended Acceptance Set

For SSPA-7 to be considered functionally demonstrated, the following should behave as expected:

```text
Positive:
  TC01 - Full happy path
  TC10 - Hidden probe list verification
  TC11 - AAP Check Mode
  TC13 - Default secure permission
  TC14 - Probe cleanup

Negative / protection:
  TC04 - Invalid version
  TC05 - Unsafe permission
  TC06 - Invalid access group
  TC09 - List verification failure
```

This set gives strong coverage of the implemented story without adding unnecessary or repetitive test cases.
