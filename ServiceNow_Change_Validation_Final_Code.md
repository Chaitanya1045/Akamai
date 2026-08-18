# ServiceNow Read-Only Change Validation — Final Ansible Code

This implementation keeps the same downstream validation logic for both mock and live ServiceNow modes.

```text
MOCK JSON
   ↓
snow_chg
   ↓
COMMON VALIDATION

LIVE ServiceNow
   ↓
servicenow.itsm.change_request_info
   ↓
records[0]
   ↓
snow_chg
   ↓
COMMON VALIDATION
```

The common validation enforces:

- Change record must exist.
- Returned change number must match the requested CHG.
- State must be `Implement`.
- Approval must be `Approved`.
- Both conditions are mandatory.
- Planned start and end must be present.
- Current UTC time must be inside the planned implementation window.
- ServiceNow validation remains read-only.
- Mock and live failures use the existing `common_logging` role.
- No Jira/story number is embedded in runtime code.
- ServiceNow host/username/password are not supplied through survey or Extra Vars. They are injected by an AAP Credential as `SN_HOST`, `SN_USERNAME`, and `SN_PASSWORD`.

---

## Repository Layout

```text
roles/
└── akamai/
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
        ├── snow_validation.yml
        └── snow_live_attempt.yml

playbooks/
└── akamai/
    └── snow_validation.yml

collections/
└── requirements.yml
```

---

# 1. `roles/akamai/defaults/main.yml`

Merge this section into the existing defaults file.

```yaml
---
# =============================================================================
# ServiceNow change validation
# =============================================================================

snow_mode: "mock"
snow_mock_scenario: "valid_implement_approved"
snow_validate_certs: true
snow_api_timeout: 30
snow_datetime_format: "%Y-%m-%d %H:%M:%S"
```

Do not define real ServiceNow credentials here.

---

# 2. `roles/akamai/vars/main.yml`

```yaml
---
# =============================================================================
# ServiceNow validation internal configuration
# =============================================================================

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

snow_live_attempt_plan:
  - attempt: 1
    delay: 0
  - attempt: 2
    delay: 2
  - attempt: 3
    delay: 4

snow_retryable_mock_statuses:
  - 408
  - 429
  - 500
  - 502
  - 503
  - 504

snow_log_stages:
  validation: "servicenow_validation"
  authentication: "servicenow_authentication"
  retrieval: "servicenow_retrieval"
  status: "servicenow_status_validation"
  window: "servicenow_window_validation"

snow_error_keys:
  prerequisite_failed: "SERVICENOW_PREREQUISITE_FAILED"
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

# 3. `roles/akamai/tasks/snow_live_attempt.yml`

```yaml
---
- name: "ServiceNow | Wait before retry attempt"
  ansible.builtin.pause:
    seconds: "{{ snow_live_attempt.delay | int }}"
  when:
    - not (_snow_live_success | bool)
    - snow_live_attempt.delay | int > 0

- name: "ServiceNow | Execute retrieval attempt"
  when:
    - not (_snow_live_success | bool)
  block:

    - name: "ServiceNow | Retrieve change request"
      servicenow.itsm.change_request_info:
        instance:
          host: "{{ lookup('ansible.builtin.env', 'SN_HOST') }}"
          username: "{{ lookup('ansible.builtin.env', 'SN_USERNAME') }}"
          password: "{{ lookup('ansible.builtin.env', 'SN_PASSWORD') }}"
          timeout: "{{ snow_api_timeout | int }}"
          validate_certs: "{{ snow_validate_certs | bool }}"
        number: "{{ akamai_chg_number }}"
        sysparm_display_value: "all"
      register: _snow_attempt_response
      changed_when: false
      no_log: true

    - name: "ServiceNow | Capture successful retrieval attempt"
      ansible.builtin.set_fact:
        _snow_live_success: true
        _snow_response: "{{ _snow_attempt_response }}"
        _snow_live_last_error: ""
        _snow_live_attempt_used: "{{ snow_live_attempt.attempt | int }}"

  rescue:

    - name: "ServiceNow | Capture failed retrieval attempt"
      ansible.builtin.set_fact:
        _snow_live_success: false
        _snow_live_last_error: >-
          {{
            ansible_failed_result.msg
            | default('ServiceNow request failed.', true)
            | string
          }}
        _snow_live_attempt_used: "{{ snow_live_attempt.attempt | int }}"

    - name: "ServiceNow | Display retry information"
      ansible.builtin.debug:
        msg: >-
          ServiceNow retrieval attempt {{ snow_live_attempt.attempt }} of
          {{ snow_live_attempt_plan | length }} failed. {{
            'A retry will be attempted.'
            if (snow_live_attempt.attempt | int) < (snow_live_attempt_plan | length)
            else 'No retries remain.'
          }}
```

---

# 4. `roles/akamai/tasks/snow_validation.yml`

```yaml
---
# =============================================================================
# ServiceNow read-only change validation
# =============================================================================

- name: "ServiceNow | Build logging context"
  ansible.builtin.set_fact:
    _snow_log_request_id: >-
      {{
        (common_logging_request_id | default('', true) | string | trim)
        if (common_logging_request_id | default('', true) | string | trim | length > 0)
        else (akamai_chg_number | default('SERVICENOW-VALIDATION', true) | string | trim)
      }}
    _snow_log_execution_id: >-
      {{
        common_logging_execution_id
        | default(awx_job_id | default(now(utc=true, fmt='%Y%m%dT%H%M%SZ'), true), true)
        | string
      }}
    _snow_log_actor: >-
      {{ common_logging_actor | default(awx_user_email | default('automation', true), true) | string }}
    _snow_log_triggered_by: >-
      {{ common_logging_triggered_by | default(awx_user_email | default('automation', true), true) | string }}

- name: "ServiceNow | Resolve logging environment"
  ansible.builtin.set_fact:
    _snow_requested_environment: >-
      {{
        common_logging_environment
        | default((akamai | default({})).get('environment', {}).get('name', 'local'), true)
        | string
        | lower
      }}

- name: "ServiceNow | Resolve safe logging environment"
  ansible.builtin.set_fact:
    _snow_log_environment: >-
      {{
        _snow_requested_environment
        if _snow_requested_environment in ['dev', 'rnd', 'qa', 'prod', 'test', 'local']
        else 'local'
      }}

- name: "ServiceNow | Require common input validation"
  block:
    - name: "ServiceNow | Check workflow input validation"
      ansible.builtin.assert:
        that:
          - akamai_input_validation_passed | default(false) | bool
        fail_msg: >-
          ServiceNow validation cannot run because common workflow input validation has not passed.
        quiet: true
  rescue:
    - name: "ServiceNow | Stop when prerequisite validation failed"
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
        common_logging_error_key_input: "{{ snow_error_keys.prerequisite_failed }}"
        common_logging_error_message: >-
          Common workflow input validation has not passed.
        common_logging_error_recommended_action: >-
          Correct the workflow input validation errors before starting ServiceNow validation.
        common_logging_error_version_info: {}

- name: "ServiceNow | Validate execution configuration"
  block:
    - name: "ServiceNow | Validate mode and business rules"
      ansible.builtin.assert:
        that:
          - snow_mode is defined
          - snow_mode in snow_allowed_modes
          - akamai_chg_number is defined
          - (akamai_chg_number | string | trim) is match('^CHG[0-9]+$')
          - (snow_required_state | string | trim | lower) == 'implement'
          - (snow_required_approval | string | trim | lower) == 'approved'
          - snow_api_timeout | int > 0
        fail_msg: "Invalid ServiceNow validation configuration."
        quiet: true

    - name: "ServiceNow | Validate mock scenario"
      ansible.builtin.assert:
        that:
          - snow_mock_scenario is defined
          - snow_mock_scenario is string
          - snow_mock_scenario | trim | length > 0
          - snow_mock_scenario is match('^[A-Za-z0-9_-]+$')
          - snow_mock_scenario in snow_allowed_mock_scenarios
        fail_msg: "Invalid ServiceNow mock scenario."
        quiet: true
      when:
        - snow_mode == 'mock'
  rescue:
    - name: "ServiceNow | Stop on invalid configuration"
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
        common_logging_error_message: "ServiceNow validation configuration is invalid."
        common_logging_error_recommended_action: >-
          Verify the change number, ServiceNow mode, mock scenario, timeout, and role configuration.
        common_logging_error_version_info: {}

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
      Started read-only ServiceNow change validation using {{ snow_mode }} data source.
    common_logging_version_info: {}

# -----------------------------------------------------------------------------
# Mock retrieval
# -----------------------------------------------------------------------------

- name: "ServiceNow | Retrieve change from mock fixture"
  when:
    - snow_mode == 'mock'
  block:
    - name: "ServiceNow | Load mock response"
      ansible.builtin.set_fact:
        _snow_raw: >-
          {{
            lookup(
              'ansible.builtin.file',
              snow_mock_dir ~ '/' ~ snow_mock_scenario ~ '.json'
            )
            | from_json
          }}
      no_log: true

    - name: "ServiceNow | Validate mock response envelope"
      ansible.builtin.assert:
        that:
          - _snow_raw is mapping
          - _snow_raw.http_status is defined
        fail_msg: "Mock ServiceNow response does not contain a valid response envelope."
        quiet: true
  rescue:
    - name: "ServiceNow | Stop when mock fixture cannot be loaded"
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
          ServiceNow mock fixture could not be loaded or contains an invalid response envelope.
        common_logging_error_recommended_action: >-
          Verify the selected mock fixture and JSON structure.
        common_logging_error_version_info: {}

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
    common_logging_error_message: "ServiceNow authentication failed."
    common_logging_error_recommended_action: >-
      Verify the ServiceNow AAP credential and ServiceNow service-account access.
    common_logging_error_version_info: {}
  when:
    - snow_mode == 'mock'
    - _snow_raw.http_status | int in [401, 403]

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
    common_logging_error_message: "Change {{ akamai_chg_number }} was not found."
    common_logging_error_recommended_action: >-
      Verify the ServiceNow change number before restarting the workflow.
    common_logging_error_version_info: {}
  when:
    - snow_mode == 'mock'
    - _snow_raw.http_status | int == 404

- name: "ServiceNow | Simulate retry handling for mock API failure"
  when:
    - snow_mode == 'mock'
    - _snow_raw.http_status | int in snow_retryable_mock_statuses
  block:
    - name: "ServiceNow | Mock attempt 1 failed"
      ansible.builtin.debug:
        msg: >-
          ServiceNow mock attempt 1 of 3 failed with HTTP {{ _snow_raw.http_status }}.
          Retrying after 2 seconds.

    - name: "ServiceNow | First mock backoff"
      ansible.builtin.pause:
        seconds: 2

    - name: "ServiceNow | Mock attempt 2 failed"
      ansible.builtin.debug:
        msg: >-
          ServiceNow mock attempt 2 of 3 failed with HTTP {{ _snow_raw.http_status }}.
          Retrying after 4 seconds.

    - name: "ServiceNow | Second mock backoff"
      ansible.builtin.pause:
        seconds: 4

    - name: "ServiceNow | Record mock retry exhaustion"
      ansible.builtin.set_fact:
        _snow_mock_terminal_status: "{{ _snow_raw.http_status | int }}"

    - name: "ServiceNow | Stop after mock API failure"
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
            if _snow_mock_terminal_status in [408, 504]
            else ('rate-limit' if _snow_mock_terminal_status == 429 else 'api')
          }}
        common_logging_error_key_input: >-
          {{
            snow_error_keys.timeout
            if _snow_mock_terminal_status in [408, 504]
            else (
              snow_error_keys.rate_limited
              if _snow_mock_terminal_status == 429
              else snow_error_keys.retrieval_failed
            )
          }}
        common_logging_error_message: "ServiceNow request failed after three simulated attempts."
        common_logging_error_recommended_action: >-
          Verify ServiceNow availability, network access, API health, and rate limits.
        common_logging_error_version_info: {}

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
      ServiceNow mock retrieval returned an unexpected HTTP status.
    common_logging_error_recommended_action: >-
      Verify the mock response or ServiceNow API behavior.
    common_logging_error_version_info: {}
  when:
    - snow_mode == 'mock'
    - _snow_raw.http_status | int != 200
    - _snow_raw.http_status | int not in [401, 403, 404]
    - _snow_raw.http_status | int not in snow_retryable_mock_statuses

- name: "ServiceNow | Validate successful mock result"
  when:
    - snow_mode == 'mock'
    - _snow_raw.http_status | int == 200
  block:
    - name: "ServiceNow | Validate mock response structure"
      ansible.builtin.assert:
        that:
          - _snow_raw.result is defined
          - _snow_raw.result is mapping
          - _snow_raw.result | length > 0
        fail_msg: "Mock ServiceNow response does not contain a valid result record."
        quiet: true

    - name: "ServiceNow | Normalize mock response"
      ansible.builtin.set_fact:
        snow_chg: "{{ _snow_raw.result }}"
        snow_fetch_source: "mock:{{ snow_mock_scenario }}"

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
        common_logging_message: "ServiceNow mock change retrieval completed successfully."
        common_logging_version_info: {}
  rescue:
    - name: "ServiceNow | Stop on invalid mock response"
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
        common_logging_error_message: "ServiceNow mock response does not contain a valid change record."
        common_logging_error_recommended_action: "Verify the mock JSON response structure."
        common_logging_error_version_info: {}

# -----------------------------------------------------------------------------
# Live credential validation
# -----------------------------------------------------------------------------

- name: "ServiceNow | Validate AAP credential injection"
  when:
    - snow_mode == 'live'
  block:
    - name: "ServiceNow | Check injected credential values"
      ansible.builtin.assert:
        that:
          - lookup('ansible.builtin.env', 'SN_HOST') | trim | length > 0
          - lookup('ansible.builtin.env', 'SN_USERNAME') | length > 0
          - lookup('ansible.builtin.env', 'SN_PASSWORD') | length > 0
        fail_msg: >-
          Required ServiceNow credential values were not injected into the Execution Environment.
        quiet: true
      no_log: true
  rescue:
    - name: "ServiceNow | Stop when AAP credential is unavailable"
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
        common_logging_error_message: "ServiceNow credentials were not injected by AAP."
        common_logging_error_recommended_action: >-
          Attach the approved ServiceNow credential to the AAP Job Template.
        common_logging_error_version_info: {}

# -----------------------------------------------------------------------------
# Live retrieval
# -----------------------------------------------------------------------------

- name: "ServiceNow | Retrieve change from live API"
  when:
    - snow_mode == 'live'
  block:
    - name: "ServiceNow | Initialize live retrieval state"
      ansible.builtin.set_fact:
        _snow_live_success: false
        _snow_live_last_error: ""
        _snow_response: {}
        _snow_live_attempt_used: 0

    - name: "ServiceNow | Execute live retrieval attempts"
      ansible.builtin.include_tasks:
        file: snow_live_attempt.yml
      loop: "{{ snow_live_attempt_plan }}"
      loop_control:
        loop_var: snow_live_attempt
        label: "attempt={{ snow_live_attempt.attempt }}"

    - name: "ServiceNow | Resolve terminal failure text"
      ansible.builtin.set_fact:
        _snow_live_error_text: "{{ _snow_live_last_error | default('') | string | lower }}"
      when:
        - not (_snow_live_success | bool)

    - name: "ServiceNow | Resolve terminal error category"
      ansible.builtin.set_fact:
        _snow_live_error_category: >-
          {{
            'auth'
            if (_snow_live_error_text is search('401|403|unauthorized|forbidden|authentication|credential'))
            else (
              'timeout'
              if (_snow_live_error_text is search('timeout|timed out|readtimeout|connecttimeout'))
              else (
                'rate-limit'
                if (_snow_live_error_text is search('429|rate limit|too many requests'))
                else 'api'
              )
            )
          }}
        _snow_live_error_key: >-
          {{
            snow_error_keys.authentication_failed
            if (_snow_live_error_text is search('401|403|unauthorized|forbidden|authentication|credential'))
            else (
              snow_error_keys.timeout
              if (_snow_live_error_text is search('timeout|timed out|readtimeout|connecttimeout'))
              else (
                snow_error_keys.rate_limited
                if (_snow_live_error_text is search('429|rate limit|too many requests'))
                else snow_error_keys.retrieval_failed
              )
            )
          }}
      when:
        - not (_snow_live_success | bool)

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
          {{ snow_log_stages.authentication if _snow_live_error_category == 'auth' else snow_log_stages.retrieval }}
        common_logging_error_actor: "{{ _snow_log_actor }}"
        common_logging_error_category_input: "{{ _snow_live_error_category }}"
        common_logging_error_key_input: "{{ _snow_live_error_key }}"
        common_logging_error_message: >-
          ServiceNow change retrieval failed after {{ _snow_live_attempt_used }} attempts.
        common_logging_error_recommended_action: >-
          Verify ServiceNow availability, AAP credential configuration, certificate trust,
          network connectivity, API permissions, and rate limits.
        common_logging_error_version_info: {}
      when:
        - not (_snow_live_success | bool)

    - name: "ServiceNow | Validate live response collection"
      ansible.builtin.assert:
        that:
          - _snow_response.records is defined
          - _snow_response.records is sequence
          - _snow_response.records is not string
        fail_msg: "ServiceNow did not return a valid change record collection."
        quiet: true

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
        common_logging_error_key_input: "{{ snow_error_keys.change_not_found }}"
        common_logging_error_message: "Change {{ akamai_chg_number }} was not found in ServiceNow."
        common_logging_error_recommended_action: >-
          Verify the supplied change number and ServiceNow read permissions.
        common_logging_error_version_info: {}
      when:
        - _snow_response.records | length == 0

    - name: "ServiceNow | Validate unique change result"
      block:
        - name: "ServiceNow | Require exactly one change record"
          ansible.builtin.assert:
            that:
              - _snow_response.records | length == 1
            fail_msg: >-
              ServiceNow returned more than one record for {{ akamai_chg_number }}.
            quiet: true
      rescue:
        - name: "ServiceNow | Stop on ambiguous live response"
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
              ServiceNow returned multiple records for the supplied change number.
            common_logging_error_recommended_action: "Verify the change number and ServiceNow data."
            common_logging_error_version_info: {}

    - name: "ServiceNow | Normalize live response"
      ansible.builtin.set_fact:
        snow_chg: "{{ _snow_response.records[0] }}"
        snow_fetch_source: "live"

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
          ServiceNow change retrieved successfully using the read-only information module.
        common_logging_version_info: {}

# -----------------------------------------------------------------------------
# Common validation for mock and live
# -----------------------------------------------------------------------------

- name: "ServiceNow | Validate change response fields"
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
        fail_msg: "ServiceNow change record is missing mandatory fields."
        quiet: true
  rescue:
    - name: "ServiceNow | Stop on missing response fields"
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
          ServiceNow response is missing one or more required change fields.
        common_logging_error_recommended_action: >-
          Verify ServiceNow field availability and service-account read permissions.
        common_logging_error_version_info: {}

- name: "ServiceNow | Normalize change identifiers"
  ansible.builtin.set_fact:
    _snow_number: >-
      {{
        (
          snow_chg.number.display_value
          if (snow_chg.number is mapping and snow_chg.number.display_value is defined)
          else (
            snow_chg.number.value
            if (snow_chg.number is mapping and snow_chg.number.value is defined)
            else snow_chg.number
          )
        )
        | default('') | string | trim
      }}
    _snow_sys_id: >-
      {{
        (
          snow_chg.sys_id.value
          if (snow_chg.sys_id is mapping and snow_chg.sys_id.value is defined)
          else (
            snow_chg.sys_id.display_value
            if (snow_chg.sys_id is mapping and snow_chg.sys_id.display_value is defined)
            else snow_chg.sys_id
          )
        )
        | default('') | string | trim
      }}

- name: "ServiceNow | Normalize state and approval"
  ansible.builtin.set_fact:
    _snow_state: >-
      {{
        (
          snow_chg.state.display_value
          if (snow_chg.state is mapping and snow_chg.state.display_value is defined)
          else (
            snow_chg.state.value
            if (snow_chg.state is mapping and snow_chg.state.value is defined)
            else snow_chg.state
          )
        )
        | default('') | string | trim | lower
      }}
    _snow_approval: >-
      {{
        (
          snow_chg.approval.display_value
          if (snow_chg.approval is mapping and snow_chg.approval.display_value is defined)
          else (
            snow_chg.approval.value
            if (snow_chg.approval is mapping and snow_chg.approval.value is defined)
            else snow_chg.approval
          )
        )
        | default('') | string | trim | lower
      }}

- name: "ServiceNow | Normalize planned window"
  ansible.builtin.set_fact:
    _snow_start_raw: >-
      {{
        (
          snow_chg.start_date.value
          if (snow_chg.start_date is mapping and snow_chg.start_date.value is defined)
          else snow_chg.start_date
        )
        | default('') | string | trim
      }}
    _snow_end_raw: >-
      {{
        (
          snow_chg.end_date.value
          if (snow_chg.end_date is mapping and snow_chg.end_date.value is defined)
          else snow_chg.end_date
        )
        | default('') | string | trim
      }}
    _snow_now_raw: "{{ now(utc=true, fmt=snow_datetime_format) }}"

- name: "ServiceNow | Validate returned change identity"
  block:
    - name: "ServiceNow | Verify change number and sys_id"
      ansible.builtin.assert:
        that:
          - _snow_number == akamai_chg_number
          - _snow_sys_id | length > 0
        fail_msg: "ServiceNow returned an invalid or unexpected change record."
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
          ServiceNow response does not match the requested change number or does not contain a sys_id.
        common_logging_error_recommended_action: >-
          Verify the ServiceNow response and requested change.
        common_logging_error_version_info: {}

- name: "ServiceNow | Validate Implement and Approved gate"
  block:
    - name: "ServiceNow | Require Implement state"
      ansible.builtin.assert:
        that:
          - _snow_state == (snow_required_state | lower)
        fail_msg: "Change is not in Implement state."
        quiet: true

    - name: "ServiceNow | Require Approved approval"
      ansible.builtin.assert:
        that:
          - _snow_approval == (snow_required_approval | lower)
        fail_msg: "Change is not Approved."
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
          ServiceNow change is in Implement state and has Approved approval.
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
          Change {{ akamai_chg_number }} must be both Implement and Approved.
          Current state={{ _snow_state }}, approval={{ _snow_approval }}.
        common_logging_error_recommended_action: >-
          Ensure the ServiceNow change is in Implement state and has Approved approval before retrying.
        common_logging_error_version_info: {}

- name: "ServiceNow | Validate planned implementation window"
  block:
    - name: "ServiceNow | Require populated implementation window"
      ansible.builtin.assert:
        that:
          - _snow_start_raw | length > 0
          - _snow_end_raw | length > 0
        fail_msg: "ServiceNow implementation window is missing."
        quiet: true

    - name: "ServiceNow | Parse implementation window"
      ansible.builtin.set_fact:
        _snow_start_datetime: "{{ _snow_start_raw | to_datetime(snow_datetime_format) }}"
        _snow_end_datetime: "{{ _snow_end_raw | to_datetime(snow_datetime_format) }}"
        _snow_now_datetime: "{{ _snow_now_raw | to_datetime(snow_datetime_format) }}"

    - name: "ServiceNow | Require valid window ordering"
      ansible.builtin.assert:
        that:
          - _snow_start_datetime < _snow_end_datetime
        fail_msg: "ServiceNow planned start must occur before planned end."
        quiet: true

    - name: "ServiceNow | Require execution inside planned window"
      ansible.builtin.assert:
        that:
          - _snow_now_datetime >= _snow_start_datetime
          - _snow_now_datetime <= _snow_end_datetime
        fail_msg: "Execution is outside the ServiceNow implementation window."
        quiet: true

    - name: "ServiceNow | Log successful window gate"
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
          Current execution time is inside the approved ServiceNow implementation window.
        common_logging_version_info: {}
  rescue:
    - name: "ServiceNow | Resolve window failure type"
      ansible.builtin.set_fact:
        _snow_window_failure_message: >-
          {{ ansible_failed_result.msg | default('ServiceNow planned implementation-window validation failed.', true) | string }}
        _snow_window_error_key: >-
          {{
            snow_error_keys.window_missing
            if (_snow_start_raw | length == 0 or _snow_end_raw | length == 0)
            else (
              snow_error_keys.outside_window
              if (
                _snow_start_datetime is defined
                and _snow_end_datetime is defined
                and _snow_now_datetime is defined
                and _snow_start_datetime < _snow_end_datetime
                and (
                  _snow_now_datetime < _snow_start_datetime
                  or _snow_now_datetime > _snow_end_datetime
                )
              )
              else snow_error_keys.window_invalid
            )
          }}

    - name: "ServiceNow | Stop on deployment-window failure"
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
        common_logging_error_message: "{{ _snow_window_failure_message }}"
        common_logging_error_recommended_action: >-
          Verify that the change contains a valid planned start and end time and run the automation
          only inside the approved window.
        common_logging_error_version_info: {}

- name: "ServiceNow | Publish validation result"
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

- name: "ServiceNow | Publish result to AAP"
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
      ServiceNow change validation completed successfully. The operation was read-only.
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

# 5. Mock Payloads

## `roles/akamai/mocks/snow/valid_implement_approved.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012345",
    "sys_id": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
    "state": "Implement",
    "approval": "Approved",
    "start_date": "2020-01-01 00:00:00",
    "end_date": "2099-12-31 23:59:59",
    "short_description": "Akamai delivery configuration promotion",
    "u_approver": "peer.reviewer",
    "assigned_to": "automation.operator",
    "requested_by": "application.team",
    "opened_by": "application.team",
    "sys_updated_on": "2026-08-04 09:15:22"
  }
}
```

## `roles/akamai/mocks/snow/invalid_not_approved.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012347",
    "sys_id": "c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6",
    "state": "Implement",
    "approval": "Not Yet Requested",
    "start_date": "2020-01-01 00:00:00",
    "end_date": "2099-12-31 23:59:59",
    "short_description": "Change has not been approved",
    "u_approver": "",
    "assigned_to": "automation.operator",
    "requested_by": "application.team"
  }
}
```

## `roles/akamai/mocks/snow/invalid_state.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012346",
    "sys_id": "b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5",
    "state": "Draft",
    "approval": "Approved",
    "start_date": "2020-01-01 00:00:00",
    "end_date": "2099-12-31 23:59:59",
    "short_description": "Change is not in implementation state",
    "u_approver": "peer.reviewer",
    "assigned_to": "automation.operator",
    "requested_by": "application.team"
  }
}
```

## `roles/akamai/mocks/snow/missing_window.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012350",
    "sys_id": "f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3",
    "state": "Implement",
    "approval": "Approved",
    "start_date": "",
    "end_date": "",
    "short_description": "No planned implementation window",
    "u_approver": "peer.reviewer",
    "assigned_to": "automation.operator",
    "requested_by": "application.team"
  }
}
```

## `roles/akamai/mocks/snow/not_found.json`

```json
{
  "http_status": 404,
  "error": "No change request record found",
  "result": {}
}
```

## `roles/akamai/mocks/snow/auth_failure.json`

```json
{
  "http_status": 401,
  "error": "User Not Authenticated",
  "result": {}
}
```

## `roles/akamai/mocks/snow/timeout.json`

```json
{
  "http_status": 504,
  "error": "Gateway Timeout",
  "result": {}
}
```

## `roles/akamai/mocks/snow/outside_window_early.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012348",
    "sys_id": "d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1",
    "state": "Implement",
    "approval": "Approved",
    "start_date": "2037-01-01 00:00:00",
    "end_date": "2037-01-01 04:00:00",
    "short_description": "Implementation window has not opened",
    "u_approver": "peer.reviewer",
    "assigned_to": "automation.operator",
    "requested_by": "application.team"
  }
}
```

## `roles/akamai/mocks/snow/outside_window_late.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012349",
    "sys_id": "e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
    "state": "Implement",
    "approval": "Approved",
    "start_date": "2021-01-01 00:00:00",
    "end_date": "2021-01-01 04:00:00",
    "short_description": "Implementation window has closed",
    "u_approver": "peer.reviewer",
    "assigned_to": "automation.operator",
    "requested_by": "application.team"
  }
}
```

---

# 6. `roles/akamai/tasks/main.yml`

Merge this into the existing task dispatcher.

```yaml
---
- name: "Validate workflow inputs"
  ansible.builtin.import_tasks:
    input_validation.yml

- name: "Validate ServiceNow change request"
  ansible.builtin.import_tasks:
    snow_validation.yml
```

---

# 7. Standalone Test Playbook

## `playbooks/akamai/snow_validation.yml`

```yaml
---
- name: "Validate ServiceNow change request"
  hosts: localhost
  connection: local
  gather_facts: false
  any_errors_fatal: true

  tasks:
    - name: "Set isolated-test prerequisite"
      ansible.builtin.set_fact:
        akamai_input_validation_passed: true

    - name: "Run ServiceNow validation"
      ansible.builtin.include_role:
        name: akamai
        tasks_from: snow_validation
```

In the integrated workflow, do not manually set `akamai_input_validation_passed: true`.

---

# 8. `collections/requirements.yml`

```yaml
---
collections:
  - name: community.general
  - name: servicenow.itsm
```

Use the collection version approved for the AAP Execution Environment if your organization pins collection versions.

---

# 9. AAP Credential Injection

The ServiceNow credential attached to the Job Template should inject:

```yaml
env:
  SN_HOST: "{{ host }}"
  SN_USERNAME: "{{ username }}"
  SN_PASSWORD: "{{ password }}"
```

The user does not pass these values through the survey or normal Extra Vars.

---

# 10. Mock Test Inputs

## Positive

```yaml
akamai_chg_number: "CHG0012345"
snow_mode: "mock"
snow_mock_scenario: "valid_implement_approved"
```

## Implement but not Approved

```yaml
akamai_chg_number: "CHG0012347"
snow_mode: "mock"
snow_mock_scenario: "invalid_not_approved"
```

## Approved but not Implement

```yaml
akamai_chg_number: "CHG0012346"
snow_mode: "mock"
snow_mock_scenario: "invalid_state"
```

## Missing Planned Window

```yaml
akamai_chg_number: "CHG0012350"
snow_mode: "mock"
snow_mock_scenario: "missing_window"
```

## Change Not Found

```yaml
akamai_chg_number: "CHG0099999"
snow_mode: "mock"
snow_mock_scenario: "not_found"
```

## Authentication Failure

```yaml
akamai_chg_number: "CHG0012345"
snow_mode: "mock"
snow_mock_scenario: "auth_failure"
```

## Timeout

```yaml
akamai_chg_number: "CHG0012345"
snow_mode: "mock"
snow_mock_scenario: "timeout"
```

## Before Planned Window

```yaml
akamai_chg_number: "CHG0012348"
snow_mode: "mock"
snow_mock_scenario: "outside_window_early"
```

## After Planned Window

```yaml
akamai_chg_number: "CHG0012349"
snow_mode: "mock"
snow_mock_scenario: "outside_window_late"
```

---

# 11. Live AAP Inputs

When ServiceNow access is available:

```yaml
akamai_chg_number: "CHG0123456"
snow_mode: "live"
```

AAP injects:

```text
SN_HOST
SN_USERNAME
SN_PASSWORD
```

The live path then normalizes `records[0]` into `snow_chg`, and all state, approval, change identity, and planned-window validation remains common with mock mode.

---

# 12. Successful Published Result

```yaml
snow_validation_result:
  result: "PASSED"
  mode: "mock"
  source: "mock:valid_implement_approved"
  change_number: "CHG0012345"
  change_sys_id: "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4"
  state: "implement"
  approval: "approved"
  planned_start_utc: "2020-01-01 00:00:00"
  planned_end_utc: "2099-12-31 23:59:59"
  checked_at_utc: "<runtime UTC timestamp>"
  read_only: true
```

In live mode, `mode` and `source` become `live`, while the downstream result contract remains the same.

---

# Final Runtime Design

```text
No ServiceNow credentials in Git
No ServiceNow credentials in Survey
No ServiceNow credentials in normal Extra Vars

AAP Credential
      ↓
SN_HOST / SN_USERNAME / SN_PASSWORD
      ↓
servicenow.itsm.change_request_info
      ↓
records[0]
      ↓
snow_chg
      ↓
Change identity valid
      AND
State = Implement
      AND
Approval = Approved
      AND
Current UTC within planned implementation window
      ↓
PASS / BLOCK
```

The live integration path is implementation-ready. Final live validation still requires the actual AAP ServiceNow credential, network connectivity, certificate trust, and confirmation of the real ServiceNow response in the target environment.
