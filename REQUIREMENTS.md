# Requirements Document

**Project:** SAP Ariba User Management Class Library  
**Prepared by:** Requirements Analyst  
**Date:** 2026-05-20  
**Status:** Draft  
**Primary Stakeholder:** Dave Rogala

---

## 1. Executive Summary

This project delivers a .NET 10.0 class library that replaces a legacy SQL/SSIS-based ETL pipeline responsible for synchronizing users into SAP Ariba. The library consolidates user data from four source systems — Workday, Fieldglass, Microsoft Entra, and an ApprovalLimits lookup table — applies defined eligibility and field-mapping business rules, detects incremental changes against a maintained current-state table, and exposes results as four async delegates that a downstream file-generation library can consume. The library is designed to run within an Azure Function and replaces a process that was incompatible with Azure and incapable of incremental (Add/Update/Delete) loads.

---

## 2. Problem Statement

### 2.1 Background

User synchronization to SAP Ariba is currently performed by a SQL/SSIS-based ETL pipeline consisting of stored procedures, table-valued functions, views, and a C# SSIS script component. The pipeline generates a full `UserConsolidated.csv` file on each run, which is then delivered to Ariba via the legacy ITK job. Source data is imported daily from Workday (HcmWorkerPublic), Fieldglass (fdg.WorkerPublic), and Microsoft Entra. The pipeline was carried forward via a "lift-and-shift" migration when the organization moved from an on-premises Ariba instance to the cloud-hosted solution, and has not been substantially refactored since.

### 2.2 Problem to Solve

The existing pipeline has three compounding problems:
1. **Azure incompatibility:** SSIS is not available in Azure, and the organization's ETL infrastructure is being migrated to Azure.
2. **Full-file loads only:** The pipeline sends a complete Source-of-Truth file on every run. The target integration platform (SAP Integration Suite on BTP) supports and prefers incremental Add/Update and Delete files, which the current process cannot produce.
3. **Maintainability:** The pipeline is difficult to troubleshoot and nearly impossible to modify systematically without risk of breaking existing behavior. The logic is spread across multiple SQL objects and a compiled SSIS script.

### 2.3 Impact of Not Solving

Failure to replace this pipeline before the Azure migration deadline would leave user synchronization with no viable execution environment. Continuing to run full-file loads also creates unnecessary load on the integration platform and increases the risk of unintended user data overwrites.

---

## 3. Goals & Success Criteria

### 3.1 Business Goals

| Goal | Metric | Target |
|------|--------|--------|
| Enable Azure-compatible user sync | Pipeline executes successfully on Azure infrastructure | 100% of runs complete without SSIS dependency |
| Incremental load support | Ariba receives only changed records per run | Add/Update and Delete files contain only delta records |
| Improved maintainability | Time to diagnose and fix a field-mapping defect | Measurably reduced vs. current SQL/SSIS process |
| Azure migration readiness | Library integrated into Azure Function before migration deadline | Complete by end of 2026 |

### 3.2 User Goals

The primary consumer of this library is the **Azure Function** (and the developers who maintain it) that orchestrates the Ariba user sync pipeline. The library must:
- Be easy to integrate via standard .NET dependency injection patterns.
- Provide clear, actionable log output when records are skipped.
- Be independently testable without requiring live connections to source systems.

### 3.3 Out of Scope

The following are explicitly excluded from this library:

- **GroupConsolidated file generation** — handled separately (may be added in a future iteration).
- **Address.csv management** — the home-address lookup table for remote workers exists and is maintained outside this library.
- **File serialization and blob storage** — handled by an existing file-generation class library.
- **SAP BTP / Integration Suite configuration** — handled separately.
- **Azure Function host** — the function that calls this library is a separate concern.
- **Delivery of files to Ariba** — out of scope; handled by SAP Integration Suite on BTP.

---

## 4. Users & Stakeholders

### 4.1 User Personas

| Persona | Description | Technical Level | Volume |
|---------|-------------|-----------------|--------|
| Azure Function / Pipeline Orchestrator | The automated process that invokes this library's delegates on a scheduled basis | N/A (system) | 1 pipeline instance |
| Library Developer / Maintainer | Engineer who builds, tests, and maintains this library | High | 1–2 engineers |
| Ariba End Users | The ~22,000 employees and contractors whose accounts are managed in Ariba | Low–Med | ~22,000 |

### 4.2 Stakeholders

| Name | Role | Involvement |
|------|------|-------------|
| Dave Rogala | Primary developer / owner | Decision-maker, implementer |

### 4.3 External Parties

| Party | Nature of Interaction |
|-------|-----------------------|
| Workday | Source system; HcmWorkerPublic data is imported daily into the database |
| SAP Fieldglass | Source system; fdg.WorkerPublic data is imported daily into the database |
| Microsoft Entra | Source system; user identity, group membership, supervisor, phone, and email data is retrieved and stored in the database |
| SAP Ariba (via BTP) | Destination system; consumes the output files produced downstream of this library |

---

## 5. Functional Requirements

### 5.1 Core Workflows

**Workflow 1: Incremental Add/Update**

1. The Azure Function invokes the appropriate Add/Update delegate (parent or child realm).
2. The library queries the consolidated database to retrieve all currently eligible users for that realm, applying eligibility rules based on Entra group membership.
3. For each eligible user, the library assembles the full user record by joining data from HcmWorkerPublic, Fieldglass fdg.WorkerPublic (for contractors), Entra, and the ApprovalLimits table, and applying all field-mapping and business logic rules.
4. The library diffs the assembled records against the current-state table to identify new users (Add) and users whose tracked fields have changed (Update).
5. Records with missing required fields are skipped; the issue is logged via `Microsoft.Extensions.Logging`.
6. The delegate returns a `Task<List<AribaUserDto>>` containing only the changed/new records.
7. After a successful run, the current-state table is updated to reflect the new state of returned records.

**Workflow 2: Incremental Delete**

1. The Azure Function invokes the appropriate Delete delegate (parent or child realm).
2. The library queries the current-state table to identify users who were previously eligible for that realm but are no longer present in the eligible user set (e.g., removed from Entra group, terminated in Workday/Fieldglass).
3. The delegate returns a `Task<List<AribaUserDto>>` containing the records to be deleted, with sufficient identity fields for Ariba to process the deletion.
4. After a successful run, the corresponding records are removed from the current-state table.

---

### 5.2 User Eligibility Rules

Eligibility is determined by membership in one of three mutually exclusive Microsoft Entra groups. The group names are configurable via `AribaUserManagementOptions` (see section 5.5); default values are shown below:

| Options Property | Default Group Name | Included in Parent Realm | Included in Child Realm | Worker Type |
|------------------|--------------------|--------------------------|-------------------------|-------------|
| `BasicUserGroupName` | Ariba Basic User | ✓ | ✓ | Employee (Workday) |
| `BasicContractorGroupName` | Ariba Basic Contractor | ✓ | ✓ | Contractor (Fieldglass) |
| `InternationalUserGroupName` | Ariba International User | ✓ | ✗ | Employee (Workday) |

A user must belong to exactly one of these groups to be included in any output. Users in none of these groups are excluded entirely.

---

### 5.3 Functional Requirements List

| ID | Requirement | Priority | Acceptance Criteria |
|----|-------------|----------|---------------------|
| FR-001 | The library SHALL expose a Parent Realm Add/Update delegate returning `Task<List<AribaUserDto>>` containing new and changed records for all eligible users (Basic User + Basic Contractor + International User) | P0 | Delegate returns only records that are new or have changed tracked fields since the last run |
| FR-002 | The library SHALL expose a Parent Realm Delete delegate returning `Task<List<AribaUserDto>>` containing records for users who were previously eligible for the parent realm but are no longer | P0 | Delegate returns only records that existed in the current-state table but are no longer eligible |
| FR-003 | The library SHALL expose a Child Realm Add/Update delegate returning `Task<List<AribaUserDto>>` containing new and changed records for Basic Users and Basic Contractors only | P0 | International Users are excluded; only new or changed records are returned |
| FR-004 | The library SHALL expose a Child Realm Delete delegate returning `Task<List<AribaUserDto>>` containing records for Basic Users and Basic Contractors who were previously eligible for the child realm but are no longer | P0 | International Users are excluded; only previously-eligible users no longer in scope are returned |
| FR-005 | The library SHALL determine user eligibility by querying Entra group membership for the three defined groups whose names are read from `AribaUserManagementOptions` (`BasicUserGroupName`, `BasicContractorGroupName`, `InternationalUserGroupName`) | P0 | Only users in one of the three configured groups appear in any delegate output |
| FR-006 | The library SHALL maintain a current-state table in the database containing the last-known values of all trackable Ariba user fields | P0 | After each successful delegate invocation, the current-state table reflects the state of all returned records |
| FR-007 | The library SHALL diff assembled user records against the current-state table to determine Add/Update vs. no-change | P0 | Records with no field changes are not returned by Add/Update delegates |
| FR-008 | The library SHALL assemble user records by joining data from HcmWorkerPublic (employees), Fieldglass fdg.WorkerPublic (contractors), Entra, and the ApprovalLimits lookup table | P0 | Output records contain correctly sourced values for all active fields per the field mapping |
| FR-009 | The library SHALL apply the JobId field rule: if the user is a member of the Entra group named by `AribaUserManagementOptions.BranchUserGroupName`, JobId is blank; otherwise JobId is the value of `AribaUserManagementOptions.DefaultJobId` | P0 | BranchUser group members have empty JobId; all others have the configured `DefaultJobId` value |
| FR-010 | The library SHALL apply the EmailAddress fallback rule: if Email is null or blank and the execution environment is not production, use the address produced by formatting `AribaUserManagementOptions.NonProdEmailFallbackTemplate` with the current environment name (e.g. `cert` or `dev`) | P0 | Non-production records with no email address receive the environment-appropriate fallback address derived from the configured template |
| FR-011 | The library SHALL apply the GenericShipTo field rule: for employees, concatenate LocationCode and BuildingFloor from HcmWorkerPublic; if `IsRemote` is true, look up the H-prefixed address identifier from the home-address table; for contractors, leave blank | P0 | Employee records have location-based ShipTo; remote employee records have H-prefixed identifier; contractor records have blank ShipTo |
| FR-012 | The library SHALL look up the ApprovalLimit value from the ApprovalLimits table using the user's LevelCode from HcmWorkerPublic | P0 | ApprovalLimit in output matches the value in the ApprovalLimits table for the user's LevelCode |
| FR-013 | The library SHALL skip any user record that is missing a required field and log a structured warning via `Microsoft.Extensions.Logging` that identifies the user and the missing field(s) | P0 | Skipped records do not appear in delegate output; a log entry is written for each skipped record identifying UniqueName and field(s) in violation |
| FR-014 | The library SHALL derive the VanillaDeliverTo field value from the assembled Name field (WorkerNameNickname + WorkerNamePreferredLast) | P0 | VanillaDeliverTo matches Name in all output records |
| FR-015 | The library SHALL populate the Name field by concatenating WorkerNameNickname and WorkerNamePreferredLast from HcmWorkerPublic in the format `"{First} {Last}"` | P0 | Name field matches the concatenation pattern for all records |
| FR-016 | The library SHALL populate the Supervisor.UniqueName field from `manager.UserPrincipalName` in Entra | P0 | Supervisor.UniqueName contains the UPN of the user's Entra manager |
| FR-017 | The library SHALL populate the PasswordAdapter and Supervisor.PasswordAdapter fields from `AribaUserManagementOptions.PasswordAdapter` | P0 | All output records contain the configured `PasswordAdapter` value for both adapter fields |
| FR-018 | The library SHALL populate the TimeZoneID field from `AribaUserManagementOptions.TimeZoneId` | P0 | All output records contain the configured `TimeZoneId` value |
| FR-019 | The library SHALL populate the PurchasingUnit field from `AribaUserManagementOptions.PurchasingUnit` | P0 | All output records contain the configured `PurchasingUnit` value |
| FR-020 | The library SHALL populate the GenericBillingAddress field from `AribaUserManagementOptions.GenericBillingAddress` | P0 | All output records contain the configured `GenericBillingAddress` value |
| FR-021 | The library SHALL populate the ImportCtrl field from `AribaUserManagementOptions.ImportCtrl` | P0 | All output records contain the configured `ImportCtrl` value |
| FR-022 | The library SHALL populate the Phone field from the first entry in Entra's `businessPhones` array, or leave blank if the array is empty | P1 | Phone is populated from the first array entry; blank when array is empty or null |
| FR-023 | The library SHALL include unit tests for all field-mapping rules, eligibility logic, diff detection, and error-handling behavior using xUnit | P0 | Test suite passes; coverage includes all business logic branches identified in FR-005 through FR-022 |
| FR-024 | The library SHALL expose an `AribaUserManagementOptions` class registered with the .NET `IOptions<T>` pattern; all constant field values, Entra group names, and the non-production email fallback template SHALL be read from this class and SHALL NOT be hardcoded | P0 | All values listed in section 5.5 are sourced from `AribaUserManagementOptions`; no literal values for these fields appear in production code; options can be overridden in tests without code changes |

---

### 5.4 Data Requirements

**AribaUserDto** — the output DTO representing one user record. Fields correspond to the Ariba user file specification:

| Field Name | Ariba Field | Type | Max Length | Source | Notes |
|------------|-------------|------|------------|--------|-------|
| UniqueName | User.UniqueName | string | 255 | Entra.UserPrincipalName | Required |
| PasswordAdapter | User.PasswordAdapter | string | 50 | `AribaUserManagementOptions.PasswordAdapter` | Required |
| Name | User.Name | string | — | HcmWorkerPublic | "{WorkerNameNickname} {WorkerNamePreferredLast}"; Required |
| EmailAddress | User.EmailAddress | string | 255 | Entra.Email | Fallback for non-prod; Required |
| JobId | User.JobProfile.JobFunction.JobId | string | 32 | Entra group | Conditional; see FR-009 |
| TimeZoneID | User.TimeZoneID | string | 50 | `AribaUserManagementOptions.TimeZoneId` | |
| Phone | User.Phone | string | 70 | Entra.businessPhones[0] | Optional |
| Supervisor.UniqueName | User.Supervisor.UniqueName | string | 255 | Entra.manager.UserPrincipalName | Required |
| Supervisor.PasswordAdapter | User.Supervisor.PasswordAdapter | string | 50 | `AribaUserManagementOptions.PasswordAdapter` | Required |
| VanillaDeliverTo | DeliverTo | string | 100 | Derived | Same as Name; nullable |
| PurchasingUnit | Accounting.ProcurementUnit.UniqueName | string | 50 | `AribaUserManagementOptions.PurchasingUnit` | |
| GenericBillingAddress | User.BillingAddressesList | string | 100 | `AribaUserManagementOptions.GenericBillingAddress` | Required |
| GenericShipTo | User.ShipTosList | string | 100 | HcmWorkerPublic / home-address table | See FR-011; Required |
| ApprovalLimit | ApprovalLimit | long | — | ApprovalLimits table | Lookup by LevelCode; nullable |
| Company | Accounting.Company.UniqueName | string | 50 | HcmWorkerPublic.CompanyCode | |
| BusinessUnit | Accounting.BusinessUnit.UniqueName | string | 50 | HcmWorkerPublic.SbuCode | Required |
| CostCenter | Accounting.CostCenter.UniqueName | string | 50 | HcmWorkerPublic.CostCenterCode | Required |
| ImportCtrl | ImportCtrl | string | 100 | `AribaUserManagementOptions.ImportCtrl` | nullable |
| AlternateEmailAddresses | User.PersistedAlternateEmailAddressesList | string | 100 | Not used | Empty; Required (Ariba expects the column) |

Fields listed as "Not Used" in the source mapping (DefaultCurrency, LocaleID, Fax, CardNumbers, ExpenseApprovalLimit, Product, Project, Account, SubAccount, Region, UserUUID, and the three custom Accounting fields) SHALL be included in the DTO with null or empty values to satisfy the Ariba file column contract.

**Current-State Table** — maintained by the library in the database. Must store the last-known value of every trackable field in AribaUserDto, keyed by UniqueName, with a realm indicator (parent/child) and a last-updated timestamp. Used exclusively for diff detection.

---

### 5.5 Configuration Options

The library SHALL expose an `AribaUserManagementOptions` class registered with the .NET `IOptions<T>` pattern. All constant field values and external identifiers SHALL be read from this class; none SHALL be hardcoded in production logic. The table below lists every configurable property, its type, its default value, and where it is consumed.

| Property | Type | Default Value | Consumed By |
|----------|------|---------------|-------------|
| `PasswordAdapter` | string | `"PasswordAdapter1"` | PasswordAdapter field; Supervisor.PasswordAdapter field (FR-017) |
| `TimeZoneId` | string | `"US/Eastern"` | TimeZoneID field (FR-018) |
| `PurchasingUnit` | string | `"MTB"` | PurchasingUnit field (FR-019) |
| `GenericBillingAddress` | string | `"00073506"` | GenericBillingAddress field (FR-020) |
| `ImportCtrl` | string | `"Both"` | ImportCtrl field (FR-021) |
| `DefaultJobId` | string | `"JF00002"` | JobId field for non-BranchUser members (FR-009) |
| `BranchUserGroupName` | string | `"a-ARBxx-BranchUser"` | JobId conditional logic — blank JobId when user is in this group (FR-009) |
| `NonProdEmailFallbackTemplate` | string | `"no-email@mtb{0}.com"` | EmailAddress fallback; `{0}` is replaced with the environment name (FR-010) |
| `BasicUserGroupName` | string | `"Ariba Basic User"` | Eligibility determination — parent and child realms (FR-005) |
| `BasicContractorGroupName` | string | `"Ariba Basic Contractor"` | Eligibility determination — parent and child realms (FR-005) |
| `InternationalUserGroupName` | string | `"Ariba International User"` | Eligibility determination — parent realm only (FR-005) |

The default values listed above reflect the current production configuration and SHALL be used as the out-of-the-box defaults when no overriding configuration is supplied. The host (Azure Function) MAY override any value via standard .NET configuration sources (appsettings.json, environment variables, Azure App Configuration, Key Vault).

---

### 5.6 Integrations

| System | Direction | Data Accessed | Notes |
|--------|-----------|---------------|-------|
| HcmWorkerPublic (Workday) | Inbound (read) | Name, CompanyCode, SbuCode, CostCenterCode, LocationCode, BuildingFloor, IsRemote, LevelCode | Data already imported into database; accessed via EF Core |
| fdg.WorkerPublic (Fieldglass) | Inbound (read) | Contractor identity fields | Data already imported into database; accessed via EF Core |
| Entra | Inbound (read) | UserPrincipalName, Email, businessPhones, manager.UserPrincipalName, group memberships | Data already imported into database; accessed via EF Core |
| ApprovalLimits table | Inbound (read) | ApprovalLimit keyed by LevelCode | Internal database table; accessed via EF Core |
| Home-address lookup table | Inbound (read) | H-prefixed address identifier for remote workers | Internal database table; accessed via EF Core |
| Existing file-generation library | Outbound (delegate) | `Task<List<AribaUserDto>>` | Library consumes the delegate; this library does not write files or interact with blob storage |

---

## 6. Non-Functional Requirements

### 6.1 Performance

- **Throughput:** Incremental runs are expected to produce fewer than 100 changed records per daily execution; the library must handle the full 22,000-user dataset for diff evaluation without timing out within a reasonable Azure Function execution window.
- **Response time:** No hard SLA defined; must complete within the Azure Function timeout configured by the host (typically 5–10 minutes for consumption plan).

### 6.2 Availability & Reliability

- **Execution model:** Invoked on-demand by an Azure Function; availability is inherited from the Azure Function host and Azure SQL.
- **Fault tolerance:** Individual record failures (missing required fields) must not abort the entire run. The library must process all eligible records and return all valid ones.
- **RTO / RPO:** Not applicable at the library level; determined by the Azure Function host and database infrastructure.

### 6.3 Security & Compliance

- **Data access:** The library accesses the database via EF Core using connection strings provided by the host environment (Azure Function app settings / Key Vault references). The library itself does not manage credentials.
- **PII:** User records contain PII (name, email, phone). The library does not persist data beyond the current-state table, which is managed in the existing secured database.
- **Audit logging:** Skipped records and processing events are logged via `Microsoft.Extensions.Logging`; the Azure Function host routes these to Application Insights. No additional audit log is required at the library level.

### 6.4 Scalability

- **Current scale:** ~22,000 users; ~100 records per incremental run.
- **Growth:** The library must remain correct if user count grows or run frequency increases. No hard upper bound defined; EF Core queries should be written to avoid full in-memory table scans where possible.

### 6.5 Accessibility & Localization

Not applicable — this is a back-end class library with no user interface.

### 6.6 Observability

- **Logging framework:** `Microsoft.Extensions.Logging` — injected via standard DI; compatible with Application Insights when the host configures the appropriate provider.
- **Log events (minimum):** Run start/end with record counts; each skipped record with UniqueName and reason; any unhandled exception before re-throw.
- **Metrics / tracing:** Inherited from the Azure Function host's Application Insights integration; no custom telemetry required at the library level.

---

## 7. Constraints

| Category | Constraint | Rationale |
|----------|------------|-----------|
| Technology | .NET 10.0 | Organizational standard for new Azure workloads |
| Technology | EF Core | Existing data access pattern used across the library ecosystem |
| Technology | `Microsoft.Extensions.Logging` | Existing logging standard; required for Application Insights compatibility |
| Technology | xUnit | Existing test framework standard |
| Technology | Must not depend on SSIS or any Windows-only runtime component | Azure migration requirement |
| Infrastructure | Azure SQL | Database hosting target post-migration |
| Integration | Output must conform to `Task<List<T>>` async delegate pattern | Required for compatibility with the existing file-generation class library |
| Timeline | MVP complete by end of 2026 | Soft deadline tied to Azure migration programme |
| Scope | Must not include file serialization, blob storage, Azure Function host code, BTP configuration, or GroupConsolidated file generation | Handled by separate libraries and tasks |

---

## 8. Assumptions

- [ ] The Workday, Fieldglass, and Entra data ingestion pipelines are already operational and populate the database on their existing schedules before this library is invoked.
- [ ] The three Entra eligibility groups (Ariba Basic User, Ariba Basic Contractor, Ariba International User) are maintained correctly and are mutually exclusive at all times.
- [ ] The existing home-address lookup table (H-prefixed identifiers for remote workers) is populated and maintained by a process outside this library.
- [ ] The ApprovalLimits table is populated and maintained by a process outside this library.
- [ ] The Azure Function host will provide the database connection string and environment identifier (prod/cert/dev) via configuration; the library does not need to resolve these itself.
- [ ] The existing file-generation library's delegate contract (`Task<List<T>>`) will not change in a breaking way during development of this library.
- [ ] Azure SQL is the target database; EF Core migrations or schema changes to support the current-state table are in scope for this library.

---

## 9. Risks & Open Questions

### 9.1 Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Azure migration deadline moves up before library is complete | Low | High | Prioritize P0 requirements; the existing SSIS pipeline can continue to run until the library is ready |
| Entra group membership data in the database is stale at time of invocation | Low | Medium | Document ingestion dependency clearly; consider adding a staleness check or warning log if Entra data is older than expected |
| Current-state table schema requires revision after initial implementation | Medium | Low | Design the current-state table with all trackable fields upfront; use EF Core migrations to manage changes |

### 9.2 Open Questions

| # | Question | Owner | Due Date |
|---|----------|-------|----------|
| 1 | ⚠️ Will the GroupConsolidated file be added to this library in a future iteration, or remain a separate library? | Dave Rogala | TBD |
| 2 | ⚠️ What is the exact Azure Function execution timeout configured for this pipeline? Relevant for sizing the full-dataset diff operation. | Dave Rogala | Before implementation |
| 3 | ⚠️ Should the current-state table track parent and child realm eligibility separately, or as a single record per user with realm flags? | Dave Rogala | Before implementation |

---

## 10. Glossary

| Term | Definition |
|------|------------|
| HcmWorkerPublic | Workday's daily worker export file; primary source of truth for employee data |
| fdg.WorkerPublic | Fieldglass's daily worker export file; source of truth for contractor data |
| Entra | Microsoft Entra ID (formerly Azure Active Directory); source of identity, group membership, supervisor, phone, and email data |
| ITK | Ariba Integration Toolkit; the legacy file-delivery mechanism being replaced by SAP BTP |
| BTP | SAP Business Technology Platform; hosts SAP Integration Suite, which is the new file delivery mechanism |
| Parent Realm | The top-level Ariba site; all eligible users are loaded here |
| Child Realm | A subordinate Ariba site; Basic Users and Basic Contractors only |
| Current-State Table | A database table maintained by this library recording the last-known Ariba field values for each user, used for incremental diff detection |
| Add/Update Delegate | An async delegate returning users who are new to Ariba or whose tracked fields have changed since the last run |
| Delete Delegate | An async delegate returning users who were previously eligible but are no longer, and should be removed from Ariba |
| LevelCode | A field in HcmWorkerPublic used to look up a user's procurement ApprovalLimit |
| IsRemote | A boolean flag in HcmWorkerPublic indicating the user works remotely; drives the GenericShipTo field logic |
| BranchUser | Users in the `a-ARBxx-BranchUser` Entra group; receive a blank JobId instead of `"JF00002"` |
| AribaUserDto | The C# data transfer object representing one Ariba user record, returned by all four delegates |

---

## Appendix

### A. Field Mapping Source Reference

The authoritative field mapping, including Ariba field names, data types, string lengths, source systems, and business logic notes, is documented in `usermappings.csv` in this repository.

### B. Four Delegate Summary

| Delegate | Realm | Operation | Eligible Groups |
|----------|-------|-----------|-----------------|
| `GetParentAddUpdateAsync` | Parent | Add/Update | Basic User, Basic Contractor, International User |
| `GetParentDeleteAsync` | Parent | Delete | Basic User, Basic Contractor, International User |
| `GetChildAddUpdateAsync` | Child | Add/Update | Basic User, Basic Contractor |
| `GetChildDeleteAsync` | Child | Delete | Basic User, Basic Contractor |

### C. Active Field Business Logic Summary

| Field | Logic Type | Rule |
|-------|------------|------|
| Name | Derived | `"{WorkerNameNickname} {WorkerNamePreferredLast}"` from HcmWorkerPublic |
| EmailAddress | Conditional | If null/blank and env ≠ prod → format `AribaUserManagementOptions.NonProdEmailFallbackTemplate` with env name |
| JobId | Conditional | Member of `AribaUserManagementOptions.BranchUserGroupName` → blank; otherwise `AribaUserManagementOptions.DefaultJobId` |
| VanillaDeliverTo | Derived | Same value as Name field |
| GenericShipTo | Conditional | Employee: LocationCode + BuildingFloor; Remote employee: H-prefixed home-address lookup; Contractor: blank |
| ApprovalLimit | Lookup | ApprovalLimits table keyed by HcmWorkerPublic.LevelCode |
