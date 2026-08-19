# SSPA-7 — Minimal Common Logging Alignment Update

## Baseline

This update is based on the previously tested SSPA-7 implementation.

**No SSPA-7 functional behavior is changed.**

The following remain unchanged:

- Temporary storage root: `/tmp`
- `defaults/main.yml` ownership of overridable inputs/defaults
- `vars/main.yml` ownership of internal constants
- Write → read → list verification
- Private mode `0600`
- Optional group-readable mode `0640`
- Naming format `{timestamp}_{version}_{status}.txt`
- Check Mode behavior
- AAP `set_stats` output
- Ephemeral `/tmp` behavior
- No persistence, retention, report generation, or email logic

The existing `common_logging` integration already follows the current shared role contract. The only required runtime correction retained from AAP testing is `hidden: true` for the hidden probe file.

---

# 1. `roles/akamai_version_push/defaults/main.yml`

Keep the existing SSPA-7 defaults exactly as below:

```yaml
# Temporary report storage configuration
report_storage_verify_io: true

# Optional shared read access
report_storage_access_group: ""

# Secure permissions
report_storage_private_file_mode: "0600"
report_storage_group_file_mode: "0640"

# Optional naming inputs
report_storage_version: ""
report_storage_status: ""
```

These remain in `defaults/main.yml` because they are intentionally overridable inputs/defaults.

---

# 2. `roles/akamai_version_push/vars/main.yml`

Keep the existing internal constants:

```yaml
report_storage_root: "/tmp"

report_storage_filename_format: >-
  {timestamp}_{version}_{status}.txt

report_storage_filename_regex: >-
  ^[0-9]{8}T[0-9]{6}Z_[0-9]+_[a-z][a-z0-9_-]{0,31}\.txt$

report_storage_version_regex: >-
  ^[0-9]+$

report_storage_status_regex: >-
  ^[a-z][a-z0-9_-]{0,31}$

report_storage_probe_prefix: ".akamai_report_probe"

report_storage_log_stages:
  validation: "report_storage_validation"
  access: "report_storage_access"
  write: "report_storage_write"
  read: "report_storage_read"
  list: "report_storage_list"
  dry_run: "report_storage_dry_run"
  naming: "report_storage_naming"

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

These remain in `vars/main.yml` because they are internal implementation contracts and are not normal AAP inputs.

---

# 3. `roles/akamai_version_push/tasks/report_storage.yml`

Keep the existing tested task flow unchanged.

## Runtime/logging context

The existing context resolution remains valid:

```yaml
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
            | default(report_storage_runtime_timestamp, true),
            true
          )
        | string
      }}

    report_storage_log_actor: >-
      {{
        common_logging_actor
        | default(
            awx_user_email | default('automation', true),
            true
          )
        | string
      }}

    report_storage_log_triggered_by: >-
      {{
        common_logging_triggered_by
        | default(
            awx_user_email | default('automation', true),
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
```

This is intentionally self-contained and does not reference `common_logging_allowed_environments` before the shared role is loaded.

---

# 4. Required Production Fix — Hidden Probe Listing

The verification probe begins with:

```text
.akamai_report_probe
```

Therefore the list task **must** contain `hidden: true`.

Use:

```yaml
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
        hidden: true
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
        common_logging_request_id: "{{ report_storage_log_request_id }}"
        common_logging_execution_id: "{{ report_storage_log_execution_id }}"
        common_logging_environment: "{{ report_storage_log_environment }}"
        common_logging_triggered_by: "{{ report_storage_log_triggered_by }}"
        common_logging_error_stage: "{{ report_storage_log_stages.list }}"
        common_logging_error_actor: "{{ report_storage_log_actor }}"
        common_logging_error_category_input: "infrastructure"
        common_logging_error_key_input: "{{ report_storage_error_keys.list_failed }}"
        common_logging_error_message: >-
          Automation could not list the temporary report artifact
          under /tmp.
        common_logging_error_recommended_action: >-
          Verify /tmp access permissions inside the
          Execution Environment.
        common_logging_error_version_info: {}
```

This is the known AAP runtime fix. No other storage logic needs to change because write and read validation were already successful.

---

# 5. Current `common_logging` Alignment

## Terminal failures

Keep using:

```yaml
ansible.builtin.include_role:
  name: common_logging
  tasks_from: fail_with_error
```

with the existing contract:

```text
common_logging_request_id
common_logging_execution_id
common_logging_environment
common_logging_triggered_by
common_logging_error_stage
common_logging_error_actor
common_logging_error_category_input
common_logging_error_key_input
common_logging_error_message
common_logging_error_recommended_action
common_logging_error_version_info
```

## Success events

Keep using:

```yaml
ansible.builtin.include_role:
  name: common_logging
  tasks_from: log_event
```

with:

```text
common_logging_request_id
common_logging_execution_id
common_logging_environment
common_logging_triggered_by
common_logging_actor
common_logging_stage
common_logging_result
common_logging_message
common_logging_version_info
```

Keep:

```yaml
common_logging_version_info: {}
```

SSPA-7 does not need to add report-storage-specific fields into the shared restricted `version_info` map.

---

# 6. Error Contract — Unchanged

```text
REPORT_STORAGE_CONFIG_INVALID
REPORT_ACCESS_GROUP_INVALID
REPORT_STORAGE_UNAVAILABLE
REPORT_STORAGE_WRITE_FAILED
REPORT_STORAGE_READ_FAILED
REPORT_STORAGE_LIST_FAILED
REPORT_STORAGE_DRY_RUN_FAILED
REPORT_NAME_INVALID
```

`common_logging` owns the standardized numeric code mapping from the supplied error category.

---

# 7. AAP Artifact Contract — Unchanged

Successful normal execution continues to publish:

```yaml
report_storage:
  root: /tmp
  filename_format: "{timestamp}_{version}_{status}.txt"
  report_filename: "<timestamp>_252_success.txt"
  report_path: "/tmp/<timestamp>_252_success.txt"
  file_mode: "0600"
  ephemeral: true
  dry_run: false
```

If version/status are not both supplied:

```yaml
report_filename: ""
report_path: ""
```

Storage validation can still succeed.

---

# 8. Runtime Flow — Unchanged

```text
AAP Execution Environment starts
        ↓
Resolve runtime/logging context
        ↓
Resolve effective permissions
        ↓
Validate configuration
        ↓
Validate optional OS group
        ↓
Validate /tmp exists
        ↓
Check Mode?
   ├── YES → validate without probe I/O
   │
   └── NO
        ↓
Create hidden probe
        ↓
Verify WRITE
        ↓
Verify permissions
        ↓
Verify READ/content
        ↓
Verify LIST (hidden: true)
        ↓
Remove probe
        ↓
If version + status are both present
        ↓
Generate timestamp_version_status.txt
        ↓
Publish report_storage using set_stats
        ↓
Log successful validation
```

---

# 9. Standalone Playbook — No Change

`playbooks/akamai_version_push/report_storage.yml`

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

# 10. Role Dispatcher

Only retain/add this if the combined role flow should run the report-storage task:

```yaml
- name: "Prepare temporary report storage"
  ansible.builtin.import_tasks:
    report_storage.yml
```

Do not replace or remove other task imports.

---

# Final Change Summary

Compared with the previously tested SSPA-7 implementation:

```text
defaults/main.yml        → no functional change
vars/main.yml            → no functional change
report_storage.yml       → same flow and contracts
common_logging calls     → same current shared contract
AAP set_stats contract   → no change
/tmp behavior            → no change
permissions              → no change
naming                   → no change
Check Mode               → no change
retention/email scope    → no change

Required retained fix:
ansible.builtin.find
    hidden: true
```

This is intentionally a **minimal-diff update**, not a redesign of SSPA-7.
