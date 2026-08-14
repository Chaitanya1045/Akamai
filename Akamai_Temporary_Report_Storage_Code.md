# Akamai Version Push — Temporary Report Storage Code

This implementation stores reports directly under `/tmp/` inside the active AAP Execution Environment.

It intentionally does **not** implement persistent storage or report retention. The generated report is expected to be consumed by downstream report-generation/email logic before the Execution Environment terminates.

No user-story IDs are referenced in the code.

---

## Repository Changes

```text
roles/
└── akamai_version_push/
    ├── defaults/
    │   └── main.yml                       # UPDATE
    ├── vars/
    │   └── main.yml                       # UPDATE
    └── tasks/
        ├── main.yml
        ├── report_storage.yml             # NEW
        ├── ssp11_servicenow_cr_validation.yml
        └── ssp15_ctask_update.yml

playbooks/
└── akamai_version_push/
    └── report_storage.yml                 # NEW
```

---

# 1. `roles/akamai_version_push/defaults/main.yml`

Append the following section to the existing file.

```yaml
# =============================================================================
# Temporary report storage configuration
#
# Reports are stored directly under /tmp inside the active AAP
# Execution Environment.
#
# The filesystem is ephemeral. Reports must be consumed by downstream
# processing, such as email delivery, before the Execution Environment ends.
# =============================================================================

# Perform real write/read/list verification before report generation.
report_storage_verify_io: true


# -----------------------------------------------------------------------------
# Optional shared read access
#
# Leave empty when reports should only be readable by the execution user.
#
# If configured, this must be an OS group available inside the Execution
# Environment. The report file will then be readable by that group.
# -----------------------------------------------------------------------------

report_storage_access_group: ""


# -----------------------------------------------------------------------------
# File permissions
#
# Private:
#   owner = read/write
#   group = none
#   others = none
#
# Shared:
#   owner = read/write
#   group = read
#   others = none
# -----------------------------------------------------------------------------

report_storage_private_file_mode: "0600"

report_storage_group_file_mode: "0640"


# -----------------------------------------------------------------------------
# Final report naming inputs
#
# These can be supplied by downstream report-generation logic.
#
# Example:
#
# report_storage_version: "252"
# report_storage_status: "success"
# -----------------------------------------------------------------------------

report_storage_version: ""

report_storage_status: ""
```

---

# 2. `roles/akamai_version_push/vars/main.yml`

Append the following section to the existing file.

```yaml
# =============================================================================
# Temporary report storage internal constants
# =============================================================================

# Reports are intentionally written directly under /tmp.
report_storage_root: "/tmp"


# -----------------------------------------------------------------------------
# Naming convention
#
# Format:
#
#   {timestamp}_{version}_{status}.txt
#
# Example:
#
#   20260814T093015Z_252_success.txt
# -----------------------------------------------------------------------------

report_storage_filename_format: >-
  {timestamp}_{version}_{status}.txt

report_storage_filename_regex: >-
  ^[0-9]{8}T[0-9]{6}Z_[0-9]+_[a-z][a-z0-9_-]{0,31}\.txt$

report_storage_version_regex: >-
  ^[0-9]+$

report_storage_status_regex: >-
  ^[a-z][a-z0-9_-]{0,31}$


# -----------------------------------------------------------------------------
# Temporary accessibility probe
# -----------------------------------------------------------------------------

report_storage_probe_prefix: ".akamai_report_probe"


# -----------------------------------------------------------------------------
# Common logging stages
# -----------------------------------------------------------------------------

report_storage_log_stages:
  validation: "report_storage_validation"
  access: "report_storage_access"
  write: "report_storage_write"
  read: "report_storage_read"
  list: "report_storage_list"
  dry_run: "report_storage_dry_run"
  naming: "report_storage_naming"


# -----------------------------------------------------------------------------
# Stable common logging error identifiers
# -----------------------------------------------------------------------------

report_storage_error_keys:
  config_invalid: "REPORT_STORAGE_CONFIG_INVALID"
  access_group_invalid: "REPORT_ACCESS_GROUP_INVALID"
  storage_unavailable: "REPORT_STORAGE_UNAVAILABLE"
  write_failed: "REPORT_STORAGE_WRITE_FAILED"
  read_failed: "REPORT_STORAGE_READ_FAILED"
  list_failed: "REPORT_STORAGE_LIST_FAILED"
  dry_run_failed: "REPORT_STORAGE_DRY_RUN_FAILED"
  naming_invalid: "REPORT_NAME_INVALID"
```

---

# 3. `roles/akamai_version_push/tasks/report_storage.yml`

```yaml
---
# =============================================================================
# Temporary report storage
#
# Reports are stored directly under /tmp inside the active AAP
# Execution Environment.
#
# Responsibilities:
#   - Validate temporary storage configuration
#   - Confirm /tmp is available
#   - Validate optional shared-access group
#   - Verify real write/read/list capability
#   - Enforce restrictive report-file permissions
#   - Define and validate the report naming convention
#   - Expose the resolved report path to downstream tasks
#   - Use common_logging for standardized success/failure handling
#
# This task does NOT:
#   - provide persistent storage
#   - implement retention
#   - send email
#   - generate report content
# =============================================================================


# =============================================================================
# Runtime context
# =============================================================================

- name: "Report Storage | Resolve runtime timestamp"
  ansible.builtin.set_fact:
    report_storage_runtime_timestamp: >-
      {{ now(utc=true, fmt='%Y%m%dT%H%M%SZ') }}


- name: "Report Storage | Resolve logging correlation"
  ansible.builtin.set_fact:
    report_storage_log_request_id: >-
      {{
        common_logging_request_id
        | default(
            'REPORT-' ~ report_storage_runtime_timestamp,
            true
          )
        | string
      }}

    report_storage_log_execution_id: >-
      {{
        common_logging_execution_id
        | default(
            awx_job_id
            | default(
                report_storage_runtime_timestamp,
                true
              ),
            true
          )
        | string
      }}

    report_storage_log_actor: >-
      {{
        common_logging_actor
        | default(
            awx_user_email
            | default('automation', true),
            true
          )
        | string
      }}

    report_storage_log_triggered_by: >-
      {{
        common_logging_triggered_by
        | default(
            awx_user_email
            | default('automation', true),
            true
          )
        | string
      }}


- name: "Report Storage | Resolve logging environment"
  ansible.builtin.set_fact:
    report_storage_requested_environment: >-
      {{
        common_logging_environment
        | default(
            (akamai | default({}))
            .get('environment', {})
            .get('name', 'local'),
            true
          )
        | string
        | lower
      }}


- name: "Report Storage | Resolve safe logging environment"
  ansible.builtin.set_fact:
    report_storage_log_environment: >-
      {{
        report_storage_requested_environment
        if report_storage_requested_environment
           in ['dev', 'rnd', 'qa', 'prod', 'test', 'local']
        else 'local'
      }}


# =============================================================================
# Resolve effective file permissions
# =============================================================================

- name: "Report Storage | Resolve effective file permissions"
  ansible.builtin.set_fact:
    report_storage_effective_file_mode: >-
      {{
        report_storage_group_file_mode
        if (report_storage_access_group | trim | length > 0)
        else report_storage_private_file_mode
      }}


# =============================================================================
# Configuration validation
# =============================================================================

- name: "Report Storage | Validate configuration"
  block:

    - name: "Report Storage | Validate fixed temporary storage root"
      ansible.builtin.assert:
        that:
          - report_storage_root == '/tmp'
        fail_msg: >-
          Temporary report storage must be /tmp.


    - name: "Report Storage | Validate file permission configuration"
      ansible.builtin.assert:
        that:
          - >-
            report_storage_private_file_mode
            is match('^[0-7]{4}$')
          - >-
            report_storage_group_file_mode
            is match('^[0-7]{4}$')
          - report_storage_private_file_mode[-1] == '0'
          - report_storage_group_file_mode[-1] == '0'
        fail_msg: >-
          Report file modes must be valid four-digit octal strings
          and must provide no access to other users.


    - name: "Report Storage | Validate optional access group format"
      ansible.builtin.assert:
        that:
          - >-
            (
              report_storage_access_group | trim | length == 0
            )
            or
            (
              report_storage_access_group
              is match('^[A-Za-z_][A-Za-z0-9_.-]{0,63}$')
            )
        fail_msg: >-
          report_storage_access_group must be empty or contain
          a valid operating-system group name.


    - name: "Report Storage | Validate filename contract"
      ansible.builtin.assert:
        that:
          - >-
            (
              report_storage_runtime_timestamp
              ~ '_1_success.txt'
            )
            is match(report_storage_filename_regex)
        fail_msg: >-
          Report filename contract is invalid.
          Required format is timestamp_version_status.txt.


  rescue:

    - name: "Report Storage | Publish configuration failure"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: >-
          {{ report_storage_log_request_id }}

        common_logging_execution_id: >-
          {{ report_storage_log_execution_id }}

        common_logging_environment: >-
          {{ report_storage_log_environment }}

        common_logging_triggered_by: >-
          {{ report_storage_log_triggered_by }}

        common_logging_error_stage: >-
          {{ report_storage_log_stages.validation }}

        common_logging_error_actor: >-
          {{ report_storage_log_actor }}

        common_logging_error_category_input: "validation"

        common_logging_error_key_input: >-
          {{ report_storage_error_keys.config_invalid }}

        common_logging_error_message: >-
          Temporary report storage configuration validation failed.

        common_logging_error_recommended_action: >-
          Correct the report storage permissions, access group,
          or filename configuration and retry.

        common_logging_error_version_info: {}


# =============================================================================
# Validate optional access group
# =============================================================================

- name: "Report Storage | Validate configured read-access group"
  when:
    - report_storage_access_group | trim | length > 0
  block:

    - name: "Report Storage | Query configured OS group"
      ansible.builtin.getent:
        database: group
        key: "{{ report_storage_access_group }}"


  rescue:

    - name: "Report Storage | Publish access-group failure"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: >-
          {{ report_storage_log_request_id }}

        common_logging_execution_id: >-
          {{ report_storage_log_execution_id }}

        common_logging_environment: >-
          {{ report_storage_log_environment }}

        common_logging_triggered_by: >-
          {{ report_storage_log_triggered_by }}

        common_logging_error_stage: >-
          {{ report_storage_log_stages.access }}

        common_logging_error_actor: >-
          {{ report_storage_log_actor }}

        common_logging_error_category_input: "infrastructure"

        common_logging_error_key_input: >-
          {{ report_storage_error_keys.access_group_invalid }}

        common_logging_error_message: >-
          Configured report read-access group is unavailable
          inside the Execution Environment.

        common_logging_error_recommended_action: >-
          Configure a valid group available inside the Execution
          Environment or leave report_storage_access_group empty.

        common_logging_error_version_info: {}


# =============================================================================
# Validate /tmp
# =============================================================================

- name: "Report Storage | Validate temporary filesystem"
  block:

    - name: "Report Storage | Inspect /tmp"
      ansible.builtin.stat:
        path: "{{ report_storage_root }}"
      register: report_storage_root_stat


    - name: "Report Storage | Confirm /tmp is available"
      ansible.builtin.assert:
        that:
          - report_storage_root_stat.stat.exists
          - report_storage_root_stat.stat.isdir
        fail_msg: >-
          /tmp is unavailable inside the Execution Environment.


  rescue:

    - name: "Report Storage | Publish temporary-storage failure"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: >-
          {{ report_storage_log_request_id }}

        common_logging_execution_id: >-
          {{ report_storage_log_execution_id }}

        common_logging_environment: >-
          {{ report_storage_log_environment }}

        common_logging_triggered_by: >-
          {{ report_storage_log_triggered_by }}

        common_logging_error_stage: >-
          {{ report_storage_log_stages.access }}

        common_logging_error_actor: >-
          {{ report_storage_log_actor }}

        common_logging_error_category_input: "infrastructure"

        common_logging_error_key_input: >-
          {{ report_storage_error_keys.storage_unavailable }}

        common_logging_error_message: >-
          Temporary report storage is unavailable.

        common_logging_error_recommended_action: >-
          Verify that /tmp is mounted and accessible inside
          the AAP Execution Environment.

        common_logging_error_version_info: {}


# =============================================================================
# Build unique verification filename
# =============================================================================

- name: "Report Storage | Resolve verification probe"
  ansible.builtin.set_fact:
    report_storage_probe_filename: >-
      {{ report_storage_probe_prefix }}_{{
        report_storage_log_execution_id
        | regex_replace('[^A-Za-z0-9._-]', '_')
      }}_{{ report_storage_runtime_timestamp }}.tmp

    report_storage_probe_content: >-
      REPORT_STORAGE_VERIFY_{{
        report_storage_log_execution_id
        | regex_replace('[^A-Za-z0-9._-]', '_')
      }}

    report_storage_probe_path: >-
      {{ report_storage_root }}/{{ report_storage_probe_prefix }}_{{
        report_storage_log_execution_id
        | regex_replace('[^A-Za-z0-9._-]', '_')
      }}_{{ report_storage_runtime_timestamp }}.tmp


# =============================================================================
# Check-mode / dry-run verification
# =============================================================================

- name: "Report Storage | Perform check-mode validation"
  when:
    - ansible_check_mode
  block:

    - name: "Report Storage | Validate sample report filename"
      ansible.builtin.assert:
        that:
          - >-
            (
              report_storage_runtime_timestamp
              ~ '_1_success.txt'
            )
            is match(report_storage_filename_regex)
        fail_msg: >-
          Report naming validation failed.


    - name: "Report Storage | Display check-mode plan"
      ansible.builtin.debug:
        msg:
          storage_root: "{{ report_storage_root }}"

          file_mode: >-
            {{ report_storage_effective_file_mode }}

          access_group: >-
            {{
              report_storage_access_group
              if report_storage_access_group | trim | length > 0
              else 'execution-user-only'
            }}

          filename_format: >-
            {{ report_storage_filename_format }}

          example_filename: >-
            {{ report_storage_runtime_timestamp }}_1_success.txt

          persistent_storage: false

          retention_enabled: false

          filesystem_changes: false


    - name: "Report Storage | Log successful check-mode validation"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: log_event
      vars:
        common_logging_request_id: >-
          {{ report_storage_log_request_id }}

        common_logging_execution_id: >-
          {{ report_storage_log_execution_id }}

        common_logging_environment: >-
          {{ report_storage_log_environment }}

        common_logging_triggered_by: >-
          {{ report_storage_log_triggered_by }}

        common_logging_actor: >-
          {{ report_storage_log_actor }}

        common_logging_stage: >-
          {{ report_storage_log_stages.dry_run }}

        common_logging_result: "success"

        common_logging_message: >-
          Temporary report storage passed check-mode validation.

        common_logging_version_info: {}


  rescue:

    - name: "Report Storage | Publish check-mode failure"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: >-
          {{ report_storage_log_request_id }}

        common_logging_execution_id: >-
          {{ report_storage_log_execution_id }}

        common_logging_environment: >-
          {{ report_storage_log_environment }}

        common_logging_triggered_by: >-
          {{ report_storage_log_triggered_by }}

        common_logging_error_stage: >-
          {{ report_storage_log_stages.dry_run }}

        common_logging_error_actor: >-
          {{ report_storage_log_actor }}

        common_logging_error_category_input: "validation"

        common_logging_error_key_input: >-
          {{ report_storage_error_keys.dry_run_failed }}

        common_logging_error_message: >-
          Temporary report storage check-mode validation failed.

        common_logging_error_recommended_action: >-
          Verify /tmp availability and report storage configuration.

        common_logging_error_version_info: {}


# =============================================================================
# Write verification
# =============================================================================

- name: "Report Storage | Verify write capability"
  when:
    - not ansible_check_mode
    - report_storage_verify_io | bool
  block:

    - name: "Report Storage | Write verification file"
      ansible.builtin.copy:
        dest: "{{ report_storage_probe_path }}"
        content: |
          {{ report_storage_probe_content }}
        group: >-
          {{
            report_storage_access_group
            if report_storage_access_group | trim | length > 0
            else omit
          }}
        mode: "{{ report_storage_effective_file_mode }}"


    - name: "Report Storage | Inspect verification file"
      ansible.builtin.stat:
        path: "{{ report_storage_probe_path }}"
      register: report_storage_probe_stat


    - name: "Report Storage | Validate verification file permissions"
      ansible.builtin.assert:
        that:
          - report_storage_probe_stat.stat.exists
          - report_storage_probe_stat.stat.isreg

          - >-
            report_storage_probe_stat.stat.mode
            ==
            report_storage_effective_file_mode

          - >-
            (
              report_storage_access_group | trim | length == 0
            )
            or
            (
              report_storage_probe_stat.stat.gr_name
              ==
              report_storage_access_group
            )
        fail_msg: >-
          Verification file was not created with the expected
          file permissions or group ownership.


  rescue:

    - name: "Report Storage | Remove incomplete verification file"
      ansible.builtin.file:
        path: "{{ report_storage_probe_path }}"
        state: absent
      failed_when: false


    - name: "Report Storage | Publish write failure"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: >-
          {{ report_storage_log_request_id }}

        common_logging_execution_id: >-
          {{ report_storage_log_execution_id }}

        common_logging_environment: >-
          {{ report_storage_log_environment }}

        common_logging_triggered_by: >-
          {{ report_storage_log_triggered_by }}

        common_logging_error_stage: >-
          {{ report_storage_log_stages.write }}

        common_logging_error_actor: >-
          {{ report_storage_log_actor }}

        common_logging_error_category_input: "infrastructure"

        common_logging_error_key_input: >-
          {{ report_storage_error_keys.write_failed }}

        common_logging_error_message: >-
          Automation could not write a temporary report artifact
          under /tmp.

        common_logging_error_recommended_action: >-
          Verify /tmp availability and write permissions for
          the Execution Environment user.

        common_logging_error_version_info: {}


# =============================================================================
# Read verification
# =============================================================================

- name: "Report Storage | Verify read capability"
  when:
    - not ansible_check_mode
    - report_storage_verify_io | bool
  block:

    - name: "Report Storage | Read verification file"
      ansible.builtin.slurp:
        src: "{{ report_storage_probe_path }}"
      register: report_storage_probe_read


    - name: "Report Storage | Validate verification-file content"
      ansible.builtin.assert:
        that:
          - >-
            (
              report_storage_probe_read.content
              | b64decode
              | trim
            )
            ==
            report_storage_probe_content
        fail_msg: >-
          Data read from temporary report storage does not match
          the data written by the automation.


  rescue:

    - name: "Report Storage | Remove verification file after read failure"
      ansible.builtin.file:
        path: "{{ report_storage_probe_path }}"
        state: absent
      failed_when: false


    - name: "Report Storage | Publish read failure"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: >-
          {{ report_storage_log_request_id }}

        common_logging_execution_id: >-
          {{ report_storage_log_execution_id }}

        common_logging_environment: >-
          {{ report_storage_log_environment }}

        common_logging_triggered_by: >-
          {{ report_storage_log_triggered_by }}

        common_logging_error_stage: >-
          {{ report_storage_log_stages.read }}

        common_logging_error_actor: >-
          {{ report_storage_log_actor }}

        common_logging_error_category_input: "infrastructure"

        common_logging_error_key_input: >-
          {{ report_storage_error_keys.read_failed }}

        common_logging_error_message: >-
          Automation could not read a temporary report artifact
          from /tmp.

        common_logging_error_recommended_action: >-
          Verify /tmp availability and read permissions for
          the Execution Environment user.

        common_logging_error_version_info: {}


# =============================================================================
# List verification
# =============================================================================

- name: "Report Storage | Verify list capability"
  when:
    - not ansible_check_mode
    - report_storage_verify_io | bool
  block:

    - name: "Report Storage | Locate verification file"
      ansible.builtin.find:
        paths: "{{ report_storage_root }}"
        patterns:
          - "{{ report_storage_probe_filename }}"
        file_type: file
        recurse: false
      register: report_storage_probe_list


    - name: "Report Storage | Validate list result"
      ansible.builtin.assert:
        that:
          - report_storage_probe_list.matched | int == 1
        fail_msg: >-
          Automation could not locate the verification artifact
          written under /tmp.


  rescue:

    - name: "Report Storage | Remove verification file after list failure"
      ansible.builtin.file:
        path: "{{ report_storage_probe_path }}"
        state: absent
      failed_when: false


    - name: "Report Storage | Publish list failure"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: >-
          {{ report_storage_log_request_id }}

        common_logging_execution_id: >-
          {{ report_storage_log_execution_id }}

        common_logging_environment: >-
          {{ report_storage_log_environment }}

        common_logging_triggered_by: >-
          {{ report_storage_log_triggered_by }}

        common_logging_error_stage: >-
          {{ report_storage_log_stages.list }}

        common_logging_error_actor: >-
          {{ report_storage_log_actor }}

        common_logging_error_category_input: "infrastructure"

        common_logging_error_key_input: >-
          {{ report_storage_error_keys.list_failed }}

        common_logging_error_message: >-
          Automation could not list the temporary report artifact
          under /tmp.

        common_logging_error_recommended_action: >-
          Verify /tmp access permissions inside the
          Execution Environment.

        common_logging_error_version_info: {}


# =============================================================================
# Remove verification artifact
# =============================================================================

- name: "Report Storage | Remove verification artifact"
  ansible.builtin.file:
    path: "{{ report_storage_probe_path }}"
    state: absent
  when:
    - not ansible_check_mode
    - report_storage_verify_io | bool


# =============================================================================
# Resolve final report path
# =============================================================================

- name: "Report Storage | Resolve final report path"
  when:
    - report_storage_version | string | trim | length > 0
    - report_storage_status | string | trim | length > 0
  block:

    - name: "Report Storage | Normalize report status"
      ansible.builtin.set_fact:
        report_storage_normalized_status: >-
          {{
            report_storage_status
            | string
            | trim
            | lower
            | regex_replace('[^a-z0-9_-]', '_')
          }}


    - name: "Report Storage | Validate filename inputs"
      ansible.builtin.assert:
        that:
          - >-
            (
              report_storage_version
              | string
              | trim
            )
            is match(report_storage_version_regex)

          - >-
            report_storage_normalized_status
            is match(report_storage_status_regex)
        fail_msg: >-
          Report version must be numeric and report status must
          resolve to a safe lowercase filename value.


    - name: "Report Storage | Build final report filename and path"
      ansible.builtin.set_fact:
        report_storage_report_filename: >-
          {{ report_storage_runtime_timestamp }}_{{ report_storage_version | string | trim }}_{{ report_storage_normalized_status }}.txt

        report_storage_report_path: >-
          {{ report_storage_root }}/{{ report_storage_runtime_timestamp }}_{{ report_storage_version | string | trim }}_{{ report_storage_normalized_status }}.txt


    - name: "Report Storage | Validate generated filename"
      ansible.builtin.assert:
        that:
          - >-
            report_storage_report_filename
            is match(report_storage_filename_regex)
        fail_msg: >-
          Generated report filename does not satisfy the required
          timestamp_version_status.txt convention.


  rescue:

    - name: "Report Storage | Publish naming failure"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: >-
          {{ report_storage_log_request_id }}

        common_logging_execution_id: >-
          {{ report_storage_log_execution_id }}

        common_logging_environment: >-
          {{ report_storage_log_environment }}

        common_logging_triggered_by: >-
          {{ report_storage_log_triggered_by }}

        common_logging_error_stage: >-
          {{ report_storage_log_stages.naming }}

        common_logging_error_actor: >-
          {{ report_storage_log_actor }}

        common_logging_error_category_input: "validation"

        common_logging_error_key_input: >-
          {{ report_storage_error_keys.naming_invalid }}

        common_logging_error_message: >-
          Report filename generation failed.

        common_logging_error_recommended_action: >-
          Supply a numeric Akamai version and a valid report status.

        common_logging_error_version_info: {}


# =============================================================================
# Publish workspace information
# =============================================================================

- name: "Report Storage | Publish temporary report storage metadata"
  ansible.builtin.set_stats:
    data:
      report_storage:
        root: "{{ report_storage_root }}"

        filename_format: >-
          {{ report_storage_filename_format }}

        report_filename: >-
          {{ report_storage_report_filename | default('') }}

        report_path: >-
          {{ report_storage_report_path | default('') }}

        file_mode: >-
          {{ report_storage_effective_file_mode }}

        ephemeral: true

        dry_run: >-
          {{ ansible_check_mode | bool }}

    per_host: false


# =============================================================================
# Final success event
# =============================================================================

- name: "Report Storage | Log successful storage validation"
  ansible.builtin.include_role:
    name: common_logging
    tasks_from: log_event
  vars:
    common_logging_request_id: >-
      {{ report_storage_log_request_id }}

    common_logging_execution_id: >-
      {{ report_storage_log_execution_id }}

    common_logging_environment: >-
      {{ report_storage_log_environment }}

    common_logging_triggered_by: >-
      {{ report_storage_log_triggered_by }}

    common_logging_actor: >-
      {{ report_storage_log_actor }}

    common_logging_stage: >-
      {{ report_storage_log_stages.access }}

    common_logging_result: "success"

    common_logging_message: >-
      Temporary report storage under /tmp passed accessibility
      and naming validation.

    common_logging_version_info: {}

  when:
    - not ansible_check_mode
```

---

# 4. `playbooks/akamai_version_push/report_storage.yml`

```yaml
---
- name: "Prepare temporary Akamai report storage"
  hosts: localhost
  connection: local
  gather_facts: false
  any_errors_fatal: true

  tasks:

    - name: "Validate temporary report storage"
      ansible.builtin.include_role:
        name: akamai_version_push
        tasks_from: report_storage
```

---

# 5. Optional Integration in `roles/akamai_version_push/tasks/main.yml`

Only add this if the combined role flow is intended to execute the report-storage task automatically.

```yaml
- name: "Prepare temporary report storage"
  ansible.builtin.import_tasks: report_storage.yml
```

Do not replace any existing task imports.

---

# Runtime Flow

```text
AAP Execution Environment starts
        ↓
Validate /tmp
        ↓
Validate optional read-access group
        ↓
Create unique probe file
        ↓
Verify WRITE
        ↓
Verify READ
        ↓
Verify LIST
        ↓
Remove probe
        ↓
Validate report filename contract
        ↓
Resolve final report path
        ↓
/tmp/<timestamp>_<version>_<status>.txt
        ↓
Downstream report-generation logic writes report
        ↓
Separate email logic attaches/sends report
        ↓
AAP execution ends
        ↓
Temporary report disappears with EE
```

---

# Important Runtime Constraint

Because `/tmp` belongs to the active Execution Environment, report-generation and email-delivery logic must consume the report before that EE terminates. If email delivery runs in a completely separate AAP job/container, the original `/tmp` file cannot be assumed to exist there.
