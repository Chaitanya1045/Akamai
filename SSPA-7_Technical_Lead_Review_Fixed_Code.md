# SSPA-7 — Technical Lead Review Fixed Implementation
This package is scoped only to temporary report storage and addresses the technical-lead review comments.
Important: `roles/akamai_version_push/tasks/main.yml` and ServiceNow/CTASK/collection files are intentionally not included. Revert those unrelated PR changes to the `dev` baseline.

---

# `LEAD_REVIEW_FIXES.md`

# SSPA-7 Technical-Lead Review Fixes

This package addresses the review table shown in the lead's screenshots.

| # | Review item | Resolution |
|---|---|---|
| 1 | PR scope | Keep only temporary report-storage files in this PR. ServiceNow and CTASK work must move to their own PRs. |
| 2 | ServiceNow playbook | Remove/revert ServiceNow validation playbook from this PR. |
| 3 | CTASK playbook | Remove/revert CTASK update playbook from this PR. |
| 4 | Collection dependency | Revert ServiceNow collection-version changes; SSPA-7 uses only `ansible.builtin` plus the existing `common_logging` role. |
| 5 | Role dispatcher | Do not import SSPA-7 from the default role entry point. Call explicitly with `tasks_from: report_storage`. |
| 6 | Missing task files | Remove unrelated ServiceNow/CTASK dispatcher references; do not add those task files to this PR. |
| 7 | Temporary lifecycle | Documented: files exist only in the active AAP Execution Environment. |
| 8 | Downstream usage | Documented: only same-job tasks can read `/tmp` directly; separate jobs require approved transfer/persistence. |
| 9 | Published metadata | Removed misleading final `report_path`; publish workspace metadata plus `storage_verified` and `report_exists`. |
| 10 | Metadata naming | Uses `workspace_path`; SSPA-7 does not claim a final report path exists. |
| 11 | Storage verification metadata | Added `storage_verified`. |
| 12 | Report existence metadata | Added `report_exists`, always false in SSPA-7 because this story does not create final report content. |
| 13 | Logging actor | Uses `report_storage_module` as `common_logging_actor` and error actor. |
| 14 | Triggered-by identity | Uses AAP launcher identity only; automation component is not used as launcher. |
| 15 | Environment fallback | Deployed jobs require dev/rnd/qa/prod. test/local fallback only with explicit `report_storage_test_mode=true`. |
| 16 | Environment source | Documented as protected inventory/fixed Job Template only; not Survey/Prompt on Launch. |
| 17 | Request-ID fallback | Deployed jobs require shared `common_logging_request_id`; fallback only in explicit test mode. |
| 18 | Execution ID | Uses shared `common_logging_execution_id`, otherwise `awx_job_id`; local fallback only in test mode. |
| 19 | Logging context | Resolved once into `report_storage_context` and reused by all events/metadata. |
| 20 | Temporary directory | Implemented dedicated `/tmp/akamai-reports/<execution_id>/` workspace. |
| 21 | Directory permissions | Implemented 0700 private or 0750 approved-group mode. |
| 22 | Path traversal | Safe execution-ID regex and explicit containment checks under `/tmp/akamai-reports`. |
| 23 | Identifier safety | Filesystem component restricted to A-Z/a-z/0-9/`.`/`_`/`-`; request ID is not used as path component. |
| 24 | Access group | Documented as protected approved config only, never Survey/Prompt on Launch. |
| 25 | Group verification | Verifies group existence, directory ownership/mode, and probe-file ownership/mode. |
| 26 | Actual report cleanup | Added explicit `report_storage_cleanup.yml` for downstream same-job cleanup. |
| 27 | Cleanup timing | Cleanup is not called from storage entry point; downstream orchestration calls it only after all consumers, preferably in `always`. |
| 28 | Check-mode semantics | Publishes `configuration_validated=true`, `storage_verified=false`, `report_exists=false`; no filesystem writes. |
| 29 | Normal-run semantics | `storage_verified=true` only after real write/read/list checks pass. |
| 30 | Audit rendering | Standalone playbook invokes `common_logging` `render_audit` from `always` when history exists. |
| 31 | Audit filename ownership | SSPA-7 no longer defines audit filenames; common logging owns audit naming. |
| 32 | Final report naming | SSPA-7 no longer generates timestamp/version/status report names; downstream report generation owns final naming/content. |
| 33 | Common-logging success | Continues using `tasks_from: log_event`. |
| 34 | Common-logging failure | Continues using `tasks_from: fail_with_error`. |
| 35 | Error classification | Configuration/protected-input failures use validation; filesystem failures use infrastructure. |
| 36 | Task-file size | Split into context, validation, provisioning, I/O, publishing, and cleanup files. |
| 37 | Encoding | Files contain clean UTF-8/ASCII text; corrupted characters removed. |
| 38 | Documentation | Added complete `docs/report_storage.md` integration contract. |
| 39 | Automated tests | Added executable normal-run and Check-Mode filesystem contract tests. |
| 40 | AAP validation | Must be executed in AAP QA after code update; capture job output/evidence before PR approval. |

## Files that should NOT be part of the SSPA-7 PR

Revert/remove from this PR if currently changed:

```text
ServiceNow validation playbooks/tasks
CTASK update playbooks/tasks
ServiceNow collection version changes
unrelated role dispatcher imports
```

## Files intended for this PR

```text
roles/akamai_version_push/defaults/main.yml              # merge SSPA-7 section
roles/akamai_version_push/vars/main.yml                  # merge SSPA-7 section
roles/akamai_version_push/tasks/report_storage.yml
roles/akamai_version_push/tasks/report_storage_context.yml
roles/akamai_version_push/tasks/report_storage_validate.yml
roles/akamai_version_push/tasks/report_storage_provision.yml
roles/akamai_version_push/tasks/report_storage_verify.yml
roles/akamai_version_push/tasks/report_storage_publish.yml
roles/akamai_version_push/tasks/report_storage_cleanup.yml
playbooks/akamai_version_push/report_storage.yml
docs/report_storage.md
tests/report_storage_contract.yml
tests/report_storage_check_mode.yml
```

Do not modify `roles/akamai_version_push/tasks/main.yml` for this story. Revert it to the `dev` branch version if the current PR added ServiceNow, CTASK, or automatic report-storage imports.

---

# `roles/akamai_version_push/defaults/main.yml`

```yaml
---
# =============================================================================
# Temporary report storage - overridable configuration
# =============================================================================

# Perform real filesystem write/read/list verification in a normal run.
report_storage_verify_io: true

# Explicit standalone/local test switch.
# Keep false for deployed AAP jobs. Do not expose this switch through Survey
# or Prompt on Launch. Enable it only in dedicated standalone test jobs.
report_storage_test_mode: false

# Optional approved OS group for shared read access.
# This value must come from protected inventory or fixed Job Template config.
# Do not expose it through Survey or Prompt on Launch.
report_storage_access_group: ""

# File permissions.
report_storage_private_file_mode: "0600"
report_storage_group_file_mode: "0640"

# Dedicated execution-directory permissions.
report_storage_private_directory_mode: "0700"
report_storage_group_directory_mode: "0750"
```

---

# `roles/akamai_version_push/vars/main.yml`

```yaml
---
# =============================================================================
# Temporary report storage - internal constants
# =============================================================================

# /tmp is the EE-local temporary filesystem. A dedicated application root and
# per-execution directory are used to isolate files and make path containment
# enforceable.
report_storage_base_root: "/tmp"
report_storage_root: "/tmp/akamai-reports"

# Stable component identity used as common_logging actor/error actor.
report_storage_module: "akamai_version_push.report_storage"

# Only the execution ID becomes a filesystem path component.
report_storage_path_component_regex: "^[A-Za-z0-9._-]{1,128}$"


# Environments allowed in deployed AAP jobs.
report_storage_deployed_environments:
  - dev
  - rnd
  - qa
  - prod

# test/local are permitted only when report_storage_test_mode=true.
report_storage_test_environments:
  - test
  - local

# Hidden probe used only to prove write/read/list access.
report_storage_probe_prefix: ".akamai_report_probe"

# common_logging stages.
report_storage_log_stages:
  context: "report_storage_context"
  validation: "report_storage_validation"
  provisioning: "report_storage_provisioning"
  write: "report_storage_write"
  read: "report_storage_read"
  list: "report_storage_list"
  check_mode: "report_storage_check_mode"
  publish: "report_storage_publish"
  cleanup: "report_storage_cleanup"

# Stable error keys. Numeric code mapping remains owned by common_logging.
report_storage_error_keys:
  context_invalid: "REPORT_STORAGE_CONTEXT_INVALID"
  config_invalid: "REPORT_STORAGE_CONFIG_INVALID"
  access_group_invalid: "REPORT_ACCESS_GROUP_INVALID"
  path_invalid: "REPORT_STORAGE_PATH_INVALID"
  storage_unavailable: "REPORT_STORAGE_UNAVAILABLE"
  write_failed: "REPORT_STORAGE_WRITE_FAILED"
  read_failed: "REPORT_STORAGE_READ_FAILED"
  list_failed: "REPORT_STORAGE_LIST_FAILED"
  cleanup_failed: "REPORT_STORAGE_CLEANUP_FAILED"
```

---

# `roles/akamai_version_push/tasks/report_storage.yml`

```yaml
---
# =============================================================================
# Temporary report storage entry point
#
# This task file is intentionally callable only through:
#
#   include_role:
#     name: akamai_version_push
#     tasks_from: report_storage
#
# It is NOT imported from the role's default tasks/main.yml entry point.
# =============================================================================

- name: "Report Storage | Resolve shared execution context"
  ansible.builtin.import_tasks: report_storage_context.yml

- name: "Report Storage | Validate configuration and path contract"
  ansible.builtin.import_tasks: report_storage_validate.yml

- name: "Report Storage | Provision temporary workspace"
  ansible.builtin.import_tasks: report_storage_provision.yml

- name: "Report Storage | Verify temporary filesystem I/O"
  ansible.builtin.import_tasks: report_storage_verify.yml

- name: "Report Storage | Publish storage metadata"
  ansible.builtin.import_tasks: report_storage_publish.yml
```

---

# `roles/akamai_version_push/tasks/report_storage_context.yml`

```yaml
---
# =============================================================================
# Resolve logging/execution context once and reuse it everywhere.
# =============================================================================

- name: "Report Storage | Resolve context timestamp"
  ansible.builtin.set_fact:
    report_storage_runtime_timestamp: >-
      {{ now(utc=true, fmt='%Y%m%dT%H%M%SZ') }}


- name: "Report Storage | Resolve shared execution context"
  ansible.builtin.set_fact:
    report_storage_context:
      timestamp_utc: "{{ report_storage_runtime_timestamp }}"

      request_id: >-
        {{
          (common_logging_request_id | default('', true) | string | trim)
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
            'REPORT-' ~ report_storage_runtime_timestamp
            if report_storage_test_mode | bool
            else ''
          )
        }}

      execution_id: >-
        {{
          (common_logging_execution_id | default('', true) | string | trim)
          if
          (
            common_logging_execution_id
            | default('', true)
            | string
            | trim
            | length > 0
          )
          else
          (
            (awx_job_id | default('', true) | string | trim)
            if
            (
              awx_job_id
              | default('', true)
              | string
              | trim
              | length > 0
            )
            else
            (
              'LOCAL-' ~ report_storage_runtime_timestamp
              if report_storage_test_mode | bool
              else ''
            )
          )
        }}

      actor: "{{ report_storage_module }}"

      triggered_by: >-
        {{
          (awx_user_email | default('', true) | string | trim)
          if
          (
            awx_user_email
            | default('', true)
            | string
            | trim
            | length > 0
          )
          else
          (
            (awx_user_name | default('', true) | string | trim)
            if
            (
              awx_user_name
              | default('', true)
              | string
              | trim
              | length > 0
            )
            else
            (
              'local-test'
              if report_storage_test_mode | bool
              else ''
            )
          )
        }}

      environment: >-
        {{
          (common_logging_environment | default('', true) | string | trim | lower)
          if
          (
            common_logging_environment
            | default('', true)
            | string
            | trim
            | length > 0
          )
          else
          (
            (
              (akamai | default({}))
              .get('environment', {})
              .get('name', '')
              | string
              | trim
              | lower
            )
            if
            (
              (akamai | default({}))
              .get('environment', {})
              .get('name', '')
              | string
              | trim
              | length > 0
            )
            else
            (
              'local'
              if report_storage_test_mode | bool
              else ''
            )
          )
        }}

      test_mode: "{{ report_storage_test_mode | bool }}"


# This is bootstrap validation. common_logging is not invoked until the context
# itself is valid, otherwise the logger would receive invalid correlation data.
- name: "Report Storage | Validate shared execution context"
  ansible.builtin.assert:
    that:
      - report_storage_context.request_id | length > 0
      - report_storage_context.request_id | length <= 256
      - report_storage_context.execution_id | length > 0
      - report_storage_context.execution_id is match(report_storage_path_component_regex)
      - report_storage_context.actor == report_storage_module
      - report_storage_context.triggered_by | length > 0
      - >-
        (
          report_storage_context.environment
          in report_storage_deployed_environments
        )
        or
        (
          report_storage_test_mode | bool
          and
          report_storage_context.environment
          in report_storage_test_environments
        )
    fail_msg: >-
      Invalid temporary-storage execution context. Deployed AAP jobs require
      a shared common_logging_request_id, a shared/AAP execution ID, the AAP
      launcher identity, and environment dev/rnd/qa/prod. test/local and local
      request/execution fallbacks are allowed only when
      report_storage_test_mode=true.
    quiet: true


- name: "Report Storage | Log validated execution context"
  ansible.builtin.include_role:
    name: common_logging
    tasks_from: log_event
  vars:
    common_logging_request_id: "{{ report_storage_context.request_id }}"
    common_logging_execution_id: "{{ report_storage_context.execution_id }}"
    common_logging_environment: "{{ report_storage_context.environment }}"
    common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
    common_logging_actor: "{{ report_storage_context.actor }}"
    common_logging_stage: "{{ report_storage_log_stages.context }}"
    common_logging_result: "success"
    common_logging_message: >-
      Temporary report-storage execution context validated.
    common_logging_version_info: {}
```

---

# `roles/akamai_version_push/tasks/report_storage_validate.yml`

```yaml
---
# =============================================================================
# Validate configuration, permissions, protected inputs, and path containment.
# =============================================================================

- name: "Report Storage | Initialize validation state"
  ansible.builtin.set_fact:
    report_storage_configuration_validated: false
    report_storage_storage_verified: false
    report_storage_report_exists: false


- name: "Report Storage | Resolve effective permissions"
  ansible.builtin.set_fact:
    report_storage_effective_file_mode: >-
      {{
        report_storage_group_file_mode
        if report_storage_access_group | default('') | string | trim | length > 0
        else report_storage_private_file_mode
      }}

    report_storage_effective_directory_mode: >-
      {{
        report_storage_group_directory_mode
        if report_storage_access_group | default('') | string | trim | length > 0
        else report_storage_private_directory_mode
      }}


- name: "Report Storage | Resolve protected workspace paths"
  ansible.builtin.set_fact:
    report_storage_execution_dir: >-
      {{ report_storage_root }}/{{ report_storage_context.execution_id }}

    report_storage_probe_filename: >-
      {{ report_storage_probe_prefix }}_{{ report_storage_context.timestamp_utc }}.tmp

    report_storage_probe_content: >-
      REPORT_STORAGE_VERIFY_{{ report_storage_context.execution_id }}

    report_storage_probe_path: >-
      {{ report_storage_root }}/{{ report_storage_context.execution_id }}/{{ report_storage_probe_prefix }}_{{ report_storage_context.timestamp_utc }}.tmp


- name: "Report Storage | Validate storage configuration"
  block:

    - name: "Report Storage | Validate approved roots"
      ansible.builtin.assert:
        that:
          - report_storage_base_root == '/tmp'
          - report_storage_root == '/tmp/akamai-reports'
        fail_msg: >-
          Temporary report storage must remain under the approved
          /tmp/akamai-reports root.
        quiet: true


    - name: "Report Storage | Validate secure permission contract"
      ansible.builtin.assert:
        that:
          - report_storage_private_file_mode == '0600'
          - report_storage_group_file_mode == '0640'
          - report_storage_private_directory_mode == '0700'
          - report_storage_group_directory_mode == '0750'
          - report_storage_effective_file_mode in ['0600', '0640']
          - report_storage_effective_directory_mode in ['0700', '0750']
        fail_msg: >-
          Temporary report-storage permissions must use file modes
          0600/0640 and directory modes 0700/0750.
        quiet: true


    - name: "Report Storage | Validate optional protected access group"
      ansible.builtin.assert:
        that:
          - >-
            (
              report_storage_access_group
              | default('')
              | string
              | trim
              | length == 0
            )
            or
            (
              report_storage_access_group
              is match('^[A-Za-z_][A-Za-z0-9_.-]{0,63}$')
            )
        fail_msg: >-
          report_storage_access_group must be empty or a valid protected OS
          group name. It must be sourced from protected inventory or fixed
          Job Template configuration, not Survey/Prompt on Launch.
        quiet: true


    - name: "Report Storage | Validate path-component safety"
      ansible.builtin.assert:
        that:
          - report_storage_context.execution_id is match(report_storage_path_component_regex)
          - "'/' not in report_storage_context.execution_id"
          - "'\\\\' not in report_storage_context.execution_id"
          - "'..' not in report_storage_context.execution_id"
          - >-
            report_storage_execution_dir
            is match('^/tmp/akamai-reports/[A-Za-z0-9._-]{1,128}$')
          - >-
            report_storage_probe_filename
            is match('^\\.akamai_report_probe_[0-9]{8}T[0-9]{6}Z\\.tmp$')
          - >-
            report_storage_probe_path
            == report_storage_execution_dir ~ '/' ~ report_storage_probe_filename
        fail_msg: >-
          Unsafe temporary-storage path component detected. Path components
          may contain only letters, numbers, period, underscore, and hyphen,
          and all resolved paths must remain under /tmp/akamai-reports.
        quiet: true


    - name: "Report Storage | Mark configuration validated"
      ansible.builtin.set_fact:
        report_storage_configuration_validated: true


    - name: "Report Storage | Log successful configuration validation"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: log_event
      vars:
        common_logging_request_id: "{{ report_storage_context.request_id }}"
        common_logging_execution_id: "{{ report_storage_context.execution_id }}"
        common_logging_environment: "{{ report_storage_context.environment }}"
        common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
        common_logging_actor: "{{ report_storage_context.actor }}"
        common_logging_stage: "{{ report_storage_log_stages.validation }}"
        common_logging_result: "success"
        common_logging_message: >-
          Temporary report-storage configuration and path contract validated.
        common_logging_version_info: {}


  rescue:

    - name: "Report Storage | Stop on configuration validation failure"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: "{{ report_storage_context.request_id }}"
        common_logging_execution_id: "{{ report_storage_context.execution_id }}"
        common_logging_environment: "{{ report_storage_context.environment }}"
        common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
        common_logging_error_stage: "{{ report_storage_log_stages.validation }}"
        common_logging_error_actor: "{{ report_storage_context.actor }}"
        common_logging_error_category_input: "validation"
        common_logging_error_key_input: "{{ report_storage_error_keys.config_invalid }}"
        common_logging_error_message: >-
          Temporary report-storage configuration or path validation failed.
        common_logging_error_recommended_action: >-
          Verify protected environment/group configuration, secure modes,
          execution identifier safety, and approved temporary root settings.
        common_logging_error_version_info: {}
```

---

# `roles/akamai_version_push/tasks/report_storage_provision.yml`

```yaml
---
# =============================================================================
# Provision dedicated temporary workspace in normal mode.
# Check mode validates configuration only and performs no filesystem writes.
# =============================================================================

- name: "Report Storage | Initialize workspace state"
  ansible.builtin.set_fact:
    report_storage_workspace_exists: false


- name: "Report Storage | Validate configured protected access group"
  when:
    - report_storage_access_group | default('') | string | trim | length > 0
  block:

    - name: "Report Storage | Query configured OS group"
      ansible.builtin.getent:
        database: group
        key: "{{ report_storage_access_group }}"

  rescue:

    - name: "Report Storage | Stop when configured access group is unavailable"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: "{{ report_storage_context.request_id }}"
        common_logging_execution_id: "{{ report_storage_context.execution_id }}"
        common_logging_environment: "{{ report_storage_context.environment }}"
        common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
        common_logging_error_stage: "{{ report_storage_log_stages.validation }}"
        common_logging_error_actor: "{{ report_storage_context.actor }}"
        common_logging_error_category_input: "validation"
        common_logging_error_key_input: "{{ report_storage_error_keys.access_group_invalid }}"
        common_logging_error_message: >-
          The protected report-storage access group is not available inside
          the Execution Environment.
        common_logging_error_recommended_action: >-
          Configure an approved OS group that exists in the Execution
          Environment, or leave report_storage_access_group empty for
          execution-user-only access.
        common_logging_error_version_info: {}


- name: "Report Storage | Validate base temporary filesystem"
  block:

    - name: "Report Storage | Inspect /tmp"
      ansible.builtin.stat:
        path: "{{ report_storage_base_root }}"
      register: report_storage_base_stat


    - name: "Report Storage | Confirm /tmp is available"
      ansible.builtin.assert:
        that:
          - report_storage_base_stat.stat.exists
          - report_storage_base_stat.stat.isdir
        fail_msg: >-
          The /tmp filesystem is unavailable inside the Execution Environment.
        quiet: true

  rescue:

    - name: "Report Storage | Stop when temporary filesystem is unavailable"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: "{{ report_storage_context.request_id }}"
        common_logging_execution_id: "{{ report_storage_context.execution_id }}"
        common_logging_environment: "{{ report_storage_context.environment }}"
        common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
        common_logging_error_stage: "{{ report_storage_log_stages.provisioning }}"
        common_logging_error_actor: "{{ report_storage_context.actor }}"
        common_logging_error_category_input: "infrastructure"
        common_logging_error_key_input: "{{ report_storage_error_keys.storage_unavailable }}"
        common_logging_error_message: >-
          The Execution Environment temporary filesystem is unavailable.
        common_logging_error_recommended_action: >-
          Verify that /tmp exists and is accessible to the AAP Execution
          Environment user.
        common_logging_error_version_info: {}


- name: "Report Storage | Provision execution workspace"
  when:
    - not ansible_check_mode
  block:

    - name: "Report Storage | Create application temporary root"
      ansible.builtin.file:
        path: "{{ report_storage_root }}"
        state: directory
        group: >-
          {{
            report_storage_access_group
            if report_storage_access_group | default('') | string | trim | length > 0
            else omit
          }}
        mode: "{{ report_storage_effective_directory_mode }}"


    - name: "Report Storage | Create per-execution temporary directory"
      ansible.builtin.file:
        path: "{{ report_storage_execution_dir }}"
        state: directory
        group: >-
          {{
            report_storage_access_group
            if report_storage_access_group | default('') | string | trim | length > 0
            else omit
          }}
        mode: "{{ report_storage_effective_directory_mode }}"


    - name: "Report Storage | Inspect per-execution temporary directory"
      ansible.builtin.stat:
        path: "{{ report_storage_execution_dir }}"
      register: report_storage_execution_dir_stat


    - name: "Report Storage | Verify directory mode and group ownership"
      ansible.builtin.assert:
        that:
          - report_storage_execution_dir_stat.stat.exists
          - report_storage_execution_dir_stat.stat.isdir
          - >-
            report_storage_execution_dir_stat.stat.mode
            == report_storage_effective_directory_mode
          - >-
            (
              report_storage_access_group
              | default('')
              | string
              | trim
              | length == 0
            )
            or
            (
              report_storage_execution_dir_stat.stat.gr_name
              == report_storage_access_group
            )
        fail_msg: >-
          Temporary report directory does not have the expected secure mode
          or approved group ownership.
        quiet: true


    - name: "Report Storage | Mark workspace available"
      ansible.builtin.set_fact:
        report_storage_workspace_exists: true


    - name: "Report Storage | Log workspace provisioning success"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: log_event
      vars:
        common_logging_request_id: "{{ report_storage_context.request_id }}"
        common_logging_execution_id: "{{ report_storage_context.execution_id }}"
        common_logging_environment: "{{ report_storage_context.environment }}"
        common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
        common_logging_actor: "{{ report_storage_context.actor }}"
        common_logging_stage: "{{ report_storage_log_stages.provisioning }}"
        common_logging_result: "success"
        common_logging_message: >-
          Dedicated temporary report workspace provisioned successfully.
        common_logging_version_info: {}

  rescue:

    - name: "Report Storage | Stop on workspace provisioning failure"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: "{{ report_storage_context.request_id }}"
        common_logging_execution_id: "{{ report_storage_context.execution_id }}"
        common_logging_environment: "{{ report_storage_context.environment }}"
        common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
        common_logging_error_stage: "{{ report_storage_log_stages.provisioning }}"
        common_logging_error_actor: "{{ report_storage_context.actor }}"
        common_logging_error_category_input: "infrastructure"
        common_logging_error_key_input: "{{ report_storage_error_keys.storage_unavailable }}"
        common_logging_error_message: >-
          Temporary report workspace could not be provisioned with the
          required permissions and ownership.
        common_logging_error_recommended_action: >-
          Verify /tmp access, approved OS group availability, and Execution
          Environment filesystem permissions.
        common_logging_error_version_info: {}
```

---

# `roles/akamai_version_push/tasks/report_storage_verify.yml`

```yaml
---
# =============================================================================
# Verify filesystem I/O.
#
# Check mode:
#   configuration_validated=true
#   storage_verified=false
#   no write/read/list operations
#
# Normal mode:
#   storage_verified becomes true only after write, read, and list checks pass.
# =============================================================================

- name: "Report Storage | Handle check-mode semantics"
  when:
    - ansible_check_mode
  block:

    - name: "Report Storage | Keep storage verification false in check mode"
      ansible.builtin.set_fact:
        report_storage_storage_verified: false
        report_storage_report_exists: false


    - name: "Report Storage | Log check-mode validation"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: log_event
      vars:
        common_logging_request_id: "{{ report_storage_context.request_id }}"
        common_logging_execution_id: "{{ report_storage_context.execution_id }}"
        common_logging_environment: "{{ report_storage_context.environment }}"
        common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
        common_logging_actor: "{{ report_storage_context.actor }}"
        common_logging_stage: "{{ report_storage_log_stages.check_mode }}"
        common_logging_result: "success"
        common_logging_message: >-
          Temporary report-storage configuration validated in Check Mode;
          filesystem I/O verification was intentionally not performed.
        common_logging_version_info: {}


- name: "Report Storage | Verify write/read/list contract"
  when:
    - not ansible_check_mode
    - report_storage_verify_io | bool
  block:

    # -------------------------------------------------------------------------
    # WRITE
    # -------------------------------------------------------------------------

    - name: "Report Storage | Verify write capability"
      block:

        - name: "Report Storage | Write hidden verification artifact"
          ansible.builtin.copy:
            dest: "{{ report_storage_probe_path }}"
            content: |
              {{ report_storage_probe_content }}
            group: >-
              {{
                report_storage_access_group
                if report_storage_access_group | default('') | string | trim | length > 0
                else omit
              }}
            mode: "{{ report_storage_effective_file_mode }}"


        - name: "Report Storage | Inspect verification artifact"
          ansible.builtin.stat:
            path: "{{ report_storage_probe_path }}"
          register: report_storage_probe_stat


        - name: "Report Storage | Verify artifact mode and approved group"
          ansible.builtin.assert:
            that:
              - report_storage_probe_stat.stat.exists
              - report_storage_probe_stat.stat.isreg
              - >-
                report_storage_probe_stat.stat.mode
                == report_storage_effective_file_mode
              - >-
                (
                  report_storage_access_group
                  | default('')
                  | string
                  | trim
                  | length == 0
                )
                or
                (
                  report_storage_probe_stat.stat.gr_name
                  == report_storage_access_group
                )
            fail_msg: >-
              Verification artifact was not created with the expected file
              mode or approved group ownership.
            quiet: true


        - name: "Report Storage | Log write verification success"
          ansible.builtin.include_role:
            name: common_logging
            tasks_from: log_event
          vars:
            common_logging_request_id: "{{ report_storage_context.request_id }}"
            common_logging_execution_id: "{{ report_storage_context.execution_id }}"
            common_logging_environment: "{{ report_storage_context.environment }}"
            common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
            common_logging_actor: "{{ report_storage_context.actor }}"
            common_logging_stage: "{{ report_storage_log_stages.write }}"
            common_logging_result: "success"
            common_logging_message: >-
              Temporary report-storage write verification passed.
            common_logging_version_info: {}

      rescue:

        - name: "Report Storage | Stop on write verification failure"
          ansible.builtin.include_role:
            name: common_logging
            tasks_from: fail_with_error
          vars:
            common_logging_request_id: "{{ report_storage_context.request_id }}"
            common_logging_execution_id: "{{ report_storage_context.execution_id }}"
            common_logging_environment: "{{ report_storage_context.environment }}"
            common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
            common_logging_error_stage: "{{ report_storage_log_stages.write }}"
            common_logging_error_actor: "{{ report_storage_context.actor }}"
            common_logging_error_category_input: "infrastructure"
            common_logging_error_key_input: "{{ report_storage_error_keys.write_failed }}"
            common_logging_error_message: >-
              Temporary report-storage write verification failed.
            common_logging_error_recommended_action: >-
              Verify Execution Environment write access and the configured
              directory/file permission contract.
            common_logging_error_version_info: {}


    # -------------------------------------------------------------------------
    # READ
    # -------------------------------------------------------------------------

    - name: "Report Storage | Verify read capability"
      block:

        - name: "Report Storage | Read verification artifact"
          ansible.builtin.slurp:
            src: "{{ report_storage_probe_path }}"
          register: report_storage_probe_read


        - name: "Report Storage | Verify artifact content"
          ansible.builtin.assert:
            that:
              - >-
                (
                  report_storage_probe_read.content
                  | b64decode
                  | trim
                )
                == report_storage_probe_content
            fail_msg: >-
              Temporary report-storage read content did not match the value
              written by the verification step.
            quiet: true


        - name: "Report Storage | Log read verification success"
          ansible.builtin.include_role:
            name: common_logging
            tasks_from: log_event
          vars:
            common_logging_request_id: "{{ report_storage_context.request_id }}"
            common_logging_execution_id: "{{ report_storage_context.execution_id }}"
            common_logging_environment: "{{ report_storage_context.environment }}"
            common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
            common_logging_actor: "{{ report_storage_context.actor }}"
            common_logging_stage: "{{ report_storage_log_stages.read }}"
            common_logging_result: "success"
            common_logging_message: >-
              Temporary report-storage read verification passed.
            common_logging_version_info: {}

      rescue:

        - name: "Report Storage | Stop on read verification failure"
          ansible.builtin.include_role:
            name: common_logging
            tasks_from: fail_with_error
          vars:
            common_logging_request_id: "{{ report_storage_context.request_id }}"
            common_logging_execution_id: "{{ report_storage_context.execution_id }}"
            common_logging_environment: "{{ report_storage_context.environment }}"
            common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
            common_logging_error_stage: "{{ report_storage_log_stages.read }}"
            common_logging_error_actor: "{{ report_storage_context.actor }}"
            common_logging_error_category_input: "infrastructure"
            common_logging_error_key_input: "{{ report_storage_error_keys.read_failed }}"
            common_logging_error_message: >-
              Temporary report-storage read verification failed.
            common_logging_error_recommended_action: >-
              Verify Execution Environment read access and file integrity.
            common_logging_error_version_info: {}


    # -------------------------------------------------------------------------
    # LIST
    # -------------------------------------------------------------------------

    - name: "Report Storage | Verify list capability"
      block:

        - name: "Report Storage | Locate hidden verification artifact"
          ansible.builtin.find:
            paths: "{{ report_storage_execution_dir }}"
            patterns:
              - "{{ report_storage_probe_filename }}"
            file_type: file
            recurse: false
            hidden: true
          register: report_storage_probe_list


        - name: "Report Storage | Verify list result"
          ansible.builtin.assert:
            that:
              - report_storage_probe_list.matched | int == 1
            fail_msg: >-
              Temporary report-storage verification artifact could not be
              listed from the execution workspace.
            quiet: true


        - name: "Report Storage | Log list verification success"
          ansible.builtin.include_role:
            name: common_logging
            tasks_from: log_event
          vars:
            common_logging_request_id: "{{ report_storage_context.request_id }}"
            common_logging_execution_id: "{{ report_storage_context.execution_id }}"
            common_logging_environment: "{{ report_storage_context.environment }}"
            common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
            common_logging_actor: "{{ report_storage_context.actor }}"
            common_logging_stage: "{{ report_storage_log_stages.list }}"
            common_logging_result: "success"
            common_logging_message: >-
              Temporary report-storage list verification passed.
            common_logging_version_info: {}

      rescue:

        - name: "Report Storage | Stop on list verification failure"
          ansible.builtin.include_role:
            name: common_logging
            tasks_from: fail_with_error
          vars:
            common_logging_request_id: "{{ report_storage_context.request_id }}"
            common_logging_execution_id: "{{ report_storage_context.execution_id }}"
            common_logging_environment: "{{ report_storage_context.environment }}"
            common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
            common_logging_error_stage: "{{ report_storage_log_stages.list }}"
            common_logging_error_actor: "{{ report_storage_context.actor }}"
            common_logging_error_category_input: "infrastructure"
            common_logging_error_key_input: "{{ report_storage_error_keys.list_failed }}"
            common_logging_error_message: >-
              Temporary report-storage list verification failed.
            common_logging_error_recommended_action: >-
              Verify Execution Environment directory access and hidden-file
              listing behavior.
            common_logging_error_version_info: {}


    - name: "Report Storage | Mark real filesystem verification successful"
      ansible.builtin.set_fact:
        report_storage_storage_verified: true
        report_storage_report_exists: false

  always:

    - name: "Report Storage | Remove verification artifact"
      ansible.builtin.file:
        path: "{{ report_storage_probe_path }}"
        state: absent
      failed_when: false


- name: "Report Storage | Record skipped I/O verification"
  when:
    - not ansible_check_mode
    - not (report_storage_verify_io | bool)
  block:

    - name: "Report Storage | Keep storage verification false"
      ansible.builtin.set_fact:
        report_storage_storage_verified: false
        report_storage_report_exists: false


    - name: "Report Storage | Log skipped I/O verification"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: log_event
      vars:
        common_logging_request_id: "{{ report_storage_context.request_id }}"
        common_logging_execution_id: "{{ report_storage_context.execution_id }}"
        common_logging_environment: "{{ report_storage_context.environment }}"
        common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
        common_logging_actor: "{{ report_storage_context.actor }}"
        common_logging_stage: "{{ report_storage_log_stages.validation }}"
        common_logging_result: "skipped"
        common_logging_message: >-
          Temporary report-storage filesystem I/O verification was disabled;
          storage_verified remains false.
        common_logging_version_info: {}
```

---

# `roles/akamai_version_push/tasks/report_storage_publish.yml`

```yaml
---
# =============================================================================
# Publish temporary-storage metadata.
#
# SSPA-7 owns temporary storage only. It does not claim that a deployment
# report exists and it does not define final report/audit filenames.
# =============================================================================

- name: "Report Storage | Build published storage metadata"
  ansible.builtin.set_fact:
    report_storage_metadata:
      base_root: "{{ report_storage_base_root }}"
      root: "{{ report_storage_root }}"
      workspace_path: "{{ report_storage_execution_dir }}"
      workspace_exists: "{{ report_storage_workspace_exists | bool }}"
      configuration_validated: "{{ report_storage_configuration_validated | bool }}"
      storage_verified: "{{ report_storage_storage_verified | bool }}"
      report_exists: false
      file_mode: "{{ report_storage_effective_file_mode }}"
      directory_mode: "{{ report_storage_effective_directory_mode }}"
      access_group: "{{ report_storage_access_group | default('') | string | trim }}"
      ephemeral: true
      same_job_only: true
      separate_job_transfer_required: true
      cleanup_required: true
      cleanup_task: "report_storage_cleanup"
      final_report_naming_owner: "downstream_report_generation"
      audit_filename_owner: "common_logging"
      request_id: "{{ report_storage_context.request_id }}"
      execution_id: "{{ report_storage_context.execution_id }}"
      environment: "{{ report_storage_context.environment }}"
      checked_at_utc: "{{ report_storage_context.timestamp_utc }}"
      check_mode: "{{ ansible_check_mode | bool }}"


- name: "Report Storage | Publish metadata to AAP"
  ansible.builtin.set_stats:
    data:
      report_storage: "{{ report_storage_metadata }}"
    per_host: false


- name: "Report Storage | Log successful metadata publication"
  ansible.builtin.include_role:
    name: common_logging
    tasks_from: log_event
  vars:
    common_logging_request_id: "{{ report_storage_context.request_id }}"
    common_logging_execution_id: "{{ report_storage_context.execution_id }}"
    common_logging_environment: "{{ report_storage_context.environment }}"
    common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
    common_logging_actor: "{{ report_storage_context.actor }}"
    common_logging_stage: "{{ report_storage_log_stages.publish }}"
    common_logging_result: "success"
    common_logging_message: >-
      Temporary report-storage metadata published. configuration_validated={{
      report_storage_configuration_validated | bool }}, storage_verified={{
      report_storage_storage_verified | bool }}, report_exists=false.
    common_logging_version_info: {}
```

---

# `roles/akamai_version_push/tasks/report_storage_cleanup.yml`

```yaml
---
# =============================================================================
# Temporary report workspace cleanup
#
# Call this task ONLY after every same-job consumer has finished reading or
# delivering the temporary report. In the integrated workflow, call it from an
# outer `always` section after report generation/email consumption completes.
# =============================================================================

- name: "Report Storage Cleanup | Skip filesystem cleanup in Check Mode"
  ansible.builtin.debug:
    msg: >-
      Check Mode: temporary report workspace cleanup was not required.
  when:
    - ansible_check_mode


- name: "Report Storage Cleanup | Validate cleanup target"
  when:
    - not ansible_check_mode
    - report_storage_execution_dir is defined
  block:

    - name: "Report Storage Cleanup | Verify cleanup path is contained"
      ansible.builtin.assert:
        that:
          - report_storage_context is defined
          - report_storage_context.execution_id is match(report_storage_path_component_regex)
          - >-
            report_storage_execution_dir
            == report_storage_root ~ '/' ~ report_storage_context.execution_id
          - >-
            report_storage_execution_dir
            is match('^/tmp/akamai-reports/[A-Za-z0-9._-]{1,128}$')
        fail_msg: >-
          Refusing cleanup because the target is outside the approved
          per-execution temporary report workspace.
        quiet: true


    - name: "Report Storage Cleanup | Remove per-execution workspace"
      ansible.builtin.file:
        path: "{{ report_storage_execution_dir }}"
        state: absent


    - name: "Report Storage Cleanup | Log cleanup success"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: log_event
      vars:
        common_logging_request_id: "{{ report_storage_context.request_id }}"
        common_logging_execution_id: "{{ report_storage_context.execution_id }}"
        common_logging_environment: "{{ report_storage_context.environment }}"
        common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
        common_logging_actor: "{{ report_storage_context.actor }}"
        common_logging_stage: "{{ report_storage_log_stages.cleanup }}"
        common_logging_result: "success"
        common_logging_message: >-
          Per-execution temporary report workspace removed after all same-job
          consumers completed.
        common_logging_version_info: {}

  rescue:

    - name: "Report Storage Cleanup | Stop on cleanup failure"
      ansible.builtin.include_role:
        name: common_logging
        tasks_from: fail_with_error
      vars:
        common_logging_request_id: "{{ report_storage_context.request_id }}"
        common_logging_execution_id: "{{ report_storage_context.execution_id }}"
        common_logging_environment: "{{ report_storage_context.environment }}"
        common_logging_triggered_by: "{{ report_storage_context.triggered_by }}"
        common_logging_error_stage: "{{ report_storage_log_stages.cleanup }}"
        common_logging_error_actor: "{{ report_storage_context.actor }}"
        common_logging_error_category_input: "infrastructure"
        common_logging_error_key_input: "{{ report_storage_error_keys.cleanup_failed }}"
        common_logging_error_message: >-
          Temporary report workspace cleanup failed.
        common_logging_error_recommended_action: >-
          Verify the cleanup target, Execution Environment permissions, and
          that all same-job consumers have completed before cleanup runs.
        common_logging_error_version_info: {}
```

---

# `playbooks/akamai_version_push/report_storage.yml`

```yaml
---
- name: "Validate temporary Akamai report storage"
  hosts: localhost
  connection: local
  gather_facts: false
  any_errors_fatal: true

  tasks:

    - name: "Report Storage | Run standalone storage validation"
      block:

        - name: "Report Storage | Validate temporary storage contract"
          ansible.builtin.include_role:
            name: akamai_version_push
            tasks_from: report_storage

      always:

        # Standalone validation has no downstream report consumer, so the
        # per-execution workspace can be removed immediately after validation.
        - name: "Report Storage | Cleanup standalone validation workspace"
          ansible.builtin.include_role:
            name: akamai_version_push
            tasks_from: report_storage_cleanup
          when:
            - report_storage_context is defined
            - report_storage_execution_dir is defined

        # Audit filename/content remains owned by common_logging.
        - name: "Report Storage | Render common audit when events exist"
          ansible.builtin.include_role:
            name: common_logging
            tasks_from: render_audit
          when:
            - common_logging_history | default([]) | length > 0
```

---

# `docs/report_storage.md`

# Temporary Report Storage Integration Contract

## Scope

This implementation provides temporary report storage only. It does not retrieve ServiceNow data, update CTASK records, generate deployment-report content, define deployment-report filenames, send email, or provide persistent retention.

The temporary storage is local to the active AAP Execution Environment.

## PR Scope

The SSPA-7 PR must contain only temporary report-storage changes. Remove ServiceNow and CTASK playbooks/tasks and revert any ServiceNow collection-version changes unless the storage code itself directly requires them.

The role default entry point must not automatically import report storage, ServiceNow validation, or CTASK work. Call temporary storage explicitly with:

```yaml
- name: Validate temporary report storage
  ansible.builtin.include_role:
    name: akamai_version_push
    tasks_from: report_storage
```

## Storage Lifecycle

The workspace is created under:

```text
/tmp/akamai-reports/<execution_id>/
```

The file system is ephemeral.

A file created in this workspace is available only within the current AAP Execution Environment. Later tasks in the same AAP job can read it directly. A separate AAP workflow job/container must not assume the same `/tmp` file exists; a separate job requires an approved transfer/persistence mechanism.

## Context Contract

For deployed AAP jobs:

- `common_logging_request_id` must be supplied by the shared workflow context.
- `common_logging_execution_id` should be the shared execution ID; `awx_job_id` is used when the shared execution ID is not supplied.
- `common_logging_triggered_by` is not synthesized from the automation component. The implementation uses only the AAP launcher identity (`awx_user_email`, then `awx_user_name`).
- `common_logging_actor` and `common_logging_error_actor` are always `akamai_version_push.report_storage`.
- Environment must resolve to `dev`, `rnd`, `qa`, or `prod`.
- Environment must come from protected inventory or fixed Job Template configuration. Do not expose it through Survey or Prompt on Launch.

For explicit standalone tests only:

```yaml
report_storage_test_mode: true
```

This switch must not be exposed through Survey/Prompt on Launch and must remain false in deployed templates. It allows `test`/`local` environment and local request/execution fallbacks.

## Protected Access Group

`report_storage_access_group` is optional. When configured, it must be an approved OS group sourced from protected inventory or fixed Job Template configuration. Do not expose it through Survey or Prompt on Launch.

The role verifies:

- group existence in the Execution Environment;
- execution-directory group ownership;
- verification-file group ownership;
- directory mode `0750`;
- file mode `0640`.

Without a group, the workspace uses:

- directory mode `0700`;
- file mode `0600`.

## Path Security

Only the execution ID becomes a path component. It is restricted to:

```text
A-Z a-z 0-9 . _ -
```

The role rejects `/`, `\`, `..`, and any resolved path outside:

```text
/tmp/akamai-reports/
```

Request IDs are correlation identifiers and are not used as filesystem path components.

## Check Mode

Check Mode validates configuration and context only. It does not create directories or perform write/read/list checks.

Published semantics:

```yaml
configuration_validated: true
storage_verified: false
report_exists: false
check_mode: true
```

## Normal Run

`storage_verified` becomes `true` only after all real filesystem checks pass:

1. write hidden probe;
2. verify secure mode/group ownership;
3. read probe and verify content;
4. list probe with `hidden: true`;
5. remove the probe.

If active I/O verification is disabled, `storage_verified` remains `false`.

## Published Metadata

SSPA-7 publishes storage capability, not a generated deployment report:

```yaml
report_storage:
  base_root: /tmp
  root: /tmp/akamai-reports
  workspace_path: /tmp/akamai-reports/<execution_id>
  workspace_exists: true
  configuration_validated: true
  storage_verified: true
  report_exists: false
  file_mode: "0600"
  directory_mode: "0700"
  access_group: ""
  ephemeral: true
  same_job_only: true
  separate_job_transfer_required: true
  cleanup_required: true
  cleanup_task: report_storage_cleanup
  final_report_naming_owner: downstream_report_generation
  audit_filename_owner: common_logging
```

`report_exists` is `false` because SSPA-7 does not create the final report.

## Naming Ownership

Audit filenames are owned by `common_logging`/the common-logging story. SSPA-7 does not define audit filenames.

Final deployment-report naming and content are owned by the downstream report-generation story. SSPA-7 provides only the secure temporary workspace.

## Cleanup Ownership and Timing

The verification probe is always removed by SSPA-7.

The final temporary report/workspace must be removed only after all same-job consumers finish reading/delivering it. Downstream orchestration should call:

```yaml
- name: Cleanup temporary report workspace
  ansible.builtin.include_role:
    name: akamai_version_push
    tasks_from: report_storage_cleanup
```

from an outer `always` section after report generation and delivery/consumption tasks complete.

Do not invoke cleanup immediately after storage validation in the integrated deployment flow, because that would remove the workspace before downstream consumers use it.

The standalone validation playbook has no downstream consumer, so it cleans up its workspace in its own `always` section.

## Common Logging

Success events use:

```text
common_logging/tasks/log_event.yml
```

Terminal storage failures use:

```text
common_logging/tasks/fail_with_error.yml
```

Configuration failures use category `validation`. Filesystem write/read/list/provisioning/cleanup failures use category `infrastructure`.

The standalone validation playbook calls `common_logging/tasks/render_audit.yml` in its `always` section when log history exists. Audit naming remains owned by common logging.

## Limitations

- No cross-job `/tmp` persistence.
- No retention policy.
- No final report content generation.
- No final report filename generation.
- No email delivery.
- No ServiceNow validation.
- No CTASK update.
- No ServiceNow collection dependency is required by this storage implementation.

---

# `docs/aap_qa_validation.md`

# AAP QA Validation Evidence Checklist

Run the corrected implementation in AAP QA before PR approval.

## Normal Run

Use a dedicated SSPA-7 Job Template that calls:

```yaml
ansible.builtin.include_role:
  name: akamai_version_push
  tasks_from: report_storage
```

Protected Job Template/inventory configuration must provide:

```yaml
common_logging_environment: qa
common_logging_request_id: <shared-workflow-request-id>
```

Do not expose environment or access group through Survey/Prompt on Launch.

Expected evidence:

```text
configuration_validated = true
storage_verified = true
report_exists = false
workspace_path = /tmp/akamai-reports/<execution_id>
file_mode = 0600 (or 0640 for approved group)
directory_mode = 0700 (or 0750 for approved group)
common_logging events emitted
verification probe removed
```

## Check Mode

Run the same Job Template with Job Type = Check.

Expected evidence:

```text
configuration_validated = true
storage_verified = false
report_exists = false
workspace_exists = false
no write/read/list probe operations
```

## Negative Tests

Capture evidence for at least:

1. Missing environment in deployed mode -> fail before storage work.
2. `local` environment without explicit test mode -> fail.
3. Missing shared request ID in deployed mode -> fail.
4. Unsafe execution ID containing `/`, `\\`, or `..` -> fail.
5. Invalid/nonexistent protected group -> validation failure.
6. Write/read/list filesystem failure -> infrastructure classification.

## PR Evidence

Attach or link:

- successful normal AAP QA job output;
- successful AAP Check Mode job output;
- one configuration-validation failure;
- one filesystem/infrastructure failure;
- common_logging/audit output;
- confirmation that ServiceNow/CTASK changes and ServiceNow collection changes are absent from the SSPA-7 PR.

---

# `tests/report_storage_contract.yml`

```yaml
---
- name: "Test temporary report storage contract"
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    report_storage_test_mode: true
    common_logging_environment: "local"
    common_logging_request_id: "REPORT-STORAGE-TEST-001"
    common_logging_execution_id: "TEST-EXEC-001"
    report_storage_verify_io: true
    report_storage_access_group: ""

  tasks:

    - name: "Report Storage Test | Execute and verify contract"
      block:

        - name: "Report Storage Test | Run storage validation"
          ansible.builtin.include_role:
            name: akamai_version_push
            tasks_from: report_storage


        - name: "Report Storage Test | Assert published semantics"
          ansible.builtin.assert:
            that:
              - report_storage_metadata.configuration_validated | bool
              - report_storage_metadata.storage_verified | bool
              - not (report_storage_metadata.report_exists | bool)
              - report_storage_metadata.workspace_exists | bool
              - report_storage_metadata.ephemeral | bool
              - report_storage_metadata.same_job_only | bool
              - report_storage_metadata.separate_job_transfer_required | bool
              - report_storage_metadata.file_mode == '0600'
              - report_storage_metadata.directory_mode == '0700'
              - >-
                report_storage_metadata.workspace_path
                == '/tmp/akamai-reports/TEST-EXEC-001'
            fail_msg: >-
              Temporary report-storage metadata contract did not match the
              required normal-run semantics.
            quiet: true


        - name: "Report Storage Test | Verify probe cleanup"
          ansible.builtin.find:
            paths: "{{ report_storage_execution_dir }}"
            patterns:
              - ".akamai_report_probe_*"
            file_type: file
            recurse: false
            hidden: true
          register: report_storage_test_probe_files


        - name: "Report Storage Test | Assert probe was removed"
          ansible.builtin.assert:
            that:
              - report_storage_test_probe_files.matched | int == 0
            fail_msg: >-
              Verification probe was not removed after filesystem validation.
            quiet: true


        - name: "Report Storage Test | Assert common logging events exist"
          ansible.builtin.assert:
            that:
              - common_logging_history is defined
              - common_logging_history | length > 0
            fail_msg: >-
              Temporary report-storage execution did not create common_logging
              event history.
            quiet: true

      always:

        - name: "Report Storage Test | Cleanup test workspace"
          ansible.builtin.include_role:
            name: akamai_version_push
            tasks_from: report_storage_cleanup
          when:
            - report_storage_execution_dir is defined


    - name: "Report Storage Test | Inspect workspace after cleanup"
      ansible.builtin.stat:
        path: "/tmp/akamai-reports/TEST-EXEC-001"
      register: report_storage_test_workspace_after_cleanup


    - name: "Report Storage Test | Assert execution workspace removed"
      ansible.builtin.assert:
        that:
          - not report_storage_test_workspace_after_cleanup.stat.exists
        fail_msg: >-
          Per-execution temporary workspace remained after explicit cleanup.
        quiet: true
```

---

# `tests/report_storage_check_mode.yml`

```yaml
---
# Run with:
#   ansible-playbook tests/report_storage_check_mode.yml --check

- name: "Test temporary report storage Check Mode semantics"
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    report_storage_test_mode: true
    common_logging_environment: "local"
    common_logging_request_id: "REPORT-STORAGE-CHECK-001"
    common_logging_execution_id: "TEST-CHECK-001"
    report_storage_verify_io: true
    report_storage_access_group: ""

  tasks:

    - name: "Report Storage Check Test | Require Ansible Check Mode"
      ansible.builtin.assert:
        that:
          - ansible_check_mode
        fail_msg: >-
          This test must be run using ansible-playbook --check or AAP Job Type
          Check.
        quiet: true


    - name: "Report Storage Check Test | Run storage validation"
      ansible.builtin.include_role:
        name: akamai_version_push
        tasks_from: report_storage


    - name: "Report Storage Check Test | Assert Check Mode metadata"
      ansible.builtin.assert:
        that:
          - report_storage_metadata.configuration_validated | bool
          - not (report_storage_metadata.storage_verified | bool)
          - not (report_storage_metadata.report_exists | bool)
          - not (report_storage_metadata.workspace_exists | bool)
          - report_storage_metadata.check_mode | bool
        fail_msg: >-
          Check Mode metadata must report configuration_validated=true,
          storage_verified=false, report_exists=false, and no created workspace.
        quiet: true
```
