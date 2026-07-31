# Akamai Config Push Automation — Structure & Variable Reference (Revision 2)

**Project:** Enable Application Teams to Self-Serve Akamai Delivery Config Pushes
**Epic:** SSPA-1 | **Stories:** SSPA-4, SSPA-8, SSPA-14
**Execution platform:** Ansible Automation Platform (AAP) Job Templates with surveys
**Purpose:** Development kickoff reference — what each folder does, what variables exist, where they live, why.

---

## 1. Top-Level Folder Map

| Path | What It Is | Why It Exists | Owner |
|---|---|---|---|
| `ansible.cfg` | Runtime configuration | Pins roles/collections paths so runs are reproducible off a clean checkout, not off one operator's environment | Dev, set once |
| `requirements.txt` | Python dependencies | Must be **baked into the AAP Execution Environment image** — pip cannot run at job time | Platform / EE team |
| `collections/requirements.yml` | Galaxy collection manifest | Versioned dependency declaration; an unpinned collection can change ServiceNow behaviour between two runs of identical code | Platform / EE team |
| `inventories/` | Environment definitions (staging, production) | Holds each environment's property catalog and network. Bound to a Job Template — see §2 | Dev + Security |
| `playbooks/` | Entry-point playbook | Orchestration only: ordering, conditionals, and the survey-variable mapping layer | Dev |
| `roles/` | Reusable task bundles | One role per external system, each independently testable | Dev |

**Removed since Revision 1:** `inventories/*/group_vars/all/vault.yml` and the `artifacts/` directory. Under AAP, secrets come from Credentials rather than vaulted repo files, and the execution environment is an ephemeral container — a diff written to local disk does not survive the job.

---

## 2. Execution Model — What Actually Separates Staging From Production

Revision 1 claimed the control was "switching the `-i` flag." Under AAP that is wrong, and stating it to a security reviewer would be a problem.

| Layer | Staging | Production | What It Prevents |
|---|---|---|---|
| Job Template | `SSPA – Akamai Push (Staging)` | `SSPA – Akamai Push (Production)` | A single template with a network dropdown would attach production credentials to a job anyone can launch "for staging" |
| Bound inventory | `inventories/staging` | `inventories/production` | Wrong property catalog cannot be loaded |
| Bound credential | Staging EdgeGrid (READ-WRITE, staging scope) | Production EdgeGrid (READ-WRITE) | Staging job physically lacks permission to activate production |
| RBAC | App teams: Execute | App teams: Execute, plus **Approval node** | Unreviewed production launch |
| Network value | Derived from `sspa_env` in inventory | Derived from `sspa_env` in inventory | **`network` is never a survey field** — it cannot be typed by a requester |

---

## 3. Variable Precedence — The Trap

AAP injects survey answers as `extra_vars`, which is the **highest** precedence in Ansible. A survey field silently beats both `group_vars` and role `defaults`.

| Source | Precedence | Used For | Example |
|---|---|---|---|
| Role `defaults/main.yml` | Lowest | Behavioural defaults meant to be overridden | `akamai_poll_interval` |
| Inventory `group_vars/all/vars.yml` | Middle | Environment facts | `sspa_property_catalog`, `sspa_akamai_network` |
| AAP Credential injection | High | Secrets | `akamai_edgegrid`, `snow_password` |
| Survey → `extra_vars` | **Highest** | Per-run request inputs | `sspa_in_chg_number` |

**Two rules that follow from this:**

1. Every survey variable is namespaced `sspa_in_*` and is **never** consumed directly by a role. The playbook `vars:` block maps `sspa_in_*` → role variables. Every override is therefore deliberate and greppable.
2. Role `defaults/` is used, never role `vars/`. `vars/` sits near the top of precedence and cannot be overridden by inventory — the classic symptom is "my group_vars value is being ignored."

---

## 4. Survey Input Contract

AAP surveys have **no conditional field logic**. Every field renders on every launch, so the reason-to-required-field mapping for the compliance record must be asserted in `preflight.yml`, not enforced by the form.

| Survey Variable | Type | Required | Why It Exists |
|---|---|---|---|
| `sspa_in_chg_number` | Text, regex `^CHG\d{7}$` | Yes | Input to the SSPA-4/SSPA-8 authorisation gate |
| `sspa_in_ctask_number` | Text | Yes | Write-back target for SSPA-14 |
| `sspa_in_app_mnemonic` | **Multiple choice** | Yes | Resolves to contract/group/property. Free text would let a typo activate someone else's property |
| `sspa_in_source_mode` | Choice: `promote_version`, `apply_rules` | Yes | Determines whether a version is created or an existing one promoted — see §11 |
| `sspa_in_source_version` | Integer | Conditional | The version to promote |
| `sspa_in_note` | Text | No | Recorded on the Akamai activation |
| `sspa_in_notify_emails` | Text (comma-separated) | No | Required by PAPI; falls back to the environment default |
| `sspa_in_dry_run` | Boolean, **default true** | Yes | Runs preflight + diff, stops before activation |
| `sspa_in_noncompliance_reason` | Choice: `NONE`, `NO_PRODUCTION_TRAFFIC`, `EMERGENCY`, `OTHER` | Prod only | Drives compliance record shape |
| `sspa_in_other_reason` | Text | If `OTHER` | Required sub-field |
| `sspa_in_customer_email` | Text | If `NONE` | Required sub-field |
| `sspa_in_unit_tested` | Boolean | If `NONE` | Required sub-field |

**Never in the survey:** any credential, `contractId` / `groupId` / `propertyId`, `network`, `ruleFormat`, poll intervals, `acknowledgeAllWarnings`, or a staging-gate override.

`peerReviewedBy` is deliberately **not** a survey field. It is read from the CHG approver. A requester certifying their own peer review is not a control.

---

## 5. Roles

| Role | Story | Writes? | Responsibility | Failure Semantics |
|---|---|---|---|---|
| `snow_chg_validate` | SSPA-4, SSPA-8 | No | Verify `state == Implement` **AND** `approval == Approved`, and that now is inside the change window | Refuses to proceed. Cannot cause an incident |
| `akamai_property` | SSPA-1 | Yes | The only role that mutates Akamai state | Triggers rollback only if an activation was actually submitted |
| `snow_ctask_update` | SSPA-14 | Yes | Append outcome + diff to the change task | `failed_when: false` — must never mask or reverse a real activation outcome |

### `akamai_property/` internals

| Path | Purpose | Design Principle |
|---|---|---|
| `library/akamai_papi.py` | EdgeGrid EG1-HMAC-SHA256 signing + all PAPI operations | Ansible auto-loads modules from `library/`. In Revision 1 this file had no declared home and would have broken on a clean checkout |
| `filter_plugins/ruletree.py` | Strips `uuid`/`templateUuid`/`templateLink`, canonicalises key order | Makes the diff readable. A noisy diff gets rubber-stamped, which is worse than no diff |
| `defaults/main.yml` | Poll interval, timeout, rule format, fast fallback | Lowest precedence, so inventory can override |
| `meta/argument_specs.yml` | Validated role interface | Turns missing required variables into a clear error, not a runtime `KeyError` |
| `tasks/main.yml` | `block` / `rescue` / `always` sequencer | Reading one short file shows the whole flow |
| `tasks/preflight.yml` | Property resolution, active-version read, 409 concurrency check, staging gate | Fail fast, fail cheap, no side effects |
| `tasks/resolve_source.yml` | Determines the candidate version | Replaces `create_version.yml` — see §11 |
| `tasks/diff.yml` | Fetch both rule trees, normalise, compare | The evidence step. This is what replaces the Akamai engineer on the change call |
| `tasks/activate.yml` | Build compliance record, submit activation | The point of no return |
| `tasks/poll_status.yml` | Poll dedicated activation endpoint | Async boundary; retry/backoff logic isolated |
| `tasks/rollback.yml` | Re-activate previous version with fast fallback | Recovery is pre-written and pre-tested, runnable standalone |

---

## 6. Non-Secret Configuration Variables

| Variable | Defined In | Consumed By | Why |
|---|---|---|---|
| `sspa_env` | inventory `group_vars` | playbook assertions | Single source of truth for which environment this is |
| `sspa_akamai_network` | inventory `group_vars` | `activate.yml`, `preflight.yml` | Derived from environment, never from survey |
| `sspa_property_catalog` | inventory `group_vars` | playbook `vars:` | Maps mnemonic → contract/group/property. Requesters pick a name; IDs are never typed |
| `sspa_require_staging_first` | inventory `group_vars` | `preflight.yml` | Policy: production requires prior staging activation |
| `sspa_notify_emails_default` | inventory `group_vars` | playbook `vars:` | PAPI requires `notifyEmails`; fallback prevents an empty list |
| `akamai_rule_format` | role `defaults` | both rule-tree GETs | **Pinned, never `latest`.** Under `latest` a schema change makes a property diff against itself |
| `akamai_poll_interval` / `akamai_poll_timeout` | role `defaults` | `poll_status.yml` | Activation is a long-running job, not request/response |
| `akamai_use_fast_fallback` | role `defaults` | `activate.yml`, `rollback.yml` | Fast rollback path |
| `akamai_fail_on_warnings` | role `defaults` | `activate.yml` | Warnings stop the run; there is no blanket acknowledgement |

## 7. Secrets — AAP Credentials, Not Vault Files

| Variable | Delivered By | Consumed By | Why Not a Survey Field |
|---|---|---|---|
| `akamai_edgegrid` (`host`, `client_token`, `client_secret`, `access_token`) | Custom AAP Credential Type | `akamai_papi.py` | Survey values are stored as job extra_vars and readable via the job detail and API |
| `snow_username` / `snow_password` | AAP Credential | both ServiceNow roles | Same |
| `snow_instance` | inventory `group_vars` | both ServiceNow roles | Not secret, but environment-specific |

## 8. Runtime Facts Derived During the Run

| Fact | Set In | Used By | Meaning |
|---|---|---|---|
| `sspa_active_version` | `preflight.yml` | `diff.yml`, `rollback.yml` | Last known good — the rollback target |
| `sspa_candidate_version` | `resolve_source.yml` | `diff.yml`, `activate.yml` | What is about to go live |
| `sspa_diff_summary` | `diff.yml` | `snow_ctask_update` | The audit evidence |
| `sspa_activation_id` | `activate.yml` | `poll_status.yml`, rescue block | Its existence is the signal that production state changed |
| `sspa_peer_reviewed_by` | `snow_chg_validate` | `activate.yml` | Compliance record field, sourced from the CHG approver |
| `sspa_result` | `main.yml` | `snow_ctask_update` | `SUCCESS` / `FAILED` / `ROLLED_BACK` |

---

## 9. Security Controls

| # | Control | Prevents |
|---|---|---|
| 1 | Separate Job Templates per environment, each with its own credential | Production credentials being reachable from a staging-intent launch |
| 2 | Network derived from inventory, not survey | A requester selecting PRODUCTION on the staging template |
| 3 | Mnemonic multiple-choice, IDs from catalog | A typo activating an unrelated property |
| 4 | CHG gate runs first, read-only | Unapproved production change |
| 5 | 409 concurrency pre-check | Colliding with an in-flight activation |
| 6 | Dry-run defaults to true | Accidental activation on first use |
| 7 | No `acknowledgeAllWarnings` | Requesters normalising away the warning gate |
| 8 | `peerReviewedBy` sourced from ServiceNow | Self-certified peer review |
| 9 | Normalised diff | A noise-filled diff being rubber-stamped |
| 10 | `no_log: true` on every credentialed task | Secrets in AAP job output |
| 11 | Rollback conditional on `sspa_activation_id` existing | Rollback firing after a harmless preflight failure |

## 10. Execution Flow

| Stage | File | Fails If |
|---|---|---|
| 1 | playbook `pre_tasks` | Mnemonic not onboarded, or network/environment mismatch |
| 2 | `snow_chg_validate` | CHG missing, not Implement, not Approved, outside window |
| 3 | `preflight.yml` | Property unresolvable, nothing active, activation in flight, staging gate unmet |
| 4 | `resolve_source.yml` | No source version, or version already live |
| 5 | `diff.yml` | Rule tree fetch fails, or diff is empty after normalisation |
| 6 | — | **Stops here if `dry_run`** |
| 7 | `activate.yml` | Compliance record rejected, warnings present, 403 on write scope |
| 8 | `poll_status.yml` | Terminal state is `FAILED` or `ABORTED`, or timeout |
| 9 | `rollback.yml` (rescue) | Only reached if an activation was submitted |
| 10 | `snow_ctask_update` | Always runs; never alters the outcome |

---

## 11. Unresolved Design Decisions

These are open questions, not implementation gaps. Both need a decision before further code is written against this structure.

| # | Question | Why It Matters |
|---|---|---|
| 1 | **Does PNC's in-house `local.akamai` collection already provide EdgeGrid signing and PAPI modules?** | The `Boundary_Protection_Automation` repo has a `local.akamai` namespace and an `aki_access_auth` role rendering the same four EdgeGrid values from `akamai.auth_list`. If it covers PAPI, `akamai_papi.py` is a second, unblessed implementation of credential signing — a maintenance and audit problem security would reasonably object to. It should then be deleted, and `collections/requirements.yml` and this document revised |
| 2 | **`promote_version` or `apply_rules`?** | Revision 1 cloned the active version and then diffed against it, which yields an empty diff by construction. Revision 2 makes both modes explicit, but only one is likely to be the real workflow. If app teams edit in the Property Manager UI, `apply_rules` is dead code and should be removed |
| 3 | **Is the diff a gate or an artifact?** | "Fail the run if changes fall outside an allowlist" and "attach the diff to the ctask" are different implementations. The epic reads like the first; the code implements the second |
| 4 | **Which repo does this live in?** | If it belongs inside `Boundary_Protection_Automation`, it inherits that repo's `ansible.cfg`, collections path and conventions, and this standalone layout is wrong |

## 12. Documentation Note for Review

Two source tickets carry acceptance criteria that do not match the confirmed business rule:

| Source | States | Implemented |
|---|---|---|
| SSPA-4 | `"Implement"` OR `"Approved"` | — |
| SSPA-8 | Only `"Implement"` is accepted | — |
| **Confirmed rule** | — | `state == Implement` **AND** `approval == Approved` — two fields, both required |

Recommend updating both ticket descriptions to match. On a compliance-facing change gate, a mismatch between documented criteria and implemented logic is what surfaces during an audit.

Separately: the SSPA-1 epic shipped with Success Criteria, Acceptance Criteria, Stakeholders and Dependencies empty. Nothing in this document substitutes for filling them in.

## 13. Story Traceability

| Story | Deliverable | Status |
|---|---|---|
| SSPA-4 | `snow_chg_validate` — state + approval gate | Code complete, untested |
| SSPA-8 | `snow_chg_validate` — window enforcement | Code complete, untested |
| SSPA-14 | `snow_ctask_update` — result + diff write-back | Code complete, untested |
| SSPA-1 | `akamai_property` — full push workflow | Code complete, blocked on §11 and the dependency list |
