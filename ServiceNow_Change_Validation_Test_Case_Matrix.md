# ServiceNow Change Validation — Complete Test Case Matrix and AAP Extra Vars

This document is aligned with the current ServiceNow validation implementation:

```text
roles/akamai/tasks/snow_validation.yml
roles/akamai/tasks/snow_live_attempt.yml
roles/akamai/mocks/snow/*.json
```

The validation contract is:

```text
Change exists
    AND
Returned CHG matches requested CHG
    AND
State = Implement
    AND
Approval = Approved
    AND
Planned start exists
    AND
Planned end exists
    AND
Planned start < planned end
    AND
Current UTC is inside the planned window
        ↓
PASS
```

Mock and live modes use the same downstream business validation after the change is normalized into `snow_chg`.

---

# 1. Standard Base Extra Vars

For mock testing, use this as the standard base and change only the CHG number and scenario for each test case.

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TEST-001"

akamai_chg_number: "CHG0012345"

snow_mode: "mock"

snow_mock_scenario: "valid_implement_approved"
```

Notes:

- `akamai.environment.name` is used by `common_logging`.
- `common_logging_request_id` is used to correlate logs for the test execution.
- `akamai_chg_number` is the requested ServiceNow Change number.
- `snow_mode` must be exactly `mock` or `live`.
- `snow_mock_scenario` is used only in mock mode.
- For successful HTTP 200 fixtures, the CHG number in Extra Vars must match the `number` inside the selected JSON fixture unless the purpose of the test is specifically to validate CHG mismatch handling.

---

# 2. Complete Mock Test Matrix

| TC | Test Scenario | CHG Input | Mock Scenario | Expected Result | Expected Error Key |
|---|---|---|---|---|---|
| TC01 | Happy path: Implement + Approved + valid window | CHG0012345 | valid_implement_approved | PASS | None |
| TC02 | Implement but not Approved | CHG0012347 | invalid_not_approved | FAIL | SERVICENOW_STATUS_INVALID |
| TC03 | Approved but not Implement | CHG0012346 | invalid_state | FAIL | SERVICENOW_STATUS_INVALID |
| TC04 | Planned window missing | CHG0012350 | missing_window | FAIL | SERVICENOW_WINDOW_MISSING |
| TC05 | Planned window has not started | CHG0012348 | outside_window_early | FAIL | SERVICENOW_OUTSIDE_WINDOW |
| TC06 | Planned window already expired | CHG0012349 | outside_window_late | FAIL | SERVICENOW_OUTSIDE_WINDOW |
| TC07 | Change not found | CHG0099999 | not_found | FAIL | SERVICENOW_CHANGE_NOT_FOUND |
| TC08 | Authentication failure | CHG0012345 | auth_failure | FAIL | SERVICENOW_AUTH_FAILED |
| TC09 | Timeout and retry exhaustion | CHG0012345 | timeout | FAIL after 3 attempts | SERVICENOW_API_TIMEOUT |
| TC10 | Invalid CHG prefix | INC0012345 | valid_implement_approved | FAIL before retrieval | SERVICENOW_CONFIG_INVALID |
| TC11 | Invalid alphanumeric CHG number | CHGABC123 | valid_implement_approved | FAIL before retrieval | SERVICENOW_CONFIG_INVALID |
| TC12 | Empty CHG number | empty string | valid_implement_approved | FAIL before retrieval | SERVICENOW_CONFIG_INVALID |
| TC13 | Invalid mode | CHG0012345 | valid_implement_approved | FAIL before retrieval | SERVICENOW_CONFIG_INVALID |
| TC14 | Invalid/nonexistent mock scenario | CHG0012345 | invalid_scenario_name | FAIL before file lookup | SERVICENOW_CONFIG_INVALID |
| TC15 | Unsafe mock path traversal attempt | CHG0012345 | ../valid_implement_approved | FAIL before file lookup | SERVICENOW_CONFIG_INVALID |
| TC16 | Requested CHG does not match returned CHG | CHG0019999 | valid_implement_approved | FAIL after normalization | SERVICENOW_RESPONSE_INVALID |
| TC17 | Case-sensitive invalid mode | CHG0012345 | valid_implement_approved | FAIL before retrieval | SERVICENOW_CONFIG_INVALID |
| TC18 | Valid mode but mock scenario case mismatch | CHG0012345 | VALID_IMPLEMENT_APPROVED | FAIL before retrieval | SERVICENOW_CONFIG_INVALID |

---

# 3. TC01 — Happy Path

Purpose:

```text
Validate that a valid ServiceNow change passes all gates.
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC01"

akamai_chg_number: "CHG0012345"

snow_mode: "mock"

snow_mock_scenario: "valid_implement_approved"
```

Expected:

```text
Mock JSON loaded                         PASS
HTTP status = 200                        PASS
Returned CHG = requested CHG             PASS
sys_id present                           PASS
State = Implement                        PASS
Approval = Approved                      PASS
Planned start present                    PASS
Planned end present                      PASS
Start < End                              PASS
Current UTC inside planned window        PASS
Read-only                                TRUE
Overall Job Result                       SUCCESS
```

Expected AAP result:

```yaml
snow_validation_result:
  result: "PASSED"
  mode: "mock"
  source: "mock:valid_implement_approved"
  change_number: "CHG0012345"
  state: "implement"
  approval: "approved"
  read_only: true
```

---

# 4. TC02 — Implement but Not Approved

Purpose:

```text
Prove that Implement alone is not sufficient.
Both Implement AND Approved are mandatory.
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC02"

akamai_chg_number: "CHG0012347"

snow_mode: "mock"

snow_mock_scenario: "invalid_not_approved"
```

Fixture condition:

```text
State      = Implement
Approval   = Not Yet Requested
```

Expected:

```text
State validation       PASS
Approval validation    FAIL
Overall                 FAIL
```

Expected error:

```text
SERVICENOW_STATUS_INVALID
category=business-rule
```

---

# 5. TC03 — Approved but Not Implement

Purpose:

```text
Prove that Approved alone is not sufficient.
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC03"

akamai_chg_number: "CHG0012346"

snow_mode: "mock"

snow_mock_scenario: "invalid_state"
```

Fixture condition:

```text
State      = Draft
Approval   = Approved
```

Expected:

```text
State validation       FAIL
Overall                 FAIL
```

Expected error:

```text
SERVICENOW_STATUS_INVALID
category=business-rule
```

---

# 6. TC04 — Missing Planned Window

Purpose:

```text
Verify that a valid state and approval cannot bypass
the mandatory implementation window.
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC04"

akamai_chg_number: "CHG0012350"

snow_mode: "mock"

snow_mock_scenario: "missing_window"
```

Fixture condition:

```text
State       = Implement
Approval    = Approved
Start date  = empty
End date    = empty
```

Expected:

```text
SERVICENOW_WINDOW_MISSING
category=business-rule
```

---

# 7. TC05 — Before Planned Window

Purpose:

```text
Verify deployment is blocked when the approved window
has not yet opened.
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC05"

akamai_chg_number: "CHG0012348"

snow_mode: "mock"

snow_mock_scenario: "outside_window_early"
```

Expected logic:

```text
Current UTC < planned start
        ↓
BLOCK
```

Expected error:

```text
SERVICENOW_OUTSIDE_WINDOW
category=business-rule
```

---

# 8. TC06 — After Planned Window

Purpose:

```text
Verify deployment is blocked when the approved
implementation window has expired.
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC06"

akamai_chg_number: "CHG0012349"

snow_mode: "mock"

snow_mock_scenario: "outside_window_late"
```

Expected logic:

```text
Current UTC > planned end
        ↓
BLOCK
```

Expected error:

```text
SERVICENOW_OUTSIDE_WINDOW
category=business-rule
```

---

# 9. TC07 — Change Not Found

Purpose:

```text
Validate that a nonexistent ServiceNow change blocks execution.
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC07"

akamai_chg_number: "CHG0099999"

snow_mode: "mock"

snow_mock_scenario: "not_found"
```

Expected error:

```text
SERVICENOW_CHANGE_NOT_FOUND
category=business-rule
```

Expected job result:

```text
FAILED
```

---

# 10. TC08 — Authentication Failure

Purpose:

```text
Validate ServiceNow authentication failure handling.
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC08"

akamai_chg_number: "CHG0012345"

snow_mode: "mock"

snow_mock_scenario: "auth_failure"
```

Expected error:

```text
SERVICENOW_AUTH_FAILED
category=auth
code=2
```

Expected job result:

```text
FAILED
```

---

# 11. TC09 — Timeout and Retry Exhaustion

Purpose:

```text
Validate the retry requirement and terminal timeout handling.
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC09"

akamai_chg_number: "CHG0012345"

snow_mode: "mock"

snow_mock_scenario: "timeout"
```

Expected execution:

```text
Attempt 1
   ↓
HTTP 504
   ↓
Wait 2 seconds

Attempt 2
   ↓
HTTP 504
   ↓
Wait 4 seconds

Attempt 3
   ↓
HTTP 504
   ↓
FAIL
```

Expected error:

```text
SERVICENOW_API_TIMEOUT
category=timeout
code=4
```

---

# 12. TC10 — Invalid CHG Prefix

Purpose:

```text
Verify invalid request input fails before mock/live retrieval.
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC10"

akamai_chg_number: "INC0012345"

snow_mode: "mock"

snow_mock_scenario: "valid_implement_approved"
```

Expected:

```text
SERVICENOW_CONFIG_INVALID
category=validation
```

No mock business validation should execute.

---

# 13. TC11 — Invalid Alphanumeric CHG

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC11"

akamai_chg_number: "CHGABC123"

snow_mode: "mock"

snow_mock_scenario: "valid_implement_approved"
```

Expected:

```text
SERVICENOW_CONFIG_INVALID
```

---

# 14. TC12 — Empty CHG Number

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC12"

akamai_chg_number: ""

snow_mode: "mock"

snow_mock_scenario: "valid_implement_approved"
```

Expected:

```text
SERVICENOW_CONFIG_INVALID
```

---

# 15. TC13 — Invalid ServiceNow Mode

Purpose:

```text
Verify only mock and live modes are accepted.
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC13"

akamai_chg_number: "CHG0012345"

snow_mode: "dummy"

snow_mock_scenario: "valid_implement_approved"
```

Expected:

```text
SERVICENOW_CONFIG_INVALID
```

---

# 16. TC14 — Invalid Mock Scenario

Purpose:

```text
Verify arbitrary/unapproved mock fixtures cannot be selected.
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC14"

akamai_chg_number: "CHG0012345"

snow_mode: "mock"

snow_mock_scenario: "invalid_scenario_name"
```

Expected:

```text
SERVICENOW_CONFIG_INVALID
```

The JSON lookup should not run.

---

# 17. TC15 — Path Traversal Attempt

Purpose:

```text
Validate that a user cannot point the mock loader outside
the approved mock scenario allow-list.
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC15"

akamai_chg_number: "CHG0012345"

snow_mode: "mock"

snow_mock_scenario: "../valid_implement_approved"
```

Expected:

```text
SERVICENOW_CONFIG_INVALID
```

The file lookup must not execute.

---

# 18. TC16 — Requested CHG vs Returned CHG Mismatch

Purpose:

```text
Prove that the automation validates the returned record
belongs to the requested CHG.
```

The selected valid fixture returns:

```text
CHG0012345
```

but the request deliberately asks for:

```text
CHG0019999
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC16"

akamai_chg_number: "CHG0019999"

snow_mode: "mock"

snow_mock_scenario: "valid_implement_approved"
```

Expected:

```text
SERVICENOW_RESPONSE_INVALID
category=api
```

Expected logic:

```text
Requested CHG = CHG0019999
Returned CHG  = CHG0012345
        ↓
Mismatch
        ↓
BLOCK
```

---

# 19. TC17 — Invalid Mode Case

The implementation currently treats mode values as exact.

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC17"

akamai_chg_number: "CHG0012345"

snow_mode: "Mock"

snow_mock_scenario: "valid_implement_approved"
```

Expected:

```text
SERVICENOW_CONFIG_INVALID
```

Only:

```text
mock
live
```

are valid.

---

# 20. TC18 — Mock Scenario Case Mismatch

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC18"

akamai_chg_number: "CHG0012345"

snow_mode: "mock"

snow_mock_scenario: "VALID_IMPLEMENT_APPROVED"
```

Expected:

```text
SERVICENOW_CONFIG_INVALID
```

This validates the explicit mock scenario allow-list.

---

# 21. Recommended Mock Showcase Set

If you do not want to demonstrate every negative configuration test during a sprint review, these are the strongest functional cases to showcase:

```text
TC01  Happy path
TC02  Implement but not Approved
TC03  Approved but not Implement
TC04  Missing planned window
TC05  Before planned window
TC06  After planned window
TC07  Change not found
TC08  Authentication failure
TC09  Timeout and retry exhaustion
TC16  Requested CHG vs returned CHG mismatch
```

These demonstrate the major functional and acceptance-criteria behaviors.

---

# 22. Live ServiceNow Test Matrix

These test cases become available when:

```text
ServiceNow API connectivity is available
AAP ServiceNow Credential is attached
Execution Environment has servicenow.itsm
Certificate trust/network routing is working
```

The ServiceNow host/username/password must **not** be passed as Extra Vars.

AAP injects:

```text
SN_HOST
SN_USERNAME
SN_PASSWORD
```

| Live TC | Real ServiceNow Condition | Expected |
|---|---|---|
| LIVE01 | Existing CHG, Implement, Approved, inside window | PASS |
| LIVE02 | Existing CHG, Implement, not Approved | SERVICENOW_STATUS_INVALID |
| LIVE03 | Existing CHG, Approved, not Implement | SERVICENOW_STATUS_INVALID |
| LIVE04 | Existing CHG, valid state/approval, before window | SERVICENOW_OUTSIDE_WINDOW |
| LIVE05 | Existing CHG, valid state/approval, after window | SERVICENOW_OUTSIDE_WINDOW |
| LIVE06 | Existing CHG with missing planned window | SERVICENOW_WINDOW_MISSING |
| LIVE07 | Nonexistent CHG | SERVICENOW_CHANGE_NOT_FOUND |
| LIVE08 | AAP ServiceNow credential not attached | SERVICENOW_AUTH_FAILED |
| LIVE09 | Invalid/expired ServiceNow credential | SERVICENOW_AUTH_FAILED |
| LIVE10 | ServiceNow/network timeout in controlled test | 3 attempts then SERVICENOW_API_TIMEOUT |
| LIVE11 | ServiceNow rate limit in controlled test | retry/final SERVICENOW_RATE_LIMITED |
| LIVE12 | Invalid/malformed ServiceNow response | SERVICENOW_RESPONSE_INVALID |

---

# 23. LIVE01 — Valid Live Change

Prerequisite:

The selected real CHG must be:

```text
State       = Implement
Approval    = Approved
Current UTC = within planned start/end
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-LIVE01"

akamai_chg_number: "CHG0123456"

snow_mode: "live"
```

Do not pass:

```text
SN_HOST
SN_USERNAME
SN_PASSWORD
```

in Extra Vars.

Expected:

```text
AAP credential detected
        ↓
servicenow.itsm.change_request_info
        ↓
Exactly one record
        ↓
Requested CHG matches returned CHG
        ↓
Implement
        AND
Approved
        ↓
Inside planned window
        ↓
PASS
```

---

# 24. LIVE02 — Implement but Not Approved

Select a real test CHG that is:

```text
State     = Implement
Approval  != Approved
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-LIVE02"

akamai_chg_number: "CHG<REAL_TEST_CHG>"

snow_mode: "live"
```

Expected:

```text
SERVICENOW_STATUS_INVALID
```

---

# 25. LIVE03 — Approved but Not Implement

Select a real CHG that is:

```text
State     != Implement
Approval  = Approved
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-LIVE03"

akamai_chg_number: "CHG<REAL_TEST_CHG>"

snow_mode: "live"
```

Expected:

```text
SERVICENOW_STATUS_INVALID
```

---

# 26. LIVE04 — Before Approved Window

Select a real CHG with:

```text
State       = Implement
Approval    = Approved
Now         < Planned Start
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-LIVE04"

akamai_chg_number: "CHG<REAL_TEST_CHG>"

snow_mode: "live"
```

Expected:

```text
SERVICENOW_OUTSIDE_WINDOW
```

---

# 27. LIVE05 — After Approved Window

Select a real CHG with:

```text
State       = Implement
Approval    = Approved
Now         > Planned End
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-LIVE05"

akamai_chg_number: "CHG<REAL_TEST_CHG>"

snow_mode: "live"
```

Expected:

```text
SERVICENOW_OUTSIDE_WINDOW
```

---

# 28. LIVE06 — Missing Planned Window

Use only a controlled test CHG where planned dates are intentionally absent.

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-LIVE06"

akamai_chg_number: "CHG<REAL_TEST_CHG>"

snow_mode: "live"
```

Expected:

```text
SERVICENOW_WINDOW_MISSING
```

---

# 29. LIVE07 — Nonexistent Change

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-LIVE07"

akamai_chg_number: "CHG<NONEXISTENT_TEST_NUMBER>"

snow_mode: "live"
```

Expected:

```text
SERVICENOW_CHANGE_NOT_FOUND
```

---

# 30. LIVE08 — AAP Credential Not Attached

Purpose:

```text
Validate fail-fast behavior when the Job Template has no
ServiceNow AAP credential attached.
```

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-LIVE08"

akamai_chg_number: "CHG0123456"

snow_mode: "live"
```

Run this only in a controlled test Job Template without the ServiceNow credential.

Expected:

```text
SERVICENOW_AUTH_FAILED
category=auth
```

The module call should not proceed.

---

# 31. LIVE09 — Invalid ServiceNow Credential

This should only be performed with an approved non-production test credential or controlled negative test configuration.

Extra Vars are unchanged:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-LIVE09"

akamai_chg_number: "CHG0123456"

snow_mode: "live"
```

Expected:

```text
Live attempt 1 failed
Live attempt 2 failed
Live attempt 3 failed
        ↓
SERVICENOW_AUTH_FAILED
```

Do not place a bad password directly in Extra Vars.

---

# 32. LIVE10 — Controlled Timeout

This requires a controlled integration test condition such as an approved unreachable test endpoint/network path.

Extra Vars:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-LIVE10"

akamai_chg_number: "CHG0123456"

snow_mode: "live"
```

Expected:

```text
Attempt 1
   ↓ timeout
Wait 2 seconds

Attempt 2
   ↓ timeout
Wait 4 seconds

Attempt 3
   ↓ timeout

SERVICENOW_API_TIMEOUT
```

Do not deliberately disrupt a production ServiceNow endpoint merely to create this test.

---

# 33. Extra-Var Validation Summary

The following inputs are intended to be supplied by normal job inputs:

```yaml
akamai_chg_number: "CHG0012345"
snow_mode: "mock"
snow_mock_scenario: "valid_implement_approved"
```

Optional logging context for testing:

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC01"
```

The following are **not** survey/Extra Var inputs:

```text
SN_HOST
SN_USERNAME
SN_PASSWORD
snow_required_state
snow_required_approval
snow_live_attempt_plan
snow_error_keys
snow_log_stages
```

`SN_HOST`, `SN_USERNAME`, and `SN_PASSWORD` come from the AAP ServiceNow credential.

The remaining values are internal role configuration/business-rule constants.

---

# 34. Expected Error Matrix

| Error Key | Meaning | Expected Category |
|---|---|---|
| SERVICENOW_CONFIG_INVALID | Invalid CHG/mode/configuration | validation |
| SERVICENOW_MOCK_INVALID | Invalid/unreadable mock fixture structure | validation |
| SERVICENOW_AUTH_FAILED | Missing/rejected ServiceNow credentials | auth |
| SERVICENOW_API_FAILED | General ServiceNow retrieval failure | api |
| SERVICENOW_API_TIMEOUT | Timeout after retry processing | timeout |
| SERVICENOW_RATE_LIMITED | ServiceNow rate limiting | rate-limit |
| SERVICENOW_CHANGE_NOT_FOUND | Requested CHG does not exist | business-rule |
| SERVICENOW_RESPONSE_INVALID | Invalid/ambiguous/mismatched response | api |
| SERVICENOW_STATUS_INVALID | State and/or approval gate failed | business-rule |
| SERVICENOW_WINDOW_MISSING | Planned start/end absent | business-rule |
| SERVICENOW_WINDOW_INVALID | Date format/order invalid | business-rule |
| SERVICENOW_OUTSIDE_WINDOW | Current time outside allowed window | business-rule |

---

# 35. Recommended Execution Order

For AAP mock validation, run in this order:

```text
TC01  Positive baseline
TC02  Implement / not Approved
TC03  Approved / not Implement
TC04  Missing window
TC05  Before window
TC06  After window
TC07  Not found
TC08  Authentication failure
TC09  Timeout / retries
TC16  CHG mismatch
TC10  Invalid CHG
TC13  Invalid mode
TC14  Invalid scenario
TC15  Path traversal
TC17  Case-sensitive mode
TC18  Case-sensitive scenario
```

This order validates the happy path first, then business gates, infrastructure/error handling, response integrity, and finally input hardening.

---

# 36. Test Evidence to Capture from AAP

For every test, capture:

```text
Job ID
Test Case ID
Extra Vars
Expected Result
Actual Result
Final Job Status
common_logging error key/result
AAP Artifacts / set_stats
Relevant task output
Execution timestamp
```

For successful validation specifically verify the AAP artifact contains:

```yaml
snow_validation_result:
  result: "PASSED"
  mode: "mock"
  source: "mock:valid_implement_approved"
  change_number: "CHG0012345"
  state: "implement"
  approval: "approved"
  read_only: true
```

For live validation verify:

```yaml
snow_validation_result:
  result: "PASSED"
  mode: "live"
  source: "live"
  read_only: true
```

---

# 37. Acceptance-Oriented Coverage Summary

The test set demonstrates:

```text
Successful ServiceNow validation
Implement AND Approved enforcement
Missing window blocking
Early-window blocking
Expired-window blocking
Change-not-found handling
Authentication failure handling
Timeout retry behavior
Read-only validation
Requested/returned CHG identity verification
Input validation
Mock scenario allow-list protection
Common logging error classification
Mock-to-live business-logic consistency
```

Live API-specific tests should be executed once the ServiceNow AAP credential, network connectivity, certificate trust, and approved live test records are available.
