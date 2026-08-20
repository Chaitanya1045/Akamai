## SSPA-7: Temporary Report Storage Validation for Akamai Automation

Adds secure temporary report-storage validation inside the active AAP Execution Environment using an execution-specific workspace under `/tmp/akamai-reports/<execution_id>/`.

### What's New

- Scoped this PR to **SSPA-7 temporary report-storage changes only**.
- Removed unrelated ServiceNow, CTASK, collection, and dispatcher changes from the SSPA-7 scope.
- Added execution-specific temporary workspace: `/tmp/akamai-reports/<execution_id>/`.
- Added `/tmp` and workspace availability validation.
- Added protected path validation to prevent traversal and unsafe path components.
- Added safe execution-ID validation using only `A-Z`, `a-z`, `0-9`, `.`, `_`, and `-`.
- Added secure private directory permissions with `0700`.
- Added optional approved group-readable directory permissions with `0750`.
- Added secure private file permissions with `0600`.
- Added optional approved group-readable file permissions with `0640`.
- Added protected access-group validation and resulting group-ownership verification.
- Added real write, read, content, and list verification using a hidden temporary probe file.
- Fixed hidden probe discovery using `hidden: true`.
- Added automatic verification-probe cleanup after I/O validation.
- Added explicit execution-workspace cleanup using `report_storage_cleanup.yml`.
- Added cleanup-path containment validation before deleting the execution workspace.
- Added truthful storage metadata:
  - `configuration_validated`
  - `storage_verified`
  - `workspace_exists`
  - `report_exists`
  - `ephemeral`
  - `same_job_only`
  - `separate_job_transfer_required`
  - `cleanup_required`
- `report_exists` remains `false` because SSPA-7 validates storage only and does not create the final report.
- Final deployment-report naming/content ownership moved to the downstream report-generation story.
- Audit filename/rendering ownership remains with `common_logging`.
- Added same-job lifecycle documentation: `/tmp` is available only inside the active AAP job/Execution Environment.
- Documented that separate AAP jobs require an approved transfer/persistent-storage mechanism.
- Added shared execution-context validation for request ID, execution ID, environment, actor, launcher, and timestamp.
- Added deployed environment protection for `dev`, `rnd`, `qa`, and `prod`.
- Limited `test`/`local` fallback behavior to explicit standalone test mode.
- Separated logging identities:
  - `actor` = `akamai_version_push.report_storage`
  - `triggered_by` = actual AAP launcher
- Reused the existing `common_logging` framework for success and terminal failure handling.
- Split the previous large implementation into smaller task files:
  - `report_storage.yml`
  - `report_storage_context.yml`
  - `report_storage_validate.yml`
  - `report_storage_provision.yml`
  - `report_storage_verify.yml`
  - `report_storage_publish.yml`
  - `report_storage_cleanup.yml`
- Added step-by-step implementation, AAP QA, lifecycle/integration, and technical-lead review documentation.
- Added updated normal-run and Check Mode contract tests.
- Normal AAP execution was re-tested successfully for context validation, provisioning, write/read/list verification, metadata publication, probe cleanup, workspace cleanup, and `common_logging`.

### Error Codes

- `REPORT_STORAGE_CONFIG_INVALID` - Invalid storage root, permission, path, identifier, or configuration contract.
- `REPORT_ACCESS_GROUP_INVALID` - Configured protected OS group is invalid or unavailable.
- `REPORT_STORAGE_UNAVAILABLE` - Temporary storage or execution workspace provisioning/validation failed.
- `REPORT_STORAGE_WRITE_FAILED` - Write or file-permission verification failed.
- `REPORT_STORAGE_READ_FAILED` - Read or content verification failed.
- `REPORT_STORAGE_LIST_FAILED` - Hidden probe list verification failed.
- `REPORT_STORAGE_CLEANUP_FAILED` - Execution-specific workspace cleanup failed.

### Test Cases

- `TC00` - PR scope contains only SSPA-7 work -> Success.
- `TC01` - Normal private workspace validation -> Success with `0700` directory and `0600` file mode.
- `TC02` - AAP QA deployed-context validation -> Validate shared request/execution/environment/launcher context.
- `TC03` - AAP Check Mode -> Expected `configuration_validated=true`, `storage_verified=false`, `workspace_exists=false`, `report_exists=false`, `check_mode=true`.
- `TC04` - I/O verification disabled -> Success with `storage_verified=false`.
- `TC05` - Missing request ID in deployed mode -> Validation failure.
- `TC06` - Missing execution ID in deployed mode -> Validation failure.
- `TC07` - Missing launcher identity in deployed mode -> Validation failure.
- `TC08` - Invalid deployed environment -> Validation failure.
- `TC09` - `local`/`test` used outside explicit test mode -> Validation failure.
- `TC10` - Explicit standalone/local test fallback -> Success.
- `TC11` - Unsafe execution ID containing `/` -> Validation failure.
- `TC12` - Unsafe traversal-style execution ID -> Validation failure.
- `TC13` - Unsafe execution ID containing `\` -> Validation failure.
- `TC14` - Execution ID exceeds allowed length -> Validation failure.
- `TC15` - Request ID exceeds allowed length -> Validation failure.
- `TC16` - Approved storage-root contract tampered -> `REPORT_STORAGE_CONFIG_INVALID`.
- `TC17` - Unsafe file-permission contract -> `REPORT_STORAGE_CONFIG_INVALID`.
- `TC18` - Unsafe directory-permission contract -> `REPORT_STORAGE_CONFIG_INVALID`.
- `TC19` - Invalid access-group syntax -> `REPORT_STORAGE_CONFIG_INVALID`.
- `TC20` - Configured OS group unavailable -> `REPORT_ACCESS_GROUP_INVALID`.
- `TC21` - Approved group-readable mode -> Success with `0750` directory and `0640` file mode.
- `TC22` - Hidden probe listing -> Success using `hidden: true`.
- `TC23` - Verification probe cleanup -> Success.
- `TC24` - Published metadata contract -> `storage_verified=true`, `workspace_exists=true`, `report_exists=false`.
- `TC25` - Same-job downstream task uses published workspace -> Success.
- `TC26` - Explicit workspace cleanup after downstream consumers -> Success.
- `TC27` - Unsafe cleanup target -> `REPORT_STORAGE_CLEANUP_FAILED`.
- `TC28` - Logging actor and AAP launcher identity separation -> Success.
- `TC29` - `common_logging` success events -> Success.
- `TC30` - Standalone audit rendering through `common_logging` -> Validate when event history exists.
- `TC31` - Controlled write failure -> `REPORT_STORAGE_WRITE_FAILED`.
- `TC32` - Controlled read failure -> `REPORT_STORAGE_READ_FAILED`.
- `TC33` - Controlled list failure -> `REPORT_STORAGE_LIST_FAILED`.
- `TC34` - Controlled workspace provisioning failure -> `REPORT_STORAGE_UNAVAILABLE`.
- `TC35` - Controlled cleanup failure -> `REPORT_STORAGE_CLEANUP_FAILED`.
- `TC36` - UTF-8 / encoding validation -> Success.
- `TC37` - Automated normal-run contract -> Success.
- `TC38` - Automated Check Mode contract -> Run using actual AAP Job Type `Check`.

### Notes

- `/tmp/akamai-reports/<execution_id>/` is temporary/ephemeral storage inside the active AAP Execution Environment.
- Direct downstream access is supported only for tasks running in the **same AAP job/Execution Environment**.
- Separate AAP jobs must use an approved transfer/persistent-storage mechanism.
- SSPA-7 validates and prepares storage only; it does **not** generate the final deployment report.
- Final report naming/content/path is owned by the downstream report-generation story.
- `common_logging` owns audit naming and rendering.
- The verification probe is deleted immediately after successful I/O verification.
- The execution workspace is deleted only after all same-job consumers complete.
- Normal AAP Run mode has been validated successfully.
- Check Mode must be validated using actual AAP **Job Type = Check**; a normal Run with `CHECK` in request/execution IDs is not Check Mode.
