# SSPA-11 — Updated Code with Separate Input and ServiceNow Validation

## Directory structure

```text
roles/AkamaiVersionConfigPush/akamai/
├── defaults/
│   └── main.yml
├── vars/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── prechecks.yml
│   ├── 01_input_validation.yml
│   └── 03_snow_validation.yml
├── files/
│   └── mocks/
│       └── snow/
│           ├── valid_implement_approved.json
│           ├── invalid_not_approved.json
│           ├── invalid_state.json
│           ├── outside_window_early.json
│           ├── outside_window_late.json
│           ├── missing_window.json
│           ├── missing_approver.json
│           ├── not_found.json
│           ├── auth_failure.json
│           └── timeout.json
└── meta/
    └── main.yml
```

## `defaults/main.yml`

```yaml
---
snow_mode: mock
snow_mock_scenario: valid_implement_approved
snow_instance: ""

snow_validate_certs: true
snow_ca_path: ""

snow_datetime_format: "%Y-%m-%d %H:%M:%S"
snow_enforce_window: true
```

## `vars/main.yml`

```yaml
---
snow_change_table: change_request

snow_required_state: implement
snow_required_approval: approved

snow_api_timeout: 30
snow_api_retries: 3
snow_api_retry_delay: 5

snow_mock_dir: "mocks/snow"

snow_approver_field: u_approver

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

## `tasks/main.yml`

```yaml
---
- name: Run Akamai prechecks
  ansible.builtin.import_tasks: prechecks.yml
  tags:
    - prechecks
```

## `tasks/prechecks.yml`

```yaml
---
- name: "Step 1 | Validate workflow inputs"
  ansible.builtin.include_tasks:
    file: 01_input_validation.yml
    apply:
      tags:
        - input_validation
        - prechecks
  tags:
    - always
    - input_validation
    - prechecks

- name: "Step 3 | Validate ServiceNow change"
  ansible.builtin.include_tasks:
    file: 03_snow_validation.yml
    apply:
      tags:
        - snow
        - prechecks
  tags:
    - snow
    - prechecks
```

## `tasks/01_input_validation.yml`

```yaml
---
- name: "01 | Validate change number input"
  ansible.builtin.assert:
    that:
      - akamai_chg_number is defined
      - akamai_chg_number is string
      - akamai_chg_number | trim | length > 0
      - akamai_chg_number | trim is match('^CHG[0-9]{7}$')
    fail_msg: >-
      Invalid or missing akamai_chg_number.
      The value must use CHG followed by exactly seven digits.
    quiet: true

- name: "01 | Validate Akamai property input"
  ansible.builtin.assert:
    that:
      - akamai_property_name is defined
      - akamai_property_name is string
      - akamai_property_name | trim | length > 0
    fail_msg: "akamai_property_name is mandatory and cannot be empty."
    quiet: true

- name: "01 | Validate candidate version input"
  ansible.builtin.assert:
    that:
      - akamai_candidate_version is defined
      - akamai_candidate_version | string | trim | length > 0
      - akamai_candidate_version | string is match('^[0-9]+$')
      - akamai_candidate_version | int > 0
    fail_msg: "akamai_candidate_version must be a positive integer."
    quiet: true

- name: "01 | Validate Akamai contract input"
  ansible.builtin.assert:
    that:
      - akamai_contract_id is defined
      - akamai_contract_id is string
      - akamai_contract_id | trim | length > 0
    fail_msg: "akamai_contract_id is mandatory and cannot be empty."
    quiet: true

- name: "01 | Validate Akamai group input"
  ansible.builtin.assert:
    that:
      - akamai_group_id is defined
      - akamai_group_id is string
      - akamai_group_id | trim | length > 0
    fail_msg: "akamai_group_id is mandatory and cannot be empty."
    quiet: true

- name: "01 | Validate target network comes from inventory"
  ansible.builtin.assert:
    that:
      - akamai_network is defined
      - akamai_network is string
      - akamai_network | lower in ['staging', 'production']
    fail_msg: >-
      akamai_network must come from inventory and be staging or production.
    quiet: true

- name: "01 | Normalize validated workflow inputs"
  ansible.builtin.set_fact:
    akamai_chg_number: "{{ akamai_chg_number | trim }}"
    akamai_property_name: "{{ akamai_property_name | trim }}"
    akamai_candidate_version: "{{ akamai_candidate_version | int }}"
    akamai_contract_id: "{{ akamai_contract_id | trim }}"
    akamai_group_id: "{{ akamai_group_id | trim }}"
    akamai_network: "{{ akamai_network | lower | trim }}"

- name: "01 | Publish common input validation result"
  ansible.builtin.set_fact:
    akamai_input_validation_passed: true
```

## `tasks/03_snow_validation.yml`

```yaml
---
- name: "03 | Require common input validation"
  ansible.builtin.assert:
    that:
      - akamai_input_validation_passed | default(false) | bool
    fail_msg: >-
      ServiceNow validation cannot run because common workflow input
      validation has not passed.
    quiet: true

- name: "03 | Validate ServiceNow execution configuration"
  ansible.builtin.assert:
    that:
      - snow_mode is defined
      - snow_mode in ['mock', 'live']
      - snow_required_state is defined
      - snow_required_state | string | trim | length > 0
      - snow_required_approval is defined
      - snow_required_approval | string | trim | length > 0
    fail_msg: "Invalid ServiceNow configuration."
    quiet: true

- name: "03 | Validate ServiceNow mock configuration"
  ansible.builtin.assert:
    that:
      - snow_mock_scenario is defined
      - snow_mock_scenario is string
      - snow_mock_scenario | trim | length > 0
      - snow_mock_scenario is match('^[A-Za-z0-9_-]+$')
    fail_msg: >-
      snow_mock_scenario is mandatory in mock mode and may contain only
      letters, numbers, underscores and hyphens.
    quiet: true
  when: snow_mode == 'mock'

- name: "03 | Retrieve ServiceNow change from mock fixture"
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

    - name: "03 | Validate mock response status"
      ansible.builtin.assert:
        that:
          - _snow_raw is mapping
          - _snow_raw.http_status is defined
          - _snow_raw.http_status | int == 200
        fail_msg: >-
          [MOCK] ServiceNow retrieval failed.
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
          Mock fixture '{{ snow_mock_scenario }}' does not contain a valid result.
        quiet: true

    - name: "03 | Normalize mock ServiceNow response"
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

- name: "03 | Retrieve ServiceNow change from live API"
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
          Live mode requires an HTTPS ServiceNow instance and valid credentials.
        quiet: true
      no_log: true

    - name: "03 | Build ServiceNow query parameters"
      ansible.builtin.set_fact:
        _snow_query_parameters:
          sysparm_query: "number={{ akamai_chg_number }}"
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
            _snow_query_parameters | urlencode
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

    - name: "03 | Require exactly one ServiceNow change"
      ansible.builtin.assert:
        that:
          - _snow_response.json is defined
          - _snow_response.json.result is defined
          - _snow_response.json.result | length == 1
        fail_msg: >-
          ServiceNow lookup for {{ akamai_chg_number }} did not return exactly
          one record.
        quiet: true
      no_log: true

    - name: "03 | Normalize live ServiceNow response"
      ansible.builtin.set_fact:
        snow_chg: "{{ _snow_response.json.result[0] }}"
        snow_fetch_source: live
      no_log: true

  rescue:
    - name: "03 | Fail live ServiceNow retrieval"
      ansible.builtin.fail:
        msg: >-
          Live ServiceNow validation failed for {{ akamai_chg_number }}
          after {{ snow_api_retries }} attempts.

- name: "03 | Validate mandatory ServiceNow fields"
  ansible.builtin.assert:
    that:
      - snow_chg is defined
      - snow_chg is mapping
      - snow_chg.number is defined
      - snow_chg.number | string | trim == akamai_chg_number
      - snow_chg.sys_id is defined
      - snow_chg.sys_id | string | trim | length > 0
      - snow_chg.state is defined
      - snow_chg.approval is defined
      - snow_chg.start_date is defined
      - snow_chg.end_date is defined
    fail_msg: "ServiceNow record is missing mandatory fields."
    quiet: true

- name: "03 | Normalize ServiceNow values"
  ansible.builtin.set_fact:
    _snow_state: >-
      {{
        (
          snow_chg.state.display_value
          if snow_chg.state is mapping and snow_chg.state.display_value is defined
          else
          (
            snow_chg.state.value
            if snow_chg.state is mapping and snow_chg.state.value is defined
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
          if snow_chg.approval is mapping and snow_chg.approval.display_value is defined
          else
          (
            snow_chg.approval.value
            if snow_chg.approval is mapping and snow_chg.approval.value is defined
            else snow_chg.approval
          )
        )
        | string
        | trim
        | lower
      }}

    _snow_start_raw: "{{ snow_chg.start_date | string | trim }}"
    _snow_end_raw: "{{ snow_chg.end_date | string | trim }}"

- name: "03 | Require Implement state and Approved approval"
  ansible.builtin.assert:
    that:
      - _snow_state == snow_required_state | lower
      - _snow_approval == snow_required_approval | lower
    fail_msg: >-
      ServiceNow gate failed for {{ akamai_chg_number }}.
      state={{ _snow_state }}, approval={{ _snow_approval }}.
    quiet: true

- name: "03 | Validate planned implementation window"
  when: snow_enforce_window | bool
  block:
    - name: "03 | Validate window is populated"
      ansible.builtin.assert:
        that:
          - _snow_start_raw | length > 0
          - _snow_end_raw | length > 0
        fail_msg: "ServiceNow implementation window is missing."
        quiet: true

    - name: "03 | Convert ServiceNow window to epoch"
      ansible.builtin.set_fact:
        _snow_start_epoch: >-
          {{
            (_snow_start_raw | to_datetime(snow_datetime_format)).strftime('%s')
            | int
          }}
        _snow_end_epoch: >-
          {{
            (_snow_end_raw | to_datetime(snow_datetime_format)).strftime('%s')
            | int
          }}
        _snow_now_epoch: "{{ now(utc=true, fmt='%s') | int }}"

    - name: "03 | Require execution inside planned window"
      ansible.builtin.assert:
        that:
          - _snow_start_epoch | int < _snow_end_epoch | int
          - _snow_now_epoch | int >= _snow_start_epoch | int
          - _snow_now_epoch | int <= _snow_end_epoch | int
        fail_msg: "Execution is outside the ServiceNow implementation window."
        quiet: true

- name: "03 | Extract ServiceNow approver"
  ansible.builtin.set_fact:
    akamai_peer_reviewed_by: >-
      {{
        snow_chg[snow_approver_field]
        | default('')
        | string
        | trim
      }}

- name: "03 | Require ServiceNow approver evidence"
  ansible.builtin.assert:
    that:
      - akamai_peer_reviewed_by | length > 0
    fail_msg: >-
      No approver identity was returned from field {{ snow_approver_field }}.
    quiet: true

- name: "03 | Publish ServiceNow validation result"
  ansible.builtin.set_fact:
    akamai_chg_validated: true
    snow_validation_result:
      story: SSPA-11
      step: 3
      result: PASSED
      mode: "{{ snow_mode }}"
      source: "{{ snow_fetch_source }}"
      change_number: "{{ snow_chg.number }}"
      change_sys_id: "{{ snow_chg.sys_id }}"
      state: "{{ _snow_state }}"
      approval: "{{ _snow_approval }}"
      planned_start_utc: "{{ _snow_start_raw }}"
      planned_end_utc: "{{ _snow_end_raw }}"
      peer_reviewed_by: "{{ akamai_peer_reviewed_by }}"
      request_method: GET
      read_only: true
```

## `meta/main.yml`

```yaml
---
galaxy_info:
  author: SSP Automation
  description: >-
    Akamai Version Configuration Push role with prechecks,
    ServiceNow validation and controlled version promotion.
  min_ansible_version: "2.15"

dependencies: []
```

## Mock fixture: `valid_implement_approved.json`

```json
{
  "http_status": 200,
  "result": {
    "number": "CHG0012345",
    "sys_id": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
    "state": "Implement",
    "approval": "Approved",
    "start_date": "2026-08-01 00:00:00",
    "end_date": "2026-08-31 23:59:59",
    "short_description": "Akamai delivery configuration promotion",
    "u_approver": "peer.reviewer"
  }
}
```

## Test command

```bash
ansible-playbook   playbooks/AkamaiVersionConfigPush/akamai/01_akamai_prechecks.yml   -i inventories/akamai/qa/aki_qa.yml   -e akamai_chg_number=CHG0012345   -e akamai_property_name=aox_qa_web   -e akamai_candidate_version=252   -e akamai_contract_id=ctr_1   -e akamai_group_id=grp_1   -e snow_mode=mock   -e snow_mock_scenario=valid_implement_approved   --tags snow
```

Important: `u_approver` remains a placeholder until the ServiceNow team confirms the real CHG approver field.
