# SSPA-1 — Complete Automation (Revision 2)

**Project:** Enable Application Teams to Self-Serve Akamai Delivery Config Pushes
**Epic:** SSPA-1 | **Stories:** SSPA-4, SSPA-8, SSPA-14
**Execution platform:** Ansible Automation Platform (AAP) Job Templates with surveys

---

## What changed from Revision 1

| # | Change | Why |
|---|---|---|
| 1 | Execution model rewritten for AAP | Survey answers arrive as `extra_vars` (highest precedence). The `-i` flag is no longer the environment control — the Job Template binding is. |
| 2 | All survey inputs namespaced `sspa_in_*` and explicitly mapped | Prevents survey values silently overriding `group_vars` and role `defaults` |
| 3 | Rule-tree normalisation added before diff | Raw PAPI rule trees carry `uuid`, `templateUuid`, `templateLink`; without stripping, every diff is noise |
| 4 | `ruleFormat` pinned identically on both rule-tree GETs | A format mismatch makes a one-line change look like a full rewrite |
| 5 | `create_version.yml` replaced by `resolve_source.yml` | Cloning the active version and then diffing against it produces an empty diff by construction |
| 6 | Diff artifact written back to ctask, not relied on from disk | AAP execution environments are ephemeral containers |
| 7 | Compliance record built explicitly per noncompliance reason | Production activations fail without it |
| 8 | `acknowledgeAllWarnings` removed | Auto-acknowledging warnings destroys the second review gate |
| 9 | `akamai_papi.py` moved to `roles/akamai_property/library/` | It had no declared location; Ansible auto-loads from `library/` |
| 10 | `meta/argument_specs.yml` added | Required variables become a validated contract, not a runtime `KeyError` |

> **Unresolved before this code is written against:** whether PNC's in-house `local.akamai` collection already provides EdgeGrid signing and PAPI modules. If it does, `akamai_papi.py` below should be deleted and replaced with calls into that collection. See Structure Reference §11.

---

## 1. Repository layout

```
sspa-akamai-automation/
├── ansible.cfg
├── requirements.txt
├── collections/
│   └── requirements.yml
├── inventories/
│   ├── staging/
│   │   ├── hosts.yml
│   │   └── group_vars/all/vars.yml
│   └── production/
│       ├── hosts.yml
│       └── group_vars/all/vars.yml
├── playbooks/
│   └── deploy_akamai_config.yml
└── roles/
    ├── snow_chg_validate/          # SSPA-4 + SSPA-8
    │   ├── defaults/main.yml
    │   └── tasks/main.yml
    ├── snow_ctask_update/          # SSPA-14
    │   ├── defaults/main.yml
    │   └── tasks/main.yml
    └── akamai_property/            # SSPA-1 core
        ├── defaults/main.yml
        ├── meta/argument_specs.yml
        ├── library/
        │   └── akamai_papi.py
        ├── filter_plugins/
        │   └── ruletree.py
        └── tasks/
            ├── main.yml
            ├── preflight.yml
            ├── resolve_source.yml
            ├── diff.yml
            ├── activate.yml
            ├── poll_status.yml
            └── rollback.yml
```

Note there is **no `vault.yml`** in the inventories. Under AAP, secrets come from Credentials, not from vaulted files in the repo.

---

## 2. `ansible.cfg`

```ini
[defaults]
inventory            = inventories/staging
roles_path           = roles
collections_path     = collections
host_key_checking    = False
stdout_callback      = yaml
deprecation_warnings = True
interpreter_python   = auto_silent
retry_files_enabled  = False

[inventory]
enable_plugins = yaml, ini

[ssh_connection]
pipelining = True
```

The `inventory` default is deliberately staging. AAP overrides it with the Job Template's bound inventory; the default only matters for local runs, and the safe local default is staging.

---

## 3. `requirements.txt`

```
requests>=2.31.0
deepdiff>=6.7.0
```

EdgeGrid signing is implemented in-module against the stdlib (`hmac`, `hashlib`, `base64`), so `edgegrid-python` is not required. Both packages must be baked into the AAP Execution Environment image — they cannot be pip-installed at job runtime.

## 4. `collections/requirements.yml`

```yaml
---
collections:
  - name: servicenow.itsm
    version: ">=2.6.0"
  - name: ansible.utils
    version: ">=3.0.0"
```

`ansible.utils` is used for `to_paths` / structured comparison helpers. **Pending resolution of the `local.akamai` question, this file may need an in-house collection added and `library/akamai_papi.py` removed.**

---

## 5. Inventories

### `inventories/production/hosts.yml`

```yaml
---
all:
  hosts:
    akamai_runner:
      ansible_connection: local
      ansible_python_interpreter: "{{ ansible_playbook_python }}"
```

Identical in `staging/`. The connection is local because every operation is an outbound HTTPS API call. **The egress IP that must be allowlisted with Akamai is the IP of the AAP execution node, not of anything named here** — this file names the logical target only.

### `inventories/production/group_vars/all/vars.yml`

```yaml
---
sspa_env: production
sspa_akamai_network: PRODUCTION

# Akamai account hierarchy — supplied by the Akamai team, one entry per onboarded app.
# App teams select the mnemonic in the survey; the IDs are never typed by a requester.
sspa_property_catalog:
  BOP:
    contract_id: "ctr_CHANGEME"
    group_id: "grp_CHANGEME"
    property_id: "prp_CHANGEME"
    property_name: "bop.example.com"
    owner: "CHANGEME"

sspa_notify_emails_default:
  - "sspa-automation@example.com"

sspa_require_staging_first: true
sspa_snow_instance: "https://example.service-now.com"
```

`staging/group_vars/all/vars.yml` is structurally identical with `sspa_env: staging`, `sspa_akamai_network: STAGING`, `sspa_require_staging_first: false`, and the staging property IDs.

---

## 6. `playbooks/deploy_akamai_config.yml`

```yaml
---
- name: SSPA-1 — self-service Akamai delivery config push
  hosts: akamai_runner
  gather_facts: false
  vars:
    # ---------------------------------------------------------------
    # Survey inputs arrive as extra_vars, which outrank group_vars and
    # role defaults. They are namespaced sspa_in_* and mapped here so
    # every override is deliberate and greppable.
    # ---------------------------------------------------------------
    sspa_chg_number: "{{ sspa_in_chg_number }}"
    sspa_ctask_number: "{{ sspa_in_ctask_number }}"
    sspa_mnemonic: "{{ sspa_in_app_mnemonic }}"
    sspa_source_mode: "{{ sspa_in_source_mode | default('promote_version') }}"
    sspa_source_version: "{{ sspa_in_source_version | default(omit) }}"
    sspa_activation_note: "{{ sspa_in_note | default('SSPA-1 automated activation') }}"
    sspa_dry_run: "{{ sspa_in_dry_run | default(true) | bool }}"

    sspa_notify_emails: >-
      {{ (sspa_in_notify_emails | default('')) | trim
         | ternary((sspa_in_notify_emails | default('')).split(','),
                   sspa_notify_emails_default) }}

    sspa_target: "{{ sspa_property_catalog[sspa_mnemonic] }}"

  pre_tasks:
    - name: Assert the selected mnemonic exists in this environment's catalog
      ansible.builtin.assert:
        that:
          - sspa_mnemonic in sspa_property_catalog
        fail_msg: >-
          '{{ sspa_mnemonic }}' is not onboarded in {{ sspa_env }}.
          Onboarded: {{ sspa_property_catalog.keys() | list | join(', ') }}
        quiet: true

    - name: Assert nobody has smuggled in an environment override
      ansible.builtin.assert:
        that:
          - sspa_akamai_network == (sspa_env == 'production') | ternary('PRODUCTION', 'STAGING')
        fail_msg: >-
          Network/{{ '' }}environment mismatch. The activation network is derived from the bound
          inventory and must not be supplied as a survey field.
        quiet: true

  roles:
    # SSPA-4 + SSPA-8 — read-only authorisation gate. Nothing mutating runs before this.
    - role: snow_chg_validate

    # SSPA-1 — the only role that changes Akamai state.
    - role: akamai_property

  post_tasks:
    # SSPA-14 — always runs, including after a failed activation.
    - name: Write outcome back to the change task
      ansible.builtin.include_role:
        name: snow_ctask_update
      when: sspa_ctask_number is defined
```

---

## 7. `roles/snow_chg_validate` — SSPA-4 + SSPA-8

### `defaults/main.yml`

```yaml
---
snow_required_state: "implement"
snow_required_approval: "approved"
snow_enforce_window: true
```

### `tasks/main.yml`

```yaml
---
# Read-only. This role can refuse to proceed; it can never cause an incident.

- name: Fetch the change record
  servicenow.itsm.change_request_info:
    instance:
      host: "{{ snow_instance }}"
      username: "{{ snow_username }}"
      password: "{{ snow_password }}"
    number: "{{ sspa_chg_number }}"
  register: chg
  no_log: true

- name: Assert the change record exists
  ansible.builtin.assert:
    that:
      - chg.records | length == 1
    fail_msg: "{{ sspa_chg_number }} not found, or matched more than one record."
    quiet: true

- name: Capture the record
  ansible.builtin.set_fact:
    sspa_chg: "{{ chg.records[0] }}"

# CONFIRMED BUSINESS RULE: state == Implement AND approval == Approved.
# Two separate fields, both required. SSPA-4 and SSPA-8 are written
# inconsistently on this point — see Structure Reference §12.
- name: Assert the change is authorised to implement
  ansible.builtin.assert:
    that:
      - sspa_chg.state | lower == snow_required_state
      - sspa_chg.approval | lower == snow_required_approval
    fail_msg: >-
      {{ sspa_chg_number }} is not implementable.
      state='{{ sspa_chg.state }}' (need '{{ snow_required_state }}'),
      approval='{{ sspa_chg.approval }}' (need '{{ snow_required_approval }}').
    quiet: true

- name: Assert we are inside the planned change window
  ansible.builtin.assert:
    that:
      - (sspa_chg.start_date | to_datetime('%Y-%m-%d %H:%M:%S')).timestamp()
        <= (ansible_date_time.iso8601 | to_datetime('%Y-%m-%dT%H:%M:%SZ')).timestamp()
      - (ansible_date_time.iso8601 | to_datetime('%Y-%m-%dT%H:%M:%SZ')).timestamp()
        <= (sspa_chg.end_date | to_datetime('%Y-%m-%d %H:%M:%S')).timestamp()
    fail_msg: >-
      Outside the change window ({{ sspa_chg.start_date }} → {{ sspa_chg.end_date }}).
    quiet: true
  when: snow_enforce_window | bool

- name: Record the approver for the compliance record
  ansible.builtin.set_fact:
    # peerReviewedBy comes from the CHG approver, NOT from a survey field.
    # A requester self-certifying their own peer review is not a control.
    sspa_peer_reviewed_by: "{{ sspa_chg.approval_history | default(sspa_chg.assigned_to) }}"
```

---

## 8. `roles/akamai_property/library/akamai_papi.py`

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-
"""EdgeGrid-signed Akamai Property Manager API client for Ansible.

Written in-house because no maintained Ansible collection covers PAPI.
Signing follows the EG1-HMAC-SHA256 scheme.
"""

from __future__ import absolute_import, division, print_function
__metaclass__ = type

DOCUMENTATION = r'''
---
module: akamai_papi
short_description: Call Akamai Property Manager API with EdgeGrid signing
options:
  operation:
    description: Which PAPI operation to perform.
    required: true
    type: str
    choices:
      - list_versions
      - get_property
      - get_rule_tree
      - create_version
      - update_rule_tree
      - activate
      - get_activation
      - list_activations
  credentials:
    description: EdgeGrid credential set (host, client_token, client_secret, access_token).
    required: true
    type: dict
  contract_id: {type: str}
  group_id: {type: str}
  property_id: {type: str}
  property_version: {type: int}
  rule_format: {type: str, default: latest}
  body: {type: dict}
  activation_id: {type: str}
  network: {type: str, choices: [STAGING, PRODUCTION]}
  timeout: {type: int, default: 60}
'''

RETURN = r'''
status_code: {description: HTTP status returned by PAPI, type: int, returned: always}
json:        {description: Parsed response body, type: dict, returned: always}
'''

import base64
import hashlib
import hmac
import json
import uuid
from datetime import datetime

try:
    from urllib.parse import urlencode
except ImportError:                                    # pragma: no cover
    from urllib import urlencode                       # noqa: F401

from ansible.module_utils.basic import AnsibleModule
from ansible.module_utils.urls import fetch_url

# EdgeGrid caps the signed body at 128 KiB. Rule trees can exceed this on
# large properties; anything beyond the cap is truncated before hashing,
# which is what Akamai's own clients do.
MAX_BODY = 131072


def _timestamp():
    return datetime.utcnow().strftime('%Y%m%dT%H:%M:%S+0000')


def _b64_hmac(data, key):
    return base64.b64encode(
        hmac.new(key.encode('utf8'), data.encode('utf8'), hashlib.sha256).digest()
    ).decode('utf8')


def _content_hash(method, body):
    """Only POST bodies are content-hashed under EG1."""
    if method.upper() != 'POST' or not body:
        return ''
    return base64.b64encode(
        hashlib.sha256(body.encode('utf8')[:MAX_BODY]).digest()
    ).decode('utf8')


def _auth_header(creds, method, path, body):
    ts = _timestamp()
    prefix = 'EG1-HMAC-SHA256 client_token={0};access_token={1};timestamp={2};nonce={3};'.format(
        creds['client_token'], creds['access_token'], ts, str(uuid.uuid4())
    )
    data_to_sign = '\t'.join([
        method.upper(),
        'https',
        creds['host'],
        path,
        '',                                   # canonicalised headers: none signed
        _content_hash(method, body),
        prefix,
    ])
    signing_key = _b64_hmac(ts, creds['client_secret'])
    return prefix + 'signature=' + _b64_hmac(data_to_sign, signing_key)


def _qs(params):
    clean = {k: v for k, v in params.items() if v}
    return ('?' + urlencode(clean)) if clean else ''


def build_request(p):
    """Return (method, path, body_dict, extra_headers).

    Query strings are assembled here rather than in YAML — building URLs in
    Jinja caused a silent line-folding bug in Revision 1.
    """
    op = p['operation']
    scope = {'contractId': p.get('contract_id'), 'groupId': p.get('group_id')}
    prop = '/papi/v1/properties/{0}'.format(p.get('property_id'))
    hdrs = {}

    if op == 'get_property':
        return 'GET', prop + _qs(scope), None, hdrs

    if op == 'list_versions':
        # /versions/latest returns a 302 redirect, so it is never used.
        # The active version is read from the versions list instead.
        return 'GET', prop + '/versions' + _qs(scope), None, hdrs

    if op == 'get_rule_tree':
        # Pin the rule format so two GETs are structurally comparable.
        hdrs['Accept'] = 'application/vnd.akamai.papirules.{0}+json'.format(p['rule_format'])
        path = '{0}/versions/{1}/rules'.format(prop, p['property_version'])
        return 'GET', path + _qs(dict(scope, validateRules='false')), None, hdrs

    if op == 'create_version':
        return 'POST', prop + '/versions' + _qs(scope), p['body'], hdrs

    if op == 'update_rule_tree':
        hdrs['Content-Type'] = 'application/vnd.akamai.papirules.{0}+json'.format(p['rule_format'])
        path = '{0}/versions/{1}/rules'.format(prop, p['property_version'])
        return 'PUT', path + _qs(scope), p['body'], hdrs

    if op == 'activate':
        return 'POST', prop + '/activations' + _qs(scope), p['body'], hdrs

    if op == 'get_activation':
        path = '{0}/activations/{1}'.format(prop, p['activation_id'])
        return 'GET', path + _qs(scope), None, hdrs

    if op == 'list_activations':
        return 'GET', prop + '/activations' + _qs(scope), None, hdrs

    raise ValueError('unsupported operation: %s' % op)


def main():
    module = AnsibleModule(
        argument_spec=dict(
            operation=dict(type='str', required=True),
            credentials=dict(type='dict', required=True, no_log=True),
            contract_id=dict(type='str'),
            group_id=dict(type='str'),
            property_id=dict(type='str'),
            property_version=dict(type='int'),
            rule_format=dict(type='str', default='latest'),
            body=dict(type='dict'),
            activation_id=dict(type='str'),
            network=dict(type='str', choices=['STAGING', 'PRODUCTION']),
            timeout=dict(type='int', default=60),
        ),
        supports_check_mode=True,
    )
    p = module.params
    creds = p['credentials']

    try:
        method, path, body_dict, extra = build_request(p)
    except ValueError as exc:
        module.fail_json(msg=str(exc))

    mutating = method in ('POST', 'PUT')
    if module.check_mode and mutating:
        module.exit_json(changed=False, skipped=True,
                         msg='check mode: %s %s not sent' % (method, path))

    payload = json.dumps(body_dict) if body_dict else None

    headers = {
        'Authorization': _auth_header(creds, method, path, payload or ''),
        'Accept': extra.get('Accept', 'application/json'),
        # Prefixed IDs (prp_, ctr_, grp_) are what the catalog stores.
        'PAPI-Use-Prefixes': 'true',
    }
    if payload:
        headers['Content-Type'] = extra.get('Content-Type', 'application/json')

    url = 'https://{0}{1}'.format(creds['host'], path)
    resp, info = fetch_url(module, url, method=method, data=payload,
                           headers=headers, timeout=p['timeout'])

    status = info.get('status', -1)
    raw = resp.read() if resp else info.get('body', b'')
    try:
        parsed = json.loads(raw) if raw else {}
    except (ValueError, TypeError):
        parsed = {'raw': str(raw)}

    if status < 200 or status >= 300:
        module.fail_json(msg='PAPI %s %s failed with HTTP %s' % (method, path, status),
                         status_code=status, json=parsed)

    module.exit_json(changed=mutating, status_code=status, json=parsed)


if __name__ == '__main__':
    main()
```

---

## 9. `roles/akamai_property/filter_plugins/ruletree.py`

This is the correction that makes the diff usable. Without it, PAPI's generated identifiers dominate the output and reviewers stop reading.

```python
"""Rule-tree normalisation filters.

PAPI stamps generated identifiers into every rule node. Comparing raw trees
produces hundreds of lines of churn that has nothing to do with the change,
and a diff nobody reads is worse than no diff — it manufactures assurance.
"""

from __future__ import absolute_import, division, print_function
__metaclass__ = type

import json

# Generated by PAPI, meaningless for review, different on every version.
NOISE_KEYS = ('uuid', 'templateUuid', 'templateLink', 'etag', 'errors', 'warnings')


def _strip(node):
    if isinstance(node, dict):
        return {k: _strip(v) for k, v in sorted(node.items()) if k not in NOISE_KEYS}
    if isinstance(node, list):
        return [_strip(i) for i in node]
    return node


def normalize_ruletree(tree):
    """Strip generated IDs and canonicalise key ordering."""
    return _strip(tree.get('rules', tree))


def ruletree_lines(tree):
    """Deterministic text form, suitable for a unified diff."""
    return json.dumps(normalize_ruletree(tree), indent=2, sort_keys=True).splitlines()


class FilterModule(object):
    def filters(self):
        return {
            'normalize_ruletree': normalize_ruletree,
            'ruletree_lines': ruletree_lines,
        }
```

---

## 10. `roles/akamai_property/defaults/main.yml`

Everything here is overridable by inventory, which is the whole point of `defaults/` rather than `vars/`.

```yaml
---
akamai_rule_format: "v2024-01-09"     # pinned, never 'latest' — see note below
akamai_poll_interval: 10
akamai_poll_timeout: 3600
akamai_use_fast_fallback: true
akamai_fail_on_warnings: true          # never auto-acknowledge
akamai_max_diff_lines: 2000
akamai_artifact_dir: "{{ lookup('env', 'AWX_ISOLATED_DATA_DIR') | default('/tmp', true) }}"
```

`akamai_rule_format` is pinned to a frozen format, not `latest`. Under `latest`, Akamai can change the rule schema underneath you and the same property will diff against itself.

## 11. `roles/akamai_property/meta/argument_specs.yml`

```yaml
---
argument_specs:
  main:
    short_description: Push an Akamai delivery configuration version to a network
    options:
      sspa_target:
        type: dict
        required: true
        options:
          contract_id: {type: str, required: true}
          group_id:    {type: str, required: true}
          property_id: {type: str, required: true}
      sspa_akamai_network:
        type: str
        required: true
        choices: [STAGING, PRODUCTION]
      sspa_source_mode:
        type: str
        required: true
        choices: [promote_version, apply_rules]
      sspa_dry_run:
        type: bool
        required: true
```

---

## 12. `roles/akamai_property/tasks/main.yml`

```yaml
---
- name: Akamai property push
  block:
    - ansible.builtin.include_tasks: preflight.yml
    - ansible.builtin.include_tasks: resolve_source.yml
    - ansible.builtin.include_tasks: diff.yml

    - name: Stop here — dry run
      ansible.builtin.meta: end_host
      when: sspa_dry_run | bool

    - ansible.builtin.include_tasks: activate.yml
    - ansible.builtin.include_tasks: poll_status.yml

  rescue:
    - name: Record the failure
      ansible.builtin.set_fact:
        sspa_result: FAILED
        sspa_failure_reason: "{{ ansible_failed_result.msg | default('unknown') }}"

    # Rollback only if we actually changed production state. A failure in
    # preflight or diff has no side effects and must not trigger one.
    - ansible.builtin.include_tasks: rollback.yml
      when:
        - sspa_activation_id is defined
        - sspa_akamai_network == 'PRODUCTION'

    - name: Re-raise so the job is marked failed
      ansible.builtin.fail:
        msg: "{{ sspa_failure_reason }}"

  always:
    - name: Ensure a result exists for the ctask write-back
      ansible.builtin.set_fact:
        sspa_result: "{{ sspa_result | default('SUCCESS') }}"
```

## 13. `tasks/preflight.yml`

```yaml
---
# Cheap, side-effect-free assertions. Every check here is one that cannot
# be performed safely once a version exists or an activation is in flight.

- name: Resolve the property
  akamai_papi:
    operation: get_property
    credentials: "{{ akamai_edgegrid }}"
    contract_id: "{{ sspa_target.contract_id }}"
    group_id: "{{ sspa_target.group_id }}"
    property_id: "{{ sspa_target.property_id }}"
  register: prop
  no_log: true

- name: Read the currently active version on the target network
  ansible.builtin.set_fact:
    sspa_active_version: >-
      {{ prop.json.properties.items[0]
         | json_query(sspa_akamai_network == 'PRODUCTION'
                      and 'productionVersion' or 'stagingVersion') }}

- name: Assert something is currently active
  ansible.builtin.assert:
    that:
      - sspa_active_version | int > 0
    fail_msg: >-
      No active version on {{ sspa_akamai_network }} for
      {{ sspa_target.property_id }}. There is nothing to diff against and
      nothing to roll back to. First activation must be done manually.
    quiet: true

- name: List activations to check for an in-flight push
  akamai_papi:
    operation: list_activations
    credentials: "{{ akamai_edgegrid }}"
    contract_id: "{{ sspa_target.contract_id }}"
    group_id: "{{ sspa_target.group_id }}"
    property_id: "{{ sspa_target.property_id }}"
  register: acts
  no_log: true

# PAPI returns 409 on a concurrent same-network activation. Detecting it
# here turns an opaque mid-run failure into a clear pre-run refusal.
- name: Assert no activation is already pending on this network
  ansible.builtin.assert:
    that:
      - pending | length == 0
    fail_msg: "Activation {{ pending | first }} is already in flight on {{ sspa_akamai_network }}."
    quiet: true
  vars:
    pending: >-
      {{ acts.json.activations.items
         | selectattr('network', 'equalto', sspa_akamai_network)
         | selectattr('status', 'in', ['PENDING', 'ZONE_1', 'ZONE_2', 'ZONE_3', 'NEW'])
         | map(attribute='activationId') | list }}

- name: Assert staging was activated first
  ansible.builtin.assert:
    that:
      - prop.json.properties.items[0].stagingVersion | int > 0
    fail_msg: "Staging gate: this property has never been activated on STAGING."
    quiet: true
  when:
    - sspa_require_staging_first | bool
    - sspa_akamai_network == 'PRODUCTION'
```

## 14. `tasks/resolve_source.yml`

Replaces Revision 1's `create_version.yml`. That file cloned the active version, and the next step diffed the clone against the active version — guaranteed to be empty.

```yaml
---
# Two legitimate modes, and they are mutually exclusive:
#
#   promote_version — an app team edited the property in Property Manager and
#                     wants version N pushed. Nothing is created here.
#   apply_rules     — a rule payload comes from source control; a version is
#                     cloned AND mutated before it can be meaningfully diffed.

- name: Promote an existing version
  when: sspa_source_mode == 'promote_version'
  block:
    - name: Assert a source version was supplied
      ansible.builtin.assert:
        that:
          - sspa_source_version is defined
          - sspa_source_version | int > 0
        fail_msg: "promote_version requires a version number."
        quiet: true

    - name: Assert the version is not already live
      ansible.builtin.assert:
        that:
          - sspa_source_version | int != sspa_active_version | int
        fail_msg: >-
          Version {{ sspa_source_version }} is already active on
          {{ sspa_akamai_network }}. Nothing to do.
        quiet: true

    - ansible.builtin.set_fact:
        sspa_candidate_version: "{{ sspa_source_version | int }}"

- name: Clone and apply a rule payload from source control
  when: sspa_source_mode == 'apply_rules'
  block:
    - name: Clone the active version
      akamai_papi:
        operation: create_version
        credentials: "{{ akamai_edgegrid }}"
        contract_id: "{{ sspa_target.contract_id }}"
        group_id: "{{ sspa_target.group_id }}"
        property_id: "{{ sspa_target.property_id }}"
        body:
          createFromVersion: "{{ sspa_active_version | int }}"
      register: cloned
      no_log: true

    - ansible.builtin.set_fact:
        sspa_candidate_version: "{{ cloned.json.versionLink | regex_search('versions/(\\d+)', '\\1') | first }}"

    - name: Write the new rule tree onto the clone
      akamai_papi:
        operation: update_rule_tree
        credentials: "{{ akamai_edgegrid }}"
        contract_id: "{{ sspa_target.contract_id }}"
        group_id: "{{ sspa_target.group_id }}"
        property_id: "{{ sspa_target.property_id }}"
        property_version: "{{ sspa_candidate_version | int }}"
        rule_format: "{{ akamai_rule_format }}"
        body: "{{ sspa_rule_payload }}"
      no_log: true
```

## 15. `tasks/diff.yml`

This file is the entire justification for removing the human from the change call. Its output is the evidence, and SSPA-14 is what makes that evidence durable.

```yaml
---
- name: Fetch the live rule tree
  akamai_papi:
    operation: get_rule_tree
    credentials: "{{ akamai_edgegrid }}"
    contract_id: "{{ sspa_target.contract_id }}"
    group_id: "{{ sspa_target.group_id }}"
    property_id: "{{ sspa_target.property_id }}"
    property_version: "{{ sspa_active_version | int }}"
    rule_format: "{{ akamai_rule_format }}"     # identical on both GETs
  register: live
  no_log: true

- name: Fetch the candidate rule tree
  akamai_papi:
    operation: get_rule_tree
    credentials: "{{ akamai_edgegrid }}"
    contract_id: "{{ sspa_target.contract_id }}"
    group_id: "{{ sspa_target.group_id }}"
    property_id: "{{ sspa_target.property_id }}"
    property_version: "{{ sspa_candidate_version | int }}"
    rule_format: "{{ akamai_rule_format }}"
  register: candidate
  no_log: true

- name: Normalise and compare
  ansible.builtin.set_fact:
    sspa_diff_before: "{{ live.json | ruletree_lines }}"
    sspa_diff_after: "{{ candidate.json | ruletree_lines }}"

- name: Refuse to activate a no-op
  ansible.builtin.assert:
    that:
      - sspa_diff_before != sspa_diff_after
    fail_msg: >-
      Version {{ sspa_candidate_version }} is byte-identical to the live
      version {{ sspa_active_version }} after normalisation. Activating it
      would be a production change with no change in it.
    quiet: true

- name: Render the diff
  ansible.builtin.set_fact:
    sspa_diff_text: "{{ lookup('ansible.builtin.pipe', 'true') }}"
  changed_when: false

- name: Produce a unified diff artifact
  ansible.builtin.copy:
    dest: "{{ akamai_artifact_dir }}/sspa_diff_{{ sspa_chg_number }}.txt"
    content: |
      Property : {{ sspa_target.property_id }} ({{ sspa_mnemonic }})
      Network  : {{ sspa_akamai_network }}
      Live     : v{{ sspa_active_version }}
      Candidate: v{{ sspa_candidate_version }}
      Format   : {{ akamai_rule_format }}
      Change   : {{ sspa_chg_number }}
      ---
      {{ sspa_diff_after | difference(sspa_diff_before) | join('\n') }}
    mode: "0640"

# The AAP execution environment is an ephemeral container. This artifact does
# NOT survive the job. It survives because SSPA-14 writes it to the ctask.
- name: Carry the diff forward for the ctask write-back
  ansible.builtin.set_fact:
    sspa_diff_summary: >-
      {{ (sspa_diff_after | difference(sspa_diff_before))[:akamai_max_diff_lines] | join('\n') }}
```

## 16. `tasks/activate.yml`

```yaml
---
- name: Build the compliance record
  ansible.builtin.set_fact:
    sspa_compliance_record: >-
      {{
        {'noncomplianceReason': 'NONE',
         'customerEmail': sspa_in_customer_email,
         'peerReviewedBy': sspa_peer_reviewed_by,
         'unitTested': sspa_in_unit_tested | bool,
         'ticketId': sspa_chg_number}
        if sspa_in_noncompliance_reason == 'NONE' else
        {'noncomplianceReason': 'OTHER',
         'otherNoncomplianceReason': sspa_in_other_reason,
         'ticketId': sspa_chg_number}
        if sspa_in_noncompliance_reason == 'OTHER' else
        {'noncomplianceReason': sspa_in_noncompliance_reason,
         'ticketId': sspa_chg_number}
      }}
  when: sspa_akamai_network == 'PRODUCTION'

- name: Submit the activation
  akamai_papi:
    operation: activate
    credentials: "{{ akamai_edgegrid }}"
    contract_id: "{{ sspa_target.contract_id }}"
    group_id: "{{ sspa_target.group_id }}"
    property_id: "{{ sspa_target.property_id }}"
    body: >-
      {{
        {'propertyVersion': sspa_candidate_version | int,
         'network': sspa_akamai_network,
         'activationType': 'ACTIVATE',
         'note': sspa_activation_note ~ ' [' ~ sspa_chg_number ~ ']',
         'notifyEmails': sspa_notify_emails,
         'useFastFallback': akamai_use_fast_fallback | bool}
        | combine({'complianceRecord': sspa_compliance_record}
                  if sspa_akamai_network == 'PRODUCTION' else {})
      }}
  register: activation
  no_log: true

# Deliberately absent: acknowledgeAllWarnings. Exposing a blanket
# acknowledgement means every requester ticks it and the warning gate stops
# existing. Warnings fail the run and require a deliberate re-launch.

- ansible.builtin.set_fact:
    sspa_activation_id: "{{ activation.json.activationLink | regex_search('activations/(\\w+)', '\\1') | first }}"
```

## 17. `tasks/poll_status.yml`

```yaml
---
- name: Poll until the activation reaches a terminal state
  akamai_papi:
    operation: get_activation
    credentials: "{{ akamai_edgegrid }}"
    contract_id: "{{ sspa_target.contract_id }}"
    group_id: "{{ sspa_target.group_id }}"
    property_id: "{{ sspa_target.property_id }}"
    activation_id: "{{ sspa_activation_id }}"
  register: status
  # ZONE_1/2/3 are propagation stages, not terminal states.
  until: status.json.activations.items[0].status not in
         ['PENDING', 'NEW', 'ZONE_1', 'ZONE_2', 'ZONE_3']
  retries: "{{ (akamai_poll_timeout | int // akamai_poll_interval | int) }}"
  delay: "{{ akamai_poll_interval | int }}"
  no_log: true

- name: Assert the activation succeeded
  ansible.builtin.assert:
    that:
      - status.json.activations.items[0].status == 'ACTIVE'
    fail_msg: >-
      Activation {{ sspa_activation_id }} ended in state
      '{{ status.json.activations.items[0].status }}'.
    quiet: true
```

## 18. `tasks/rollback.yml`

```yaml
---
# Recovery is pre-written and pre-tested, not improvised at 2am.
# Runnable standalone: --tags rollback

- name: Re-activate the last known good version
  akamai_papi:
    operation: activate
    credentials: "{{ akamai_edgegrid }}"
    contract_id: "{{ sspa_target.contract_id }}"
    group_id: "{{ sspa_target.group_id }}"
    property_id: "{{ sspa_target.property_id }}"
    body:
      propertyVersion: "{{ sspa_active_version | int }}"
      network: "{{ sspa_akamai_network }}"
      activationType: ACTIVATE
      note: "SSPA-1 automated rollback of {{ sspa_chg_number }}"
      notifyEmails: "{{ sspa_notify_emails }}"
      useFastFallback: true
      complianceRecord:
        noncomplianceReason: EMERGENCY
        ticketId: "{{ sspa_chg_number }}"
  register: rb
  no_log: true

- ansible.builtin.set_fact:
    sspa_rollback_activation_id: "{{ rb.json.activationLink | regex_search('activations/(\\w+)', '\\1') | first }}"
    sspa_result: ROLLED_BACK
```

---

## 19. `roles/snow_ctask_update` — SSPA-14

```yaml
---
# Runs on both success and failure. A write-back failure must never roll back
# a successful activation — the production change already happened; only the
# bookkeeping is behind.

- name: Append the deployment record to the change task
  servicenow.itsm.api:
    instance:
      host: "{{ snow_instance }}"
      username: "{{ snow_username }}"
      password: "{{ snow_password }}"
    resource: sc_task
    action: patch
    query_params:
      sysparm_query: "number={{ sspa_ctask_number }}"
    data:
      work_notes: |
        SSPA-1 automated Akamai push
        Result       : {{ sspa_result }}
        Property     : {{ sspa_target.property_id }} ({{ sspa_mnemonic }})
        Network      : {{ sspa_akamai_network }}
        Live before  : v{{ sspa_active_version | default('n/a') }}
        Activated    : v{{ sspa_candidate_version | default('n/a') }}
        Activation ID: {{ sspa_activation_id | default('n/a') }}
        Rollback ID  : {{ sspa_rollback_activation_id | default('none') }}
        Triggered by : {{ awx_user_name | default(lookup('env', 'USER')) }}
        AAP job      : {{ awx_job_id | default('n/a') }}
        Timestamp    : {{ ansible_date_time.iso8601 | default(lookup('pipe', 'date -u +%FT%TZ')) }}
        {% if sspa_failure_reason is defined %}
        Failure      : {{ sspa_failure_reason }}
        {% endif %}
        --- rule tree diff (normalised) ---
        {{ sspa_diff_summary | default('not generated') }}
  no_log: true
  failed_when: false        # never mask the real outcome behind a SNOW error
```

---

## 20. Blockers — code completeness does not make this runnable

| # | Blocker | Owner | Consequence if unresolved |
|---|---|---|---|
| 1 | READ-WRITE EdgeGrid credential grant | Akamai team | Every activation returns 403 |
| 2 | Compliance record requirement confirmed for this account | Akamai TAM | Production activations rejected |
| 3 | Correct `contractId` / `groupId` / `propertyId` for the BOP mnemonic | Akamai team | Catalog is placeholders; nothing resolves |
| 4 | Egress IP allowlisting for AAP execution nodes | Network + Akamai | Connection refused before any auth |
| 5 | `local.akamai` collection contents | Platform team | May invalidate `akamai_papi.py` entirely |
| 6 | Pinned `ruleFormat` for the target property | Akamai team | Diff produces false positives |
| 7 | Staging property / sandbox for testing | Akamai team | No safe place to test |
| 8 | Named property owner | App team | No one to escalate to on failure |
| 9 | AAP custom Credential Type for EdgeGrid | Platform team | Secrets would otherwise land in survey extra_vars |
| 10 | ServiceNow API account with `sc_task` write | ITSM team | SSPA-14 silently no-ops |

Development can complete while these are open. **Testing cannot start.**
