# SSPA-11 — Update for `common_logging_allowed_environments` Undefined Variable

## Issue Observed in AAP

AAP failed in:

```text
roles/akamai_version_push/tasks/snow_validation.yml
```

with:

```text
'common_logging_allowed_environments' is undefined
```

The failure happened before ServiceNow mock/live validation because `snow_validation.yml` referenced a variable owned by the `common_logging` role before that role had been loaded.

---

# Change 1 — `roles/akamai_version_push/vars/main.yml`

Add:

```yaml
# =============================================================================
# Supported logging environments
# =============================================================================

snow_allowed_logging_environments:
  - dev
  - rnd
  - qa
  - prod
  - test
  - local
```

Relevant updated section:

```yaml
---
snow_allowed_modes:
  - mock
  - live

snow_required_state: "implement"
snow_required_approval: "approved"

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

snow_retryable_mock_statuses:
  - 408
  - 429
  - 500
  - 502
  - 503
  - 504

snow_live_attempt_plan:
  - attempt: 1
    delay: 0
  - attempt: 2
    delay: 2
  - attempt: 3
    delay: 4

snow_allowed_logging_environments:
  - dev
  - rnd
  - qa
  - prod
  - test
  - local

snow_log_stages:
  validation: "servicenow_validation"
  authentication: "servicenow_authentication"
  retrieval: "servicenow_retrieval"
  status: "servicenow_status_validation"
  window: "servicenow_window_validation"

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

# Change 2 — `roles/akamai_version_push/tasks/snow_validation.yml`

Replace:

```yaml
- name: "ServiceNow | Resolve safe logging environment"
  ansible.builtin.set_fact:
    _snow_log_environment: >-
      {{
        _snow_requested_environment
        if
        _snow_requested_environment
        in common_logging_allowed_environments
        else
        'local'
      }}
```

with:

```yaml
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
```

---

# Updated Logging Context Section

```yaml
---
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

- name: "ServiceNow | Initialize validation state"
  ansible.builtin.set_fact:
    akamai_chg_validated: false
```

---

# No Changes Required for This Issue

No change is required in:

```text
snow_live_attempt.yml
defaults/main.yml
mock JSON payloads
AAP survey inputs
AAP ServiceNow credential configuration
```

---

# Expected Result

With:

```yaml
akamai:
  environment:
    name: "qa"
```

the task:

```text
ServiceNow | Resolve safe logging environment
```

should resolve:

```yaml
_snow_log_environment: "qa"
```

and continue to ServiceNow validation.

If an unsupported environment is supplied, it falls back to:

```yaml
_snow_log_environment: "local"
```

---

# Root Cause

Before:

```text
snow_validation.yml
        ↓
common_logging_allowed_environments
        ↓
common_logging role defaults not loaded
        ↓
UNDEFINED VARIABLE
```

After:

```text
akamai_version_push/vars/main.yml
        ↓
snow_allowed_logging_environments
        ↓
snow_validation.yml
        ↓
safe environment resolved
```

This removes the role-variable load-order dependency.
