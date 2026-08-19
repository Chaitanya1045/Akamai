# SSPA-11 — ServiceNow Change Validation Test Cases & Test Matrix

## Purpose

This test set is intentionally limited to the **core behavior implemented for SSPA-11**.  
It avoids duplicate or low-value tests and focuses on:

- Input/configuration validation
- ServiceNow authentication handling
- Change existence
- Required state = `Implement`
- Required approval = `Approved`
- Planned implementation window validation
- Retry/timeout behavior
- AAP credential injection
- Successful live retrieval

---

# 1. Recommended Test Matrix

| TC ID | Scenario | Mode | Main Input / Setup | Expected Result | Expected Error Key / Output | Priority |
|---|---|---|---|---|---|---|
| TC01 | Valid change request | Mock | `CHG0012345`, `valid_implement_approved` | PASS | `snow_validation_result.result = PASSED` | P1 |
| TC02 | Invalid CHG number format | Mock | `12345` or `INC0012345` | FAIL before mock/API processing | `SERVICENOW_CONFIG_INVALID` | P1 |
| TC03 | Invalid mock scenario | Mock | `snow_mock_scenario: invalid_test` | FAIL during local configuration validation | `SERVICENOW_CONFIG_INVALID` | P2 |
| TC04 | ServiceNow authentication failure | Mock | `auth_failure` | FAIL | `SERVICENOW_AUTH_FAILED` | P1 |
| TC05 | Change not found | Mock | `not_found` | FAIL | `SERVICENOW_CHANGE_NOT_FOUND` | P1 |
| TC06 | Change not in Implement state | Mock | `CHG0012346`, `invalid_state` | FAIL | `SERVICENOW_STATUS_INVALID` | P1 |
| TC07 | Change not Approved | Mock | `CHG0012347`, `invalid_not_approved` | FAIL | `SERVICENOW_STATUS_INVALID` | P1 |
| TC08 | Planned implementation window missing | Mock | `CHG0012350`, `missing_window` | FAIL | `SERVICENOW_WINDOW_MISSING` | P1 |
| TC09 | Execution before implementation window | Mock | `CHG0012348`, `outside_window_early` | FAIL | `SERVICENOW_OUTSIDE_WINDOW` | P1 |
| TC10 | Execution after implementation window | Mock | `CHG0012349`, `outside_window_late` | FAIL | `SERVICENOW_OUTSIDE_WINDOW` | P1 |
| TC11 | ServiceNow timeout / retry exhaustion | Mock | `timeout` | 3 simulated attempts, then FAIL | `SERVICENOW_API_TIMEOUT` | P1 |
| TC12 | AAP ServiceNow credential not attached/injected | Live | `snow_mode: live`, no `snow_host/snow_username/snow_password` | FAIL before API call | `SERVICENOW_AUTH_FAILED` | P1 |
| TC13 | Valid live ServiceNow change | Live | Valid CHG + valid AAP Credential | PASS | `snow_validation_result.result = PASSED`, `source = live` | P1 |
| TC14 | Live CHG returns zero records | Live | Valid format CHG that does not exist | FAIL | `SERVICENOW_CHANGE_NOT_FOUND` | P1 |
| TC15 | Live response contains multiple matching records | Live | Controlled/Mocked live response with more than one record | FAIL | `SERVICENOW_RESPONSE_INVALID` | P2 |

> **P1** = required for functional acceptance / demonstration  
> **P2** = useful defensive validation, but not necessary to spend equal demo time on

---

# 2. Detailed Test Cases

## TC01 — Valid Change Request

### Objective
Verify the complete successful SSPA-11 business-validation path.

### Inputs

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC01"

akamai_chg_number: "CHG0012345"

snow_mode: "mock"

snow_mock_scenario: "valid_implement_approved"
```

### Fixture

```text
valid_implement_approved.json
```

### Validated Conditions

```text
CHG exists
    ↓
Returned CHG matches requested CHG
    ↓
sys_id exists
    ↓
State = Implement
    ↓
Approval = Approved
    ↓
Planned start/end exist
    ↓
Current execution time is inside window
    ↓
PASS
```

### Expected Result

```text
Job Status: SUCCESS

akamai_chg_validated: true

snow_validation_result:
  result: PASSED
  mode: mock
  source: mock:valid_implement_approved
  change_number: CHG0012345
  state: implement
  approval: approved
  read_only: true
```

### Why this test is required
This is the primary positive test and proves the complete SSPA-11 validation flow.

---

## TC02 — Invalid Change Number Format

### Objective
Verify fail-fast validation before any mock lookup or ServiceNow API activity.

### Inputs

```yaml
akamai_chg_number: "12345"

snow_mode: "mock"

snow_mock_scenario: "valid_implement_approved"
```

Alternative invalid values:

```text
INC0012345
CHGABC123
CHG-0012345
```

### Expected Result

```text
Job Status: FAILED
Error Key: SERVICENOW_CONFIG_INVALID
Category: validation
```

### Expected Behavior
The execution must stop before ServiceNow/mock processing.

### Why this test is required
It proves that invalid business input is rejected locally and does not reach an external system.

---

## TC03 — Invalid Mock Scenario

### Objective
Verify local validation of the mock scenario name.

### Inputs

```yaml
akamai_chg_number: "CHG0012345"

snow_mode: "mock"

snow_mock_scenario: "invalid_test"
```

### Expected Result

```text
Job Status: FAILED
Error Key: SERVICENOW_CONFIG_INVALID
Category: validation
```

### Why this test is useful
It confirms that only approved mock scenarios can be selected.

---

## TC04 — Authentication Failure

### Objective
Verify ServiceNow authentication failure handling.

### Inputs

```yaml
akamai_chg_number: "CHG0012345"

snow_mode: "mock"

snow_mock_scenario: "auth_failure"
```

### Mock Response

```text
HTTP 401
```

### Expected Result

```text
Job Status: FAILED
Error Key: SERVICENOW_AUTH_FAILED
Category: auth
```

### Why this test is required
Authentication failure must be clearly distinguishable from business-rule failures such as invalid state or approval.

---

## TC05 — Change Not Found

### Objective
Verify behavior when the supplied CHG does not exist.

### Inputs

```yaml
akamai_chg_number: "CHG0012351"

snow_mode: "mock"

snow_mock_scenario: "not_found"
```

### Mock Response

```text
HTTP 404
```

### Expected Result

```text
Job Status: FAILED
Error Key: SERVICENOW_CHANGE_NOT_FOUND
Category: business-rule
```

### Why this test is required
Deployment must not continue if the change record does not exist.

---

## TC06 — Change Not in Implement State

### Objective
Verify the mandatory ServiceNow state gate.

### Inputs

```yaml
akamai_chg_number: "CHG0012346"

snow_mode: "mock"

snow_mock_scenario: "invalid_state"
```

### Fixture Condition

```text
state    = Draft
approval = Approved
```

### Expected Result

```text
Job Status: FAILED
Error Key: SERVICENOW_STATUS_INVALID
Category: business-rule
```

### Why this test is required
The change must be in `Implement` state before deployment can continue.

---

## TC07 — Change Not Approved

### Objective
Verify the mandatory approval gate independently of the state gate.

### Inputs

```yaml
akamai_chg_number: "CHG0012347"

snow_mode: "mock"

snow_mock_scenario: "invalid_not_approved"
```

### Fixture Condition

```text
state    = Implement
approval = Not Yet Requested
```

### Expected Result

```text
Job Status: FAILED
Error Key: SERVICENOW_STATUS_INVALID
Category: business-rule
```

### Why this test is required
This proves that `Implement` alone is not sufficient.  
Both conditions must be satisfied:

```text
State = Implement
AND
Approval = Approved
```

---

## TC08 — Missing Planned Implementation Window

### Objective
Verify that planned start and planned end are mandatory.

### Inputs

```yaml
akamai_chg_number: "CHG0012350"

snow_mode: "mock"

snow_mock_scenario: "missing_window"
```

### Fixture Condition

```text
start_date = ""
end_date   = ""
```

### Expected Result

```text
Job Status: FAILED
Error Key: SERVICENOW_WINDOW_MISSING
Category: business-rule
```

### Why this test is required
The automation must not deploy without an approved implementation window.

---

## TC09 — Execution Before Approved Window

### Objective
Verify that deployment cannot start before the ServiceNow planned start time.

### Inputs

```yaml
akamai_chg_number: "CHG0012348"

snow_mode: "mock"

snow_mock_scenario: "outside_window_early"
```

### Expected Result

```text
Job Status: FAILED
Error Key: SERVICENOW_OUTSIDE_WINDOW
Category: business-rule
```

### Why this test is required
It validates the lower boundary of the implementation-window gate.

---

## TC10 — Execution After Approved Window

### Objective
Verify that deployment cannot start after the ServiceNow planned end time.

### Inputs

```yaml
akamai_chg_number: "CHG0012349"

snow_mode: "mock"

snow_mock_scenario: "outside_window_late"
```

### Expected Result

```text
Job Status: FAILED
Error Key: SERVICENOW_OUTSIDE_WINDOW
Category: business-rule
```

### Why this test is required
It validates the upper boundary of the implementation-window gate.

---

## TC11 — ServiceNow Timeout / Retry Exhaustion

### Objective
Verify retry behavior and final timeout classification.

### Inputs

```yaml
akamai_chg_number: "CHG0012352"

snow_mode: "mock"

snow_mock_scenario: "timeout"
```

### Mock Response

```text
HTTP 504
```

### Expected Flow

```text
Attempt 1
   ↓
Failure
   ↓
Wait 2 seconds
   ↓
Attempt 2
   ↓
Failure
   ↓
Wait 4 seconds
   ↓
Attempt 3
   ↓
Failure
   ↓
Terminal failure
```

### Expected Result

```text
Job Status: FAILED
Error Key: SERVICENOW_API_TIMEOUT
Category: timeout
```

### Why this test is required
It proves that temporary API failures are retried and that the automation fails in a controlled way after retries are exhausted.

---

# 3. Live Validation Test Cases

## TC12 — AAP Credential Missing

### Objective
Verify that live mode cannot proceed without ServiceNow credentials injected by AAP.

### Inputs

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-LIVE-TC12"

akamai_chg_number: "CHG0123456"

snow_mode: "live"
```

### Setup

Do **not** attach the ServiceNow AAP Custom Credential.

Therefore these variables are unavailable:

```text
snow_host
snow_username
snow_password
```

### Expected Result

```text
Job Status: FAILED
Error Key: SERVICENOW_AUTH_FAILED
Category: auth
```

### Important Validation

```text
No ServiceNow API call should be made.
```

### Why this test is required
It proves that credentials are mandatory and are not expected from Git, defaults, vars, or survey input.

---

## TC13 — Valid Live ServiceNow Change

### Objective
Verify the complete live ServiceNow path.

### Inputs

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-LIVE-TC13"

akamai_chg_number: "<VALID_TEST_CHG>"

snow_mode: "live"
```

### AAP Setup

Attach the approved ServiceNow Custom Credential that injects:

```text
snow_host
snow_username
snow_password
```

### Required ServiceNow Test Data

The live CHG must satisfy:

```text
Exists
State = Implement
Approval = Approved
Current time is inside planned start/end window
```

### Expected Result

```text
Job Status: SUCCESS

snow_validation_result:
  result: PASSED
  mode: live
  source: live
  change_number: <VALID_TEST_CHG>
  state: implement
  approval: approved
  read_only: true
```

### Why this test is required
This is the final integration proof that the mock-tested business logic also works against the actual ServiceNow API.

---

## TC14 — Live Change Not Found

### Objective
Verify the live `records: []` path.

### Inputs

```yaml
akamai_chg_number: "<VALID_FORMAT_NON_EXISTENT_CHG>"

snow_mode: "live"
```

### Expected ServiceNow Module Result

```yaml
records: []
```

### Expected Result

```text
Job Status: FAILED
Error Key: SERVICENOW_CHANGE_NOT_FOUND
Category: business-rule
```

### Why this test is required
It confirms that live ServiceNow response handling matches the mock 404/not-found business behavior.

---

## TC15 — Multiple Live Records Returned

### Objective
Verify defensive handling if more than one record is returned for the supplied change number.

### Setup

Controlled integration/mock response:

```yaml
records:
  - {...}
  - {...}
```

### Expected Result

```text
Job Status: FAILED
Error Key: SERVICENOW_RESPONSE_INVALID
Category: api
```

### Priority

```text
P2
```

### Why this test exists
A CHG number should uniquely resolve to one change record.  
This protects the workflow from ambiguous ServiceNow responses.

This is a defensive test and does not need the same showcase emphasis as TC01–TC14.

---

# 4. Minimum Test Set for Sprint / Story Demonstration

If time is limited, these are the tests that provide the most value:

| Demo Order | Test | What It Proves |
|---|---|---|
| 1 | TC01 | Complete successful validation path |
| 2 | TC06 | State gate |
| 3 | TC07 | Approval gate |
| 4 | TC08 | Planned window is mandatory |
| 5 | TC09 or TC10 | Execution must be inside approved window |
| 6 | TC05 | CHG must exist |
| 7 | TC04 | Authentication failure is classified correctly |
| 8 | TC11 | Retry + timeout behavior |
| 9 | TC12 | AAP Credential injection is mandatory for live mode |
| 10 | TC13 | End-to-end live ServiceNow validation |

TC02, TC03 and TC15 can be shown only if the team specifically asks about defensive/input validation.

---

# 5. Functional Coverage Mapping

| SSPA-11 Requirement | Covered By |
|---|---|
| Validate CHG input format | TC02 |
| ServiceNow authentication handling | TC04, TC12 |
| Verify CHG exists | TC05, TC14 |
| Require `Implement` state | TC06 |
| Require `Approved` approval | TC07 |
| Validate planned start/end exist | TC08 |
| Prevent execution before window | TC09 |
| Prevent execution after window | TC10 |
| Successful complete validation | TC01 |
| API timeout/retry handling | TC11 |
| AAP Credential-based live authentication | TC12, TC13 |
| Successful live ServiceNow retrieval | TC13 |
| Protect against ambiguous response | TC15 |
| Standardized common logging failure path | TC02–TC12, TC14–TC15 |
| Publish successful result to AAP | TC01, TC13 |

---

# 6. Expected Error Matrix

| Failure Type | Error Category | Error Key |
|---|---|---|
| Invalid configuration / CHG format | `validation` | `SERVICENOW_CONFIG_INVALID` |
| ServiceNow credentials/authentication failure | `auth` | `SERVICENOW_AUTH_FAILED` |
| CHG does not exist | `business-rule` | `SERVICENOW_CHANGE_NOT_FOUND` |
| Invalid ServiceNow response | `api` | `SERVICENOW_RESPONSE_INVALID` |
| State or approval invalid | `business-rule` | `SERVICENOW_STATUS_INVALID` |
| Implementation window missing | `business-rule` | `SERVICENOW_WINDOW_MISSING` |
| Implementation window invalid | `business-rule` | `SERVICENOW_WINDOW_INVALID` |
| Execution outside approved window | `business-rule` | `SERVICENOW_OUTSIDE_WINDOW` |
| ServiceNow timeout | `timeout` | `SERVICENOW_API_TIMEOUT` |
| ServiceNow rate limiting | `rate-limit` | `SERVICENOW_RATE_LIMITED` |
| General API failure | `api` | `SERVICENOW_API_FAILED` |

---

# 7. Test Execution Evidence to Capture

For each test, capture only the evidence needed to prove the expected behavior:

1. AAP Job ID
2. Test Case ID
3. Extra Vars used
4. Selected mock scenario or `live`
5. Final AAP Job Status
6. Expected error key for negative tests
7. `snow_validation_result` artifact for successful tests
8. Relevant `common_logging` terminal message

Avoid capturing passwords, credential values, tokens, or full sensitive ServiceNow responses.

---

# 8. Final Recommended Acceptance Set

For SSPA-11 to be considered functionally demonstrated, the following should pass as expected:

```text
Positive:
  TC01 - Mock happy path
  TC13 - Live happy path, when ServiceNow access is available

Business gates:
  TC05 - CHG not found
  TC06 - Invalid state
  TC07 - Not approved
  TC08 - Missing window
  TC09 - Before window
  TC10 - After window

Technical/error handling:
  TC04 - Authentication failure
  TC11 - Timeout/retry exhaustion
  TC12 - Missing AAP credential injection
```

The remaining cases are useful defensive coverage but do not need to dominate the story demonstration.
