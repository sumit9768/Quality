---
title: Usage Telemetry DA05 Gold Layer Logical Model
description: Logical dimensional model for Microsoft 365 usage, migration assessment, trends, and post-migration verification
author: MA Toolkit Analytics
ms.date: 2026-09-23
ms.topic: reference
keywords:
  - usage telemetry
  - dimensional model
  - Microsoft 365 migration
  - Databricks
  - Power BI
---

## Section 1: Business capability matrix

The model uses a detailed daily person-to-asset fact as the reusable calculation base
and a smaller daily asset aggregate for Power BI. This prevents distinct-user measures
from being summed across dates while avoiding direct semantic-model queries over raw
audit events.

| Requirement | Fact or bridge | Conformed dimensions | Measures and outputs |
|---|---|---|---|
| 1. Asset usage facts | `fact_person_asset_activity_daily`, `fact_asset_usage_daily` | `dim_date`, `dim_asset`, `dim_operation`, `dim_tenant`, `dim_evidence_source` | Activity volume, daily users, active users 7/30/90 days, usage trend, last read, last write |
| 2. Read versus write | Both usage facts | `dim_operation` | Read, Write, Share, Admin, Unclassified counts and Read/Write Ratio |
| 3. Usage by person | `fact_person_asset_activity_daily`, `bridge_person_account` | `dim_people`, `dim_user`, `dim_tenant`, `dim_organization` | Distinct people, internal/guest/external usage, home tenant, resource tenant, cross-tenant activity |
| 4. Person activity profile | `fact_person_sign_in_daily`, `fact_person_activity_snapshot` | `dim_people`, `dim_user`, `dim_asset`, `dim_device`, `dim_tenant`, `dim_date` | Activity status, sign-ins, tenant count, application count, device count, last activity, Dormancy Score |
| 5. Application and solution usage | Both usage facts | `dim_asset`, `dim_operation`, `dim_people`, `dim_tenant` | Flow runs, app launches, report views, distinct people, last execution |
| 6. Usage trends | `fact_asset_usage_daily`, `fact_person_activity_snapshot` | `dim_date`, `dim_asset`, `dim_tenant`, `dim_organization` | Week-over-week, month-over-month, rolling average, segment trend, historical profile |
| 7. Metadata usage fallback | `fact_asset_usage_daily` | `dim_evidence_source`, `dim_operation`, `dim_asset`, `dim_date` | Evidence timestamp, evidence priority, observed versus inferred activity, coverage status |
| 8. Post-migration verification | `fact_asset_migration_verification`, `bridge_asset_migration` | `dim_asset` in source and target roles, `dim_tenant`, `dim_date`, `dim_migration_wave`, `dim_operation` | Source and target usage, usage delta, usage retention, reactivation, verification status |
| Identity resolution support | `bridge_person_account`, `fact_person_activity_snapshot` | `dim_people`, `dim_user`, `dim_tenant`, `dim_organization` | Resolved account count, unresolved identity count, identity confidence, active-person evidence |
| Dependency analysis | `bridge_asset_relationship`, `bridge_asset_person_role`, `bridge_asset_organization` | `dim_asset`, `dim_people`, `dim_organization`, `dim_tenant` | Upstream and downstream dependency counts, shared owners, cross-organization dependencies |
| Migration readiness | Usage facts, profile snapshot, asset relationship bridges | All conformed dimensions | Migration Readiness Score, dormancy, change velocity, dependency complexity, mapping coverage |

## Section 2: Dimensional model overview

### Modeling principles

The recommended logical model follows these principles:

* Preserve one declared grain per fact table.
* Use surrogate keys on facts and keep natural identifiers in dimensions for lineage.
* Reuse the existing `dim_date`, `dim_tenant`, `dim_organization`, `dim_user`,
  `dim_people`, `dim_group`, and `dim_spo_site` entities.
* Treat a person and a tenant account as different entities. One person can have source,
  target, guest, and external-tenant accounts.
* Represent SharePoint sites, Teams, Power BI artifacts, Power Apps, Power Automate
  flows, and mailboxes through one conformed `dim_asset`.
* Keep the detailed Gold grain at one person, asset, operation class, and day. Do not
  expose raw audit events as the primary Power BI fact.
* Persist exact trailing-window distinct counts in the asset aggregate. Daily distinct
  counts cannot be added to produce 7, 30, or 90-day active users.
* Record the evidence source and quality on every usage row. Audit, Graph usage, and
  metadata evidence are not interchangeable.
* Retain unknown dimension members for unresolved people, assets, operations, devices,
  and tenants. Do not discard valid telemetry because enrichment is late.
* Use bridges for many-to-many identity, ownership, organization, dependency, and
  source-to-target mapping relationships.

### Fact constellation

| Fact | Type | Primary analytical role |
|---|---|---|
| `fact_person_asset_activity_daily` | Transactional daily aggregate | Reusable person attribution, exact distinct-person windows, cross-tenant usage |
| `fact_asset_usage_daily` | Periodic daily aggregate | Power BI asset reporting, trailing windows, trends, last read/write |
| `fact_person_sign_in_daily` | Transactional daily aggregate | Sign-in, application, resource tenant, and device profile |
| `fact_person_activity_snapshot` | Periodic snapshot | Dormancy, identity resolution, and person-level migration assessment |
| `fact_asset_migration_verification` | Periodic comparison snapshot | Source-to-target usage verification after migration |

### Conformed asset scope

`dim_asset` uses one row structure across workloads while preserving workload-specific
identifiers and attributes. `asset_type` must use controlled values.

| Workload | Primary asset types | Current reusable source |
|---|---|---|
| SharePoint and OneDrive | `SharePointSite`, `OneDriveSite` | Gold `dim_spo_site`; Silver `spo_sites` |
| Teams | `Team`, optionally `TeamChannel` | Gold `shared_data_sets`; Silver `teams`, `teams_team_details`, `teams_channels` |
| Power BI and Fabric | `PowerBIWorkspace`, `PowerBIReport`, `PowerBIDashboard`, `PowerBIDataset`, `PowerBIDataflow`, `PowerBIApp` | Silver `powerbi_apps`, `powerbi_workspaces`; nested workspace scan payloads require flattening |
| Power Apps | `PowerApp` | Silver `powerplat_apps` |
| Power Automate | `PowerAutomateFlow` | Silver `powerplat_flows`, `powerplat_flow_metadata` |
| Exchange Online | `Mailbox` | Silver `mailboxes`; `exo_mailbox_statistics` requires a Silver implementation |
| Authentication context | `EntraApplication`, `EntraResource` | Silver `applications`, `service_principals`; sign-in `app_id` and `resource_id` |

Authentication-context assets extend the six migration workloads without changing the
meaning of the primary asset types. They allow sign-in application usage to use the
same conformed asset key.

## Section 3: Fact tables

### `fact_person_asset_activity_daily`

| Property | Design |
|---|---|
| Business purpose | Canonical daily activity by resolved person and asset. Supports person attribution, exact active-user windows, cross-tenant usage, application impact, and rollup into all asset usage outputs. |
| Grain | One row per `activity_date_key`, `resource_tenant_key`, `person_key`, `account_key`, `asset_key`, `operation_key`, and `evidence_source_key`. |
| Foreign keys | Date, resource tenant, person, tenant account, asset, operation, evidence source; optional device for telemetry that reliably identifies one device. |
| Additive measures | `activity_count`, `successful_activity_count`, `failed_activity_count`, `cross_tenant_activity_count`. |
| Non-additive attributes | `first_activity_at`, `last_activity_at`, `identity_resolution_confidence`, `asset_resolution_confidence`, `is_cross_tenant`, `is_inferred_activity`. |
| Source Silver tables | `audit_entra`, `audit_exchange`, `audit_general`, `audit_sharepoint`; fallback inputs from `spo_sites`, `mailboxes`, and future Graph usage tables. |
| Power BI use | Import or Direct Lake only for controlled drill-through and distinct-person calculations. Default reports use `fact_asset_usage_daily`. |

The account key identifies the tenant account that performed the activity. The person
key identifies the resolved individual. This separation preserves guest and external
usage without creating duplicate people.

### `fact_asset_usage_daily`

| Property | Design |
|---|---|
| Business purpose | Performance-optimized asset usage fact for daily reporting, rolling active-user windows, workload comparisons, and trend analysis. |
| Grain | One row per `activity_date_key`, `resource_tenant_key`, `asset_key`, `operation_key`, and `evidence_source_key`. `operation_key` resolves to one operation class. |
| Foreign keys | Date, resource tenant, asset, operation, evidence source; optional primary organization for a governed single-valued segment. |
| Additive measures | `activity_volume`, `successful_activity_volume`, `failed_activity_volume`, `cross_tenant_activity_volume`. |
| Semi-additive measures | `daily_distinct_people`, `active_people_7d`, `active_people_30d`, `active_people_90d`, `last_activity_at`. |
| Quality measures | `resolved_person_count`, `unresolved_person_count`, `resolved_asset_event_count`, `unresolved_asset_event_count`, `coverage_pct`. |
| Source Silver tables | Derived primarily from `fact_person_asset_activity_daily`; metadata from `spo_sites`; mailbox fallback from future Silver mailbox statistics; future Graph usage report tables. |
| Power BI use | Primary DA05 fact. Measures filter `dim_operation.operation_class` to return Last Read Timestamp, Last Write Timestamp, and class-specific trends. |

`active_people_7d`, `active_people_30d`, and `active_people_90d` are exact distinct
counts across the full trailing window. They are not sums of `daily_distinct_people`.
When Power BI aggregates multiple assets, a semantic measure must calculate from the
person fact or use a separately computed group aggregate. Summing asset-level distinct
counts would double-count people who used more than one asset.

### `fact_person_sign_in_daily`

| Property | Design |
|---|---|
| Business purpose | Daily sign-in behavior by person, tenant account, application or resource, and device. Feeds activity status, application usage, dormant-account analysis, and identity resolution. |
| Grain | One row per `sign_in_date_key`, `resource_tenant_key`, `person_key`, `account_key`, client `asset_key`, resource `asset_key`, `device_key`, `sign_in_outcome`, `is_interactive`, and `client_app_used`. |
| Foreign keys | Date, resource tenant, person, tenant account, client application asset, resource asset, device. |
| Additive measures | `sign_in_attempt_count`, `successful_sign_in_count`, `failed_sign_in_count`, `risky_sign_in_count`. |
| Semi-additive measures | `first_sign_in_at`, `last_sign_in_at`. |
| Source Silver tables | `sign_in_logs`; `users`, `devices`, and `intune_devices` for enrichment. |
| Current limitation | Silver `sign_in_logs` does not retain the Graph `status`, `deviceDetail`, or home-tenant fields. Outcome and device measures require schema enrichment. |

### `fact_person_activity_snapshot`

| Property | Design |
|---|---|
| Business purpose | A point-in-time profile of a person's activity and account posture for migration assessment, identity resolution, and dormant-account analysis. |
| Grain | One row per `snapshot_date_key`, `person_key`, and `resource_tenant_key`. |
| Foreign keys | Snapshot date, person, resource tenant, primary organization, optional migration wave. |
| Measures | `activity_count_7d`, `activity_count_30d`, `activity_count_90d`, `active_asset_count_30d`, `active_application_count_30d`, `active_device_count_30d`, `active_tenant_count_30d`, `sign_in_count_30d`, `days_since_last_activity`, `days_since_last_sign_in`, `dormancy_score`, `resolved_account_count`. |
| Status attributes | `activity_status`, `identity_status`, `has_cross_tenant_activity`, `has_source_activity`, `has_target_activity`, `is_dormant_candidate`. |
| Source Silver tables | Derived from both daily person facts; `users`, `devices`, `intune_devices`, `user_mapping_candidates`; current Gold `dim_people` and `user_map`. |
| Snapshot cadence | Daily for the most recent 90 days, weekly for the current year, and month-end thereafter. |

### `fact_asset_migration_verification`

| Property | Design |
|---|---|
| Business purpose | Compares source and target usage for a mapped asset after migration and records whether adoption and activity meet verification thresholds. |
| Grain | One row per `verification_date_key`, `asset_migration_key`, `operation_key`, and comparison window. |
| Foreign keys | Verification date, source asset, target asset, source tenant, target tenant, migration wave, operation, evidence source. |
| Measures | `source_activity_volume`, `target_activity_volume`, `source_active_people`, `target_active_people`, `activity_delta`, `activity_delta_pct`, `active_people_delta`, `usage_retention_pct`, `cross_tenant_activity_pct`, `days_to_first_target_use`. |
| Status attributes | `verification_status`, `threshold_profile`, `source_coverage_status`, `target_coverage_status`, `is_comparable`. |
| Source Silver tables | Derived from `fact_asset_usage_daily` and `bridge_asset_migration`; mapping candidates or workload-specific migration maps supply source-to-target relationships. |
| Comparison windows | Standard rows for 7, 30, and 90 days, plus a configurable pre-migration baseline period. |

The verification fact must not compare unequal evidence. For example, Audit-based
source volume and Metadata Fallback target recency are not comparable. Such rows remain
visible with `is_comparable = false`.

## Section 4: Dimension tables

### Core dimensions

| Dimension | Business purpose | Key and important attributes | SCD recommendation | Current reuse |
|---|---|---|---|---|
| `dim_date` | Calendar, fiscal, ISO week, month, and trailing-window filtering | `date_key` integer in `yyyyMMdd` form; date, day, week, ISO year, month, quarter, fiscal fields | Type 0, fixed members | Reuse existing Gold `dim_date` |
| `dim_tenant` | Conformed resource and home tenant identity | `tenant_key` surrogate; `source_key`, Entra tenant GUID, tenant name, migration role, region, default domain, valid dates | Type 2 for role and descriptive history | Extend existing Gold `dim_tenant` without creating a parallel tenant dimension |
| `dim_organization` | CDO or business organization segmentation | Existing `org_id`; organization name, configured domains, region, segment, active flag | Type 2 for reporting history | Reuse existing Gold `dim_organization` |
| `dim_people` | Canonical individual independent of tenant accounts | Existing `person_id`; display name, employee ID, person type, home tenant, primary organization, identity status, confidence, valid dates | Type 2 | Broaden existing Gold `dim_people` beyond licensed source users; preserve current `person_id` where available |
| `dim_user` | Tenant-specific account identity | Existing `user_silver_id`; Entra object ID, UPN, mail, user type, account enabled, source key, tenant, guest state, external state, valid dates | Type 2 for status and identity attributes | Reuse existing Gold `dim_user`; enrich from Silver `users` |
| `dim_asset` | Conformed migration and usage asset across all workloads | `asset_key` surrogate; workload, asset type, tenant, source object ID, stable natural key hash, name, URL, parent identifiers, owner, state, created/modified dates, sensitivity, lifecycle, valid dates | Type 2 | New conformed presentation entity seeded from existing workload dimensions and Silver asset tables |
| `dim_operation` | Versioned mapping from raw audit operation to reporting class | `operation_key`; normalized workload, raw operation, operation class, activity family, read/write effect, user-impact flag, synthetic/fallback flag, effective dates, mapping status | Type 2 | New reference dimension |
| `dim_evidence_source` | Distinguishes observed and inferred usage | `evidence_source_key`; source type, source system, priority, confidence tier, supports-person-attribution, supports-volume, supports-operation, active flag | Type 1 | New small reference dimension |
| `dim_device` | Conformed device used by a person or sign-in | `device_key`; Entra, Intune, and MDE identifiers, OS, model, management, compliance, trust, ownership, active state, valid dates | Type 2 | New Gold dimension from Silver `devices`, `intune_devices`, and `mde_devices` |
| `dim_migration_wave` | Migration program, wave, cutover, and verification context | `migration_wave_key`; program, wave, source tenant, target tenant, planned and actual cutover dates, owner, status, threshold profile | Type 2 | New dimension; no current authoritative wave entity was found |

Type 2 dimensions require `valid_from`, `valid_to`, and `is_current`. Facts resolve the
dimension version valid on the activity date. Unknown members use key `0`; unresolved
members use a separate negative or reserved key so data quality can distinguish
missing enrichment from genuinely unknown business values.

### `dim_asset` workload attributes

The conformed dimension keeps common descriptive attributes in the star. Highly
workload-specific technical properties remain in existing workload dimensions or
extension tables and relate one-to-one through `asset_key`.

| Asset type | Common source identifiers | Selected attributes |
|---|---|---|
| SharePoint site | `site_silver_id`, SharePoint `id`, normalized URL | Personal site, hub site, template, storage, sharing capability, lock state, sensitivity |
| Team | `team_id`, Entra group object ID | Visibility, archived flag, member count, linked primary site |
| Power BI artifact | Tenant plus artifact GUID | Workspace, artifact subtype, capacity, state, endorsement, parent workspace |
| Power App | Tenant plus `app_object_id` | Environment name, app type, premium or custom API flags, owner, publish date |
| Power Automate flow | Tenant plus `flow_object_id` | Environment name, state, creator, workflow ID, modified date |
| Mailbox | Tenant plus Exchange GUID | Recipient type, primary SMTP, archive, hold state |
| Entra application or resource | Tenant plus application or service-principal ID | Display name, publisher, client/resource role, enabled state |

The durable natural key is a hash of normalized `tenant_id`, `asset_type`, and
workload-native object ID. Display name, URL, UPN, or SMTP address must not form the
durable key because each can change.

### Bridge and factless fact tables

| Table | Grain | Purpose and source | Required relationship behavior |
|---|---|---|---|
| `bridge_person_account` | One row per person, tenant account, identity role, and validity interval | Maps source, target, guest, and external accounts to one person. Seed from Gold `dim_people`, `user_map`, and Silver `user_mapping_candidates`. | Many-to-many; filter from person to account and then to facts. Include match method, confidence, approval status, home tenant, and valid dates. |
| `bridge_asset_person_role` | One row per asset, person/account, role, and validity interval | Ownership, membership, permission, creator, administrator, or audience. Source from group ownership/membership, SPO site users, Power BI users, Power Platform role assignments, and mailbox permissions. | Many-to-many. Do not use bidirectional relationships in the default Power BI model. |
| `bridge_asset_organization` | One row per asset and organization association | Segment attribution where owners or members span organizations. Generalizes current Gold `shared_data_set_organizations`. | Many-to-many with confidence and attribution method; identify one governed primary association separately. |
| `bridge_asset_relationship` | One row per from-asset, to-asset, relationship type, and validity interval | Contains, depends on, uses data from, invokes, secures, publishes, or belongs to. Sources include Teams-to-site, Power BI dataset datasource, flow connection, and Power Platform solution-component relationships. | Role-play `dim_asset` as from and to. Keep both relationships single direction in Power BI. |
| `bridge_asset_migration` | One row per source asset, target asset, migration wave, and mapping version | Source-to-target mapping, mapping method, confidence, status, cutover, and verification eligibility. | Role-play `dim_asset` as source and target. Only approved mappings feed comparable verification facts. |

### Operation taxonomy

`dim_operation` resolves a case-normalized `(workload, operation)` pair. Exact mappings
take precedence over approved prefix or regular-expression mappings. Every new value
first lands on the Unclassified member and generates a stewardship alert.

| Workload | Representative operations | Operation class | Activity family |
|---|---|---|---|
| SharePoint | `FileAccessed`, `FilePreviewed`, `FileDownloaded`, `PageViewed` | Read | Content consumption |
| SharePoint | `FileUploaded`, `FileModified`, `FileDeleted`, `FileMoved`, `FileRenamed` | Write | Content change |
| SharePoint | `SharingSet`, `SharingInvitationCreated`, `AddedToSecureLink` | Share | Sharing |
| SharePoint | `SiteCollectionAdminAdded`, `PermissionLevelAdded`, `SiteCollectionCreated` | Admin | Administration |
| Exchange | `MailItemsAccessed`, `MessageBind` | Read | Mailbox access |
| Exchange | `Send`, `SendAs`, `SendOnBehalf`, `MoveToDeletedItems`, `SoftDelete`, `HardDelete` | Write | Mailbox action |
| Exchange | Mailbox delegation operations | Share | Delegation |
| Exchange | `Set-Mailbox`, `Add-MailboxPermission`, `Remove-MailboxPermission` | Admin | Administration |
| Teams | Message or chat retrieval operations | Read | Collaboration consumption |
| Teams | `MessageSent`, `MessageUpdated`, `MessageDeleted` | Write | Collaboration change |
| Teams | `MemberAdded`, `MemberRemoved`, link or invitation operations | Share | Membership or sharing |
| Teams | `TeamCreated`, `TeamDeleted`, `ChannelAdded`, `ChannelDeleted`, `TabAdded` | Admin | Administration |
| Power BI | `ViewReport`, `ViewDashboard`, `ViewTile`, export and analyze operations | Read | Analytics consumption |
| Power BI | `CreateReport`, `EditReport`, `DeleteReport`, `RefreshDataset` | Write | Analytics authoring |
| Power BI | Report, dashboard, app, or workspace share operations | Share | Sharing |
| Power BI | Workspace membership, gateway, capacity, or tenant-setting operations | Admin | Administration |
| Power Apps | App launch or play operations | Read | App launch |
| Power Apps | Create, edit, save, publish, or delete app operations | Write | App authoring |
| Power Apps | App share and permission operations | Share | Sharing |
| Power Apps | Environment, policy, owner, or administrative operations | Admin | Administration |
| Power Automate | Flow run or trigger operations | Write | Flow execution |
| Power Automate | Create, edit, enable, disable, or delete flow operations | Write | Flow authoring |
| Power Automate | Flow share and permission operations | Share | Sharing |
| Power Automate | Environment, policy, owner, or administrative operations | Admin | Administration |
| Any | Unknown value or an unmatched workload-operation pair | Unclassified | Data quality control |

The representative values establish the logical taxonomy, not a final exhaustive
Microsoft operation catalog. Production mappings must be validated against observed
tenant operation values and current Microsoft audit documentation before activation.

### Evidence source taxonomy

| Source type | Priority | Person attribution | Volume support | Recommended use |
|---|---:|---|---|---|
| Audit | 1 | Yes, after identity resolution | Yes | Authoritative event, operation, and person activity |
| Graph Usage | 2 | Depends on report | Yes, usually aggregated | Fill audit coverage gaps and provide Microsoft-published usage totals |
| Metadata Fallback | 3 | Usually no | No | Infer recency from last-modified or last-logon metadata |

Only one source is primary for a metric at a given asset and date. Lower-priority
evidence may fill a null but must not be added to higher-priority volume.

## Section 5: Relationships

### Cardinalities

| From | To | Cardinality | Notes |
|---|---|---|---|
| Each usage fact | `dim_date` | Many to one | Use role-playing date keys for activity, snapshot, cutover, and verification dates |
| Each usage fact | `dim_tenant` | Many to one | Resource tenant is active; home tenant can be an inactive role or reached through person/account |
| Person facts | `dim_people` | Many to one | Canonical individual |
| Person facts | `dim_user` | Many to one | Tenant account used for the activity |
| Asset facts | `dim_asset` | Many to one | Conformed asset |
| Usage facts | `dim_operation` | Many to one | Operation class is an attribute of the operation version |
| Usage facts | `dim_evidence_source` | Many to one | Audit, Graph Usage, or Metadata Fallback |
| Sign-in fact | `dim_device` | Many to one | Unknown when the event has no device identifier |
| Verification fact | `bridge_asset_migration` | Many to one | Carries the approved source-to-target mapping version |
| Bridges | Their dimensions | Many to one on each side | Keep single-direction relationships in the semantic model |

### Power BI relationship guidance

* Use single-direction filtering from dimensions to facts.
* Create separate role-playing semantic dimensions for Source Asset and Target Asset
  over `dim_asset` in migration verification reports.
* Create Home Tenant and Resource Tenant roles over `dim_tenant`.
* Avoid direct fact-to-fact relationships. Shared dimensions provide conformance.
* Do not enable automatic bidirectional filtering on bridges. Use explicit DAX or
  calculation groups for bridge-aware analysis.
* Hide all surrogate keys, technical natural keys, validity columns, and quality-control
  fields from report authors unless they support a diagnostic page.

## Section 6: Derived metrics

### Active Users 7 Days

For asset `A`, operation class `C`, and as-of date `D`:

```text
COUNT DISTINCT person_key
WHERE asset_key = A
  AND operation_class = C
  AND activity_date BETWEEN D - 6 days AND D
  AND activity_count > 0
```

Use resolved people, not raw accounts. Include internal, guest, and external people.
Report unresolved account activity separately.

### Active Users 30 Days

```text
COUNT DISTINCT person_key
WHERE asset_key = A
  AND operation_class = C
  AND activity_date BETWEEN D - 29 days AND D
  AND activity_count > 0
```

Apply the same identity, person-type, and unresolved-account rules as Active Users 7
Days.

### Active Users 90 Days

```text
COUNT DISTINCT person_key
WHERE asset_key = A
  AND operation_class = C
  AND activity_date BETWEEN D - 89 days AND D
  AND activity_count > 0
```

Apply the same identity, person-type, and unresolved-account rules as Active Users 7
Days. Use this window as the default long-horizon activity signal for migration
assessment.

### Usage Trend Percentage

```text
current_period_volume = SUM(activity_volume in selected complete period)
prior_period_volume   = SUM(activity_volume in immediately preceding equal period)
usage_trend_pct       = 100 * (current_period_volume - prior_period_volume)
                       / NULLIF(prior_period_volume, 0)
```

Return blank when the prior period is zero and expose `is_new_usage = true` when the
current period is positive. This avoids reporting an infinite or misleading percentage.
Weekly trends use ISO week and ISO year. Monthly trends compare complete calendar
months unless the report explicitly selects month-to-date.

### Last Read Timestamp

```text
MAX(last_activity_at)
WHERE operation_class = "Read"
  AND is_primary_evidence = true
```

Last Write Timestamp uses the same logic with `operation_class = "Write"`. Metadata
fallback can populate a write-like timestamp only when no Audit or Graph Usage evidence
exists, and the report must display its evidence source.

### Read/Write Ratio

```text
read_write_ratio =
    SUM(activity_volume WHERE operation_class = "Read")
    / NULLIF(SUM(activity_volume WHERE operation_class = "Write"), 0)
```

Expose read and write volumes beside the ratio. A blank denominator means no observed
writes, not zero read intensity.

### Cross-Tenant Usage Percentage

```text
cross_tenant_usage_pct =
    100 * SUM(cross_tenant_activity_count)
    / NULLIF(SUM(activity_count), 0)
```

An activity is cross-tenant when the resolved person's home tenant differs from the
resource tenant. Also publish a distinct-person variant using cross-tenant people
divided by all active people.

### Dormancy Score

The recommended score ranges from 0 for highly active to 100 for highly dormant:

```text
recency_component =
    60 * MIN(days_since_last_activity, 180) / 180

frequency_component =
    25 * (1 - MIN(activity_count_30d / active_frequency_target, 1))

sign_in_component =
    15 * MIN(days_since_last_sign_in, 90) / 90

dormancy_score =
    recency_component + frequency_component + sign_in_component
```

Use a configurable `active_frequency_target` by persona or license segment. Set
`activity_status` from governed thresholds, for example Active, Low Activity, Dormant,
Never Observed, and Insufficient Coverage. Do not classify Insufficient Coverage as
Dormant.

### Migration Readiness Score

The score ranges from 0 to 100, where 100 represents the strongest readiness:

```text
migration_readiness_score =
    0.30 * identity_mapping_score
  + 0.25 * asset_mapping_score
  + 0.20 * dependency_readiness_score
  + 0.15 * activity_stability_score
  + 0.10 * telemetry_coverage_score
```

Component guidance:

* Identity mapping measures approved person and owner mappings.
* Asset mapping measures approved source-to-target asset mappings.
* Dependency readiness decreases with unresolved or cross-boundary dependencies.
* Activity stability decreases with recent write spikes or high change velocity.
* Telemetry coverage measures resolved assets, people, operation taxonomy, and usable
  evidence.

Store component scores with the total. Weights are configuration, not hard-coded DAX.
A missing mandatory component makes the total status Insufficient Evidence rather than
silently treating the component as zero.

### Application measures

| Measure | Definition |
|---|---|
| Flow Runs | Activity volume for `asset_type = PowerAutomateFlow` and `activity_family = Flow execution` |
| App Launches | Activity volume for `asset_type = PowerApp` and `activity_family = App launch` |
| Report Views | Activity volume for `asset_type = PowerBIReport` and `activity_family = Analytics consumption` |
| Distinct Users | Exact distinct `person_key` in the selected scope, calculated from the person-asset fact |
| Usage Retention Percentage | `100 * target comparison-period volume / NULLIF(source baseline volume, 0)` |

## Section 7: Star schema diagram

### Core usage star

```text
                         DimDate
                            |
DimTenant ---- FactAssetUsageDaily ---- DimAsset
                            |
                      DimOperation
                            |
                   DimEvidenceSource
```

### Person activity star

```text
DimDate ------------------------------+
DimTenant ----------------------------+
DimPeople ---- FactPersonAssetActivityDaily ---- DimAsset
DimUser ------------------------------+
DimOperation -------------------------+
DimEvidenceSource --------------------+
```

### Sign-in and profile stars

```text
DimDate -----------------------+
DimTenant ---------------------+
DimPeople ---- FactPersonSignInDaily ---- DimAsset (Client Application)
DimUser -----------------------+           |
DimDevice ---------------------+           +---- DimAsset (Resource)

DimDate -----------------------+
DimTenant ---- FactPersonActivitySnapshot ---- DimPeople
DimOrganization ---------------+
DimMigrationWave --------------+
```

### Migration verification star

```text
DimDate -------------------------------+
DimMigrationWave ----------------------+
DimOperation --------------------------+
DimTenant (Source) --------------------+
DimTenant (Target) --------------------+
DimAsset (Source) ---- BridgeAssetMigration ---- DimAsset (Target)
                              |
                 FactAssetMigrationVerification
```

### Bridge constellation

```text
DimPeople ---- BridgePersonAccount ---- DimUser ---- DimTenant
    |
    +---- BridgeAssetPersonRole ---- DimAsset

DimOrganization ---- BridgeAssetOrganization ---- DimAsset

DimAsset (From) ---- BridgeAssetRelationship ---- DimAsset (To)
```

## Section 8: Physical design recommendations

### Delta table layout

| Table | Partition or clustering recommendation | Rationale |
|---|---|---|
| `fact_person_asset_activity_daily` | Partition by `activity_month`; liquid cluster by `resource_tenant_key`, `asset_key`, `person_key` where supported | Month pruning controls file counts while clustering supports asset and person predicates |
| `fact_asset_usage_daily` | Partition by `activity_month`; liquid cluster by `resource_tenant_key`, `asset_key`, `operation_key` | Primary Power BI access path |
| `fact_person_sign_in_daily` | Partition by `sign_in_month`; liquid cluster by `resource_tenant_key`, `person_key`, `asset_key` | Supports person and application profile queries |
| `fact_person_activity_snapshot` | Partition by `snapshot_month`; cluster by `resource_tenant_key`, `person_key` | Efficient current and historical profile access |
| `fact_asset_migration_verification` | Partition by `verification_month`; cluster by `migration_wave_key`, source asset, target asset | Verification reports filter by wave and mapped assets |
| Type 2 dimensions and bridges | Do not date-partition unless volume proves it necessary; cluster large bridges by primary dimension keys | Avoid small partitions and preserve point lookups |

Prefer Databricks liquid clustering over deep static partition hierarchies when the
runtime and Unity Catalog configuration support it. Never partition by person, asset,
operation, or tenant GUID because their cardinality produces small files.

### Incremental processing

1. Deduplicate append-only audit records by `audit_id` and sign-ins by `sign_in_id`
   before Gold aggregation.
2. Apply an event-time watermark and accept late arrivals for at least 14 days. Make the
   interval configurable because Management Activity API delivery can be delayed.
3. Resolve identity, asset, operation, and tenant keys before daily aggregation. Route
   unresolved values to reserved members and reprocess them after mappings arrive.
4. Merge only changed daily person-asset keys into
   `fact_person_asset_activity_daily`.
5. Rebuild affected `fact_asset_usage_daily` dates and all trailing snapshots whose
   7, 30, or 90-day windows include a changed date.
6. Use Delta Change Data Feed or equivalent changed-key manifests to avoid scanning the
   entire detailed fact.
7. Recompute the latest person activity snapshot daily. Materialize weekly and
   month-end rows according to the retention schedule.
8. Recompute migration verification only for active waves, changed mappings, or changed
   source and target usage windows.
9. Track source completeness by tenant, workload, and date. Do not publish a completed
   trend period until required ingestion batches are complete or explicitly waived.

### Power BI strategy

* Use `fact_asset_usage_daily` for standard visuals and aggregations.
* Keep `fact_person_asset_activity_daily` out of broad report pages. Use it for exact
  distinct-person measures, drill-through, and controlled composite models.
* Add aggregation mappings from the person fact to the asset fact when the selected
  dimensions match the aggregate grain.
* Use calculation groups for rolling 7/30/90 days, prior period, source versus target,
  and evidence-source display.
* Configure incremental refresh by activity or snapshot date. Keep a short refresh
  range that covers the late-arrival interval and the 90-day rolling recomputation
  horizon.
* Apply object-level or row-level security to person-identifiable facts. Most users
  should consume asset aggregates without UPN, IP address, or raw identity fields.

### Snapshot and retention

| Data product | Hot retention | Historical retention | Notes |
|---|---|---|---|
| Silver audit and sign-in detail | 24 months | Up to 7 years in lower-cost storage, subject to legal and privacy policy | Immutable, deduplicated event evidence |
| Daily person-asset activity | 36 months | 7 years where migration evidence requires it | Pseudonymize or restrict person identifiers after the hot period |
| Daily asset usage | 7 years | 7 years or program lifetime | Small enough for long-term trend analysis |
| Person activity snapshot | 90 days daily, current year weekly | Month-end snapshots for 7 years | Avoid retaining every daily person profile indefinitely |
| Migration verification | Program lifetime plus 7 years | Archive with migration evidence | Preserve threshold version and mapping version |
| Type 2 dimensions and bridges | Full validity history | Same as dependent facts | Never delete a version referenced by retained facts |

Retention values are architectural defaults. Tenant contracts, legal holds, privacy
requirements, and regional data policies take precedence.

### Data quality controls

Publish daily controls for:

* Unclassified operation percentage by workload
* Unresolved person and asset percentages
* Duplicate event identifiers
* Late-arriving event volume
* Audit-to-Graph variance where both are available
* Metadata-only asset percentage
* Source and target evidence comparability
* Orphaned source-to-target mappings
* Active-window recomputation freshness

Recommended release thresholds include zero duplicate event IDs, zero orphaned foreign
keys after unknown-member substitution, and explicitly governed limits for
Unclassified operations and unresolved assets.

## Section 9: Assumptions and data gaps

### Reusable workspace assets

The current workspace already provides:

* Append-only Silver `audit_entra`, `audit_exchange`, `audit_general`, and
  `audit_sharepoint` tables.
* Append-only Silver `sign_in_logs`.
* Silver identity, tenant-account, Teams, SharePoint, Exchange, Power BI, Power
  Platform, Entra device, Intune, and MDE metadata.
* Existing Gold date, tenant, organization, user, people, group, SharePoint site,
  shared-data-set, and identity-mapping entities.
* Existing source and target environment semantics through `source_key`,
  `environment`, tenant configuration, and user mapping candidates.

### Blocking data gaps

| Gap | Impact | Required remediation before implementation |
|---|---|---|
| Silver audit tables parse only the common audit envelope | Workload-specific asset IDs, URLs, file details, Team/channel IDs, mailbox identifiers, Power BI artifact IDs, Power Platform resource IDs, and some actor context are unavailable | Preserve raw audit JSON or parse workload-specific payload extensions into a normalized Silver audit activity table |
| Audit `user_id` is not a guaranteed Entra object ID | Person attribution can fail for UPNs, app identities, system actors, or historical aliases | Add an identity-resolution stage using object ID, normalized UPN/mail/proxy history, audit user type, service principal, and approved manual mapping |
| Home tenant is not retained for usage actors | Guest versus external-tenant classification and cross-tenant usage cannot be authoritative | Retain home tenant or external tenant identifiers from audit, sign-in, and directory guest metadata |
| Silver `users` does not expose `externalUserState` from its parsed schema | Guest lifecycle and invitation status are incomplete | Add guest state, invitation status, issuer, and creation-type fields to Silver identity output |
| Silver sign-ins omit `status`, `deviceDetail`, location, and home-tenant context | Successful versus failed sign-ins and sign-in device usage cannot be calculated reliably | Expand the Graph sign-in schema and selected output fields |
| No Graph usage report entities are registered | Graph Usage fallback cannot be populated | Add registered Bronze and Silver entities for required Microsoft 365 usage reports with report period and refresh metadata |
| `exo_mailbox_statistics` is registered in Bronze but has no implemented Silver table | Mailbox last-logon fallback, item count, and size are unavailable in Gold | Implement typed Silver mailbox statistics and join by tenant plus Exchange GUID |
| Power BI workspace scans remain nested in `_record` | Report, dashboard, dataset, and dataflow assets are not fully conformed | Flatten nested scan artifacts to stable Silver tables before creating their `dim_asset` members |
| Power Apps and flows provide inventory but not run or launch history | Flow Runs and App Launches depend on audit payload resolution | Parse the corresponding audit operations and resource identifiers or ingest dedicated usage APIs |
| No authoritative migration-wave entity was found | Verification thresholds and cutover windows have no governed owner | Add migration program and wave configuration with source tenant, target tenant, cutover, and threshold profile |
| No approved asset mapping entity was found across all workloads | Post-migration comparison cannot pair source and target assets | Create governed source-to-target asset mapping outputs and retain mapping versions |

### Modeling assumptions

* `source_key` identifies a tenant-level source and does not identify an organization.
  Organization attribution uses `org_id`.
* `tenant_id` is an Entra tenant identifier, not an organization key.
* Audit is authoritative when complete and correctly resolved.
* Graph Usage fills an aggregate coverage gap but does not overwrite higher-quality
  person-level audit evidence.
* Metadata timestamps indicate recency, not event volume or a known active user.
* A Power Automate flow execution is classified as Write because it executes an action
  and no Execute operation class is in the DA05 requirement.
* Unknown and Unclassified records remain in totals and quality reports.
* Active-user metrics count canonical people. A separate active-account measure may be
  published for identity operations.
* Service principals and system accounts are excluded from Active Users by default and
  reported through separate non-person activity measures.
* Source-to-target comparisons require equivalent windows, time zones, operation
  taxonomy versions, and evidence types.

### Recommended implementation sequence

1. Enrich Silver audit, sign-in, mailbox statistics, Graph usage, and Power BI artifact
   schemas.
2. Establish identity and asset resolution with unknown-member and stewardship queues.
3. Publish conformed dimensions and bridges.
4. Build the daily person-asset and sign-in facts.
5. Build the asset aggregate and person profile snapshot.
6. Add governed asset mappings and migration verification.
7. Publish Power BI aggregation, rolling-window, quality, and verification measures.
