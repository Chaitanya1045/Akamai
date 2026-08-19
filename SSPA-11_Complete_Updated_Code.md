# SSPA-11 — Complete Updated ServiceNow Change Validation Code

## Final Design

This version uses the same business-validation logic for both mock and live ServiceNow data.

```text
                    INPUT
                      │
              akamai_chg_number
                      │
          ┌───────────┴───────────┐
          │                       │
        MOCK                     LIVE
          │                       │
   Local JSON fixture      ServiceNow ITSM module
          │                       │
          └───────────┬───────────┘
                      │
                   snow_chg
                      │
                      ▼
             COMMON VALIDATION
                      │
        CHG identity / sys_id
                      │
              State = Implement
                      │
             Approval = Approved
                      │
          Implementation window
                      │
                      ▼
                     PASS
```

The source-specific code only retrieves and normalizes the ServiceNow record into `snow_chg`.

The common validation file does **not** contain separate mock/live business rules.

The previous `snow_live_attempt.yml` helper is no longer required in this design.

---

# Directory Structure

```text
roles/
└── akamai_version_push/
    ├── defaults/
    │   └── main.yml
    ├── vars/
    │   └── main.yml
    ├── tasks/
    │   ├── main.yml
    │   ├── snow_fetch.yml
    │   └── snow_validation.yml
    └── mocks/
        └── snow/
            ├── auth_failure.json
            ├── invalid_not_approved.json
            ├── invalid_state.json
            ├── missing_window.json
            ├── not_found.json
            ├── outside_window_early.json
            ├── outside_window_late.json
            ├── timeout.json
            └── valid_implement_approved.json

playbooks/
└── akamai_version_push/
    └── snow_validation.yml

collections/
└── requirements.yml
```

---

# 1. `roles/akamai_version_push/defaults/main.yml`

Merge this section into the existing defaults file.

```yaml
---
# =============================================================================
# ServiceNow change validation
# =============================================================================

# Supported values:
#   mock - use local JSON fixtures
#   live - retrieve the change from ServiceNow
snow_mode: "mock"

# Used only when snow_mode == mock.
snow_mock_scenario: "valid_implement_approved"

# Validate the ServiceNow TLS certificate.
snow_validate_certs: true

# API connection timeout in seconds.
snow_api_timeout: 30

# Raw ServiceNow date format used for implementation-window comparison.
snow_datetime_format: "%Y-%m-%d %H:%M:%S"
```

Do not store the following variables in defaults:

```text
snow_host
snow_username
snow_password
```

They are supplied at runtime by the AAP ServiceNow credential.

---

# 2. `roles/akamai_version_push/vars/main.yml`

Merge this section into the existing vars file.

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
# Mock HTTP statuses treated as transient/retryable
# =============================================================================

snow_retryable_mock_statuses:
  - 408
  - 429
  - 500
  - 502
  - 503
  - 504


# =============================================================================
# Live ServiceNow retry configuration
# =============================================================================

snow_live_retry_count: 3
snow_live_retry_delay: 2


# =============================================================================
# Supported logging environments
#
# Keep this local to akamai_version_push.
# Do not reference common_logging_allowed_environments before the
# common_logging role is loaded.
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

# 3. `roles/akamai_version_push/tasks/main.yml`

Keep the existing role dispatcher and add the ServiceNow retrieval and validation tasks in sequence.

```yaml
---
# Existing task imports for other stories remain here.

- name: "Retrieve ServiceNow change data"
  ansible.builtin.import_tasks:
    snow_fetch.yml

- name: "Validate ServiceNow change request"
  ansible.builtin.import_tasks:
    snow_validation.yml

# Existing task imports for other stories continue here.
```

The important ordering is:

```text
snow_fetch.yml
      ↓
creates snow_chg
      ↓
snow_validation.yml
      ↓
common business validation
```

---

# 4. `roles/akamai_version_push/tasks/snow_fetch.yml`

This file owns **data retrieval only**.

Both mock and live paths must produce the same variable:

```text
snow_chg
```

```yaml
---
# =============================================================================
# ServiceNow change retrieval
#
# MOCK:
#   Local JSON fixture
#       ↓
#   snow_chg
#
# LIVE:
#   servicenow.itsm.change_request_info
#       ↓
#   records[0]
#       ↓
#   snow_chg
#
# No state / approval / implementation-window business validation is
# performed in this file.
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
# Resolve requested environment
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
        | trim
        | lower
      }}


# =============================================================================
# Resolve safe logging environment
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
# Initialize state
# =============================================================================

- name: "ServiceNow | Initialize retrieval state"
  ansible.builtin.set_fact:
    akamai_chg_validated: false
    snow_chg: {}
    snow_fetch_source: ""


# =============================================================================
# Validate local configuration before any external API call
# =============================================================================

- name: "ServiceNow | Validate local configuration"
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

          - snow_api_timeout | int > 0

          - snow_live_retry_count | int > 0

          - snow_live_retry_delay | int >= 0

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


# =============================================================================
# Log retrieval start
# =============================================================================

- name: "ServiceNow | Log retrieval start"
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
      Started read-only ServiceNow change retrieval
      using {{ snow_mode }} data source.

    common_logging_version_info: {}


# =============================================================================
# MOCK RETRIEVAL
# =============================================================================

- name: "ServiceNow | Retrieve mock change"
  when:
    - snow_mode == 'mock'
  block:

    - name: "ServiceNow | Read mock fixture"
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

        common_logging_error_key_input: >-
          {{ snow_error_keys.mock_invalid }}

        common_logging_error_message: >-
          ServiceNow mock fixture could not be loaded
          or contains an invalid response envelope.

        common_logging_error_recommended_action: >-
          Verify the selected mock fixture and JSON structure.

        common_logging_error_version_info: {}


# =============================================================================
# MOCK - authentication failure
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

    common_logging_error_key_input: >-
      {{ snow_error_keys.authentication_failed }}

    common_logging_error_message: >-
      ServiceNow authentication failed.

    common_logging_error_recommended_action: >-
      Verify the ServiceNow AAP credential and
      ServiceNow service-account access.

    common_logging_error_version_info: {}

  when:
    - snow_mode == 'mock'
    - _snow_raw.http_status | int in [401, 403]


# =============================================================================
# MOCK - change not found
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

    common_logging_error_key_input: >-
      {{ snow_error_keys.change_not_found }}

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
# MOCK - retryable/transient failure
# =============================================================================

- name: "ServiceNow | Handle mock transient API failure"
  when:
    - snow_mode == 'mock'
    - >-
      _snow_raw.http_status
      | int
      in snow_retryable_mock_statuses
  block:

    - name: "ServiceNow | Display simulated retry attempts"
      ansible.builtin.debug:
        msg: >-
          Simulated ServiceNow attempt {{ item }} of
          {{ snow_live_retry_count }} failed with
          HTTP {{ _snow_raw.http_status }}.
      loop: >-
        {{
          range(
            1,
            (snow_live_retry_count | int) + 1
          )
          | list
        }}


    - name: "ServiceNow | Resolve simulated terminal status"
      ansible.builtin.set_fact:

        _snow_mock_terminal_status: >-
          {{
            _snow_raw.http_status
            | int
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

        common_logging_error_category_input: >-
          {{
            'timeout'
            if
            _snow_mock_terminal_status
            in [408, 504]
            else
            (
              'rate-limit'
              if
              _snow_mock_terminal_status == 429
              else
              'api'
            )
          }}

        common_logging_error_key_input: >-
          {{
            snow_error_keys.timeout
            if
            _snow_mock_terminal_status
            in [408, 504]
            else
            (
              snow_error_keys.rate_limited
              if
              _snow_mock_terminal_status == 429
              else
              snow_error_keys.retrieval_failed
            )
          }}

        common_logging_error_message: >-
          ServiceNow request failed after
          {{ snow_live_retry_count }}
          simulated attempts.

        common_logging_error_recommended_action: >-
          Verify ServiceNow availability, network access,
          API health and rate limits.

        common_logging_error_version_info: {}


# =============================================================================
# MOCK - unexpected API status
# =============================================================================

- name: "ServiceNow | Handle unexpected mock API status"
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

    common_logging_error_key_input: >-
      {{ snow_error_keys.retrieval_failed }}

    common_logging_error_message: >-
      ServiceNow mock request returned unexpected
      HTTP status {{ _snow_raw.http_status }}.

    common_logging_error_recommended_action: >-
      Verify the mock response and expected
      ServiceNow API behavior.

    common_logging_error_version_info: {}

  when:
    - snow_mode == 'mock'
    - _snow_raw.http_status | int != 200
    - _snow_raw.http_status | int not in [401, 403, 404]
    - >-
      _snow_raw.http_status
      | int
      not in snow_retryable_mock_statuses


# =============================================================================
# MOCK - normalize successful response to common contract
# =============================================================================

- name: "ServiceNow | Normalize successful mock response"
  when:
    - snow_mode == 'mock'
    - _snow_raw.http_status | int == 200
  block:

    - name: "ServiceNow | Validate mock change record"
      ansible.builtin.assert:
        that:

          - _snow_raw.result is defined

          - _snow_raw.result is mapping

          - _snow_raw.result | length > 0

        fail_msg: >-
          ServiceNow mock response does not
          contain a valid change record.

        quiet: true


    - name: "ServiceNow | Set common change object from mock response"
      ansible.builtin.set_fact:

        snow_chg: "{{ _snow_raw.result }}"

        snow_fetch_source: >-
          mock:{{ snow_mock_scenario }}

      no_log: true


  rescue:

    - name: "ServiceNow | Stop on invalid mock change record"
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

        common_logging_error_key_input: >-
          {{ snow_error_keys.response_invalid }}

        common_logging_error_message: >-
          ServiceNow mock response does not
          contain a valid change record.

        common_logging_error_recommended_action: >-
          Verify the selected mock JSON structure.

        common_logging_error_version_info: {}


# =============================================================================
# LIVE - validate AAP credential injection
#
# AAP Custom Credential injects:
#
#   snow_host
#   snow_username
#   snow_password
# =============================================================================

- name: "ServiceNow | Validate live credential injection"
  when:
    - snow_mode == 'live'
  block:

    - name: "ServiceNow | Require AAP-injected credentials"
      ansible.builtin.assert:
        that:

          - snow_host is defined

          - >-
            snow_host
            | default('')
            | string
            | trim
            | length > 0

          - snow_username is defined

          - >-
            snow_username
            | default('')
            | string
            | trim
            | length > 0

          - snow_password is defined

          - >-
            snow_password
            | default('')
            | string
            | length > 0

        fail_msg: >-
          Required ServiceNow credential values
          were not injected by AAP.

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

        common_logging_error_key_input: >-
          {{ snow_error_keys.authentication_failed }}

        common_logging_error_message: >-
          ServiceNow credentials were not injected by AAP.

        common_logging_error_recommended_action: >-
          Attach the approved ServiceNow credential
          to the AAP Job Template.

        common_logging_error_version_info: {}


# =============================================================================
# LIVE - retrieve change from ServiceNow
# =============================================================================

- name: "ServiceNow | Retrieve live change"
  when:
    - snow_mode == 'live'
  block:

    - name: "ServiceNow | Retrieve change request"
      servicenow.itsm.change_request_info:
        instance:

          host: "{{ snow_host }}"

          username: "{{ snow_username }}"

          password: "{{ snow_password }}"

          timeout: "{{ snow_api_timeout | float }}"

          validate_certs: "{{ snow_validate_certs | bool }}"

        number: "{{ akamai_chg_number }}"

        # Return raw and display values.
        sysparm_display_value: "all"

      register: _snow_response

      changed_when: false

      retries: "{{ snow_live_retry_count | int }}"

      delay: "{{ snow_live_retry_delay | int }}"

      until:
        - _snow_response is succeeded

      no_log: true


  rescue:

    - name: "ServiceNow | Capture live retrieval failure"
      ansible.builtin.set_fact:

        _snow_live_error_text: >-
          {{
            ansible_failed_result.msg
            | default(
                'ServiceNow request failed.',
                true
              )
            | string
            | lower
          }}

      no_log: true


    - name: "ServiceNow | Classify live retrieval failure"
      ansible.builtin.set_fact:

        _snow_live_error_category: >-
          {{
            'auth'
            if
            (
              _snow_live_error_text
              is search(
                '401|403|unauthorized|forbidden|authentication|credential'
              )
            )
            else
            (
              'timeout'
              if
              (
                _snow_live_error_text
                is search(
                  'timeout|timed out|readtimeout|connecttimeout'
                )
              )
              else
              (
                'rate-limit'
                if
                (
                  _snow_live_error_text
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
              _snow_live_error_text
              is search(
                '401|403|unauthorized|forbidden|authentication|credential'
              )
            )
            else
            (
              snow_error_keys.timeout
              if
              (
                _snow_live_error_text
                is search(
                  'timeout|timed out|readtimeout|connecttimeout'
                )
              )
              else
              (
                snow_error_keys.rate_limited
                if
                (
                  _snow_live_error_text
                  is search(
                    '429|rate limit|too many requests'
                  )
                )
                else
                snow_error_keys.retrieval_failed
              )
            )
          }}


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
            if
            _snow_live_error_category == 'auth'
            else
            snow_log_stages.retrieval
          }}

        common_logging_error_actor: "{{ _snow_log_actor }}"

        common_logging_error_category_input: >-
          {{ _snow_live_error_category }}

        common_logging_error_key_input: >-
          {{ _snow_live_error_key }}

        common_logging_error_message: >-
          ServiceNow change retrieval failed after
          {{ snow_live_retry_count }} attempts.

        common_logging_error_recommended_action: >-
          Verify AAP credentials, ServiceNow availability,
          certificate trust, network connectivity,
          API permissions and rate limits.

        common_logging_error_version_info: {}


# =============================================================================
# LIVE - validate records returned by ServiceNow
# =============================================================================

- name: "ServiceNow | Validate live response collection"
  when:
    - snow_mode == 'live'
  block:

    - name: "ServiceNow | Require records collection"
      ansible.builtin.assert:
        that:

          - _snow_response.records is defined

          - _snow_response.records is sequence

          - _snow_response.records is not string

        fail_msg: >-
          ServiceNow did not return a valid
          change record collection.

        quiet: true


  rescue:

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

        common_logging_error_category_input: "api"

        common_logging_error_key_input: >-
          {{ snow_error_keys.response_invalid }}

        common_logging_error_message: >-
          ServiceNow returned an invalid change response.

        common_logging_error_recommended_action: >-
          Verify the ServiceNow response structure
          and service-account read permissions.

        common_logging_error_version_info: {}


# =============================================================================
# LIVE - zero records
# =============================================================================

- name: "ServiceNow | Stop when live change is not found"
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

    common_logging_error_key_input: >-
      {{ snow_error_keys.change_not_found }}

    common_logging_error_message: >-
      Change {{ akamai_chg_number }}
      was not found in ServiceNow.

    common_logging_error_recommended_action: >-
      Verify the supplied change number and
      ServiceNow read permissions.

    common_logging_error_version_info: {}

  when:
    - snow_mode == 'live'
    - _snow_response.records | length == 0


# =============================================================================
# LIVE - require unique result
#
# The module documentation notes that number may not be unique,
# so protect the workflow from an ambiguous response.
# =============================================================================

- name: "ServiceNow | Validate unique live change result"
  when:
    - snow_mode == 'live'
    - _snow_response.records | length > 0
  block:

    - name: "ServiceNow | Require exactly one matching change"
      ansible.builtin.assert:
        that:

          - _snow_response.records | length == 1

        fail_msg: >-
          ServiceNow returned more than one record
          for {{ akamai_chg_number }}.

        quiet: true


  rescue:

    - name: "ServiceNow | Stop on ambiguous live result"
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

        common_logging_error_key_input: >-
          {{ snow_error_keys.response_invalid }}

        common_logging_error_message: >-
          ServiceNow returned multiple records
          for the supplied change number.

        common_logging_error_recommended_action: >-
          Verify the ServiceNow change number
          and source data.

        common_logging_error_version_info: {}


# =============================================================================
# LIVE - normalize successful response to common contract
# =============================================================================

- name: "ServiceNow | Set common change object from live response"
  ansible.builtin.set_fact:

    snow_chg: "{{ _snow_response.records[0] }}"

    snow_fetch_source: "live"

  when:
    - snow_mode == 'live'
    - _snow_response.records | length == 1

  no_log: true


# =============================================================================
# Confirm common retrieval contract
# =============================================================================

- name: "ServiceNow | Validate common retrieval contract"
  block:

    - name: "ServiceNow | Require normalized change object"
      ansible.builtin.assert:
        that:

          - snow_chg is defined

          - snow_chg is mapping

          - snow_chg | length > 0

          - snow_fetch_source | default('') | length > 0

        fail_msg: >-
          ServiceNow retrieval did not produce
          the required common change object.

        quiet: true


  rescue:

    - name: "ServiceNow | Stop when common retrieval contract is invalid"
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

        common_logging_error_key_input: >-
          {{ snow_error_keys.response_invalid }}

        common_logging_error_message: >-
          ServiceNow retrieval did not produce
          a valid normalized change record.

        common_logging_error_recommended_action: >-
          Verify the selected data source and
          returned ServiceNow record structure.

        common_logging_error_version_info: {}


# =============================================================================
# Log successful retrieval
# =============================================================================

- name: "ServiceNow | Log successful retrieval"
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
      ServiceNow change retrieval completed successfully
      from {{ snow_fetch_source }}.

    common_logging_version_info: {}
```

---

# 5. `roles/akamai_version_push/tasks/snow_validation.yml`

This file contains the **single common business-validation path**.

There are no separate mock/live validation branches here.

```yaml
---
# =============================================================================
# COMMON ServiceNow change validation
#
# Input contract:
#
#   snow_chg
#
# snow_fetch.yml guarantees that both mock and live sources produce this
# same object before this file executes.
# =============================================================================


# =============================================================================
# Validate role-required business configuration
# =============================================================================

- name: "ServiceNow | Validate required business configuration"
  block:

    - name: "ServiceNow | Require configured deployment gates"
      ansible.builtin.assert:
        that:

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

        fail_msg: >-
          ServiceNow business-gate configuration is invalid.

        quiet: true


  rescue:

    - name: "ServiceNow | Stop on invalid business configuration"
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
          ServiceNow business-gate configuration is invalid.

        common_logging_error_recommended_action: >-
          Verify the configured required state and approval values.

        common_logging_error_version_info: {}


# =============================================================================
# Validate mandatory fields returned by either source
# =============================================================================

- name: "ServiceNow | Validate mandatory change fields"
  block:

    - name: "ServiceNow | Require mandatory fields"
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
          one or more mandatory fields.

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

        common_logging_error_key_input: >-
          {{ snow_error_keys.response_invalid }}

        common_logging_error_message: >-
          ServiceNow response is missing one or more
          required change fields.

        common_logging_error_recommended_action: >-
          Verify ServiceNow field availability
          and service-account read permissions.

        common_logging_error_version_info: {}


# =============================================================================
# Normalize identifiers
# =============================================================================

- name: "ServiceNow | Normalize identifiers"
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


# =============================================================================
# Normalize state and approval
#
# Prefer display values for human-readable business rules.
# =============================================================================

- name: "ServiceNow | Normalize state and approval"
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


# =============================================================================
# Normalize planned implementation window
#
# Prefer raw "value" when sysparm_display_value=all is returned by live SNOW.
# =============================================================================

- name: "ServiceNow | Normalize planned implementation window"
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
# Normalize optional metadata
# =============================================================================

- name: "ServiceNow | Normalize optional metadata"
  ansible.builtin.set_fact:

    _snow_short_description: >-
      {{
        (
          snow_chg.short_description.display_value
          if
          (
            snow_chg.short_description is defined
            and
            snow_chg.short_description is mapping
            and
            snow_chg.short_description.display_value is defined
          )
          else
          (
            snow_chg.short_description.value
            if
            (
              snow_chg.short_description is defined
              and
              snow_chg.short_description is mapping
              and
              snow_chg.short_description.value is defined
            )
            else
            (
              snow_chg.short_description
              if
              snow_chg.short_description is defined
              else
              ''
            )
          )
        )
        | default('')
        | string
        | trim
      }}

    _snow_assigned_to: >-
      {{
        (
          snow_chg.assigned_to.display_value
          if
          (
            snow_chg.assigned_to is defined
            and
            snow_chg.assigned_to is mapping
            and
            snow_chg.assigned_to.display_value is defined
          )
          else
          (
            snow_chg.assigned_to.value
            if
            (
              snow_chg.assigned_to is defined
              and
              snow_chg.assigned_to is mapping
              and
              snow_chg.assigned_to.value is defined
            )
            else
            (
              snow_chg.assigned_to
              if
              snow_chg.assigned_to is defined
              else
              ''
            )
          )
        )
        | default('')
        | string
        | trim
      }}

    _snow_requested_by: >-
      {{
        (
          snow_chg.requested_by.display_value
          if
          (
            snow_chg.requested_by is defined
            and
            snow_chg.requested_by is mapping
            and
            snow_chg.requested_by.display_value is defined
          )
          else
          (
            snow_chg.requested_by.value
            if
            (
              snow_chg.requested_by is defined
              and
              snow_chg.requested_by is mapping
              and
              snow_chg.requested_by.value is defined
            )
            else
            (
              snow_chg.requested_by
              if
              snow_chg.requested_by is defined
              else
              ''
            )
          )
        )
        | default('')
        | string
        | trim
      }}

    _snow_opened_by: >-
      {{
        (
          snow_chg.opened_by.display_value
          if
          (
            snow_chg.opened_by is defined
            and
            snow_chg.opened_by is mapping
            and
            snow_chg.opened_by.display_value is defined
          )
          else
          (
            snow_chg.opened_by.value
            if
            (
              snow_chg.opened_by is defined
              and
              snow_chg.opened_by is mapping
              and
              snow_chg.opened_by.value is defined
            )
            else
            (
              snow_chg.opened_by
              if
              snow_chg.opened_by is defined
              else
              ''
            )
          )
        )
        | default('')
        | string
        | trim
      }}

    _snow_assignment_group: >-
      {{
        (
          snow_chg.assignment_group.display_value
          if
          (
            snow_chg.assignment_group is defined
            and
            snow_chg.assignment_group is mapping
            and
            snow_chg.assignment_group.display_value is defined
          )
          else
          (
            snow_chg.assignment_group.value
            if
            (
              snow_chg.assignment_group is defined
              and
              snow_chg.assignment_group is mapping
              and
              snow_chg.assignment_group.value is defined
            )
            else
            (
              snow_chg.assignment_group
              if
              snow_chg.assignment_group is defined
              else
              ''
            )
          )
        )
        | default('')
        | string
        | trim
      }}


# =============================================================================
# Validate returned CHG identity
# =============================================================================

- name: "ServiceNow | Validate change identity"
  block:

    - name: "ServiceNow | Require expected CHG and sys_id"
      ansible.builtin.assert:
        that:

          - _snow_number == akamai_chg_number

          - _snow_sys_id | length > 0

        fail_msg: >-
          ServiceNow returned an invalid or
          unexpected change record.

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

        common_logging_error_key_input: >-
          {{ snow_error_keys.response_invalid }}

        common_logging_error_message: >-
          ServiceNow response does not match the
          requested change number or does not
          contain a sys_id.

        common_logging_error_recommended_action: >-
          Verify the supplied change number and
          returned ServiceNow record.

        common_logging_error_version_info: {}


# =============================================================================
# State + approval gate
#
# BOTH must pass.
# =============================================================================

- name: "ServiceNow | Validate Implement and Approved gate"
  block:

    - name: "ServiceNow | Require Implement state"
      ansible.builtin.assert:
        that:

          - >-
            _snow_state
            ==
            (
              snow_required_state
              | string
              | trim
              | lower
            )

        fail_msg: >-
          Change is not in Implement state.

        quiet: true


    - name: "ServiceNow | Require Approved approval"
      ansible.builtin.assert:
        that:

          - >-
            _snow_approval
            ==
            (
              snow_required_approval
              | string
              | trim
              | lower
            )

        fail_msg: >-
          Change is not Approved.

        quiet: true


    - name: "ServiceNow | Log successful state and approval gate"
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

        common_logging_error_key_input: >-
          {{ snow_error_keys.status_invalid }}

        common_logging_error_message: >-
          Change {{ akamai_chg_number }}
          must be both Implement and Approved.
          Current state={{ _snow_state }},
          approval={{ _snow_approval }}.

        common_logging_error_recommended_action: >-
          Ensure the ServiceNow change is in
          Implement state and has Approved approval.

        common_logging_error_version_info: {}


# =============================================================================
# Planned implementation window
# =============================================================================

- name: "ServiceNow | Validate planned implementation window"
  block:

    - name: "ServiceNow | Require populated planned window"
      ansible.builtin.assert:
        that:

          - _snow_start_raw | length > 0

          - _snow_end_raw | length > 0

        fail_msg: >-
          ServiceNow implementation window is missing.

        quiet: true


    - name: "ServiceNow | Parse planned implementation window"
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

          - >-
            _snow_start_datetime
            <
            _snow_end_datetime

        fail_msg: >-
          ServiceNow planned start must occur
          before planned end.

        quiet: true


    - name: "ServiceNow | Require execution inside approved window"
      ansible.builtin.assert:
        that:

          - >-
            _snow_now_datetime
            >=
            _snow_start_datetime

          - >-
            _snow_now_datetime
            <=
            _snow_end_datetime

        fail_msg: >-
          Execution is outside the ServiceNow
          implementation window.

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
          Current execution time is inside
          the approved ServiceNow implementation window.

        common_logging_version_info: {}


  rescue:

    - name: "ServiceNow | Resolve planned-window failure"
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

        _snow_window_failure_message: >-
          {{
            ansible_failed_result.msg
            | default(
                'ServiceNow planned implementation-window validation failed.',
                true
              )
            | string
          }}


    - name: "ServiceNow | Stop on planned-window failure"
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

        common_logging_error_key_input: >-
          {{ _snow_window_error_key }}

        common_logging_error_message: >-
          {{ _snow_window_failure_message }}

        common_logging_error_recommended_action: >-
          Verify that the change contains a valid
          planned start and end time and run the
          automation only inside the approved window.

        common_logging_error_version_info: {}


# =============================================================================
# Publish successful validation result
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

      short_description: "{{ _snow_short_description }}"

      assigned_to: "{{ _snow_assigned_to }}"

      requested_by: "{{ _snow_requested_by }}"

      opened_by: "{{ _snow_opened_by }}"

      assignment_group: "{{ _snow_assignment_group }}"

      planned_start_utc: "{{ _snow_start_raw }}"

      planned_end_utc: "{{ _snow_end_raw }}"

      checked_at_utc: "{{ _snow_now_raw }}"

      read_only: true


- name: "ServiceNow | Publish validation result to AAP"
  ansible.builtin.set_stats:
    data:

      snow_validation_result: >-
        {{ snow_validation_result }}

    per_host: false


# =============================================================================
# Final successful common_logging event
# =============================================================================

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


# =============================================================================
# Human-readable AAP summary
# =============================================================================

- name: "ServiceNow | Display validation summary"
  ansible.builtin.debug:
    msg:

      result: "{{ snow_validation_result.result }}"

      source: "{{ snow_validation_result.source }}"

      change_number: "{{ snow_validation_result.change_number }}"

      state: "{{ snow_validation_result.state }}"

      approval: "{{ snow_validation_result.approval }}"

      short_description: >-
        {{ snow_validation_result.short_description }}

      assigned_to: >-
        {{ snow_validation_result.assigned_to }}

      requested_by: >-
        {{ snow_validation_result.requested_by }}

      opened_by: >-
        {{ snow_validation_result.opened_by }}

      assignment_group: >-
        {{ snow_validation_result.assignment_group }}

      planned_start_utc: >-
        {{ snow_validation_result.planned_start_utc }}

      planned_end_utc: >-
        {{ snow_validation_result.planned_end_utc }}

      read_only: true
```

---

# 6. Mock JSON Payloads

The successful/business-rule fixtures use the same general CHG structure and change only the values required for the scenario.

---

## `roles/akamai_version_push/mocks/snow/valid_implement_approved.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012345",
    "sys_id": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
    "short_description": "Akamai property version deployment",
    "description": "Deploy approved Akamai property version through AAP automation.",
    "state": "Implement",
    "approval": "Approved",
    "start_date": "2020-01-01 00:00:00",
    "end_date": "2099-12-31 23:59:59",
    "requested_by": "application.team",
    "opened_by": "application.team",
    "assigned_to": "automation.operator",
    "assignment_group": "Akamai Operations",
    "implementation_plan": "Execute Akamai version promotion through AAP automation.",
    "backout_plan": "Rollback is handled through the approved rollback workflow.",
    "risk": "Low",
    "impact": "3 - Low",
    "priority": "4 - Low",
    "sys_created_on": "2026-08-01 08:30:00",
    "sys_updated_on": "2026-08-19 10:15:00"
  }
}
```

---

## `roles/akamai_version_push/mocks/snow/invalid_state.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012346",
    "sys_id": "b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5",
    "short_description": "Akamai property version deployment",
    "description": "Deploy approved Akamai property version through AAP automation.",
    "state": "Draft",
    "approval": "Approved",
    "start_date": "2020-01-01 00:00:00",
    "end_date": "2099-12-31 23:59:59",
    "requested_by": "application.team",
    "opened_by": "application.team",
    "assigned_to": "automation.operator",
    "assignment_group": "Akamai Operations",
    "implementation_plan": "Execute Akamai version promotion through AAP automation.",
    "backout_plan": "Rollback is handled through the approved rollback workflow.",
    "risk": "Low",
    "impact": "3 - Low",
    "priority": "4 - Low",
    "sys_created_on": "2026-08-01 08:30:00",
    "sys_updated_on": "2026-08-19 10:15:00"
  }
}
```

---

## `roles/akamai_version_push/mocks/snow/invalid_not_approved.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012347",
    "sys_id": "c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6",
    "short_description": "Akamai property version deployment",
    "description": "Deploy approved Akamai property version through AAP automation.",
    "state": "Implement",
    "approval": "Not Yet Requested",
    "start_date": "2020-01-01 00:00:00",
    "end_date": "2099-12-31 23:59:59",
    "requested_by": "application.team",
    "opened_by": "application.team",
    "assigned_to": "automation.operator",
    "assignment_group": "Akamai Operations",
    "implementation_plan": "Execute Akamai version promotion through AAP automation.",
    "backout_plan": "Rollback is handled through the approved rollback workflow.",
    "risk": "Low",
    "impact": "3 - Low",
    "priority": "4 - Low",
    "sys_created_on": "2026-08-01 08:30:00",
    "sys_updated_on": "2026-08-19 10:15:00"
  }
}
```

---

## `roles/akamai_version_push/mocks/snow/outside_window_early.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012348",
    "sys_id": "d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1",
    "short_description": "Akamai property version deployment",
    "description": "Deploy approved Akamai property version through AAP automation.",
    "state": "Implement",
    "approval": "Approved",
    "start_date": "2037-01-01 00:00:00",
    "end_date": "2037-01-01 04:00:00",
    "requested_by": "application.team",
    "opened_by": "application.team",
    "assigned_to": "automation.operator",
    "assignment_group": "Akamai Operations",
    "implementation_plan": "Execute Akamai version promotion through AAP automation.",
    "backout_plan": "Rollback is handled through the approved rollback workflow.",
    "risk": "Low",
    "impact": "3 - Low",
    "priority": "4 - Low",
    "sys_created_on": "2026-08-01 08:30:00",
    "sys_updated_on": "2026-08-19 10:15:00"
  }
}
```

---

## `roles/akamai_version_push/mocks/snow/outside_window_late.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012349",
    "sys_id": "e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
    "short_description": "Akamai property version deployment",
    "description": "Deploy approved Akamai property version through AAP automation.",
    "state": "Implement",
    "approval": "Approved",
    "start_date": "2021-01-01 00:00:00",
    "end_date": "2021-01-01 04:00:00",
    "requested_by": "application.team",
    "opened_by": "application.team",
    "assigned_to": "automation.operator",
    "assignment_group": "Akamai Operations",
    "implementation_plan": "Execute Akamai version promotion through AAP automation.",
    "backout_plan": "Rollback is handled through the approved rollback workflow.",
    "risk": "Low",
    "impact": "3 - Low",
    "priority": "4 - Low",
    "sys_created_on": "2026-08-01 08:30:00",
    "sys_updated_on": "2026-08-19 10:15:00"
  }
}
```

---

## `roles/akamai_version_push/mocks/snow/missing_window.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012350",
    "sys_id": "f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3",
    "short_description": "Akamai property version deployment",
    "description": "Deploy approved Akamai property version through AAP automation.",
    "state": "Implement",
    "approval": "Approved",
    "start_date": "",
    "end_date": "",
    "requested_by": "application.team",
    "opened_by": "application.team",
    "assigned_to": "automation.operator",
    "assignment_group": "Akamai Operations",
    "implementation_plan": "Execute Akamai version promotion through AAP automation.",
    "backout_plan": "Rollback is handled through the approved rollback workflow.",
    "risk": "Low",
    "impact": "3 - Low",
    "priority": "4 - Low",
    "sys_created_on": "2026-08-01 08:30:00",
    "sys_updated_on": "2026-08-19 10:15:00"
  }
}
```

---

## `roles/akamai_version_push/mocks/snow/not_found.json`

```json
{
  "http_status": 404,
  "error": "No change request record found",
  "result": {}
}
```

---

## `roles/akamai_version_push/mocks/snow/auth_failure.json`

```json
{
  "http_status": 401,
  "error": "User Not Authenticated",
  "result": {}
}
```

---

## `roles/akamai_version_push/mocks/snow/timeout.json`

```json
{
  "http_status": 504,
  "error": "Gateway Timeout",
  "result": {}
}
```

---

# 7. Standalone SSPA-11 Playbook

## `playbooks/akamai_version_push/snow_validation.yml`

Use this playbook when testing only SSPA-11 so unrelated role tasks do not execute.

```yaml
---
- name: "Validate ServiceNow change request"
  hosts: localhost

  connection: local

  gather_facts: false

  any_errors_fatal: true

  tasks:

    - name: "Retrieve ServiceNow change data"
      ansible.builtin.include_role:
        name: akamai_version_push
        tasks_from: snow_fetch

    - name: "Run common ServiceNow change validation"
      ansible.builtin.include_role:
        name: akamai_version_push
        tasks_from: snow_validation
```

---

# 8. `collections/requirements.yml`

Merge the ServiceNow collection into the existing requirements file.

```yaml
---
collections:

  - name: community.general

  - name: servicenow.itsm
```

If the project pins collection versions, use the version approved for the AAP Execution Environment.

---

# 9. AAP Custom Credential Type

The ServiceNow credential must inject the connection values at runtime.

## Input Configuration

```yaml
fields:

  - id: host
    type: string
    label: ServiceNow Host

  - id: username
    type: string
    label: ServiceNow Username

  - id: password
    type: string
    label: ServiceNow Password
    secret: true

required:
  - host
  - username
  - password
```

## Injector Configuration

```yaml
extra_vars:

  snow_host: "{{ host }}"

  snow_username: "{{ username }}"

  snow_password: "{{ password }}"
```

The actual AAP credential contains:

```text
ServiceNow Host:
https://<servicenow-instance>

ServiceNow Username:
<service-account>

ServiceNow Password:
********
```

Do not put the actual values in Git, role defaults, role vars, or survey inputs.

---

# 10. Mock AAP Extra Vars

## Happy Path

```yaml
akamai:
  environment:
    name: qa

common_logging_request_id: "SNOW-TC01"

akamai_chg_number: "CHG0012345"

snow_mode: "mock"

snow_mock_scenario: "valid_implement_approved"
```

Expected result:

```text
SUCCESS
```

Expected artifact:

```yaml
snow_validation_result:
  result: PASSED
  mode: mock
  source: mock:valid_implement_approved
  change_number: CHG0012345
  state: implement
  approval: approved
  short_description: Akamai property version deployment
  assigned_to: automation.operator
  requested_by: application.team
  opened_by: application.team
  assignment_group: Akamai Operations
  read_only: true
```

---

# 11. Mock Negative Test Inputs

## Invalid State

```yaml
akamai:
  environment:
    name: qa

common_logging_request_id: "SNOW-TC06"

akamai_chg_number: "CHG0012346"

snow_mode: "mock"

snow_mock_scenario: "invalid_state"
```

Expected:

```text
SERVICENOW_STATUS_INVALID
```

---

## Not Approved

```yaml
akamai:
  environment:
    name: qa

common_logging_request_id: "SNOW-TC07"

akamai_chg_number: "CHG0012347"

snow_mode: "mock"

snow_mock_scenario: "invalid_not_approved"
```

Expected:

```text
SERVICENOW_STATUS_INVALID
```

---

## Before Window

```yaml
akamai:
  environment:
    name: qa

common_logging_request_id: "SNOW-TC09"

akamai_chg_number: "CHG0012348"

snow_mode: "mock"

snow_mock_scenario: "outside_window_early"
```

Expected:

```text
SERVICENOW_OUTSIDE_WINDOW
```

---

## After Window

```yaml
akamai:
  environment:
    name: qa

common_logging_request_id: "SNOW-TC10"

akamai_chg_number: "CHG0012349"

snow_mode: "mock"

snow_mock_scenario: "outside_window_late"
```

Expected:

```text
SERVICENOW_OUTSIDE_WINDOW
```

---

## Missing Window

```yaml
akamai:
  environment:
    name: qa

common_logging_request_id: "SNOW-TC08"

akamai_chg_number: "CHG0012350"

snow_mode: "mock"

snow_mock_scenario: "missing_window"
```

Expected:

```text
SERVICENOW_WINDOW_MISSING
```

---

## Authentication Failure

```yaml
akamai:
  environment:
    name: qa

common_logging_request_id: "SNOW-TC04"

akamai_chg_number: "CHG0012345"

snow_mode: "mock"

snow_mock_scenario: "auth_failure"
```

Expected:

```text
SERVICENOW_AUTH_FAILED
```

---

## Change Not Found

```yaml
akamai:
  environment:
    name: qa

common_logging_request_id: "SNOW-TC05"

akamai_chg_number: "CHG0012351"

snow_mode: "mock"

snow_mock_scenario: "not_found"
```

Expected:

```text
SERVICENOW_CHANGE_NOT_FOUND
```

---

## Timeout

```yaml
akamai:
  environment:
    name: qa

common_logging_request_id: "SNOW-TC11"

akamai_chg_number: "CHG0012352"

snow_mode: "mock"

snow_mock_scenario: "timeout"
```

Expected:

```text
SERVICENOW_API_TIMEOUT
```

---

# 12. Live AAP Extra Vars

Once the ServiceNow service account and API connectivity are available:

```yaml
akamai:
  environment:
    name: qa

common_logging_request_id: "SNOW-LIVE-001"

akamai_chg_number: "<VALID_TEST_CHG>"

snow_mode: "live"
```

The attached AAP credential supplies:

```text
snow_host
snow_username
snow_password
```

The survey/Extra Vars do not contain credentials.

---

# 13. Mock vs Live Responsibility

## Mock path

```text
mock JSON
   ↓
snow_fetch.yml
   ↓
snow_chg
```

## Live path

```text
AAP Credential
   ↓
servicenow.itsm.change_request_info
   ↓
records[0]
   ↓
snow_fetch.yml
   ↓
snow_chg
```

## Common validation

```text
snow_chg
   ↓
snow_validation.yml
   ↓
mandatory fields
   ↓
CHG identity
   ↓
Implement
   ↓
Approved
   ↓
implementation window
   ↓
PASS
```

This is the key design point:

```text
Different retrieval source
        +
Same normalized contract
        +
Same validation logic
```

---

# 14. Remove Old Helper File

With this refactored structure, remove:

```text
roles/akamai_version_push/tasks/snow_live_attempt.yml
```

There should be no import/reference to `snow_live_attempt.yml`.

Search the role for:

```text
snow_live_attempt
```

Expected result:

```text
0 references
```

---

# 15. Environment Variable Fix

Search the entire `akamai_version_push` role for:

```text
common_logging_allowed_environments
```

Expected result:

```text
0 references
```

SSPA-11 must use:

```text
snow_allowed_logging_environments
```

before `common_logging` is included.

This prevents the AAP error:

```text
'common_logging_allowed_environments' is undefined
```

---

# 16. Variable Ownership

| Variable | Source |
|---|---|
| `akamai_chg_number` | AAP Survey / workflow input |
| `akamai.environment.name` | AAP / workflow environment input |
| `snow_mode` | defaults / test input |
| `snow_mock_scenario` | defaults / test input |
| `snow_api_timeout` | `defaults/main.yml` |
| `snow_validate_certs` | `defaults/main.yml` |
| `snow_required_state` | `vars/main.yml` |
| `snow_required_approval` | `vars/main.yml` |
| `snow_live_retry_count` | `vars/main.yml` |
| `snow_live_retry_delay` | `vars/main.yml` |
| `snow_allowed_logging_environments` | `vars/main.yml` |
| `snow_host` | AAP ServiceNow Credential |
| `snow_username` | AAP ServiceNow Credential |
| `snow_password` | AAP ServiceNow Credential |

---

# 17. Important Live Validation Note

The mock path can validate the full business-rule matrix without ServiceNow connectivity.

The live path still requires controlled integration testing for:

```text
AAP credential injection
ServiceNow service account
Network connectivity from the EE
Certificate trust
ServiceNow API permissions
Actual returned field structure
Actual date/time semantics
```

Live testing should prove that the ServiceNow response is correctly converted to `snow_chg`.

The state, approval and implementation-window rules are then executed by the same `snow_validation.yml` already exercised with mocks.

---

# 18. Final Implementation Summary

```text
snow_fetch.yml
    ├── validates local configuration
    ├── mock retrieval
    ├── live credential validation
    ├── live API retrieval
    ├── retry/error handling
    └── produces snow_chg

snow_validation.yml
    ├── common mandatory field validation
    ├── field normalization
    ├── metadata normalization
    ├── CHG identity validation
    ├── Implement state validation
    ├── Approved approval validation
    ├── implementation-window validation
    └── AAP result publication
```

This is the recommended SSPA-11 structure for the PR because mock and live data use the same downstream validation logic.
