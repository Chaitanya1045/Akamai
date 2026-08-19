# SSPA-11 — Updated Code for Logging Environment Variable Fix

## Issue

AAP was failing because `snow_validation.yml` still referenced:

```yaml
common_logging_allowed_environments
```

before the `common_logging` role had loaded its defaults.

The fix is to keep the allowed environment list local to the `akamai_version_push` role and use it consistently everywhere in SSPA-11.

---

# 1. `roles/akamai_version_push/vars/main.yml`

Add / retain the following ServiceNow configuration.

```yaml
---
# =============================================================================
# ServiceNow internal validation configuration
# =============================================================================

snow_allowed_modes:
  - mock
  - live


# =============================================================================
# Mandatory ServiceNow deployment gates
# =============================================================================

snow_required_state: "implement"
snow_required_approval: "approved"


# =============================================================================
# Mock configuration
# =============================================================================

snow_mock_dir: "{{ role_path }}/mocks/snow"

snow_allowed_mock_scenarios:
  - auth_failure
  - invalid_not_approved
  - invalid_state
  - missing_window
  - not_found
  - outside_window_early
  - outside_window_late
  - timeout
  - valid_implement_approved


# =============================================================================
# Retryable mock statuses
# =============================================================================

snow_retryable_mock_statuses:
  - 408
  - 429
  - 500
  - 502
  - 503
  - 504


# =============================================================================
# Live API retry plan
# =============================================================================

snow_live_attempt_plan:
  - attempt: 1
    delay: 0

  - attempt: 2
    delay: 2

  - attempt: 3
    delay: 4


# =============================================================================
# Supported logging environments
#
# Keep this local to akamai_version_push so ServiceNow validation does not
# depend on common_logging role defaults being loaded first.
# =============================================================================

snow_allowed_logging_environments:
  - dev
  - rnd
  - qa
  - prod
  - test
  - local


# =============================================================================
# common_logging stages
# =============================================================================

snow_log_stages:
  validation: "servicenow_validation"
  authentication: "servicenow_authentication"
  retrieval: "servicenow_retrieval"
  status: "servicenow_status_validation"
  window: "servicenow_window_validation"


# =============================================================================
# Standardized ServiceNow error keys
# =============================================================================

snow_error_keys:
  config_invalid: "SERVICENOW_CONFIG_INVALID"
  mock_invalid: "SERVICENOW_MOCK_INVALID"
  authentication_failed: "SERVICENOW_AUTH_FAILED"
  retrieval_failed: "SERVICENOW_API_FAILED"
  timeout: "SERVICENOW_API_TIMEOUT"
  rate_limited: "SERVICENOW_RATE_LIMITED"
  change_not_found: "SERVICENOW_CHANGE_NOT_FOUND"
  response_invalid: "SERVICENOW_RESPONSE_INVALID"
  status_invalid: "SERVICENOW_STATUS_INVALID"
  window_missing: "SERVICENOW_WINDOW_MISSING"
  window_invalid: "SERVICENOW_WINDOW_INVALID"
  outside_window: "SERVICENOW_OUTSIDE_WINDOW"
```

---

# 2. `roles/akamai_version_push/tasks/snow_validation.yml`

## Updated logging environment section

```yaml
---
# =============================================================================
# ServiceNow read-only change validation
# =============================================================================


# =============================================================================
# Build logging context
# =============================================================================

- name: "ServiceNow | Build logging context"
  ansible.builtin.set_fact:

    _snow_log_request_id: >-
      {{
        (
          common_logging_request_id
          | default('', true)
          | string
          | trim
        )
        if
        (
          common_logging_request_id
          | default('', true)
          | string
          | trim
          | length > 0
        )
        else
        (
          akamai_chg_number
          | default(
              'SERVICENOW-VALIDATION',
              true
            )
          | string
          | trim
        )
      }}

    _snow_log_execution_id: >-
      {{
        common_logging_execution_id
        | default(
            awx_job_id
            | default(
                now(
                  utc=true,
                  fmt='%Y%m%dT%H%M%SZ'
                ),
                true
              ),
            true
          )
        | string
      }}

    _snow_log_actor: >-
      {{
        common_logging_actor
        | default(
            awx_user_email
            | default(
                'automation',
                true
              ),
            true
          )
        | string
      }}

    _snow_log_triggered_by: >-
      {{
        common_logging_triggered_by
        | default(
            awx_user_email
            | default(
                'automation',
                true
              ),
            true
          )
        | string
      }}


# =============================================================================
# Resolve requested logging environment
# =============================================================================

- name: "ServiceNow | Resolve logging environment"
  ansible.builtin.set_fact:

    _snow_requested_environment: >-
      {{
        common_logging_environment
        | default(
            (
              akamai
              | default({})
            )
            .get(
              'environment',
              {}
            )
            .get(
              'name',
              'local'
            ),
            true
          )
        | string
        | lower
      }}


# =============================================================================
# Resolve safe logging environment
#
# IMPORTANT:
# Use snow_allowed_logging_environments.
# Do not reference common_logging_allowed_environments here.
# =============================================================================

- name: "ServiceNow | Resolve safe logging environment"
  ansible.builtin.set_fact:

    _snow_log_environment: >-
      {{
        _snow_requested_environment
        if
        _snow_requested_environment
        in snow_allowed_logging_environments
        else
        'local'
      }}


# =============================================================================
# Initial state
# =============================================================================

- name: "ServiceNow | Initialize validation state"
  ansible.builtin.set_fact:
    akamai_chg_validated: false
```

---

# 3. Update the Local Configuration Validation Task

The AAP failure also showed another old reference inside:

```text
ServiceNow | Validate environment, mode and change number
```

Replace that task with the following:

```yaml
- name: "ServiceNow | Validate execution configuration"
  block:

    - name: "ServiceNow | Validate environment, mode and change number"
      ansible.builtin.assert:
        that:

          - snow_mode is defined

          - snow_mode in snow_allowed_modes

          - akamai_chg_number is defined

          - >-
            (
              akamai_chg_number
              | string
              | trim
            )
            is match(
              '^CHG[0-9]+$'
            )

          - >-
            _snow_requested_environment
            in snow_allowed_logging_environments

          - >-
            (
              snow_required_state
              | string
              | trim
              | lower
            )
            == 'implement'

          - >-
            (
              snow_required_approval
              | string
              | trim
              | lower
            )
            == 'approved'

          - snow_api_timeout | int > 0

        fail_msg: >-
          Invalid ServiceNow validation configuration.

        quiet: true


    - name: "ServiceNow | Validate mock scenario"
      ansible.builtin.assert:
        that:

          - snow_mock_scenario is defined

          - snow_mock_scenario is string

          - snow_mock_scenario | trim | length > 0

          - >-
            snow_mock_scenario
            is match(
              '^[A-Za-z0-9_-]+$'
            )

          - >-
            snow_mock_scenario
            in snow_allowed_mock_scenarios

        fail_msg: >-
          Invalid ServiceNow mock scenario.

        quiet: true

      when:
        - snow_mode == 'mock'


  rescue:

    - name: "ServiceNow | Stop on invalid local configuration"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:

        common_logging_request_id: "{{ _snow_log_request_id }}"
        common_logging_execution_id: "{{ _snow_log_execution_id }}"
        common_logging_environment: "{{ _snow_log_environment }}"
        common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

        common_logging_error_stage: "{{ snow_log_stages.validation }}"
        common_logging_error_actor: "{{ _snow_log_actor }}"

        common_logging_error_category_input: "validation"

        common_logging_error_key_input: >-
          {{ snow_error_keys.config_invalid }}

        common_logging_error_message: >-
          ServiceNow validation configuration is invalid.

        common_logging_error_recommended_action: >-
          Verify the environment, change number,
          ServiceNow mode, mock scenario,
          timeout and retry configuration.

        common_logging_error_version_info: {}
```

---

# 4. Important Search-and-Replace Check

Search the entire `akamai_version_push` role for:

```text
common_logging_allowed_environments
```

There should be **zero references** remaining inside the SSPA-11 implementation.

All SSPA-11 environment validation must use:

```text
snow_allowed_logging_environments
```

---

# 5. Expected AAP Input

```yaml
akamai:
  environment:
    name: qa

common_logging_request_id: SNOW-TC01

akamai_chg_number: CHG0012345

snow_mode: mock

snow_mock_scenario: valid_implement_approved
```

---

# 6. Expected Behavior After Fix

For:

```yaml
akamai:
  environment:
    name: qa
```

the role should resolve:

```yaml
_snow_requested_environment: "qa"

_snow_log_environment: "qa"
```

The task:

```text
ServiceNow | Validate environment, mode and change number
```

should also pass because:

```text
qa
```

is included in:

```yaml
snow_allowed_logging_environments:
  - dev
  - rnd
  - qa
  - prod
  - test
  - local
```

The execution should then continue into the mock/live retrieval path instead of failing with:

```text
common_logging_allowed_environments is undefined
```

---

# 7. Root Cause Summary

Before:

```text
snow_validation.yml
        ↓
common_logging_allowed_environments
        ↓
common_logging role defaults not yet loaded
        ↓
undefined variable
```

After:

```text
akamai_version_push/vars/main.yml
        ↓
snow_allowed_logging_environments
        ↓
snow_validation.yml
        ↓
environment validation
        ↓
common_logging integration
```

This removes the role-default load-order dependency and keeps the SSPA-11 validation self-contained.
