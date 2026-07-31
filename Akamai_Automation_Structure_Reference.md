# Akamai Config Push Automation — Project Structure & Variable Reference

**Project:** Enable Application Teams to Self-Serve Akamai Delivery Config Pushes
**Purpose of this document:** Development kickoff reference — what each folder does, what variables exist, where they live, and why.

---

## 1. Top-Level Folder Map

| Folder / File | What It Is | Why It Exists | Who Touches It |
|---|---|---|---|
| `ansible.cfg` | Ansible runtime configuration | Points Ansible at our roles, collections, inventory, and vault password so no CLI flags are needed on every run | Set once by dev, rarely changed |
| `requirements.txt` | Python package list | The EdgeGrid signing library and HTTP client must exist on the AAP execution node | Platform/EE team |
| `collections/` | Ansible Galaxy collection list | Declares `servicenow.itsm` dependency — must be baked into the AAP Execution Environment | Platform/EE team |
| `inventories/` | Environment definitions (staging, production) | **Physical guardrail** — separates staging and prod credentials so prod cannot be touched from the staging inventory | Dev + Security |
| `playbooks/` | Entry-point playbook | The single command AAP runs; orchestrates all roles in order | Dev |
| `roles/` | Reusable task bundles | Each role = one logical responsibility, independently testable | Dev |
| `artifacts/` | Diff output storage | Where version-comparison diffs are written for human review and audit evidence | Read-only for reviewers; gitignored |

---

## 2. `inventories/` — Environment Separation

### Why two inventories instead of one variable?

Because Akamai is an API (not SSH servers), the inventory's only job is to **separate environments**. Switching from staging to production means switching the `-i` flag. Since each inventory holds its own vault with its own Akamai API client credentials, a staging run **physically cannot** activate production — the credentials it loads don't have that permission. This is defence-in-depth, not just policy.

| Path | Contents | Encrypted? | Notes |
|---|---|---|---|
| `inventories/staging/hosts.yml` | `localhost` only | No | Akamai is an API, so all tasks run locally |
| `inventories/staging/group_vars/all/vars.yml` | Non-secret config | No | Property IDs, network target, poll settings |
| `inventories/staging/group_vars/all/vault.yml` | Credentials | **Yes** (ansible-vault) | 4 EdgeGrid values + SNOW service account |
| `inventories/production/…` | Same structure | Same | Different property/API-client values |

---

## 3. `roles/` — What Each Role Does

| Role | Reads or Writes? | Responsibility |
|---|---|---|
| `snow_chg_validate` | **READ ONLY** | Gate check before deployment: is the CHG approved, in Implement state, and inside its window? |
| `akamai_property` | Read + Write (Akamai) | Create version → diff → activate → poll → rollback |
| `snow_ctask_update` | **WRITE** | Append deployment result to the ctask work notes after execution |

### 3.1 `roles/snow_chg_validate/`

| File | Purpose |
|---|---|
| `tasks/main.yml` | Query CHG via `servicenow.itsm.change_request_info`, then three assertions: state, approval, time window |

**Design constraint:** this role must not modify any ServiceNow record. It only reads. If any check fails, the play stops immediately with a message naming which condition failed and what the actual value was.

### 3.2 `roles/akamai_property/`

| File | Step | Purpose |
|---|---|---|
| `defaults/main.yml` | — | Safe fallback values so the role never runs with undefined vars |
| `tasks/main.yml` | — | Orchestrator: includes the task files below in order |
| `tasks/preflight.yml` | 0 | Render temp `.edgerc` from vault; **concurrency check** (fail if another activation is PENDING); fetch current active versions |
| `tasks/create_version.yml` | 1 | Clone the currently-active version into a new editable version |
| `tasks/diff.yml` | 2 | Fetch both rule trees, produce a unified diff, save to `artifacts/` |
| `tasks/activate.yml` | 3 | POST the activation to STAGING or PRODUCTION; capture activation ID |
| `tasks/poll_status.yml` | 4 | Poll every 10s up to 60 min until `ACTIVE` / `FAILED` / `ABORTED` |
| `tasks/rollback.yml` | Failure path | Re-activate the previous stable version (Fast Fallback if inside the 60-min window) |
| `templates/edgerc.j2` | — | Renders the temporary credentials file from vault variables at runtime |
| `files/akamai_papi.py` | — | Python EdgeGrid signing wrapper — Ansible's `uri` module **cannot** sign EdgeGrid requests |

### 3.3 `roles/snow_ctask_update/`

| File | Purpose |
|---|---|
| `tasks/main.yml` | Build a formatted deployment report and append it to the ctask `work_notes` field |

**Why `work_notes` and not `comments`:** `work_notes` is the internal, append-only journal. `comments` is customer-visible. Deployment logs and error traces belong in the internal field.

---

## 4. Variable Reference — Complete Table

### 4.1 Non-Secret Variables (`vars.yml`)

| Variable | Example Value | Where Defined | Used By | Why It Exists |
|---|---|---|---|---|
| `akamai_network` | `PRODUCTION` | `vars.yml` per inventory | All Akamai tasks, main playbook | Decides target network **and** whether the SNOW gate runs at all |
| `environment_name` | `production` | `vars.yml` | Logging/summary | Human-readable label in output |
| `akamai_property_id` | `prp_123456` | `vars.yml` | Every PAPI call | Identifies which config rulebook we're changing |
| `akamai_contract_id` | `ctr_1-ABC123` | `vars.yml` | Every PAPI call | **Mandatory query param** — omitting it returns 404 even when the property exists |
| `akamai_group_id` | `grp_98765` | `vars.yml` | Every PAPI call | Same as above — permission/billing boundary resolution |
| `akamai_host` | `akab-xxxx.luna.akamaiapis.net` | `vars.yml` | `edgerc.j2` | Per-API-client endpoint; not a shared Akamai URL |
| `snow_instance` | `pncint.service-now.com` | `vars.yml` | Both SNOW roles | Target ServiceNow instance |
| `activation_poll_interval` | `10` | `vars.yml` | `poll_status.yml` | Seconds between status checks — matches design spec |
| `activation_poll_max_attempts` | `360` | `vars.yml` | `poll_status.yml` | 360 × 10s = 60-minute timeout — matches design spec |
| `rollback_on_failure` | `true` | `vars.yml` | Main playbook `rescue` block | Kill-switch to disable auto-rollback during testing |
| `use_fast_fallback` | `true` | `vars.yml` | `activate.yml`, `rollback.yml` | Enables Akamai's 60-min instant-revert window |
| `edgerc_path` | `/tmp/.edgerc_akamai_<epoch>` | `vars.yml` | `preflight.yml`, cleanup | Temp credential file path; unique per run to avoid collisions |
| `artifacts_dir` | `<project>/artifacts` | `vars.yml` | `diff.yml` | Where diffs are written for review |
| `akamai_notify_email` | `team@pnc.com` | `defaults/main.yml` | `activate.yml` | Akamai sends activation notifications here |

### 4.2 Secret Variables (`vault.yml` — encrypted)

| Variable | What It Is | Used By | Security Notes |
|---|---|---|---|
| `vault_akamai_client_token` | Public credential identifier | `edgerc.j2` | Not highly sensitive but kept with the set |
| `vault_akamai_access_token` | Links credential to API grants | `edgerc.j2` | Rotate together with the others |
| `vault_akamai_client_secret` | **The signing secret** | `edgerc.j2` | Shown **once** at creation in Akamai IAM — irrecoverable if lost |
| `vault_snow_username` | SNOW service account | Both SNOW roles | Dedicated service account, not a personal login |
| `vault_snow_password` | SNOW service account password | Both SNOW roles | Or swap for OAuth `client_id`/`client_secret` |

### 4.3 Runtime Variables (passed via `-e` at job launch)

| Variable | Required When | Example | Purpose |
|---|---|---|---|
| `chg_number` | Production only | `CHG0012345` | The change record to validate against |
| `ctask_number` | Production only | `CTASK0098765` | The change task to write results into |

### 4.4 Derived Facts (set during the run, not configured)

| Fact | Set By | Consumed By | Meaning |
|---|---|---|---|
| `akamai_current_production_version` | `preflight.yml` | `create_version.yml`, `rollback.yml` | Version currently live on prod |
| `akamai_current_staging_version` | `preflight.yml` | `create_version.yml`, `rollback.yml` | Version currently live on staging |
| `akamai_base_version` | `create_version.yml` | `diff.yml`, ctask report | The version we cloned from |
| `akamai_new_version` | `create_version.yml` | `activate.yml`, `diff.yml`, ctask report | The newly created version number |
| `akamai_activation_id` | `activate.yml` | `poll_status.yml`, ctask report | Handle used to poll activation progress |
| `akamai_activation_final_status` | `poll_status.yml` | Main playbook, ctask report | `ACTIVE` / `FAILED` / `ABORTED` |
| `deployment_status` | Main playbook | `snow_ctask_update` | `PASS` or `FAIL` written to ctask |
| `rollback_performed` | Main playbook `rescue` | ctask report | Whether rollback ran |
| `rollback_target_version` | `rollback.yml` | ctask report | Version we reverted to |
| `chg_state`, `chg_approval`, `chg_start`, `chg_end` | `snow_chg_validate` | Assertions in the same role | The four fields the gate check evaluates |

---

## 5. Security Controls Built Into the Design

| Control | Where Implemented | What It Prevents |
|---|---|---|
| Credentials encrypted at rest | `vault.yml` via `ansible-vault` | Secrets in plaintext in Git |
| Temp `.edgerc` deleted after run | `post_tasks` cleanup, always executes | Credentials lingering on the execution node |
| `.edgerc` written with mode `0600` | `preflight.yml` template task | Other users on the host reading credentials |
| `no_log: true` on credential-touching tasks | All SNOW/Akamai calls | Secrets leaking into AAP job output and logs |
| Separate API client per environment | Separate `vault.yml` per inventory | A staging run activating production |
| Concurrency preflight | `preflight.yml` | Two activations racing on the same property |
| `artifacts/` gitignored | `.gitignore` | Config content and diffs entering source control |
| CHG gate is read-only | `snow_chg_validate` design | Automation accidentally mutating change records |

---

## 6. Execution Flow — Ordered

| # | Stage | Runs When | Fails The Job If… |
|---|---|---|---|
| 1 | Gather facts | Always | — |
| 2 | CHG validation | `akamai_network == PRODUCTION` | State ≠ implement, approval ≠ approved, or outside window |
| 3 | Render `.edgerc` | Always | Vault vars missing |
| 4 | Concurrency preflight | Always | Another activation already `PENDING` |
| 5 | Fetch active versions | Always | Akamai API unreachable / 404 (check contract+group IDs) |
| 6 | Create new version | Always | Clone rejected by PAPI |
| 7 | Generate diff | Always | Rule tree fetch fails |
| 8 | Activate | Always | Validation errors in rule tree, or 409 concurrency |
| 9 | Poll status | Always | Terminal status ≠ `ACTIVE`, or 60-min timeout |
| 10 | Rollback | Only on failure, if `rollback_on_failure` | Rollback activation itself fails → escalate to human |
| 11 | Update ctask | Production + `ctask_number` provided | Retried 3×, then warns (does not mask the deployment result) |
| 12 | Delete `.edgerc` | Always, even on failure | — |
| 13 | Final status | Always | Play exits non-zero if `deployment_status == FAIL` |

---

## 7. Open Dependencies — Blockers Before Development Can Complete

| # | Item | Owner | Why It Blocks | Risk If Unresolved |
|---|---|---|---|---|
| 1 | Exact SNOW field names for state / approval / window | ServiceNow admin | Field names are instance-customisable | Gate check silently reads wrong fields |
| 2 | Actual value format of `state` and `approval` | ServiceNow admin | Could be `implement`, `Implement`, or a numeric code | False rejections on every run |
| 3 | Timezone of SNOW date fields | ServiceNow admin | Window comparison must be like-for-like | Off-by-hours pass/fail at window edges |
| 4 | ctask table name confirmation | ServiceNow admin | May be `change_task` or custom | Write step fails at runtime |
| 5 | `servicenow.itsm` in the AAP Execution Environment | Platform / EE team | Collection must be in the EE image | Playbook fails on first task |
| 6 | `contractId` + `groupId` for target BOP mnemonic | Akamai TAM | Mandatory on every PAPI call | 404 on every Akamai call |
| 7 | READ-WRITE API client credentials (per environment) | Akamai TAM / IAM admin | Read-only cannot activate | 403 at the activation step |
| 8 | Egress IP allowlisting for the AAP node | Network team | Outbound to `*.luna.akamaiapis.net` | Connection timeouts |
| 9 | Compliance record requirement for activations | Akamai TAM | Some accounts mandate it on prod activations | Activation rejected on first prod run |
| 10 | SNOW service account + role grants | ServiceNow admin | Needs read on `change_request`, write on `change_task` | 403 on gate check or ctask update |

---

## 8. Requirements Clarification — Applied

The change-gate condition was documented inconsistently in the original requirements. Two different phrasings appeared:

- One version implied `"Implement" OR "Approved"` — suggesting either value of a single field was acceptable.
- Another version stated `Only "Implement" status is accepted` — with no mention of approval at all.

**Confirmed business rule, implemented in this design:**

```
state    == "Implement"   AND
approval == "Approved"
```

These are **two separate fields on the change record**, and **both** must be true. A change sitting in Implement state without approval must be blocked, and an approved change outside its Implement state must also be blocked.

**Recommendation:** update the written requirement text to match, so the documented acceptance criteria and the implemented logic agree. On a compliance-facing production gate, a mismatch between documented criteria and shipped behaviour is exactly the kind of discrepancy that surfaces during audit review.
