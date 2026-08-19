# SSPA-11 — Updated ServiceNow Change Validation

## Purpose of This Revision

This revision aligns SSPA-11 with the latest `common_logging` contract and makes the mock/live design explicit:

```text
Different retrieval source
        ↓
Same normalized variable: snow_chg
        ↓
One common validation path
```

The main structural change is:

```text
OLD
tasks/
├── snow_validation.yml
└── snow_live_attempt.yml

NEW
tasks/
├── snow_validation.yml
└── snow_fetch.yml
```

`snow_fetch.yml` contains **both mock and live retrieval only**.

`snow_validation.yml` contains the **single common business validation path** used by both sources.

There is no separate mock validation and live validation.

---

# 1. Final Execution Design

```text
                     snow_validation.yml
                             │
                             │ local/config validation
                             ▼
                        snow_fetch.yml
                    ┌────────┴────────┐
                    │                 │
                  MOCK               LIVE
                    │                 │
              JSON fixture      ServiceNow API
                    │                 │
                    └────────┬────────┘
                             │
                           snow_chg
                             │
                             ▼
                 COMMON BUSINESS VALIDATION
                             │
                   Mandatory fields
                             │
                   CHG identity/sys_id
                             │
                    State = Implement
                             │
                   Approval = Approved
                             │
              Planned implementation window
                             │
                  Current time in window
                             │
                             ▼
                           PASS
                             │
                  snow_validation_result
                             │
                             ▼
                        AAP set_stats
```

The only source-specific behavior is retrieval, authentication, transport/API error handling, and normalization into `snow_chg`.

---

# 2. `roles/akamai_version_push/defaults/main.yml`

Merge this section into the existing defaults file.

```yaml
---
# =============================================================================
# ServiceNow change validation
# =============================================================================

# Data source:
#   mock - local JSON fixture
#   live - ServiceNow API
snow_mode: "mock"

# Used only when snow_mode == mock.
snow_mock_scenario: "valid_implement_approved"

# TLS certificate validation for live ServiceNow calls.
snow_validate_certs: true

# API timeout in seconds.
snow_api_timeout: 30

# ServiceNow raw date/time format expected by the common validator.
snow_datetime_format: "%Y-%m-%d %H:%M:%S"
```

Do not define or store these in defaults:

```text
snow_host
snow_username
snow_password
```

They are supplied by the AAP Custom Credential at runtime.

---

# 3. `roles/akamai_version_push/vars/main.yml`

Merge this section into the existing vars file.

```yaml
---
# =============================================================================
# ServiceNow internal configuration
# =============================================================================

snow_allowed_modes:
  - mock
  - live


# =============================================================================
# Mandatory business gates
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


snow_retryable_mock_statuses:
  - 408
  - 429
  - 500
  - 502
  - 503
  - 504


# =============================================================================
# Live retry configuration
# =============================================================================

snow_live_retries: 3
snow_live_retry_delay: 2


# =============================================================================
# common_logging stages
#
# Must remain lowercase snake_case because common_logging validates stage format.
# =============================================================================

snow_log_stages:
  validation: "servicenow_validation"
  authentication: "servicenow_authentication"
  retrieval: "servicenow_retrieval"
  status: "servicenow_status_validation"
  window: "servicenow_window_validation"


# =============================================================================
# Stable semantic error keys
#
# Numeric error codes are NOT maintained here.
# common_logging owns category -> numeric-code mapping.
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

# 4. `roles/akamai_version_push/tasks/main.yml`

Retain the existing dispatcher and use only the SSPA-11 entry point below.

```yaml
- name: "Validate ServiceNow change request"
  ansible.builtin.import_tasks:
    snow_validation.yml
```

Do not import `snow_fetch.yml` directly from `main.yml`.

`snow_validation.yml` owns the complete SSPA-11 flow and calls `snow_fetch.yml` internally.

---

# 5. `roles/akamai_version_push/tasks/snow_fetch.yml`

This file contains **source-specific retrieval only**.

It must finish successfully with the same output contract:

```text
snow_chg
snow_fetch_source
```

No `Implement`, `Approved`, CHG identity, or implementation-window business validation is performed here.

```yaml
---
# =============================================================================
# ServiceNow data retrieval adapter
#
# MOCK:
#   JSON fixture -> snow_chg
#
# LIVE:
#   ServiceNow API -> records[0] -> snow_chg
#
# OUTPUT CONTRACT FOR BOTH:
#   snow_chg
#   snow_fetch_source
#
# Business validation is NOT performed in this file.
# =============================================================================


# =============================================================================
# MOCK RETRIEVAL
# =============================================================================

- name: "ServiceNow | Retrieve change from mock source"
  when:
    - snow_mode == 'mock'
  block:

    - name: "ServiceNow | Read selected mock fixture"
      ansible.builtin.set_fact:
        _snow_raw: >-
          {{
            lookup(
              'ansible.builtin.file',
              snow_mock_dir
              ~ '/'
              ~ snow_mock_scenario
              ~ '.json'
            )
            | from_json
          }}
      no_log: true


    - name: "ServiceNow | Validate mock response envelope"
      ansible.builtin.assert:
        that:
          - _snow_raw is mapping
          - _snow_raw.http_status is defined
        fail_msg: >-
          Mock ServiceNow response does not contain
          a valid response envelope.
        quiet: true


  rescue:

    - name: "ServiceNow | Stop on invalid mock fixture"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: "{{ _snow_log_request_id }}"
        common_logging_execution_id: "{{ _snow_log_execution_id }}"
        common_logging_environment: "{{ _snow_log_environment }}"
        common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

        common_logging_error_stage: "{{ snow_log_stages.retrieval }}"
        common_logging_error_actor: "{{ _snow_log_actor }}"
        common_logging_error_category_input: "validation"
        common_logging_error_key_input: "{{ snow_error_keys.mock_invalid }}"

        common_logging_error_message: >-
          ServiceNow mock fixture could not be loaded
          or contains an invalid response envelope.

        common_logging_error_recommended_action: >-
          Verify the selected mock scenario and JSON fixture structure.

        common_logging_error_version_info: {}


# =============================================================================
# MOCK AUTHENTICATION FAILURE
# =============================================================================

- name: "ServiceNow | Handle mock authentication failure"
  ansible.builtin.include_role:
    name: common_logging
    tasks_from: fail_with_error
  vars:
    common_logging_request_id: "{{ _snow_log_request_id }}"
    common_logging_execution_id: "{{ _snow_log_execution_id }}"
    common_logging_environment: "{{ _snow_log_environment }}"
    common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

    common_logging_error_stage: "{{ snow_log_stages.authentication }}"
    common_logging_error_actor: "{{ _snow_log_actor }}"
    common_logging_error_category_input: "auth"
    common_logging_error_key_input: "{{ snow_error_keys.authentication_failed }}"

    common_logging_error_message: >-
      ServiceNow authentication failed.

    common_logging_error_recommended_action: >-
      Verify the ServiceNow AAP credential and
      service-account access.

    common_logging_error_version_info: {}

  when:
    - snow_mode == 'mock'
    - _snow_raw.http_status | int in [401, 403]


# =============================================================================
# MOCK CHANGE NOT FOUND
# =============================================================================

- name: "ServiceNow | Handle mock change not found"
  ansible.builtin.include_role:
    name: common_logging
    tasks_from: fail_with_error
  vars:
    common_logging_request_id: "{{ _snow_log_request_id }}"
    common_logging_execution_id: "{{ _snow_log_execution_id }}"
    common_logging_environment: "{{ _snow_log_environment }}"
    common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

    common_logging_error_stage: "{{ snow_log_stages.retrieval }}"
    common_logging_error_actor: "{{ _snow_log_actor }}"
    common_logging_error_category_input: "business-rule"
    common_logging_error_key_input: "{{ snow_error_keys.change_not_found }}"

    common_logging_error_message: >-
      Change {{ akamai_chg_number }} was not found.

    common_logging_error_recommended_action: >-
      Verify the ServiceNow change number before
      restarting the workflow.

    common_logging_error_version_info: {}

  when:
    - snow_mode == 'mock'
    - _snow_raw.http_status | int == 404


# =============================================================================
# MOCK RETRYABLE FAILURE
#
# This simulates the terminal outcome after retry exhaustion.
# Business validation is not entered.
# =============================================================================

- name: "ServiceNow | Handle mock retryable API failure"
  when:
    - snow_mode == 'mock'
    - _snow_raw.http_status | int in snow_retryable_mock_statuses
  block:

    - name: "ServiceNow | Display simulated retry attempt 1"
      ansible.builtin.debug:
        msg: >-
          ServiceNow mock attempt 1 of 3 failed
          with HTTP {{ _snow_raw.http_status }}.


    - name: "ServiceNow | Display simulated retry attempt 2"
      ansible.builtin.debug:
        msg: >-
          ServiceNow mock attempt 2 of 3 failed
          with HTTP {{ _snow_raw.http_status }}.


    - name: "ServiceNow | Display simulated retry attempt 3"
      ansible.builtin.debug:
        msg: >-
          ServiceNow mock attempt 3 of 3 failed
          with HTTP {{ _snow_raw.http_status }}.


    - name: "ServiceNow | Resolve mock terminal API classification"
      ansible.builtin.set_fact:
        _snow_mock_error_category: >-
          {{
            'timeout'
            if (_snow_raw.http_status | int) in [408, 504]
            else
            (
              'rate-limit'
              if (_snow_raw.http_status | int) == 429
              else
              'api'
            )
          }}

        _snow_mock_error_key: >-
          {{
            snow_error_keys.timeout
            if (_snow_raw.http_status | int) in [408, 504]
            else
            (
              snow_error_keys.rate_limited
              if (_snow_raw.http_status | int) == 429
              else
              snow_error_keys.retrieval_failed
            )
          }}


    - name: "ServiceNow | Stop after simulated retry exhaustion"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: "{{ _snow_log_request_id }}"
        common_logging_execution_id: "{{ _snow_log_execution_id }}"
        common_logging_environment: "{{ _snow_log_environment }}"
        common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

        common_logging_error_stage: "{{ snow_log_stages.retrieval }}"
        common_logging_error_actor: "{{ _snow_log_actor }}"
        common_logging_error_category_input: "{{ _snow_mock_error_category }}"
        common_logging_error_key_input: "{{ _snow_mock_error_key }}"

        common_logging_error_message: >-
          ServiceNow request failed after three
          simulated retrieval attempts.

        common_logging_error_recommended_action: >-
          Verify ServiceNow availability, network connectivity,
          API health and rate limits.

        common_logging_error_version_info: {}


# =============================================================================
# MOCK UNEXPECTED HTTP FAILURE
# =============================================================================

- name: "ServiceNow | Handle unexpected mock API failure"
  ansible.builtin.include_role:
    name: common_logging
    tasks_from: fail_with_error
  vars:
    common_logging_request_id: "{{ _snow_log_request_id }}"
    common_logging_execution_id: "{{ _snow_log_execution_id }}"
    common_logging_environment: "{{ _snow_log_environment }}"
    common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

    common_logging_error_stage: "{{ snow_log_stages.retrieval }}"
    common_logging_error_actor: "{{ _snow_log_actor }}"
    common_logging_error_category_input: "api"
    common_logging_error_key_input: "{{ snow_error_keys.retrieval_failed }}"

    common_logging_error_message: >-
      ServiceNow mock request returned an unexpected API response.

    common_logging_error_recommended_action: >-
      Verify the mock response and expected ServiceNow API behavior.

    common_logging_error_version_info: {}

  when:
    - snow_mode == 'mock'
    - _snow_raw.http_status | int != 200
    - _snow_raw.http_status | int not in [401, 403, 404]
    - _snow_raw.http_status | int not in snow_retryable_mock_statuses


# =============================================================================
# NORMALIZE SUCCESSFUL MOCK RESPONSE
# =============================================================================

- name: "ServiceNow | Normalize successful mock retrieval"
  when:
    - snow_mode == 'mock'
    - _snow_raw.http_status | int == 200
  block:

    - name: "ServiceNow | Validate mock result"
      ansible.builtin.assert:
        that:
          - _snow_raw.result is defined
          - _snow_raw.result is mapping
          - _snow_raw.result | length > 0
        fail_msg: >-
          ServiceNow mock response does not contain
          a valid change record.
        quiet: true


    - name: "ServiceNow | Publish mock retrieval contract"
      ansible.builtin.set_fact:
        snow_chg: "{{ _snow_raw.result }}"
        snow_fetch_source: "mock:{{ snow_mock_scenario }}"
      no_log: true


  rescue:

    - name: "ServiceNow | Stop on invalid mock result"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: "{{ _snow_log_request_id }}"
        common_logging_execution_id: "{{ _snow_log_execution_id }}"
        common_logging_environment: "{{ _snow_log_environment }}"
        common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

        common_logging_error_stage: "{{ snow_log_stages.retrieval }}"
        common_logging_error_actor: "{{ _snow_log_actor }}"
        common_logging_error_category_input: "api"
        common_logging_error_key_input: "{{ snow_error_keys.response_invalid }}"

        common_logging_error_message: >-
          ServiceNow mock response does not contain
          a valid change record.

        common_logging_error_recommended_action: >-
          Verify the selected mock JSON response structure.

        common_logging_error_version_info: {}


- name: "ServiceNow | Log successful mock retrieval"
  ansible.builtin.include_role:
    name: common_logging
    tasks_from: log_event
  vars:
    common_logging_request_id: "{{ _snow_log_request_id }}"
    common_logging_execution_id: "{{ _snow_log_execution_id }}"
    common_logging_environment: "{{ _snow_log_environment }}"
    common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

    common_logging_actor: "{{ _snow_log_actor }}"
    common_logging_stage: "{{ snow_log_stages.retrieval }}"
    common_logging_result: "success"

    common_logging_message: >-
      ServiceNow change data was retrieved from the mock source.

    common_logging_version_info: {}

  when:
    - snow_mode == 'mock'
    - _snow_raw.http_status | int == 200


# =============================================================================
# LIVE AAP CREDENTIAL VALIDATION
# =============================================================================

- name: "ServiceNow | Validate live credential injection"
  when:
    - snow_mode == 'live'
  block:

    - name: "ServiceNow | Require AAP-injected ServiceNow credentials"
      ansible.builtin.assert:
        that:
          - snow_host is defined
          - snow_host | default('') | string | trim | length > 0

          - snow_username is defined
          - snow_username | default('') | string | trim | length > 0

          - snow_password is defined
          - snow_password | default('') | string | length > 0

        fail_msg: >-
          Required ServiceNow credential values were not injected by AAP.

        quiet: true

      no_log: true


  rescue:

    - name: "ServiceNow | Stop when live credential is unavailable"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: "{{ _snow_log_request_id }}"
        common_logging_execution_id: "{{ _snow_log_execution_id }}"
        common_logging_environment: "{{ _snow_log_environment }}"
        common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

        common_logging_error_stage: "{{ snow_log_stages.authentication }}"
        common_logging_error_actor: "{{ _snow_log_actor }}"
        common_logging_error_category_input: "auth"
        common_logging_error_key_input: "{{ snow_error_keys.authentication_failed }}"

        common_logging_error_message: >-
          ServiceNow credentials were not injected by AAP.

        common_logging_error_recommended_action: >-
          Attach the approved ServiceNow credential
          to the AAP Job Template.

        common_logging_error_version_info: {}


# =============================================================================
# LIVE RETRIEVAL
#
# Raw module failures remain internal/no_log.
# Only sanitized messages are sent to common_logging.
# =============================================================================

- name: "ServiceNow | Retrieve change from live source"
  when:
    - snow_mode == 'live'
  block:

    - name: "ServiceNow | Retrieve change request"
      servicenow.itsm.change_request_info:
        instance:
          host: "{{ snow_host }}"
          username: "{{ snow_username }}"
          password: "{{ snow_password }}"
          timeout: "{{ snow_api_timeout | int }}"
          validate_certs: "{{ snow_validate_certs | bool }}"

        number: "{{ akamai_chg_number }}"
        sysparm_display_value: "all"

      register: _snow_live_response

      changed_when: false

      retries: "{{ snow_live_retries | int }}"
      delay: "{{ snow_live_retry_delay | int }}"
      until: _snow_live_response is succeeded

      no_log: true


  rescue:

    - name: "ServiceNow | Classify live retrieval failure internally"
      ansible.builtin.set_fact:
        _snow_live_error_text: >-
          {{
            ansible_failed_result.msg
            | default('', true)
            | string
            | lower
          }}

        _snow_live_error_category: >-
          {{
            'auth'
            if
            (
              ansible_failed_result.msg
              | default('', true)
              | string
              | lower
              is search(
                '401|403|unauthorized|forbidden|authentication|credential'
              )
            )
            else
            (
              'timeout'
              if
              (
                ansible_failed_result.msg
                | default('', true)
                | string
                | lower
                is search(
                  'timeout|timed out|readtimeout|connecttimeout'
                )
              )
              else
              (
                'rate-limit'
                if
                (
                  ansible_failed_result.msg
                  | default('', true)
                  | string
                  | lower
                  is search(
                    '429|rate limit|too many requests'
                  )
                )
                else
                'api'
              )
            )
          }}

        _snow_live_error_key: >-
          {{
            snow_error_keys.authentication_failed
            if
            (
              ansible_failed_result.msg
              | default('', true)
              | string
              | lower
              is search(
                '401|403|unauthorized|forbidden|authentication|credential'
              )
            )
            else
            (
              snow_error_keys.timeout
              if
              (
                ansible_failed_result.msg
                | default('', true)
                | string
                | lower
                is search(
                  'timeout|timed out|readtimeout|connecttimeout'
                )
              )
              else
              (
                snow_error_keys.rate_limited
                if
                (
                  ansible_failed_result.msg
                  | default('', true)
                  | string
                  | lower
                  is search(
                    '429|rate limit|too many requests'
                  )
                )
                else
                snow_error_keys.retrieval_failed
              )
            )
          }}

      no_log: true


    - name: "ServiceNow | Stop after live retrieval failure"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: "{{ _snow_log_request_id }}"
        common_logging_execution_id: "{{ _snow_log_execution_id }}"
        common_logging_environment: "{{ _snow_log_environment }}"
        common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

        common_logging_error_stage: >-
          {{
            snow_log_stages.authentication
            if _snow_live_error_category == 'auth'
            else snow_log_stages.retrieval
          }}

        common_logging_error_actor: "{{ _snow_log_actor }}"
        common_logging_error_category_input: "{{ _snow_live_error_category }}"
        common_logging_error_key_input: "{{ _snow_live_error_key }}"

        common_logging_error_message: >-
          ServiceNow change retrieval failed after
          the configured retry attempts.

        common_logging_error_recommended_action: >-
          Verify the AAP ServiceNow credential, ServiceNow availability,
          certificate trust, network connectivity, API permissions
          and rate limits.

        common_logging_error_version_info: {}


# =============================================================================
# LIVE RESPONSE CONTRACT
# =============================================================================

- name: "ServiceNow | Validate live retrieval response"
  when:
    - snow_mode == 'live'
  block:

    - name: "ServiceNow | Require valid records collection"
      ansible.builtin.assert:
        that:
          - _snow_live_response.records is defined
          - _snow_live_response.records is sequence
          - _snow_live_response.records is not string
        fail_msg: >-
          ServiceNow did not return a valid change-record collection.
        quiet: true


    - name: "ServiceNow | Require change to exist"
      ansible.builtin.assert:
        that:
          - _snow_live_response.records | length > 0
        fail_msg: >-
          The requested ServiceNow change does not exist.
        quiet: true


    - name: "ServiceNow | Require unique change result"
      ansible.builtin.assert:
        that:
          - _snow_live_response.records | length == 1
        fail_msg: >-
          ServiceNow returned an ambiguous change response.
        quiet: true


  rescue:

    - name: "ServiceNow | Resolve live response contract failure"
      ansible.builtin.set_fact:
        _snow_live_contract_not_found: >-
          {{
            _snow_live_response.records is defined
            and
            _snow_live_response.records is sequence
            and
            _snow_live_response.records is not string
            and
            (_snow_live_response.records | length == 0)
          }}


    - name: "ServiceNow | Stop on invalid live response"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: "{{ _snow_log_request_id }}"
        common_logging_execution_id: "{{ _snow_log_execution_id }}"
        common_logging_environment: "{{ _snow_log_environment }}"
        common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

        common_logging_error_stage: "{{ snow_log_stages.retrieval }}"
        common_logging_error_actor: "{{ _snow_log_actor }}"

        common_logging_error_category_input: >-
          {{
            'business-rule'
            if _snow_live_contract_not_found | bool
            else 'api'
          }}

        common_logging_error_key_input: >-
          {{
            snow_error_keys.change_not_found
            if _snow_live_contract_not_found | bool
            else snow_error_keys.response_invalid
          }}

        common_logging_error_message: >-
          {{
            'The requested ServiceNow change was not found.'
            if _snow_live_contract_not_found | bool
            else
            'ServiceNow returned an invalid or ambiguous change response.'
          }}

        common_logging_error_recommended_action: >-
          Verify the supplied change number, ServiceNow data,
          and service-account read permissions.

        common_logging_error_version_info: {}


# =============================================================================
# NORMALIZE SUCCESSFUL LIVE RESPONSE
# =============================================================================

- name: "ServiceNow | Publish live retrieval contract"
  ansible.builtin.set_fact:
    snow_chg: "{{ _snow_live_response.records[0] }}"
    snow_fetch_source: "live"
  when:
    - snow_mode == 'live'
  no_log: true


- name: "ServiceNow | Log successful live retrieval"
  ansible.builtin.include_role:
    name: common_logging
    tasks_from: log_event
  vars:
    common_logging_request_id: "{{ _snow_log_request_id }}"
    common_logging_execution_id: "{{ _snow_log_execution_id }}"
    common_logging_environment: "{{ _snow_log_environment }}"
    common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

    common_logging_actor: "{{ _snow_log_actor }}"
    common_logging_stage: "{{ snow_log_stages.retrieval }}"
    common_logging_result: "success"

    common_logging_message: >-
      ServiceNow change data was retrieved successfully
      from the live read-only API.

    common_logging_version_info: {}

  when:
    - snow_mode == 'live'
```

---

# 6. `roles/akamai_version_push/tasks/snow_validation.yml`

This is the SSPA-11 entry point.

Important design rule:

> After `snow_fetch.yml` completes, the business validation below does not branch on mock/live mode.

```yaml
---
# =============================================================================
# ServiceNow change validation
#
# Retrieval:
#   snow_fetch.yml
#
# Common retrieval contract:
#   snow_chg
#   snow_fetch_source
#
# The business rules after retrieval are identical for mock and live.
# =============================================================================


# =============================================================================
# BUILD COMMON LOGGING CONTEXT
# =============================================================================

- name: "ServiceNow | Build logging identity context"
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
# ENVIRONMENT
#
# Missing environment:
#   standalone/local execution -> local
#
# Invalid supplied environment:
#   do not continue silently.
#
# A safe local value is used only so common_logging can emit the
# standardized configuration failure.
# =============================================================================

- name: "ServiceNow | Resolve requested logging environment"
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
        | trim
      }}


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


# =============================================================================
# INITIAL STATE
# =============================================================================

- name: "ServiceNow | Initialize validation state"
  ansible.builtin.set_fact:
    akamai_chg_validated: false


# =============================================================================
# LOCAL CONFIGURATION VALIDATION
# =============================================================================

- name: "ServiceNow | Validate execution configuration"
  block:

    - name: "ServiceNow | Validate environment, mode and change number"
      ansible.builtin.assert:
        that:

          - >-
            _snow_requested_environment
            in common_logging_allowed_environments

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
            (
              snow_required_state
              | string
              | lower
              | trim
            )
            == 'implement'

          - >-
            (
              snow_required_approval
              | string
              | lower
              | trim
            )
            == 'approved'

          - snow_api_timeout | int > 0

          - snow_live_retries | int > 0

          - snow_live_retry_delay | int >= 0

        fail_msg: >-
          Invalid ServiceNow validation configuration.

        quiet: true


    - name: "ServiceNow | Validate mock scenario selection"
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
        common_logging_error_key_input: "{{ snow_error_keys.config_invalid }}"

        common_logging_error_message: >-
          ServiceNow validation configuration is invalid.

        common_logging_error_recommended_action: >-
          Verify the environment, change number, ServiceNow mode,
          mock scenario, timeout and retry configuration.

        common_logging_error_version_info: {}


# =============================================================================
# LOG VALIDATION START
# =============================================================================

- name: "ServiceNow | Log validation start"
  ansible.builtin.include_role:
    name: common_logging
    tasks_from: log_event
  vars:
    common_logging_request_id: "{{ _snow_log_request_id }}"
    common_logging_execution_id: "{{ _snow_log_execution_id }}"
    common_logging_environment: "{{ _snow_log_environment }}"
    common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

    common_logging_actor: "{{ _snow_log_actor }}"
    common_logging_stage: "{{ snow_log_stages.validation }}"
    common_logging_result: "success"

    common_logging_message: >-
      Started read-only ServiceNow change validation.

    common_logging_version_info: {}


# =============================================================================
# RETRIEVE DATA
#
# MOCK and LIVE converge into:
#
#   snow_chg
#   snow_fetch_source
# =============================================================================

- name: "ServiceNow | Retrieve change data"
  ansible.builtin.include_tasks:
    file: snow_fetch.yml


# =============================================================================
# COMMON BUSINESS VALIDATION
#
# NO mock/live branching below this point.
# =============================================================================


# =============================================================================
# MANDATORY FIELDS
# =============================================================================

- name: "ServiceNow | Validate mandatory returned fields"
  block:

    - name: "ServiceNow | Require common change contract"
      ansible.builtin.assert:
        that:
          - snow_chg is defined
          - snow_chg is mapping

          - snow_chg.number is defined
          - snow_chg.sys_id is defined
          - snow_chg.state is defined
          - snow_chg.approval is defined
          - snow_chg.start_date is defined
          - snow_chg.end_date is defined

        fail_msg: >-
          ServiceNow change record is missing
          mandatory validation fields.

        quiet: true


  rescue:

    - name: "ServiceNow | Stop on missing mandatory fields"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: "{{ _snow_log_request_id }}"
        common_logging_execution_id: "{{ _snow_log_execution_id }}"
        common_logging_environment: "{{ _snow_log_environment }}"
        common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

        common_logging_error_stage: "{{ snow_log_stages.retrieval }}"
        common_logging_error_actor: "{{ _snow_log_actor }}"
        common_logging_error_category_input: "api"
        common_logging_error_key_input: "{{ snow_error_keys.response_invalid }}"

        common_logging_error_message: >-
          ServiceNow change data is missing one or more
          required validation fields.

        common_logging_error_recommended_action: >-
          Verify ServiceNow field availability,
          response structure and service-account read permissions.

        common_logging_error_version_info: {}


# =============================================================================
# NORMALIZE COMMON CHANGE CONTRACT
#
# Works with:
#   mock scalar values
#   live sysparm_display_value=all mappings
# =============================================================================

- name: "ServiceNow | Normalize common change identifiers"
  ansible.builtin.set_fact:

    _snow_number: >-
      {{
        (
          snow_chg.number.display_value
          if
          (
            snow_chg.number is mapping
            and
            snow_chg.number.display_value is defined
          )
          else
          (
            snow_chg.number.value
            if
            (
              snow_chg.number is mapping
              and
              snow_chg.number.value is defined
            )
            else
            snow_chg.number
          )
        )
        | default('')
        | string
        | trim
      }}

    _snow_sys_id: >-
      {{
        (
          snow_chg.sys_id.value
          if
          (
            snow_chg.sys_id is mapping
            and
            snow_chg.sys_id.value is defined
          )
          else
          (
            snow_chg.sys_id.display_value
            if
            (
              snow_chg.sys_id is mapping
              and
              snow_chg.sys_id.display_value is defined
            )
            else
            snow_chg.sys_id
          )
        )
        | default('')
        | string
        | trim
      }}


- name: "ServiceNow | Normalize common state and approval"
  ansible.builtin.set_fact:

    _snow_state: >-
      {{
        (
          snow_chg.state.display_value
          if
          (
            snow_chg.state is mapping
            and
            snow_chg.state.display_value is defined
          )
          else
          (
            snow_chg.state.value
            if
            (
              snow_chg.state is mapping
              and
              snow_chg.state.value is defined
            )
            else
            snow_chg.state
          )
        )
        | default('')
        | string
        | trim
        | lower
      }}

    _snow_approval: >-
      {{
        (
          snow_chg.approval.display_value
          if
          (
            snow_chg.approval is mapping
            and
            snow_chg.approval.display_value is defined
          )
          else
          (
            snow_chg.approval.value
            if
            (
              snow_chg.approval is mapping
              and
              snow_chg.approval.value is defined
            )
            else
            snow_chg.approval
          )
        )
        | default('')
        | string
        | trim
        | lower
      }}


- name: "ServiceNow | Normalize common implementation window"
  ansible.builtin.set_fact:

    _snow_start_raw: >-
      {{
        (
          snow_chg.start_date.value
          if
          (
            snow_chg.start_date is mapping
            and
            snow_chg.start_date.value is defined
          )
          else
          snow_chg.start_date
        )
        | default('')
        | string
        | trim
      }}

    _snow_end_raw: >-
      {{
        (
          snow_chg.end_date.value
          if
          (
            snow_chg.end_date is mapping
            and
            snow_chg.end_date.value is defined
          )
          else
          snow_chg.end_date
        )
        | default('')
        | string
        | trim
      }}

    _snow_now_raw: >-
      {{
        now(
          utc=true,
          fmt=snow_datetime_format
        )
      }}


# =============================================================================
# CHANGE IDENTITY GATE
# =============================================================================

- name: "ServiceNow | Validate returned change identity"
  block:

    - name: "ServiceNow | Require matching CHG and sys_id"
      ansible.builtin.assert:
        that:
          - _snow_number == akamai_chg_number
          - _snow_sys_id | length > 0

        fail_msg: >-
          ServiceNow returned an invalid or unexpected change record.

        quiet: true


  rescue:

    - name: "ServiceNow | Stop on invalid change identity"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: "{{ _snow_log_request_id }}"
        common_logging_execution_id: "{{ _snow_log_execution_id }}"
        common_logging_environment: "{{ _snow_log_environment }}"
        common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

        common_logging_error_stage: "{{ snow_log_stages.retrieval }}"
        common_logging_error_actor: "{{ _snow_log_actor }}"
        common_logging_error_category_input: "api"
        common_logging_error_key_input: "{{ snow_error_keys.response_invalid }}"

        common_logging_error_message: >-
          ServiceNow response does not match the requested change
          or does not contain a valid sys_id.

        common_logging_error_recommended_action: >-
          Verify the requested change number and ServiceNow response.

        common_logging_error_version_info: {}


# =============================================================================
# IMPLEMENT + APPROVED GATE
# =============================================================================

- name: "ServiceNow | Validate Implement and Approved gate"
  block:

    - name: "ServiceNow | Require Implement state"
      ansible.builtin.assert:
        that:
          - _snow_state == (snow_required_state | lower)
        fail_msg: >-
          Change is not in Implement state.
        quiet: true


    - name: "ServiceNow | Require Approved approval"
      ansible.builtin.assert:
        that:
          - _snow_approval == (snow_required_approval | lower)
        fail_msg: >-
          Change is not Approved.
        quiet: true


    - name: "ServiceNow | Log successful status gate"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: log_event
      vars:
        common_logging_request_id: "{{ _snow_log_request_id }}"
        common_logging_execution_id: "{{ _snow_log_execution_id }}"
        common_logging_environment: "{{ _snow_log_environment }}"
        common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

        common_logging_actor: "{{ _snow_log_actor }}"
        common_logging_stage: "{{ snow_log_stages.status }}"
        common_logging_result: "success"

        common_logging_message: >-
          ServiceNow change is in Implement state
          and has Approved approval.

        common_logging_version_info: {}


  rescue:

    - name: "ServiceNow | Stop on invalid state or approval"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: "{{ _snow_log_request_id }}"
        common_logging_execution_id: "{{ _snow_log_execution_id }}"
        common_logging_environment: "{{ _snow_log_environment }}"
        common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

        common_logging_error_stage: "{{ snow_log_stages.status }}"
        common_logging_error_actor: "{{ _snow_log_actor }}"
        common_logging_error_category_input: "business-rule"
        common_logging_error_key_input: "{{ snow_error_keys.status_invalid }}"

        common_logging_error_message: >-
          Change {{ akamai_chg_number }} must be both
          Implement and Approved.

        common_logging_error_recommended_action: >-
          Ensure the ServiceNow change is in Implement state
          and has Approved approval before retrying.

        common_logging_error_version_info: {}


# =============================================================================
# IMPLEMENTATION WINDOW GATE
# =============================================================================

- name: "ServiceNow | Validate planned implementation window"
  block:

    - name: "ServiceNow | Require populated implementation window"
      ansible.builtin.assert:
        that:
          - _snow_start_raw | length > 0
          - _snow_end_raw | length > 0

        fail_msg: >-
          ServiceNow implementation window is missing.

        quiet: true


    - name: "ServiceNow | Parse implementation window"
      ansible.builtin.set_fact:

        _snow_start_datetime: >-
          {{
            _snow_start_raw
            | to_datetime(
                snow_datetime_format
              )
          }}

        _snow_end_datetime: >-
          {{
            _snow_end_raw
            | to_datetime(
                snow_datetime_format
              )
          }}

        _snow_now_datetime: >-
          {{
            _snow_now_raw
            | to_datetime(
                snow_datetime_format
              )
          }}


    - name: "ServiceNow | Require valid window ordering"
      ansible.builtin.assert:
        that:
          - _snow_start_datetime < _snow_end_datetime

        fail_msg: >-
          ServiceNow planned start must occur before planned end.

        quiet: true


    - name: "ServiceNow | Require execution inside planned window"
      ansible.builtin.assert:
        that:
          - _snow_now_datetime >= _snow_start_datetime
          - _snow_now_datetime <= _snow_end_datetime

        fail_msg: >-
          Execution is outside the ServiceNow implementation window.

        quiet: true


    - name: "ServiceNow | Log successful implementation-window gate"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: log_event
      vars:
        common_logging_request_id: "{{ _snow_log_request_id }}"
        common_logging_execution_id: "{{ _snow_log_execution_id }}"
        common_logging_environment: "{{ _snow_log_environment }}"
        common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

        common_logging_actor: "{{ _snow_log_actor }}"
        common_logging_stage: "{{ snow_log_stages.window }}"
        common_logging_result: "success"

        common_logging_message: >-
          Current execution time is inside the approved
          ServiceNow implementation window.

        common_logging_version_info: {}


  rescue:

    - name: "ServiceNow | Resolve implementation-window failure"
      ansible.builtin.set_fact:

        _snow_window_error_key: >-
          {{
            snow_error_keys.window_missing
            if
            (
              _snow_start_raw | length == 0
              or
              _snow_end_raw | length == 0
            )
            else
            (
              snow_error_keys.outside_window
              if
              (
                _snow_start_datetime is defined
                and
                _snow_end_datetime is defined
                and
                _snow_now_datetime is defined
                and
                _snow_start_datetime < _snow_end_datetime
                and
                (
                  _snow_now_datetime < _snow_start_datetime
                  or
                  _snow_now_datetime > _snow_end_datetime
                )
              )
              else
              snow_error_keys.window_invalid
            )
          }}


    - name: "ServiceNow | Stop on implementation-window failure"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: "{{ _snow_log_request_id }}"
        common_logging_execution_id: "{{ _snow_log_execution_id }}"
        common_logging_environment: "{{ _snow_log_environment }}"
        common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

        common_logging_error_stage: "{{ snow_log_stages.window }}"
        common_logging_error_actor: "{{ _snow_log_actor }}"
        common_logging_error_category_input: "business-rule"
        common_logging_error_key_input: "{{ _snow_window_error_key }}"

        common_logging_error_message: >-
          ServiceNow planned implementation-window validation failed.

        common_logging_error_recommended_action: >-
          Verify that the change contains a valid planned start
          and end time and run the automation only inside
          the approved implementation window.

        common_logging_error_version_info: {}


# =============================================================================
# SUCCESS RESULT
# =============================================================================

- name: "ServiceNow | Build successful validation result"
  ansible.builtin.set_fact:

    akamai_chg_validated: true

    snow_validation_result:
      result: "PASSED"
      mode: "{{ snow_mode }}"
      source: "{{ snow_fetch_source }}"
      change_number: "{{ _snow_number }}"
      change_sys_id: "{{ _snow_sys_id }}"
      state: "{{ _snow_state }}"
      approval: "{{ _snow_approval }}"
      planned_start_utc: "{{ _snow_start_raw }}"
      planned_end_utc: "{{ _snow_end_raw }}"
      checked_at_utc: "{{ _snow_now_raw }}"
      read_only: true


- name: "ServiceNow | Publish successful result to AAP"
  ansible.builtin.set_stats:
    data:
      snow_validation_result: "{{ snow_validation_result }}"
    per_host: false


- name: "ServiceNow | Log successful validation"
  ansible.builtin.include_role:
    name: common_logging
    tasks_from: log_event
  vars:
    common_logging_request_id: "{{ _snow_log_request_id }}"
    common_logging_execution_id: "{{ _snow_log_execution_id }}"
    common_logging_environment: "{{ _snow_log_environment }}"
    common_logging_triggered_by: "{{ _snow_log_triggered_by }}"

    common_logging_actor: "{{ _snow_log_actor }}"
    common_logging_stage: "{{ snow_log_stages.validation }}"
    common_logging_result: "success"

    common_logging_message: >-
      ServiceNow change validation completed successfully.
      The operation was read-only.

    common_logging_version_info: {}


- name: "ServiceNow | Display validation summary"
  ansible.builtin.debug:
    msg:
      result: "{{ snow_validation_result.result }}"
      source: "{{ snow_validation_result.source }}"
      change_number: "{{ snow_validation_result.change_number }}"
      state: "{{ snow_validation_result.state }}"
      approval: "{{ snow_validation_result.approval }}"
      planned_start_utc: "{{ snow_validation_result.planned_start_utc }}"
      planned_end_utc: "{{ snow_validation_result.planned_end_utc }}"
      read_only: true
```

---

# 7. Mock Fixtures

The existing fixture structure can remain because the common normalization layer supports scalar mock fields and mapping-style live fields.

## `mocks/snow/valid_implement_approved.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012345",
    "sys_id": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
    "state": "Implement",
    "approval": "Approved",
    "start_date": "2020-01-01 00:00:00",
    "end_date": "2099-12-31 23:59:59"
  }
}
```

## `mocks/snow/invalid_state.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012346",
    "sys_id": "b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5",
    "state": "Draft",
    "approval": "Approved",
    "start_date": "2020-01-01 00:00:00",
    "end_date": "2099-12-31 23:59:59"
  }
}
```

## `mocks/snow/invalid_not_approved.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012347",
    "sys_id": "c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6",
    "state": "Implement",
    "approval": "Not Yet Requested",
    "start_date": "2020-01-01 00:00:00",
    "end_date": "2099-12-31 23:59:59"
  }
}
```

## `mocks/snow/missing_window.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012350",
    "sys_id": "f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3",
    "state": "Implement",
    "approval": "Approved",
    "start_date": "",
    "end_date": ""
  }
}
```

## `mocks/snow/not_found.json`

```json
{
  "http_status": 404,
  "error": "No change request record found",
  "result": {}
}
```

## `mocks/snow/auth_failure.json`

```json
{
  "http_status": 401,
  "error": "User Not Authenticated",
  "result": {}
}
```

## `mocks/snow/timeout.json`

```json
{
  "http_status": 504,
  "error": "Gateway Timeout",
  "result": {}
}
```

## `mocks/snow/outside_window_early.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012348",
    "sys_id": "d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1",
    "state": "Implement",
    "approval": "Approved",
    "start_date": "2037-01-01 00:00:00",
    "end_date": "2037-01-01 04:00:00"
  }
}
```

## `mocks/snow/outside_window_late.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012349",
    "sys_id": "e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
    "state": "Implement",
    "approval": "Approved",
    "start_date": "2021-01-01 00:00:00",
    "end_date": "2021-01-01 04:00:00"
  }
}
```

---

# 8. Standalone SSPA-11 Playbook

## `playbooks/akamai_version_push/snow_validation.yml`

```yaml
---
- name: "Validate ServiceNow change request"
  hosts: localhost
  connection: local
  gather_facts: false
  any_errors_fatal: true

  tasks:

    - name: "Run ServiceNow change validation"
      ansible.builtin.include_role:
        name: akamai_version_push
        tasks_from: snow_validation
```

---

# 9. AAP Custom Credential Contract

The AAP ServiceNow Custom Credential should inject:

```text
snow_host
snow_username
snow_password
```

Example injector:

```yaml
extra_vars:
  snow_host: "{{ host }}"
  snow_username: "{{ username }}"
  snow_password: "{{ password }}"
```

These values must not be stored in:

```text
defaults/main.yml
vars/main.yml
inventory
survey
Git
```

The live module consumes them only at runtime:

```yaml
instance:
  host: "{{ snow_host }}"
  username: "{{ snow_username }}"
  password: "{{ snow_password }}"
```

---

# 10. `collections/requirements.yml`

Ensure the approved ServiceNow collection is present in the Execution Environment.

```yaml
---
collections:
  - name: community.general

  - name: servicenow.itsm
```

If the project pins collection versions, use the version approved by the AAP/EE team.

---

# 11. Mock AAP Extra Vars

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-TC01"

akamai_chg_number: "CHG0012345"

snow_mode: "mock"

snow_mock_scenario: "valid_implement_approved"
```

---

# 12. Live AAP Extra Vars

```yaml
akamai:
  environment:
    name: "qa"

common_logging_request_id: "SNOW-LIVE-001"

akamai_chg_number: "CHG0123456"

snow_mode: "live"
```

The ServiceNow credential is attached to the Job Template and supplies the authentication variables.

---

# 13. `common_logging` Alignment Changes Made

## Change 1 — Removed separate live-validation appearance

The previous:

```text
snow_validation.yml
snow_live_attempt.yml
```

was replaced by:

```text
snow_validation.yml
snow_fetch.yml
```

`snow_fetch.yml` contains both retrieval sources.

This makes the intended architecture explicit:

```text
mock retrieval ─┐
                ├─> snow_chg ─> same validator
live retrieval ─┘
```

---

## Change 2 — No local numeric error-code mapping

SSPA-11 supplies only categories such as:

```text
validation
business-rule
auth
api
rate-limit
timeout
```

`common_logging` owns the numeric mapping.

---

## Change 3 — Stable uppercase error keys retained

Examples:

```text
SERVICENOW_AUTH_FAILED
SERVICENOW_STATUS_INVALID
SERVICENOW_OUTSIDE_WINDOW
```

These are compatible with the latest `common_logging` error-key contract.

---

## Change 4 — `common_logging_version_info` remains empty

The latest shared role permits only approved Akamai/Jira/version metadata keys.

ServiceNow-specific fields such as:

```text
change_number
approval
state
start_date
end_date
```

are not part of that allowed map.

Therefore SSPA-11 correctly uses:

```yaml
common_logging_version_info: {}
```

---

## Change 5 — Raw ServiceNow errors are not logged

Raw module failures remain internal under `no_log: true`.

Only sanitized messages are passed to `common_logging`.

This protects against accidental credential/token/request leakage and complies with the shared sensitive-text validation.

---

## Change 6 — Invalid environment no longer silently continues as `local`

Missing environment can use `local` for standalone execution.

If a non-supported environment is explicitly supplied, configuration validation fails.

A safe `local` value is used only to allow `common_logging` itself to emit that standardized configuration failure.

---

## Change 7 — Shared `log_event` / `fail_with_error` contract is reused

No SSPA-11-specific audit-event structure is created.

Successful/non-terminal events use:

```text
common_logging/tasks/log_event.yml
```

Terminal failures use:

```text
common_logging/tasks/fail_with_error.yml
```

---

# 14. Business Logic Is Now Identical for Mock and Live

After the fetch task finishes, there is no business-validation condition like:

```yaml
when: snow_mode == 'mock'
```

or:

```yaml
when: snow_mode == 'live'
```

The same code validates:

```text
Mandatory fields
CHG identity
sys_id
Implement state
Approved approval
Planned start/end
Window ordering
Current execution inside window
```

This is the key requirement behind the refactor.

---

# 15. Testing Impact

Mock testing continues to cover the business-rule permutations:

```text
Valid CHG
Wrong state
Not approved
Missing window
Before window
After window
Not found
Authentication failure
Timeout
```

Live testing should verify the integration boundary:

```text
AAP Credential injection
ServiceNow connectivity
TLS trust
ServiceNow module execution
Actual records[] structure
Actual display/raw field structure
Normalization into snow_chg
Successful passage through the SAME common validator
```

There is no need to create a second copy of the entire business-rule test suite for live mode because the validator is shared.

---

# 16. Controlled Live-Test Note

Before declaring the live path fully validated, confirm the actual ServiceNow response for:

```text
state
approval
start_date
end_date
```

when using:

```yaml
sysparm_display_value: "all"
```

The common normalization code supports both scalar mock values and `{value, display_value}`-style fields, but the actual date/time structure and timezone semantics should be confirmed with the first controlled live CHG.

---

# Final File Layout

```text
roles/
└── akamai_version_push/
    ├── defaults/
    │   └── main.yml
    ├── vars/
    │   └── main.yml
    ├── mocks/
    │   └── snow/
    │       ├── auth_failure.json
    │       ├── invalid_not_approved.json
    │       ├── invalid_state.json
    │       ├── missing_window.json
    │       ├── not_found.json
    │       ├── outside_window_early.json
    │       ├── outside_window_late.json
    │       ├── timeout.json
    │       └── valid_implement_approved.json
    └── tasks/
        ├── main.yml
        ├── snow_fetch.yml
        └── snow_validation.yml

playbooks/
└── akamai_version_push/
    └── snow_validation.yml
```

The previous `snow_live_attempt.yml` is no longer required after this refactor.
