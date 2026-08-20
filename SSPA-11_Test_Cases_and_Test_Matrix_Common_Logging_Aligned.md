# SSPA-11 — Test Cases and Test Matrix

## Test Strategy

Mock and live use different retrieval mechanisms only:

```text
Mock JSON ────┐
              ├──> snow_chg ───> SAME business validation
Live API ─────┘
```

Business-rule permutations are therefore tested primarily through mock fixtures, while live tests focus on the integration boundary: AAP credential injection, ServiceNow connectivity, actual module response shape, and live retrieval behavior.

---

# Test Matrix

| TC ID | Scenario | Mode | Input / Setup | Expected Result | Expected Error Key |
|---|---|---|---|---|---|
| TC01 | Valid Implement + Approved CHG | Mock | `valid_implement_approved` | PASS | — |
| TC02 | Invalid CHG number format | Mock | `INC0012345` | FAIL before retrieval | `SERVICENOW_CONFIG_INVALID` |
| TC03 | Invalid mock scenario | Mock | Unsupported scenario | FAIL before fixture retrieval | `SERVICENOW_CONFIG_INVALID` |
| TC04 | Authentication failure | Mock | `auth_failure` | FAIL | `SERVICENOW_AUTH_FAILED` |
| TC05 | CHG not found | Mock | `not_found` | FAIL | `SERVICENOW_CHANGE_NOT_FOUND` |
| TC06 | State not Implement | Mock | `invalid_state` | FAIL | `SERVICENOW_STATUS_INVALID` |
| TC07 | Approval not Approved | Mock | `invalid_not_approved` | FAIL | `SERVICENOW_STATUS_INVALID` |
| TC08 | Planned window missing | Mock | `missing_window` | FAIL | `SERVICENOW_WINDOW_MISSING` |
| TC09 | Current time before planned start | Mock | `outside_window_early` | FAIL | `SERVICENOW_OUTSIDE_WINDOW` |
| TC10 | Current time after planned end | Mock | `outside_window_late` | FAIL | `SERVICENOW_OUTSIDE_WINDOW` |
| TC11 | Timeout / retry exhaustion | Mock | `timeout` | Retry then FAIL | `SERVICENOW_API_TIMEOUT` |
| TC12 | AAP ServiceNow credential missing | Live | No credential attached | FAIL before API call | `SERVICENOW_AUTH_FAILED` |
| TC13 | Valid live ServiceNow CHG | Live | Valid credential + valid CHG | PASS | — |
| TC14 | Live CHG not found | Live | Valid-format nonexistent CHG | FAIL | `SERVICENOW_CHANGE_NOT_FOUND` |
| TC15 | Invalid/ambiguous live response | Live | Invalid records response or >1 record | FAIL | `SERVICENOW_RESPONSE_INVALID` |

---

# TC01 — Valid CHG / Happy Path

```yaml
common_logging_request_id: "SNOW-TC01"

akamai_chg_number: "CHG0012345"

snow_mode: "mock"

snow_mock_scenario: "valid_implement_approved"
```

Expected flow:

```text
Fixture loaded
   ↓
HTTP 200
   ↓
snow_chg created
   ↓
Returned CHG matches requested CHG
   ↓
sys_id exists
   ↓
State = Implement
   ↓
Approval = Approved
   ↓
Window present and valid
   ↓
Current time inside window
   ↓
PASS
```

Expected result:

```yaml
snow_validation_result:
  result: PASSED
  mode: mock
  source: mock:valid_implement_approved
  change_number: CHG0012345
  state: implement
  approval: approved
  read_only: true
```

---

# TC02 — Invalid CHG Number Format

```yaml
common_logging_request_id: "SNOW-TC02"

akamai_chg_number: "INC0012345"

snow_mode: "mock"

snow_mock_scenario: "valid_implement_approved"
```

Expected:

```text
FAIL before mock/API retrieval
SERVICENOW_CONFIG_INVALID
category = validation
```

---

# TC03 — Invalid Mock Scenario

```yaml
common_logging_request_id: "SNOW-TC03"

akamai_chg_number: "CHG0012345"

snow_mode: "mock"

snow_mock_scenario: "unsupported_scenario"
```

Expected:

```text
FAIL before fixture retrieval
SERVICENOW_CONFIG_INVALID
```

---

# TC04 — Authentication Failure

```yaml
common_logging_request_id: "SNOW-TC04"

akamai_chg_number: "CHG0012345"

snow_mode: "mock"

snow_mock_scenario: "auth_failure"
```

Expected:

```text
FAIL
SERVICENOW_AUTH_FAILED
category = auth
```

This simulates ServiceNow HTTP 401/403 behavior.

---

# TC05 — CHG Not Found

```yaml
common_logging_request_id: "SNOW-TC05"

akamai_chg_number: "CHG0012351"

snow_mode: "mock"

snow_mock_scenario: "not_found"
```

Expected:

```text
FAIL
SERVICENOW_CHANGE_NOT_FOUND
```

The automation must not continue when the CHG cannot be retrieved.

---

# TC06 — State Is Not Implement

```yaml
common_logging_request_id: "SNOW-TC06"

akamai_chg_number: "CHG0012346"

snow_mode: "mock"

snow_mock_scenario: "invalid_state"
```

Fixture condition:

```text
state    = Draft
approval = Approved
```

Expected:

```text
FAIL
SERVICENOW_STATUS_INVALID
```

---

# TC07 — Approval Is Not Approved

```yaml
common_logging_request_id: "SNOW-TC07"

akamai_chg_number: "CHG0012347"

snow_mode: "mock"

snow_mock_scenario: "invalid_not_approved"
```

Fixture condition:

```text
state    = Implement
approval = Not Yet Requested
```

Expected:

```text
FAIL
SERVICENOW_STATUS_INVALID
```

The business gate is:

```text
State = Implement
AND
Approval = Approved
```

---

# TC08 — Planned Window Missing

```yaml
common_logging_request_id: "SNOW-TC08"

akamai_chg_number: "CHG0012350"

snow_mode: "mock"

snow_mock_scenario: "missing_window"
```

Fixture:

```text
start_date = ""
end_date   = ""
```

Expected:

```text
FAIL
SERVICENOW_WINDOW_MISSING
```

---

# TC09 — Execution Before Planned Window

```yaml
common_logging_request_id: "SNOW-TC09"

akamai_chg_number: "CHG0012348"

snow_mode: "mock"

snow_mock_scenario: "outside_window_early"
```

Condition:

```text
NOW < planned_start
```

Expected:

```text
FAIL
SERVICENOW_OUTSIDE_WINDOW
```

---

# TC10 — Execution After Planned Window

```yaml
common_logging_request_id: "SNOW-TC10"

akamai_chg_number: "CHG0012349"

snow_mode: "mock"

snow_mock_scenario: "outside_window_late"
```

Condition:

```text
NOW > planned_end
```

Expected:

```text
FAIL
SERVICENOW_OUTSIDE_WINDOW
```

---

# TC11 — Timeout / Retry Exhaustion

```yaml
common_logging_request_id: "SNOW-TC11"

akamai_chg_number: "CHG0012352"

snow_mode: "mock"

snow_mock_scenario: "timeout"
```

Expected flow:

```text
Attempt 1
   ↓
Failure
   ↓
Backoff
   ↓
Attempt 2
   ↓
Failure
   ↓
Backoff
   ↓
Terminal failure
```

Expected:

```text
SERVICENOW_API_TIMEOUT
category = timeout
```

---

# TC12 — AAP Credential Missing

```yaml
common_logging_request_id: "SNOW-LIVE-TC12"

akamai_chg_number: "CHG0123456"

snow_mode: "live"
```

Do not attach the ServiceNow AAP credential.

Expected:

```text
snow_host     unavailable
snow_username unavailable
snow_password unavailable

FAIL before ServiceNow API call
SERVICENOW_AUTH_FAILED
```

---

# TC13 — Valid Live ServiceNow CHG

Preconditions:

```text
AAP ServiceNow credential attached
CHG exists
State = Implement
Approval = Approved
Current time inside planned implementation window
```

Extra Vars:

```yaml
common_logging_request_id: "SNOW-LIVE-TC13"

akamai_chg_number: "<VALID_LIVE_CHG>"

snow_mode: "live"
```

Expected flow:

```text
AAP credential injection
        ↓
ServiceNow connectivity
        ↓
servicenow.itsm.change_request_info
        ↓
records[0]
        ↓
snow_chg
        ↓
SAME common validator
        ↓
Implement + Approved
        ↓
Window validation
        ↓
PASS
```

Expected result:

```yaml
snow_validation_result:
  result: PASSED
  mode: live
  source: live
  change_number: <VALID_LIVE_CHG>
  state: implement
  approval: approved
  read_only: true
```

---

# TC14 — Live CHG Not Found

Use a valid-format CHG that does not exist.

```yaml
common_logging_request_id: "SNOW-LIVE-TC14"

akamai_chg_number: "<NON_EXISTENT_VALID_FORMAT_CHG>"

snow_mode: "live"
```

Expected module result conceptually:

```yaml
records: []
```

Expected:

```text
FAIL
SERVICENOW_CHANGE_NOT_FOUND
```

---

# TC15 — Invalid / Ambiguous Live Response

Scenario A: response does not provide a valid `records` sequence.

Expected:

```text
FAIL
SERVICENOW_RESPONSE_INVALID
```

Scenario B: more than one record is returned for the CHG number.

Expected:

```text
FAIL
SERVICENOW_RESPONSE_INVALID
```

The automation must never arbitrarily select one of multiple records.

---

# Minimum Regression Set After common_logging Alignment

| Priority | TC | Purpose |
|---|---|---|
| 1 | TC01 | Successful end-to-end mock validation |
| 2 | TC06 | State gate remains unchanged |
| 3 | TC07 | Approval gate remains unchanged |
| 4 | TC08 | Window is mandatory |
| 5 | TC09 | Early-window execution is blocked |
| 6 | TC10 | Late-window execution is blocked |
| 7 | TC05 | Not-found handling remains correct |
| 8 | TC11 | Retry/timeout behavior remains correct |
| 9 | TC12 | AAP credential dependency is enforced |
| 10 | TC13 | Live ServiceNow integration works with same validator |

---

# Expected Business Gate

Deployment may proceed only when all of the following are true:

```text
CHG exists
AND
returned CHG matches requested CHG
AND
sys_id exists
AND
State = Implement
AND
Approval = Approved
AND
planned start/end are present
AND
planned start < planned end
AND
current execution time is inside the planned window
```

Any failure must terminate through `common_logging/fail_with_error.yml`.

---

# Mock vs Live Validation Requirement

There must be no separate mock-only or live-only state/approval/window validator.

```text
Mock fixture --------┐
                     ├──> snow_chg
Live API response ---┘
                          ↓
                   identity validation
                          ↓
                   Implement + Approved
                          ↓
                    window validation
                          ↓
                        PASS
```

This is what makes the mock test suite meaningful for the eventual live API path.

---

# Example Successful Mock Input

```yaml
common_logging_request_id: "SNOW-TC01"

akamai_chg_number: "CHG0012345"

snow_mode: "mock"

snow_mock_scenario: "valid_implement_approved"
```

`akamai.environment.name` is not required for standalone testing; logging falls back to `local`.

---

# Example Live Input

```yaml
common_logging_request_id: "SNOW-LIVE-001"

akamai_chg_number: "<VALID_TEST_CHG>"

snow_mode: "live"
```

AAP must inject:

```text
snow_host
snow_username
snow_password
```

Do not pass ServiceNow credentials through Extra Vars or the survey.
