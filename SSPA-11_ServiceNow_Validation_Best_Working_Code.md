# SSPA-11 — ServiceNow Validation (Step 3) — Best Working Code

This version keeps the existing project structure while correcting the key issues found in the reviewed draft.

## 1. `roles/AkamaiVersionConfigPush/akamai/defaults/main.yml`

Append:

```yaml
---
# ============================================================================
# SSPA-11 — ServiceNow read-only change validation
# ============================================================================

# Supported modes:
#   mock - local JSON fixtures
#   live - ServiceNow Table API
snow_mode: mock

snow_mock_scenario: valid_implement_approved

# Example:
# https://pncdev.service-now.com
snow_instance: ""

snow_validate_certs: true
snow_ca_path: ""

# ServiceNow values may differ by instance.
# Keep these overridable until confirmed against the PNC instance.
snow_required_state: implement
snow_required_approval: approved

snow_enforce_window: true

# ServiceNow normally stores date/time values in UTC in this format.
snow_datetime_format: "%Y-%m-%d %H:%M:%S"
```

## 2. `roles/AkamaiVersionConfigPush/akamai/vars/main.yml`

Append:

```yaml
---
# ============================================================================
# SSPA-11 constants
#
# These values belong in vars because survey users must not override them.
# ============================================================================

snow_change_table: change_request

snow_api_timeout: 30
snow_api_retries: 3
snow_api_retry_delay: 5

snow_mock_dir: "mocks/snow"

snow_change_fields:
  - number
  - sys_id
  - state
  - approval
  - start_date
  - end_date
  - short_description
  - assigned_to
  - requested_by
  - opened_by
  - sys_updated_on
  - u_approver
```

## 3. `roles/AkamaiVersionConfigPush/akamai/meta/argument_specs.yml`

Add these under the existing `main.options` block:

```yaml
      snow_mode:
        type: str
        required: false
        default: mock
        choices:
          - mock
          - live
        description:
          - ServiceNow transport mode.
          - Mock loads a local fixture.
          - Live performs a read-only ServiceNow GET request.

      snow_mock_scenario:
        type: str
        required: false
        default: valid_implement_approved
        description:
          - Fixture filename without the JSON extension.
          - Used only when snow_mode is mock.

      snow_instance:
        type: str
        required: false
        description:
          - ServiceNow base URL.
          - Required when snow_mode is live.

      snow_validate_certs:
        type: bool
        required: false
        default: true
        description:
          - Validate the ServiceNow TLS certificate.

      snow_ca_path:
        type: str
        required: false
        default: ""
        description:
          - Optional path to the internal CA certificate bundle.

      snow_enforce_window:
        type: bool
        required: false
        default: true
        description:
          - Require execution to occur inside the planned change window.
```

## 4. `roles/AkamaiVersionConfigPush/akamai/tasks/03_snow_validation.yml`

```yaml
---
# ============================================================================
# SSPA-11 — ServiceNow API integration
#
# Purpose:
#   1. Fetch the supplied ServiceNow change.
#   2. Require state=Implement AND approval=Approved.
#   3. Require current UTC time inside the planned implementation window.
#   4. Carry the validated change information into later Akamai tasks.
#
# This implementation is read-only:
#   - mock mode reads a local JSON fixture
#   - live mode performs one HTTP GET
#   - no POST, PUT, PATCH or DELETE task exists in this file
# ============================================================================

- name: "03 | Validate ServiceNow change input"
  ansible.builtin.assert:
    that:
      - akamai_chg_number is defined
      - akamai_chg_number is string
      - akamai_chg_number | trim is match('^CHG[0-9]{7}$')
      - snow_mode in ['mock', 'live']
    fail_msg: >-
      Invalid ServiceNow input.
      akamai_chg_number must match CHG followed by exactly seven digits,
      and snow_mode must be mock or live.
    quiet: true

- name: "03 | Validate mock scenario name"
  ansible.builtin.assert:
    that:
      - snow_mock_scenario is defined
      - snow_mock_scenario is string
      - snow_mock_scenario | trim | length > 0
      - snow_mock_scenario is match('^[A-Za-z0-9_-]+$')
    fail_msg: >-
      snow_mock_scenario contains unsupported characters.
      Only letters, numbers, underscores and hyphens are allowed.
    quiet: true
  when: snow_mode == 'mock'

- name: "03 | Fetch ServiceNow change from mock fixture"
  when: snow_mode == 'mock'
  block:
    - name: "03 | Load mock ServiceNow response"
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

    - name: "03 | Validate mock HTTP response"
      ansible.builtin.assert:
        that:
          - _snow_raw is mapping
          - _snow_raw.http_status is defined
          - _snow_raw.http_status | int == 200
        fail_msg: >-
          [MOCK] ServiceNow request failed.
          HTTP status={{ _snow_raw.http_status | default('unknown') }},
          error={{ _snow_raw.error | default('no error detail') }}.
        quiet: true

    - name: "03 | Validate mock response structure"
      ansible.builtin.assert:
        that:
          - _snow_raw.result is defined
          - _snow_raw.result is mapping
          - _snow_raw.result | length > 0
        fail_msg: >-
          Mock fixture '{{ snow_mock_scenario }}' does not contain a valid
          ServiceNow result object.
        quiet: true

    - name: "03 | Normalize mock response"
      ansible.builtin.set_fact:
        snow_chg: "{{ _snow_raw.result }}"
        snow_fetch_source: "mock:{{ snow_mock_scenario }}"
      no_log: true

  rescue:
    - name: "03 | Fail mock ServiceNow retrieval"
      ansible.builtin.fail:
        msg: >-
          ServiceNow mock retrieval failed for scenario
          '{{ snow_mock_scenario }}'.

          Confirm that this file exists and contains valid JSON:
          roles/AkamaiVersionConfigPush/akamai/files/mocks/snow/{{
          snow_mock_scenario }}.json

- name: "03 | Fetch ServiceNow change from live API"
  when: snow_mode == 'live'
  block:
    - name: "03 | Validate live ServiceNow configuration"
      ansible.builtin.assert:
        that:
          - snow_instance is defined
          - snow_instance is string
          - snow_instance | trim is match('^https://[^/]+(?:/.*)?$')
          - snow_username is defined
          - snow_username is string
          - snow_username | length > 0
          - snow_password is defined
          - snow_password is string
          - snow_password | length > 0
        fail_msg: >-
          Live mode requires an HTTPS snow_instance and non-empty
          snow_username/snow_password values supplied through the attached
          AAP Vault credential.
        quiet: true
      no_log: true

    - name: "03 | Build ServiceNow query parameters"
      ansible.builtin.set_fact:
        _snow_query_params:
          sysparm_query: "number={{ akamai_chg_number | trim }}"
          sysparm_fields: "{{ snow_change_fields | join(',') }}"
          sysparm_limit: "2"
          sysparm_display_value: "true"
          sysparm_exclude_reference_link: "true"

    - name: "03 | GET ServiceNow change record"
      ansible.builtin.uri:
        url: >-
          {{
            snow_instance | regex_replace('/+$', '')
          }}/api/now/table/{{ snow_change_table }}?{{
            _snow_query_params | urlencode
          }}
        method: GET
        user: "{{ snow_username }}"
        password: "{{ snow_password }}"
        force_basic_auth: true
        headers:
          Accept: application/json
        return_content: true
        status_code:
          - 200
        timeout: "{{ snow_api_timeout | int }}"
        validate_certs: "{{ snow_validate_certs | bool }}"
        ca_path: >-
          {{
            snow_ca_path
            if snow_ca_path | default('') | trim | length > 0
            else omit
          }}
      register: _snow_response
      retries: "{{ snow_api_retries | int }}"
      delay: "{{ snow_api_retry_delay | int }}"
      until:
        - _snow_response is succeeded
        - _snow_response.status | default(0) | int == 200
      no_log: true

    - name: "03 | Validate ServiceNow response envelope"
      ansible.builtin.assert:
        that:
          - _snow_response.json is defined
          - _snow_response.json is mapping
          - _snow_response.json.result is defined
          - _snow_response.json.result is sequence
        fail_msg: >-
          ServiceNow returned an invalid response for
          {{ akamai_chg_number }}.
        quiet: true
      no_log: true

    - name: "03 | Require exactly one matching change"
      ansible.builtin.assert:
        that:
          - _snow_response.json.result | length == 1
        fail_msg: >-
          ServiceNow change lookup for {{ akamai_chg_number }} returned
          {{ _snow_response.json.result | length }} records.

          Zero records means the change does not exist or the API account
          cannot read it. More than one record is an unexpected data issue.
        quiet: true
      no_log: true

    - name: "03 | Normalize live response"
      ansible.builtin.set_fact:
        snow_chg: "{{ _snow_response.json.result[0] }}"
        snow_fetch_source: "live"
      no_log: true

  rescue:
    - name: "03 | Fail live ServiceNow retrieval"
      ansible.builtin.fail:
        msg: >-
          Live ServiceNow validation failed for {{ akamai_chg_number }} after
          {{ snow_api_retries }} attempts.

          Check the ServiceNow URL, API account, Vault credential,
          certificate trust, network path and table permissions.

- name: "03 | Validate required ServiceNow fields"
  ansible.builtin.assert:
    that:
      - snow_chg is defined
      - snow_chg is mapping
      - snow_chg.number is defined
      - snow_chg.number | string | trim == akamai_chg_number | trim
      - snow_chg.sys_id is defined
      - snow_chg.sys_id | string | trim | length > 0
      - snow_chg.state is defined
      - snow_chg.approval is defined
      - snow_chg.start_date is defined
      - snow_chg.end_date is defined
    fail_msg: >-
      ServiceNow record for {{ akamai_chg_number }} is missing one or more
      required fields: number, sys_id, state, approval, start_date or end_date.
    quiet: true

- name: "03 | Normalize state and approval"
  ansible.builtin.set_fact:
    _snow_state: >-
      {{
        (
          snow_chg.state.display_value
          if snow_chg.state is mapping
             and snow_chg.state.display_value is defined
          else
          (
            snow_chg.state.value
            if snow_chg.state is mapping
               and snow_chg.state.value is defined
            else snow_chg.state
          )
        )
        | string
        | trim
        | lower
      }}

    _snow_approval: >-
      {{
        (
          snow_chg.approval.display_value
          if snow_chg.approval is mapping
             and snow_chg.approval.display_value is defined
          else
          (
            snow_chg.approval.value
            if snow_chg.approval is mapping
               and snow_chg.approval.value is defined
            else snow_chg.approval
          )
        )
        | string
        | trim
        | lower
      }}

    _snow_start_raw: >-
      {{
        (
          snow_chg.start_date.display_value
          if snow_chg.start_date is mapping
             and snow_chg.start_date.display_value is defined
          else
          (
            snow_chg.start_date.value
            if snow_chg.start_date is mapping
               and snow_chg.start_date.value is defined
            else snow_chg.start_date
          )
        )
        | string
        | trim
      }}

    _snow_end_raw: >-
      {{
        (
          snow_chg.end_date.display_value
          if snow_chg.end_date is mapping
             and snow_chg.end_date.display_value is defined
          else
          (
            snow_chg.end_date.value
            if snow_chg.end_date is mapping
               and snow_chg.end_date.value is defined
            else snow_chg.end_date
          )
        )
        | string
        | trim
      }}

- name: "03 | Require Implement state and Approved approval"
  ansible.builtin.assert:
    that:
      - _snow_state == snow_required_state | lower
      - _snow_approval == snow_required_approval | lower
    fail_msg: >-
      SERVICENOW GATE FAILED for {{ akamai_chg_number }}.

      Current state='{{ _snow_state }}';
      required state='{{ snow_required_state | lower }}'.

      Current approval='{{ _snow_approval }}';
      required approval='{{ snow_required_approval | lower }}'.

      Deployment is blocked.
    success_msg: >-
      ServiceNow authorization gate passed for {{ akamai_chg_number }}.
      State={{ _snow_state }}, approval={{ _snow_approval }}.
    quiet: true

- name: "03 | Validate planned implementation window exists"
  ansible.builtin.assert:
    that:
      - _snow_start_raw | length > 0
      - _snow_end_raw | length > 0
    fail_msg: >-
      SERVICENOW GATE FAILED for {{ akamai_chg_number }} because its planned
      implementation start or end time is empty.
    quiet: true
  when: snow_enforce_window | bool

- name: "03 | Convert planned window to UTC epoch"
  when: snow_enforce_window | bool
  block:
    - name: "03 | Parse ServiceNow UTC timestamps"
      ansible.builtin.set_fact:
        _snow_start_epoch: >-
          {{
            (
              _snow_start_raw
              | to_datetime(snow_datetime_format)
            ).strftime('%s')
            | int
          }}

        _snow_end_epoch: >-
          {{
            (
              _snow_end_raw
              | to_datetime(snow_datetime_format)
            ).strftime('%s')
            | int
          }}

        _snow_now_epoch: "{{ now(utc=true, fmt='%s') | int }}"

        _snow_now_display: >-
          {{ now(utc=true, fmt='%Y-%m-%d %H:%M:%S') }}

    - name: "03 | Validate window ordering"
      ansible.builtin.assert:
        that:
          - _snow_start_epoch | int < _snow_end_epoch | int
        fail_msg: >-
          SERVICENOW GATE FAILED for {{ akamai_chg_number }}.
          Planned start '{{ _snow_start_raw }}' must be earlier than planned
          end '{{ _snow_end_raw }}'.
        quiet: true

    - name: "03 | Require current UTC time inside planned window"
      ansible.builtin.assert:
        that:
          - _snow_now_epoch | int >= _snow_start_epoch | int
          - _snow_now_epoch | int <= _snow_end_epoch | int
        fail_msg: >-
          SERVICENOW GATE FAILED for {{ akamai_chg_number }} because execution
          is outside the planned implementation window.

          Current UTC time: {{ _snow_now_display }}
          Planned start: {{ _snow_start_raw }}
          Planned end: {{ _snow_end_raw }}
        success_msg: >-
          Current UTC time is inside the ServiceNow implementation window.
        quiet: true

  rescue:
    - name: "03 | Fail invalid ServiceNow date format"
      ansible.builtin.fail:
        msg: >-
          ServiceNow returned an unsupported implementation-window format for
          {{ akamai_chg_number }}.

          Expected format: {{ snow_datetime_format }}
          Received start: {{ _snow_start_raw }}
          Received end: {{ _snow_end_raw }}

          Confirm whether the ServiceNow API returns UTC database values or
          display values before enabling live mode.

- name: "03 | Normalize ServiceNow approver"
  ansible.builtin.set_fact:
    akamai_peer_reviewed_by: >-
      {{
        (
          snow_chg.u_approver.display_value
          if snow_chg.u_approver is defined
             and snow_chg.u_approver is mapping
             and snow_chg.u_approver.display_value is defined
          else
          (
            snow_chg.u_approver.value
            if snow_chg.u_approver is defined
               and snow_chg.u_approver is mapping
               and snow_chg.u_approver.value is defined
            else
            snow_chg.u_approver | default('')
          )
        )
        | string
        | trim
      }}

- name: "03 | Require ServiceNow approver evidence"
  ansible.builtin.assert:
    that:
      - akamai_peer_reviewed_by | length > 0
    fail_msg: >-
      ServiceNow change {{ akamai_chg_number }} passed state and approval
      validation, but no approver identity was returned.

      Confirm the correct approver field with the ServiceNow team and replace
      'u_approver' in snow_change_fields and the approver-normalization task.
    quiet: true

- name: "03 | Publish ServiceNow validation result"
  ansible.builtin.set_fact:
    akamai_chg_validated: true

    snow_validation_result:
      story: "SSPA-11"
      step: 3
      result: "PASSED"

      mode: "{{ snow_mode }}"
      source: "{{ snow_fetch_source }}"

      change_number: "{{ snow_chg.number }}"
      change_sys_id: "{{ snow_chg.sys_id }}"

      state: "{{ _snow_state }}"
      approval: "{{ _snow_approval }}"
      authorization_rule: "state == Implement AND approval == Approved"

      planned_start_utc: "{{ _snow_start_raw }}"
      planned_end_utc: "{{ _snow_end_raw }}"
      evaluated_at_utc: >-
        {{ now(utc=true, fmt='%Y-%m-%d %H:%M:%S') }}

      peer_reviewed_by: "{{ akamai_peer_reviewed_by }}"

      request_method: "GET"
      read_only: true

- name: "03 | Log ServiceNow validation audit"
  ansible.builtin.debug:
    msg:
      - "Story: {{ snow_validation_result.story }}"
      - "Step: {{ snow_validation_result.step }}"
      - "Result: {{ snow_validation_result.result }}"
      - "Mode: {{ snow_validation_result.mode }}"
      - "Change: {{ snow_validation_result.change_number }}"
      - "State: {{ snow_validation_result.state }}"
      - "Approval: {{ snow_validation_result.approval }}"
      - "Window: {{ snow_validation_result.planned_start_utc }} -> {{ snow_validation_result.planned_end_utc }} UTC"
      - "Peer reviewed by: {{ snow_validation_result.peer_reviewed_by }}"
      - "ServiceNow request method: GET"
      - "No ServiceNow record was modified by this task file."
```

## 5. `roles/AkamaiVersionConfigPush/akamai/tasks/prechecks.yml`

Insert this in the correct step sequence:

```yaml
- name: "Step 3 | Validate ServiceNow change"
  ansible.builtin.include_tasks:
    file: 03_snow_validation.yml
    apply:
      tags:
        - snow
        - prechecks
  tags:
    - always
    - snow
    - prechecks
```

## 6. Mock fixtures

Directory:

```text
roles/AkamaiVersionConfigPush/akamai/files/mocks/snow/
```

### `valid_implement_approved.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012345",
    "sys_id": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
    "state": "Implement",
    "approval": "Approved",
    "start_date": "2026-08-05 00:00:00",
    "end_date": "2026-08-06 23:59:59",
    "short_description": "Akamai delivery configuration promotion",
    "u_approver": "peer.reviewer",
    "assigned_to": "automation.operator",
    "requested_by": "application.team",
    "opened_by": "application.team",
    "sys_updated_on": "2026-08-04 09:15:22"
  }
}
```

### `invalid_not_approved.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012347",
    "sys_id": "c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6",
    "state": "Implement",
    "approval": "Not Yet Requested",
    "start_date": "2026-08-05 00:00:00",
    "end_date": "2026-08-06 23:59:59",
    "short_description": "Akamai delivery configuration promotion",
    "u_approver": "",
    "assigned_to": "automation.operator",
    "requested_by": "application.team",
    "opened_by": "application.team",
    "sys_updated_on": "2026-08-04 09:15:22"
  }
}
```

### `invalid_state.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012346",
    "sys_id": "b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5",
    "state": "Draft",
    "approval": "Approved",
    "start_date": "2026-08-05 00:00:00",
    "end_date": "2026-08-06 23:59:59",
    "short_description": "Change is not in implementation state",
    "u_approver": "peer.reviewer",
    "assigned_to": "automation.operator",
    "requested_by": "application.team",
    "opened_by": "application.team",
    "sys_updated_on": "2026-08-04 09:15:22"
  }
}
```

### `outside_window_early.json`

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
    "requested_by": "application.team",
    "opened_by": "application.team",
    "sys_updated_on": "2026-08-04 09:15:22"
  }
}
```

### `outside_window_late.json`

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
    "requested_by": "application.team",
    "opened_by": "application.team",
    "sys_updated_on": "2026-08-04 09:15:22"
  }
}
```

### `missing_window.json`

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
    "requested_by": "application.team",
    "opened_by": "application.team",
    "sys_updated_on": "2026-08-04 09:15:22"
  }
}
```

### `not_found.json`

```json
{
  "http_status": 404,
  "error": "No change_request record found",
  "result": {}
}
```

### `auth_failure.json`

```json
{
  "http_status": 401,
  "error": "User Not Authenticated",
  "result": {}
}
```

### `timeout.json`

```json
{
  "http_status": 504,
  "error": "Gateway Timeout",
  "result": {}
}
```

## 7. Test commands

### Valid mock

```bash
ansible-playbook \
  playbooks/AkamaiVersionConfigPush/akamai/01_akamai_prechecks.yml \
  -i inventories/akamai/qa/aki_qa.yml \
  -e akamai_chg_number=CHG0012345 \
  -e snow_mode=mock \
  -e snow_mock_scenario=valid_implement_approved \
  --tags snow
```

### Implement but not approved

```bash
ansible-playbook \
  playbooks/AkamaiVersionConfigPush/akamai/01_akamai_prechecks.yml \
  -i inventories/akamai/qa/aki_qa.yml \
  -e akamai_chg_number=CHG0012347 \
  -e snow_mode=mock \
  -e snow_mock_scenario=invalid_not_approved \
  --tags snow
```

### Approved but wrong state

```bash
ansible-playbook \
  playbooks/AkamaiVersionConfigPush/akamai/01_akamai_prechecks.yml \
  -i inventories/akamai/qa/aki_qa.yml \
  -e akamai_chg_number=CHG0012346 \
  -e snow_mode=mock \
  -e snow_mock_scenario=invalid_state \
  --tags snow
```

### Live execution

```bash
ansible-playbook \
  playbooks/AkamaiVersionConfigPush/akamai/01_akamai_prechecks.yml \
  -i inventories/akamai/qa/aki_qa.yml \
  -e akamai_chg_number=CHG0012345 \
  -e snow_mode=live \
  -e snow_instance=https://pncdev.service-now.com \
  --vault-id svm-ansible-vault@prompt \
  --tags snow
```

## 8. Validation before merge

```bash
ansible-playbook \
  playbooks/AkamaiVersionConfigPush/akamai/01_akamai_prechecks.yml \
  -i inventories/akamai/qa/aki_qa.yml \
  --syntax-check
```

```bash
ansible-lint roles/AkamaiVersionConfigPush/akamai/
```

```bash
yamllint roles/AkamaiVersionConfigPush/akamai/
```

## Important unresolved item

The actual ServiceNow approver field must be confirmed with the PNC ServiceNow team. This implementation uses `u_approver` as an explicit placeholder. Do not replace it with `assigned_to` or `requested_by`; those fields are not proof of approval.
