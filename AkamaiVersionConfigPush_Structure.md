# AkamaiVersionConfigPush — Ansible Project Structure

**Project:** SSP Automation (SSPA epic) — self-service Akamai delivery config push
**Model:** Promote-only. Akamai developers create property versions manually. This automation **reads, diffs, activates, and polls**. It never creates or edits a version.
**Authoritative spec:** Confluence *Automated Deployment Workflow* — 22-step flowchart.

**Layout per lead's direction:** two separate main folders — **`roles/AkamaiVersionConfigPush/`** and **`playbooks/AkamaiVersionConfigPush/`** — each containing an `akamai` namespace holding its own files.

---

## 0. Read this before anything else

Four items. The first two change the shape of the project, so they are not optional.

### 0.1 The nested role folder breaks role resolution — `ansible.cfg` must be patched

Ansible's `roles_path` **does not recurse**. With the role at `roles/AkamaiVersionConfigPush/akamai/`, a playbook that says `roles: - akamai` will fail with *"the role 'akamai' was not found"*, because Ansible only looks one level deep inside each entry in `roles_path`.

**This is the single most likely reason your first run fails.** The fix is to add the intermediate folder to `roles_path` explicitly:

```ini
# ansible.cfg  (repo root)
[defaults]
roles_path = roles:roles/AkamaiVersionConfigPush
collections_path = collections
stdout_callback = yaml
```

Every future `roles/<MainFolder>/` added by another team needs its own entry. That is the recurring cost of this layout — worth stating to the lead so it is a conscious tradeoff, not a surprise six months from now.

**Alternative if you do not want to touch `ansible.cfg`:** use `ansible.builtin.import_role` with an explicit relative path instead of the `roles:` keyword. It is uglier and lint flags it. Patching `roles_path` is the cleaner option.

### 0.2 Step 10 cannot live inside a role

The flowchart's **step 10 "AAP Approval Gate — Await approval"** is an **AAP Workflow Approval Node**. A playbook cannot pause for a human decision and resume later. If all 22 steps are built as one role invoked by one playbook, step 10 becomes either a fake auto-pass or a broken `pause` task.

**Consequence:** three playbooks / three AAP Job Templates, stitched together by an AAP Workflow Template with an approval node between JT #1 and JT #2.

```
[ JT #1: 01_akamai_prechecks.yml ]   <- steps 1-9
              |
              v
     +------------------+
     | AAP APPROVAL NODE|            <- step 10 (native AAP, not code)
     +------------------+
              | approved
              v
[ JT #2: 02_akamai_deploy.yml ]      <- steps 11-17
              | on failure
              v
[ JT #3: 03_akamai_rollback.yml ]    <- steps 18-22
```

### 0.3 Files that must be deleted from the previous layout

| File in old structure | Why it must go |
|---|---|
| `library/akamai_papi.py` | Reimplements EdgeGrid signing and PAPI calls that `local.akamai` already provides. Duplicating a vendored internal collection creates a second thing to maintain and audit. |
| `tasks/resolve_source.yml` | Leftover from the abandoned source-version model. Under promote-only there is no source to resolve. |
| `tasks/create_version.yml` | Confirm it is gone. Creating versions violates the hard architectural constraint. |
| `tasks/preflight.yml` (as named) | Split into numbered validation files so every task file traces to a flowchart step. |

### 0.4 Open decision for the lead — the ServiceNow roles

Collapsing `snow_chg_validate` and `snow_ctask_update` into the single `akamai` role satisfies the namespacing request, but those two are **generic ServiceNow gates** any PNC automation could reuse. Folding them in makes them Akamai-only.

If the lead wants them preserved as reusable, they become a second role inside the same main folder:
`roles/AkamaiVersionConfigPush/snow_gates/`. The structure below shows them folded in, per the instruction as given.

---

## 1. Directory tree

```
SSP_Automation/                          <- existing PNC repo root
|
|-- ansible.cfg                          roles_path MUST include the nested dir (see 0.1)
|-- .ansible-lint                        profile: production
|-- .gitignore                           *.retry, .edgerc, artifacts/
|-- README.md
|
|-- collections/
|   `-- requirements.yml                 community.general, servicenow.itsm
|                                        (local.akamai is vendored - document here)
|
|-- inventories/
|   `-- akamai/
|       |-- qa/
|       |   `-- aki_qa.yml               host QA   -> environment.name: STAGING
|       `-- prod/
|           `-- aki_prod.yml             host PROD -> environment.name: PRODUCTION
|                                        <- network is DERIVED HERE. Never a survey field.
|
|
|-- roles/
|   `-- AkamaiVersionConfigPush/         ############ MAIN FOLDER 1 - ROLE SIDE ############
|       `-- akamai/                      <- role name
|           |
|           |-- README.md                role purpose, inputs, invocation, caveats
|           |
|           |-- defaults/
|           |   `-- main.yml             overridable inputs (lowest precedence)
|           |
|           |-- vars/
|           |   `-- main.yml             constants that must NOT be overridden
|           |                            (poll 10s, max 360 checks, 60 min, PAPI paths)
|           |
|           |-- meta/
|           |   |-- main.yml             galaxy_info, dependencies: []
|           |   `-- argument_specs.yml   type/required/choices validation at role entry
|           |
|           |-- filter_plugins/
|           |   `-- ruletree.py          rule-tree diff ONLY - keep only if local.akamai
|           |                            has no diff module. Verify first (see 5.3)
|           |
|           |-- templates/
|           |   |-- predeploy_report.j2      step 8
|           |   |-- deployment_report.j2     step 15
|           |   `-- notification.j2          step 16
|           |
|           `-- tasks/
|               |-- main.yml             router - dispatches on akamai_stage
|               |
|               |-- prechecks.yml        == entry point for playbook 01 ==
|               |   |-- 02_local_validation.yml
|               |   |-- 03_snow_validation.yml
|               |   |-- 04_jira_validation.yml
|               |   |-- 05_approver_routing.yml
|               |   |-- 06_akamai_metadata.yml
|               |   |-- 07_version_comparison.yml
|               |   |-- 08_predeploy_report.yml
|               |   `-- 09_send_report.yml
|               |
|               |-- deploy.yml           == entry point for playbook 02 ==
|               |   |-- 11_determine_path.yml
|               |   |-- 11a_activate.yml      (included 1x or 2x - see section 4)
|               |   |-- 12_monitor.yml
|               |   |-- 13_snow_notes.yml
|               |   |-- 14_post_validation.yml
|               |   |-- 15_reporting.yml
|               |   |-- 16_notifications.yml
|               |   `-- 17_audit_log.yml
|               |
|               `-- rollback.yml         == entry point for playbook 03 ==
|                   |-- 20_identify_stable.yml
|                   |-- 21_redeploy_stable.yml
|                   `-- 22_validate_and_close.yml
|
|
`-- playbooks/
    `-- AkamaiVersionConfigPush/         ########## MAIN FOLDER 2 - PLAYBOOK SIDE ##########
        `-- akamai/                      <- akamai namespace
            |
            |-- group_vars/
            |   `-- all.yml              non-secret shared vars for these playbooks
            |
            |-- 01_akamai_prechecks.yml      <- STEPS 1-9   (AAP Job Template #1)
            |-- 02_akamai_deploy.yml         <- STEPS 11-17 (AAP Job Template #2)
            `-- 03_akamai_rollback.yml       <- STEPS 18-22 (JT #3, also manual entry)
```

**How the two main folders connect:**

```
playbooks/AkamaiVersionConfigPush/akamai/02_akamai_deploy.yml
        |   roles: - akamai        (resolved via roles_path, see 0.1)
        v
roles/AkamaiVersionConfigPush/akamai/tasks/main.yml
        |   include_tasks by akamai_stage
        v
   prechecks.yml | deploy.yml | rollback.yml
```

---

## 2. Flowchart step -> file mapping

| Step | Flowchart box | Implemented in | Notes |
|---|---|---|---|
| 1 | Request Intake (AAP Survey) | **AAP Survey on JT #1** | Not code. `network` is **not** a survey field. |
| 2 | Local Validation | `tasks/02_local_validation.yml` | Required fields, Jira format, version format, payload completeness |
| 3 | ServiceNow Validation | `tasks/03_snow_validation.yml` | `state == Implement` **AND** `approval == Approved` — two separate fields, checked simultaneously. Plus change-window check. Read-only. |
| 4 | Jira API Validation | `tasks/04_jira_validation.yml` | Issue exists, accessible, status retrievable, project valid |
| 5 | Approver Check & Team Routing | `tasks/05_approver_routing.yml` | `peerReviewedBy` derived from the **CHG approver**, never a survey checkbox |
| 6 | Akamai Metadata Retrieval | `tasks/06_akamai_metadata.yml` | latest / staging / production version. **Must persist `productionVersion` here** — see 5.2 |
| 7 | Version Comparison | `tasks/07_version_comparison.yml` | requested vs staging vs production, upgrade-path check |
| 8 | Pre-Deployment Comparison Report | `tasks/08_predeploy_report.yml` + `predeploy_report.j2` | Diff goes to **ctask work notes**, not local disk — EEs are ephemeral |
| 9 | Send Report to Application Team | `tasks/09_send_report.yml` | App team, deployment owner, approvers (optional) |
| **10** | **AAP Approval Gate** | **AAP Workflow Approval Node** | **No task file. Split point between JT #1 and JT #2.** |
| 11 | Deployment Execution | `tasks/11_determine_path.yml` + `11a_activate.yml` | See section 4 |
| 12 | Deployment Monitoring | `tasks/12_monitor.yml` | Poll every 10s, up to 60 min, max 360 checks |
| 13 | Update Deployment Notes (SNOW) | `tasks/13_snow_notes.yml` | Mandatory: Version Number, Triggered By, Triggered On, Status (PASS/FAIL). Optional: rollback info, error logs, deployment details. Append-only. |
| 14 | Post-Deployment Validation | `tasks/14_post_validation.yml` | Validation scan, health/readiness/compliance, smoke tests, capture results |
| 15 | Reporting | `tasks/15_reporting.yml` + `deployment_report.j2` | Generate, capture metadata & outcome, publish status, store, mark complete |
| 16 | Notifications | `tasks/16_notifications.yml` + `notification.j2` | Success/failure notifications, SNOW update, email/collab |
| 17 | Audit and Logging | `tasks/17_audit_log.yml` | Traceability, operational visibility, compliance |
| 18 | Rollback Triggers | **`rescue:` block in `deploy.yml`** | Not a task file — exit conditions. See 5.1 |
| 19 | Rollback Needed? (decision) | `when:` on the rescue | Decision, not a step |
| 20 | Identify Previously Deployed Stable Version | `tasks/20_identify_stable.yml` | Reads the value captured at step 6 |
| 21 | Redeploy Previous Stable Version | `tasks/21_redeploy_stable.yml` | Execute rollback deployment, validate activation, verify app functionality |
| 22 | Validate Rollback Completion, Update Report, Task & Close | `tasks/22_validate_and_close.yml` | Validate, update CHG, rollback notification, update report, close request |

---

## 3. Folder-by-folder rationale

### 3.1 Repo-level files

| Path | Purpose | What breaks without it |
|---|---|---|
| `ansible.cfg` | Sets `roles_path` (including the nested main folder), `collections_path`, stdout callback | The role does not resolve — see 0.1 |
| `.ansible-lint` | `profile: production` — FQCN everywhere, `name:` on every task, no bare vars | Merges are lint-gated in this repo; code fails CI |
| `.gitignore` | Excludes `.edgerc`, `artifacts/`, `*.retry` | Credentials leak into version control |
| `README.md` | How to run, required survey vars, JT/workflow wiring | Next engineer cannot run it without asking you |

### 3.2 `collections/`

Declares `community.general` and `servicenow.itsm`, and **documents** that `local.akamai` is vendored at `collections/ansible_collections/local/akamai`. Pinning makes the Execution Environment reproducible — an unpinned collection can change ServiceNow module behaviour between two runs of identical code.

### 3.3 `inventories/akamai/`

The target model — where this runs and what is true about that environment. Two inventories, each defining one host with `akamai.environment.name`.

**This is the only place the network is defined.** A production push requires selecting the production inventory, so a wrong-environment deployment requires a visibly wrong selection, not a mistyped dropdown.

Credentials come from **AAP Credential Types** injected at runtime — not vault files in the inventory.

### 3.4 Main folder 1 — `roles/AkamaiVersionConfigPush/akamai/`

| Folder | Purpose |
|---|---|
| `defaults/main.yml` | Lowest precedence — anything a survey or inventory may legitimately override: property id, requested version, requestor, ticket ids, dry-run toggle |
| `vars/main.yml` | Highest precedence — constants that must **not** be overridable: poll interval, max checks, timeout, PAPI paths |
| `meta/main.yml` | `galaxy_info`, `dependencies: []` |
| `meta/argument_specs.yml` | Validates types/required/choices **at role entry**, so a bad survey value fails before any API call rather than halfway through |
| `filter_plugins/ruletree.py` | Rule-tree diff only. Conditional — see 5.3 |
| `templates/` | Report and notification bodies, kept out of task files so formatting changes do not touch logic |
| `tasks/` | One file per flowchart step, numbered for traceability |

### 3.5 Main folder 2 — `playbooks/AkamaiVersionConfigPush/akamai/`

Three thin entrypoints. No logic — each sets `akamai_stage` and includes the role, so all logic stays in one lintable place.

```yaml
# playbooks/AkamaiVersionConfigPush/akamai/02_akamai_deploy.yml
---
- name: AkamaiVersionConfigPush - deployment execution
  hosts: "{{ target_env | default('PROD') }}"
  gather_facts: false
  vars:
    akamai_stage: deploy
  roles:
    - akamai
```

`group_vars/all.yml` sits beside the playbooks and is auto-loaded relative to the playbook directory. Non-secret shared values only — credentials come from AAP.

---

## 4. Step 11 — the branching activation

The flowchart branches at "Requested Version = Current Staging Version?":

- **Yes** -> Push Directly to Production -> Validate Activation
- **No**  -> Push to Staging -> Validate Activation -> Push to Production -> Validate Activation

`11a_activate.yml` is a **single parameterised file**, included once or twice:

```yaml
# tasks/11_determine_path.yml (shape only)
- name: Push to staging first when requested version is not already on staging
  ansible.builtin.include_tasks: 11a_activate.yml
  vars:
    target_network: STAGING
  when: requested_version != current_staging_version

- name: Push to production
  ansible.builtin.include_tasks: 11a_activate.yml
  vars:
    target_network: PRODUCTION
```

**Do not write two near-identical activate files.** You will fix a bug in one and not the other, and the one you miss will be the production path.

---

## 5. Open items to resolve before writing YAML

### 5.1 Carrying state across the approval node

JT #1 produces artifacts JT #2 needs: the diff, the captured last-known-stable production version, the resolved approver. Crossing an approval node requires **`set_stats`**.

**Verify:** does your AAP configuration permit `set_stats` artifact passing between workflow nodes? If restricted, the split needs a different carrier — most likely the ctask work notes, which you are writing to anyway.

### 5.2 Where "last known stable version" comes from

Step 20 must read a value **captured at step 6 and persisted before any activation**. Reading it after a failed push risks reading the broken version and rolling forward into the failure. Make this an explicit output contract of `06_akamai_metadata.yml`.

### 5.3 Does `local.akamai` provide a rule-tree diff?

If yes, `filter_plugins/ruletree.py` is dead code before it is written — delete the folder. If no, the plugin is justified because it is genuinely absent, not because it is convenient.

### 5.4 Verify `local.akamai.properties_info` return keys

`productionVersion` / `stagingVersion` / `latestVersion` are **inferred**, not confirmed. Check the module's `RETURN` block and correct `06_akamai_metadata.yml` if they differ.

### 5.5 Other standing blockers

- READ-WRITE credential grant
- Compliance record account-level requirement, confirmed against the production account
- Service account API client for AAP
- `cacert-pnc.pem` baked into the AAP Execution Environment image
- A promotable candidate version on `prp_591325` so activation-path testing can proceed

---

## 6. Design rules this structure enforces

| Rule | How the structure enforces it |
|---|---|
| Promote-only | No `create_version.yml`, no `resolve_source.yml`, no PAPI write modules outside activation |
| `network` never comes from a survey | Only defined in `inventories/akamai/{qa,prod}/` |
| `peerReviewedBy` comes from the CHG approver | Derived in `05_approver_routing.yml`, not an input |
| No blanket warning acknowledgment | `acknowledgeAllWarnings: true` is prohibited. Warnings require an explicit decision in `11a_activate.yml` |
| Rollback = redeploy last-known-stable | `21_redeploy_stable.yml` activates a previously stable version; never reverts or edits one |
| Artifacts survive the ephemeral EE | Reports written to ctask work notes, not `artifacts/` on disk |
| No reimplementation of `local.akamai` | No `library/` folder in the role |
| Every task traces to the spec | Numbered filenames map 1:1 to flowchart steps |

---

## 7. Validation before commit

Run in the PNC venv — this cannot be validated outside it because `local.akamai` is an internal collection:

```bash
ansible-lint playbooks/AkamaiVersionConfigPush/akamai/

ansible-playbook --syntax-check \
  -i inventories/akamai/qa/aki_qa.yml \
  playbooks/AkamaiVersionConfigPush/akamai/01_akamai_prechecks.yml
```

If the syntax check reports *"the role 'akamai' was not found"*, `roles_path` in `ansible.cfg` is missing the nested main folder. See section 0.1.

Then a `--check` dry run against QA before any live activation.
