---
title: Usage Telemetry DA05 Must-Have Physical Model
description: Requirement-grouped physical model for asset usage, operation classification, and usage by person
author: MA Toolkit Analytics
ms.date: 2026-09-23
ms.topic: reference
keywords:
  - usage telemetry
  - physical data model
  - Microsoft 365 migration
  - Databricks
  - Power BI
---

## Scope and status legend

This focused model covers the three must-have DA05 requirements:

1. Asset Usage Facts
2. Read versus Write
3. Usage by Person

It reuses the current Silver and Gold schemas wherever their grain fits. Dummy values
are synthetic, but every source column shown exists in the current pipeline unless the
column is explicitly marked as a required modification.

| Status | Meaning |
|---|---|
| `Existing, reused` | The current table and column can be used without changing its grain |
| `Existing, modified` | Keep the current business entity and key, but add columns or broaden its population |
| `New` | DA05 requires a new Gold table |
| `Gap` | The current source does not contain enough information to populate the requirement authoritatively |

The examples use the configured development tenant `madev1` with tenant ID
`2b1943ef-f207-4cd2-884f-3d1c420dbe58`. Hash values are abbreviated for readability.
Production keys continue to use the current MD5 conventions.

Workspace evidence:

* [Silver pipeline](../../src/analytics/databricks/dlt/pl_silver.py)
* [Gold pipeline](../../src/analytics/databricks/dlt/pl_gold.py)
* [Development tenant configuration](../../config/dev/tenants.json)
* [Comprehensive DA05 logical model](./usage-telemetry-da05-logical-model.md)

## Requirement 1: Asset Usage Facts

### Requirement explanation

For every SharePoint site, Team, Power BI asset, Power App, Power Automate flow, and
mailbox, report daily usage by operation class. Required outputs are:

* Activity Volume
* Active Users over trailing 7, 30, and 90-day windows
* Usage Trend
* Last Read Timestamp
* Last Write Timestamp

The design uses two facts:

1. `fact_person_asset_activity_daily` preserves the person and raw operation needed for
   exact distinct-person calculations.
2. `fact_asset_usage_daily` is the smaller Power BI fact at the required daily asset and
   operation-class grain.

Daily distinct-user counts cannot be added across dates. The trailing 7, 30, and 90-day
counts are calculated from the person fact and persisted on the asset fact.

### Dummy business example

On 22 September 2026, the `Finance Hub` SharePoint site has:

* Two Read actions by Alice
* One Read action by a guest account for Bob
* One Write action by Alice

The expected asset result is:

| Asset | Class | Volume | Daily users | Active 7d | Active 30d | Active 90d | Last activity |
|---|---|---:|---:|---:|---:|---:|---|
| Finance Hub | Read | 3 | 2 | 2 | 2 | 2 | 2026-09-22 10:15 UTC |
| Finance Hub | Write | 1 | 1 | 1 | 1 | 1 | 2026-09-22 10:10 UTC |

To show the pattern generalizes past SharePoint, this document also carries a second
asset through every table: a Microsoft Team named `Sales Team`. On 22 September 2026,
the `Sales Team` has:

* One `MessagesListed` (Read) action by Alice
* One `MessageSent` (Write) action by Alice
* One `MessagesListed` (Read) action by the guest account for Bob
* One `MemberAdded` (Admin) action by Alice

The expected asset result is:

| Asset | Class | Volume | Daily users | Active 7d | Active 30d | Active 90d | Last activity |
|---|---|---:|---:|---:|---:|---:|---|
| Sales Team | Read | 2 | 2 | 2 | 2 | 2 | 2026-09-22 11:10 UTC |
| Sales Team | Write | 1 | 1 | 1 | 1 | 1 | 2026-09-22 11:05 UTC |
| Sales Team | Admin | 1 | 1 | 1 | 1 | 1 | 2026-09-22 11:15 UTC |

The Teams example also exercises the Admin class, which the SharePoint example above
does not produce, so between the two assets every populated operation class in this
document (Read, Write, Admin) has at least one real fact row.

### Existing metadata that supports the requirement

| Layer | Current table | Current columns used | How it supports DA05 | Status |
|---|---|---|---|---|
| Silver | `audit_entra`, `audit_exchange`, `audit_general`, `audit_sharepoint` | `audit_id`, `user_id`, `operation`, `workload`, `result_status`, `organization_id`, `object_id`, `user_type`, `created_at`, `environment`, `source_key`, `batch_id` | Event count, event time, actor, workload, raw operation, generic object locator | Existing, reused |
| Silver | `spo_sites` | `site_silver_id`, `sharepoint_id`, `source_key`, `environment`, `web_url`, `display_name`, `source_modified_at` | SharePoint and OneDrive asset identity and metadata | Existing, reused |
| Silver | `teams`, `teams_team_details` | `team_id`, `team_silver_id`, `entra_object_id`, `source_key`, `environment`, `display_name`, `is_archived`, `last_updated_at` | Team asset identity and lifecycle | Existing, reused |
| Silver | `mailboxes` | `mailbox_id`, `exchange_guid`, `external_directory_object_id`, `source_key`, `environment`, `display_name`, `primary_smtp_address`, `recipient_type`, `last_updated_at` | Mailbox asset identity and metadata | Existing, reused |
| Silver | `powerbi_apps`, `powerbi_workspaces` | `app_id`, `app_object_id`, `workspace_id`, `workspace_object_id`, `name`, `source_key`, `environment`, `last_updated_at`, `_record` | Power BI application and workspace identity; nested artifacts remain in `_record` | Existing, reused |
| Silver | `powerplat_apps` | `app_id`, `app_object_id`, `env_name`, `display_name`, `source_key`, `environment`, `last_modified_at` | Power App identity and metadata | Existing, reused |
| Silver | `powerplat_flows` | `flow_id`, `flow_object_id`, `env_name`, `display_name`, `source_key`, `environment`, `last_modified_at` | Power Automate flow identity and metadata | Existing, reused |
| Gold | `dim_date` | `date_key`, `date`, `year`, `quarter`, `month`, `week_of_year`, `iso_year` | Date slicing and trailing-window boundaries | Existing, reused |
| Gold | `dim_tenant` | `source_key`, `tenant_id`, `tenant_role`, `gold_loaded_at` | Resource tenant and source/target role | Existing, reused |
| Gold | `dim_spo_site` | `site_silver_id`, `source_key`, `environment`, `web_url`, `display_name`, `source_modified_at` | Existing SharePoint asset dimension | Existing, reused |
| Gold | `shared_data_sets` | `shared_data_set_id`, `shared_data_set_type`, `team_silver_id`, `source_key`, `environment`, `display_name` | Existing Team, M365 Group, and standalone-site conformed entity | Existing, reused |

### Current source limitation

The four Silver audit tables retain only a common envelope. `object_id` can support some
SharePoint URL matching, but it does not consistently identify Team, Power BI,
Power Platform, or mailbox assets. The physical Gold model is valid, but complete asset
population requires workload-specific audit fields to be retained or parsed in Silver.

Power BI reports, dashboards, datasets, and dataflows also need stable flattened Silver
tables because the current workspace scan stores nested artifacts in `_record`.

### Tables needed for Requirement 1

| Table | Type | Status | Requirement role |
|---|---|---|---|
| `dim_date` | Dimension | Existing, reused | Daily and trailing-window time intelligence |
| `dim_tenant` | Dimension | Existing, reused | Resource tenant |
| `dim_spo_site`, `shared_data_sets` | Workload dimensions | Existing, reused | Source metadata for SharePoint and Team asset members |
| `dim_asset` | Conformed dimension | New | One common asset key across all six workloads |
| `dim_operation_class` | Dimension | New | Read, Write, Share, Admin, Unclassified |
| `dim_evidence_source` | Dimension | New | Audit, Graph Usage, or Metadata Fallback |
| `fact_person_asset_activity_daily` | Fact | New | Person-level daily calculation base |
| `fact_asset_usage_daily` | Fact | New | Primary Power BI asset fact |

### Physical model for Requirement 1

#### `dim_asset` (`New`)

| Column | Databricks type | Null | Source or rule |
|---|---|---:|---|
| `asset_key` | `STRING` | No | PK, MD5 of `source_key`, `asset_type`, and native ID |
| `source_key` | `STRING` | No | FK to existing `dim_tenant.source_key` |
| `environment` | `STRING` | No | Existing Silver `environment` |
| `workload` | `STRING` | No | Normalized source workload |
| `asset_type` | `STRING` | No | SharePointSite, Team, PowerBIReport, PowerApp, PowerAutomateFlow, Mailbox |
| `native_asset_id` | `STRING` | No | Workload-native object ID |
| `source_dimension` | `STRING` | No | Source table name such as `dim_spo_site` |
| `source_dimension_key` | `STRING` | No | Existing workload-dimension key |
| `asset_name` | `STRING` | No | Existing display or asset name |
| `asset_locator` | `STRING` | Yes | URL, SMTP address, or workspace/app locator |
| `parent_asset_key` | `STRING` | Yes | Optional parent Team, workspace, or solution |
| `asset_status` | `STRING` | Yes | Active, Archived, Disabled, Deleted |
| `source_created_at` | `TIMESTAMP` | Yes | Existing source creation timestamp |
| `source_modified_at` | `TIMESTAMP` | Yes | Existing source modification timestamp |
| `valid_from` | `TIMESTAMP` | No | Type 2 start |
| `valid_to` | `TIMESTAMP` | No | Type 2 end |
| `is_current` | `BOOLEAN` | No | Type 2 current flag |
| `gold_loaded_at` | `TIMESTAMP` | No | Gold load time |

#### `dim_evidence_source` (`New`)

| Column | Databricks type | Null | Source or rule |
|---|---|---:|---|
| `evidence_source_key` | `TINYINT` | No | PK |
| `evidence_source` | `STRING` | No | Audit, Graph Usage, Metadata Fallback |
| `source_priority` | `TINYINT` | No | Audit 1, Graph Usage 2, Metadata Fallback 3 |
| `supports_person_attribution` | `BOOLEAN` | No | Governance metadata |
| `supports_activity_volume` | `BOOLEAN` | No | Governance metadata |
| `supports_operation_class` | `BOOLEAN` | No | Governance metadata |

#### `fact_person_asset_activity_daily` (`New`)

| Column | Databricks type | Null | Source or rule |
|---|---|---:|---|
| `activity_date_key` | `INT` | No | FK to `dim_date`, derived from audit `created_at` |
| `activity_month` | `DATE` | No | Partition column, first day of month |
| `source_key` | `STRING` | No | FK to `dim_tenant`, from audit `source_key` |
| `person_id` | `STRING` | No | FK to modified `dim_people` or unresolved member |
| `user_silver_id` | `STRING` | No | FK to existing `dim_user` |
| `asset_key` | `STRING` | No | FK to new `dim_asset` |
| `operation_key` | `STRING` | No | FK to new `dim_operation` |
| `operation_class_key` | `TINYINT` | No | FK to new `dim_operation_class` |
| `evidence_source_key` | `TINYINT` | No | FK to `dim_evidence_source` |
| `activity_count` | `BIGINT` | No | Count of deduplicated audit IDs |
| `successful_activity_count` | `BIGINT` | No | Count where `result_status` is successful |
| `failed_activity_count` | `BIGINT` | No | Count where `result_status` is unsuccessful |
| `first_activity_at` | `TIMESTAMP` | Yes | Minimum audit `created_at` |
| `last_activity_at` | `TIMESTAMP` | Yes | Maximum audit `created_at` |
| `is_cross_tenant` | `BOOLEAN` | Yes | Null when home tenant is unavailable |
| `cross_tenant_basis` | `STRING` | No | HomeTenant, GuestHeuristic, SameTenant, Unknown |
| `identity_resolution_confidence_pct` | `DECIMAL(5,2)` | No | Resolution score |
| `asset_resolution_confidence_pct` | `DECIMAL(5,2)` | No | Resolution score |
| `gold_loaded_at` | `TIMESTAMP` | No | Gold load time |

Logical primary key:

```text
(activity_date_key, source_key, person_id, user_silver_id,
 asset_key, operation_key, evidence_source_key)
```

#### `fact_asset_usage_daily` (`New`)

| Column | Databricks type | Null | Source or rule |
|---|---|---:|---|
| `activity_date_key` | `INT` | No | FK to `dim_date` |
| `activity_month` | `DATE` | No | Partition column |
| `source_key` | `STRING` | No | FK to `dim_tenant` |
| `asset_key` | `STRING` | No | FK to `dim_asset` |
| `operation_class_key` | `TINYINT` | No | FK to `dim_operation_class` |
| `evidence_source_key` | `TINYINT` | No | FK to `dim_evidence_source` |
| `activity_volume` | `BIGINT` | No | Sum of person fact `activity_count` |
| `daily_distinct_people` | `BIGINT` | No | Exact distinct `person_id` on the day |
| `active_people_7d` | `BIGINT` | No | Exact distinct people from date minus 6 through date |
| `active_people_30d` | `BIGINT` | No | Exact distinct people from date minus 29 through date |
| `active_people_90d` | `BIGINT` | No | Exact distinct people from date minus 89 through date |
| `last_activity_at` | `TIMESTAMP` | Yes | Latest activity for this class through the date |
| `cross_tenant_activity_volume` | `BIGINT` | No | Activity where `is_cross_tenant = true` |
| `resolved_person_count` | `BIGINT` | No | Resolved person rows |
| `unresolved_person_count` | `BIGINT` | No | Unresolved person rows |
| `coverage_pct` | `DECIMAL(5,2)` | No | Resolved volume divided by total volume |
| `gold_loaded_at` | `TIMESTAMP` | No | Gold load time |

Logical primary key:

```text
(activity_date_key, source_key, asset_key,
 operation_class_key, evidence_source_key)
```

Usage Trend is calculated from additive daily rows rather than stored as a second source
of truth:

```text
usage_trend_pct =
    100 * (current_period_activity - prior_equal_period_activity)
    / NULLIF(prior_equal_period_activity, 0)
```

If the prior period is zero, return blank and expose an `is_new_usage` indicator.

### Requirement 1 dummy data from current schemas

#### Existing Gold rows used as inputs

`dim_date` already contains 22 September 2026 because its current range is 2020 through
2030.

| date_key | date | year | month | week_of_year | iso_year |
|---:|---|---:|---:|---:|---:|
| 20260922 | 2026-09-22 | 2026 | 9 | 39 | 2026 |

`dim_tenant` can contain this row from the existing development tenant configuration.

| source_key | tenant_id | tenant_role |
|---|---|---|
| `madev1` | `2b1943ef-f207-4cd2-884f-3d1c420dbe58` | source |

`dim_spo_site` can contain this schema-aligned dummy site.

| site_silver_id | source_key | environment | web_url | display_name | source_modified_at |
|---|---|---|---|---|---|
| `site_finance_hash` | `madev1` | source | `https://xtlab1.sharepoint.com/sites/finance` | Finance Hub | 2026-09-20 08:00:00 |

Silver `teams` can contain this schema-aligned dummy team. The columns shown are the
exact output columns of the Silver `teams` table.

| team_id | entra_object_id | display_name | mail | visibility | source_key | environment | source_created_at |
|---|---|---|---|---|---|---|---|
| `madev1_grp-sales-001` | `grp-sales-001` | Sales Team | `salesteam@xtlab1.onmicrosoft.com` | Private | `madev1` | source | 2025-01-10 09:00:00 |

Silver `teams_team_details` supplies the richer payload, including the URL used later as
the asset locator.

| team_silver_id | entra_object_id | display_name | visibility | is_archived | members_count | web_url | source_key | environment |
|---|---|---|---:|---|---:|---|---|---|
| `madev1_grp-sales-001` | `grp-sales-001` | Sales Team | Private | false | 6 | `https://teams.microsoft.com/l/team/19%3ateamchannel_xxx/conversations?groupId=grp-sales-001` | `madev1` | source |

The existing Gold `shared_data_sets` table already joins these two Silver tables for
type `team`. This row can be produced by the current pipeline logic without any change.

| shared_data_set_id | shared_data_set_type | display_name | source_key | environment | entra_group_id | team_silver_id | member_count | is_archived | visibility |
|---|---|---|---|---|---|---|---:|---|---|
| `sds_sales_team_hash` | team | Sales Team | `madev1` | source | `grp-sales-001` | `madev1_grp-sales-001` | 6 | false | Private |

#### New `dim_asset` row derived from `dim_spo_site`

| asset_key | source_key | workload | asset_type | native_asset_id | source_dimension | source_dimension_key | asset_name | asset_locator |
|---|---|---|---|---|---|---|---|---|
| `asset_finance_hash` | `madev1` | SharePoint | SharePointSite | `site-guid-001` | `dim_spo_site` | `site_finance_hash` | Finance Hub | `https://xtlab1.sharepoint.com/sites/finance` |

The new asset row does not duplicate SharePoint metadata. It provides the common
cross-workload key and points back to the existing `dim_spo_site` row.

#### New `dim_asset` row derived from `shared_data_sets`

| asset_key | source_key | workload | asset_type | native_asset_id | source_dimension | source_dimension_key | asset_name | asset_locator |
|---|---|---|---|---|---|---|---|---|
| `asset_sales_team_hash` | `madev1` | MicrosoftTeams | Team | `grp-sales-001` | `shared_data_sets` | `sds_sales_team_hash` | Sales Team | `https://teams.microsoft.com/l/team/19%3ateamchannel_xxx/conversations?groupId=grp-sales-001` |

Two assets now exist in `dim_asset` side by side. Both point back to their own existing
workload dimension instead of copying its attributes:

| asset_key | asset_type | source_dimension | source_dimension_key |
|---|---|---|---|
| `asset_finance_hash` | SharePointSite | `dim_spo_site` | `site_finance_hash` |
| `asset_sales_team_hash` | Team | `shared_data_sets` | `sds_sales_team_hash` |

#### New `dim_evidence_source` rows

All three governed evidence sources are populated even though the dummy activity in
this document only uses Audit. Graph Usage and Metadata Fallback are shown so the
dimension is never empty, and so Section 7 of the requirement's metadata fallback
logic has a real row to point at.

| evidence_source_key | evidence_source | source_priority | supports_person_attribution | supports_activity_volume | supports_operation_class |
|---:|---|---:|---|---|---|
| 1 | Audit | 1 | true | true | true |
| 2 | Graph Usage | 2 | false | true | false |
| 3 | Metadata Fallback | 3 | false | false | false |

Graph Usage reports (for example, the Microsoft 365 usage report APIs) return activity
counts per resource but do not reliably attribute Read versus Write or a specific
person when reports are configured for privacy, so both of those support flags are
`false`. Metadata Fallback only carries a last-modified or last-logon timestamp, so it
cannot support activity volume, operation class, or person attribution at all; it exists
purely to produce the Last Activity fields described in DA05 requirement 7.

## Requirement 2: Read versus Write

### Requirement explanation

Every raw audit operation must map to one governed operation class:

* Read
* Write
* Share
* Admin
* Unclassified

Read and Write cannot be inferred reliably from the workload alone. The model needs an
explicit versioned operation taxonomy. Raw operations remain available for audit and
drill-through, while Power BI uses the smaller class dimension.

### Dummy business example

For the Finance Hub:

* `FileAccessed` and `FileDownloaded` map to Read.
* `FileModified` maps to Write.
* Three Read events divided by one Write event produce a Read/Write Ratio of `3.0`.

For the Sales Team:

* `MessagesListed` maps to Read.
* `MessageSent` maps to Write.
* `MemberAdded` maps to Admin, which is excluded from the Read/Write Ratio.
* Two Read events divided by one Write event produce a Read/Write Ratio of `2.0`.

### Existing metadata that supports the requirement

| Layer | Current table | Current columns used | How it supports DA05 | Status |
|---|---|---|---|---|
| Silver | All four `audit_*` tables | `operation`, `workload`, `record_type`, `result_status`, `created_at` | Supplies raw operation and workload | Existing, reused |
| Gold | None | None | No current governed operation taxonomy exists | Gap |

### Tables needed for Requirement 2

| Table | Type | Status | Requirement role |
|---|---|---|---|
| `dim_operation_class` | Dimension | New | Five governed reporting categories |
| `dim_operation` | Dimension | New | Maps raw workload-operation pairs to a class |
| `fact_person_asset_activity_daily` | Fact | New, shared with Requirement 1 | Preserves the raw operation |
| `fact_asset_usage_daily` | Fact | New, shared with Requirement 1 | Aggregates to operation class |

### Physical model for Requirement 2

#### `dim_operation_class` (`New`)

| Column | Databricks type | Null | Source or rule |
|---|---|---:|---|
| `operation_class_key` | `TINYINT` | No | PK |
| `operation_class` | `STRING` | No | Read, Write, Share, Admin, Unclassified |
| `sort_order` | `TINYINT` | No | Report ordering |
| `description` | `STRING` | No | Governed business definition |
| `is_user_activity` | `BOOLEAN` | No | Distinguishes user activity from administrative changes |

#### `dim_operation` (`New`)

| Column | Databricks type | Null | Source or rule |
|---|---|---:|---|
| `operation_key` | `STRING` | No | PK, MD5 of normalized workload and operation |
| `workload` | `STRING` | No | Existing audit `workload` |
| `raw_operation` | `STRING` | No | Existing audit `operation` |
| `normalized_operation` | `STRING` | No | Lowercase trimmed operation |
| `operation_class_key` | `TINYINT` | No | FK to `dim_operation_class` |
| `activity_family` | `STRING` | Yes | Content consumption, content change, sharing, administration |
| `mapping_status` | `STRING` | No | Approved, Proposed, Unclassified |
| `mapping_rule` | `STRING` | No | Exact, Prefix, RegularExpression, Default |
| `valid_from` | `TIMESTAMP` | No | Type 2 start |
| `valid_to` | `TIMESTAMP` | No | Type 2 end |
| `is_current` | `BOOLEAN` | No | Type 2 current flag |
| `gold_loaded_at` | `TIMESTAMP` | No | Gold load time |

### Requirement 2 dummy data from current schemas

#### Existing Silver audit rows

These rows use only columns present in the current Silver `audit_sharepoint` schema.

| audit_id | user_id | operation | workload | result_status | object_id | created_at | environment | source_key |
|---|---|---|---|---|---|---|---|---|
| `evt-001` | `alice@xtlab1.onmicrosoft.com` | FileAccessed | SharePoint | Succeeded | `https://xtlab1.sharepoint.com/sites/finance/Shared Documents/Budget.xlsx` | 2026-09-22 09:00:00 | source | `madev1` |
| `evt-002` | `alice@xtlab1.onmicrosoft.com` | FileDownloaded | SharePoint | Succeeded | `https://xtlab1.sharepoint.com/sites/finance/Shared Documents/Budget.xlsx` | 2026-09-22 10:05:00 | source | `madev1` |
| `evt-003` | `alice@xtlab1.onmicrosoft.com` | FileModified | SharePoint | Succeeded | `https://xtlab1.sharepoint.com/sites/finance/Shared Documents/Budget.xlsx` | 2026-09-22 10:10:00 | source | `madev1` |
| `evt-004` | `bob_ext#EXT#@xtlab1.onmicrosoft.com` | FileAccessed | SharePoint | Succeeded | `https://xtlab1.sharepoint.com/sites/finance/Shared Documents/Budget.xlsx` | 2026-09-22 10:15:00 | source | `madev1` |

These rows use only columns present in the current Silver `audit_general` schema. The
`object_id` for Teams events is the Entra group ID rather than a URL, which is why
`dim_asset` resolves Teams assets by `native_asset_id` instead of `asset_locator`.

| audit_id | user_id | operation | workload | result_status | object_id | created_at | environment | source_key |
|---|---|---|---|---|---|---|---|---|
| `evt-101` | `alice@xtlab1.onmicrosoft.com` | MessagesListed | MicrosoftTeams | Succeeded | `grp-sales-001` | 2026-09-22 11:00:00 | source | `madev1` |
| `evt-102` | `alice@xtlab1.onmicrosoft.com` | MessageSent | MicrosoftTeams | Succeeded | `grp-sales-001` | 2026-09-22 11:05:00 | source | `madev1` |
| `evt-103` | `bob_ext#EXT#@xtlab1.onmicrosoft.com` | MessagesListed | MicrosoftTeams | Succeeded | `grp-sales-001` | 2026-09-22 11:10:00 | source | `madev1` |
| `evt-104` | `alice@xtlab1.onmicrosoft.com` | MemberAdded | MicrosoftTeams | Succeeded | `grp-sales-001` | 2026-09-22 11:15:00 | source | `madev1` |

#### New operation dimensions

| operation_class_key | operation_class | description |
|---:|---|---|
| 0 | Unclassified | Requires taxonomy review |
| 1 | Read | Consumes content without changing it |
| 2 | Write | Creates, updates, deletes, sends, or executes content |
| 3 | Share | Grants or distributes access |
| 4 | Admin | Changes configuration, policy, membership, or administration |

| operation_key | workload | raw_operation | operation_class_key | activity_family | mapping_status |
|---|---|---|---:|---|---|
| `op_access_hash` | SharePoint | FileAccessed | 1 | Content consumption | Approved |
| `op_download_hash` | SharePoint | FileDownloaded | 1 | Content consumption | Approved |
| `op_modify_hash` | SharePoint | FileModified | 2 | Content change | Approved |
| `op_teams_messageslisted_hash` | MicrosoftTeams | MessagesListed | 1 | Content consumption | Approved |
| `op_teams_messagesent_hash` | MicrosoftTeams | MessageSent | 2 | Content change | Approved |
| `op_teams_memberadded_hash` | MicrosoftTeams | MemberAdded | 4 | Administration | Approved |

#### New asset fact rows

| activity_date_key | source_key | asset_key | operation_class_key | evidence_source_key | activity_volume | daily_distinct_people | active_people_7d | active_people_30d | active_people_90d | last_activity_at |
|---:|---|---|---:|---:|---:|---:|---:|---:|---:|---|
| 20260922 | `madev1` | `asset_finance_hash` | 1 | 1 | 3 | 2 | 2 | 2 | 2 | 2026-09-22 10:15:00 |
| 20260922 | `madev1` | `asset_finance_hash` | 2 | 1 | 1 | 1 | 1 | 1 | 1 | 2026-09-22 10:10:00 |
| 20260922 | `madev1` | `asset_sales_team_hash` | 1 | 1 | 2 | 2 | 2 | 2 | 2 | 2026-09-22 11:10:00 |
| 20260922 | `madev1` | `asset_sales_team_hash` | 2 | 1 | 1 | 1 | 1 | 1 | 1 | 2026-09-22 11:05:00 |
| 20260922 | `madev1` | `asset_sales_team_hash` | 4 | 1 | 1 | 1 | 1 | 1 | 1 | 2026-09-22 11:15:00 |

Interpretation:

* Last Read Timestamp is the Read row's `last_activity_at`.
* Last Write Timestamp is the Write row's `last_activity_at`.
* Read/Write Ratio for Finance Hub is `3 / 1 = 3.0`.
* Read/Write Ratio for Sales Team is `2 / 1 = 2.0`; the Admin row is excluded from the
  ratio because Admin is neither Read nor Write.
* New unmatched operations use class `0` until taxonomy stewardship approves them.

## Requirement 3: Usage by Person

### Requirement explanation

Usage must resolve to a unique person while preserving the tenant account used for the
activity. The model must distinguish:

* Internal people using member accounts
* External people using guest accounts
* External-tenant members
* The person's home tenant
* The resource tenant where the asset is hosted
* Cross-tenant usage

A person and a tenant account are different business entities. One person can have a
source member account, a target member account, and multiple guest accounts.

### Dummy business example

Alice is an internal `madev1` member. Bob uses a `madev1` guest account. Current schema
metadata can identify Bob as Guest, but it cannot authoritatively identify his home
tenant. Bob's cross-tenant status therefore remains unverified until a home-tenant field
is ingested.

### Existing metadata that supports the requirement

| Layer | Current table | Current columns used | How it supports DA05 | Status |
|---|---|---|---|---|
| Silver | `users` | `user_id`, `entra_object_id`, `source_key`, `environment`, `user_principal_name`, `display_name`, `mail`, `proxy_addresses`, `user_type`, `account_enabled`, `employee_id` | Tenant-account identity and Member/Guest classification | Existing, reused |
| Silver | All four `audit_*` tables | `user_id`, `user_type`, `organization_id`, `source_key`, `environment` | Actor value and resource tenant | Existing, reused |
| Silver | `user_mapping_candidates` | `source_user_id`, `target_user_id`, match fields and status | Source-to-target identity resolution | Existing, reused |
| Gold | `dim_user` | `user_silver_id`, `user_id`, `entra_object_id`, `source_key`, `environment`, `user_principal_name`, `display_name`, `mail`, `user_type`, `account_enabled`, `org_id` | Existing account dimension | Existing, modified |
| Gold | `dim_people` | `person_id`, `source_user_id`, `target_user_id`, source and target tenant keys, identity attributes, organization, migration path and status | Existing canonical migration person | Existing, modified |
| Gold | `user_map` | `source_user_id`, `target_user_id`, `match_context`, `match_score`, `match_type`, `mapping_status` | Approved source-to-target mapping | Existing, reused |

### Current source limitations

The parsed Entra user schema contains `externalUserState`, but Silver `users` does not
select it. Neither Silver `users` nor the common audit output retains an authoritative
home tenant ID for guest users.

Current Gold `dim_people` includes licensed, non-resource source users. It does not
include every guest, external person, or target-only person observed in telemetry.

These are explicit modifications:

* Add `external_user_state` and authoritative `home_tenant_id` to Silver `users` when a
  supported source is available.
* Extend `dim_user` with guest state and home tenant.
* Broaden `dim_people` to include observed external people while preserving existing
  `person_id` values for current members.
* Add `bridge_person_account` to represent one person with multiple tenant accounts.

Until home tenant is available, store `is_cross_tenant = NULL` and
`cross_tenant_basis = 'Unknown'`. Do not infer an authoritative tenant ID from the UPN
domain.

### Tables needed for Requirement 3

| Table | Type | Status | Requirement role |
|---|---|---|---|
| `dim_tenant` | Dimension | Existing, modified | Continue source/target rows and allow external home-tenant rows when discovered |
| `dim_user` | Dimension | Existing, modified | Preserve account grain and add guest/home-tenant attributes |
| `dim_people` | Dimension | Existing, modified | Broaden the person population beyond licensed source users |
| `user_map` | Mapping | Existing, reused | Seed approved source-to-target account links |
| `bridge_person_account` | Bridge | New | Resolve many tenant accounts to one person |
| `fact_person_asset_activity_daily` | Fact | New, shared with Requirement 1 | Attribute activity to person and account |

### Physical model for Requirement 3

#### `dim_tenant` (`Existing, modified`)

Retain `source_key` as the primary key to avoid breaking current Gold relationships.

| Column | Databricks type | Change | Description |
|---|---|---|---|
| `source_key` | `STRING` | Existing | Current PK |
| `tenant_id` | `STRING` | Existing | Entra tenant GUID |
| `tenant_role` | `STRING` | Modified values | Extend current source/target values with external |
| `tenant_name` | `STRING` | Add | Friendly tenant name |
| `default_domain` | `STRING` | Add | Optional verified domain |
| `discovery_source` | `STRING` | Add | Configuration, GuestIdentity, Audit, Manual |
| `gold_loaded_at` | `TIMESTAMP` | Existing | Gold load time |

External tenant rows require an authoritative tenant GUID. They are not created from an
email domain alone.

#### `dim_user` (`Existing, modified`)

Keep the existing `user_silver_id` primary key and all current identity, mailbox,
license, OneDrive, and organization attributes.

| Column | Databricks type | Change | Description |
|---|---|---|---|
| `user_silver_id` | `STRING` | Existing | PK |
| `user_id` | `STRING` | Existing | Silver compound account ID |
| `entra_object_id` | `STRING` | Existing | Entra object ID |
| `source_key` | `STRING` | Existing | Resource tenant |
| `environment` | `STRING` | Existing | Source or target |
| `user_principal_name` | `STRING` | Existing | Normalized UPN |
| `mail` | `STRING` | Existing | Normalized mail |
| `display_name` | `STRING` | Existing | Display name |
| `user_type` | `STRING` | Existing | Member or Guest |
| `account_enabled` | `BOOLEAN` | Existing | Enabled state |
| `org_id` | `STRING` | Existing | Source organization |
| `external_user_state` | `STRING` | Add | PendingAcceptance or Accepted |
| `home_tenant_id` | `STRING` | Add | Authoritative home tenant GUID |
| `home_tenant_source_key` | `STRING` | Add | FK to external row in `dim_tenant` |
| `last_updated_at` | `TIMESTAMP` | Existing | Silver source timestamp |
| `gold_loaded_at` | `TIMESTAMP` | Existing | Gold load time |

#### `dim_people` (`Existing, modified`)

Keep the existing `person_id` and all current migration attributes.

| Column | Databricks type | Change | Description |
|---|---|---|---|
| `person_id` | `STRING` | Existing | PK |
| `source_user_id` | `STRING` | Existing | Source account |
| `target_user_id` | `STRING` | Existing | Target account |
| `source_tenant_key` | `STRING` | Existing | Source tenant |
| `target_tenant_key` | `STRING` | Existing | Target tenant |
| `organization_id` | `STRING` | Existing | Existing organization FK |
| `display_name_source` | `STRING` | Existing | Source display name |
| `primary_email_source` | `STRING` | Existing | Source email |
| `migration_path` | `STRING` | Existing | Existing migration path |
| `migration_status` | `STRING` | Existing | Existing migration status |
| `person_category` | `STRING` | Add | Internal, External, ServicePrincipal, Unresolved |
| `home_tenant_id` | `STRING` | Add | Authoritative home tenant GUID |
| `home_tenant_source_key` | `STRING` | Add | FK to `dim_tenant` when known |
| `identity_status` | `STRING` | Add | Resolved, Ambiguous, Unresolved |
| `identity_confidence_pct` | `DECIMAL(5,2)` | Add | Confidence from 0 to 100 |
| `last_updated_at` | `TIMESTAMP` | Existing | Source timestamp |
| `gold_loaded_at` | `TIMESTAMP` | Existing | Gold load time |

The modified population rule includes a person observed in DA05 activity even when that
person is not a licensed source member. Existing migration persons keep their current
keys.

#### `bridge_person_account` (`New`)

| Column | Databricks type | Null | Source or rule |
|---|---|---:|---|
| `person_account_key` | `STRING` | No | PK, MD5 of person, user, role, valid start |
| `person_id` | `STRING` | No | FK to modified `dim_people` |
| `user_silver_id` | `STRING` | No | FK to modified `dim_user` |
| `identity_role` | `STRING` | No | SourceMember, TargetMember, Guest, ExternalMember |
| `match_method` | `STRING` | No | ObjectId, EmployeeIdAndUpn, NormalizedUpn, GuestIssuer, Manual |
| `match_confidence_pct` | `DECIMAL(5,2)` | No | Confidence from 0 to 100 |
| `mapping_status` | `STRING` | No | Approved, Candidate, Rejected, Unresolved |
| `valid_from` | `TIMESTAMP` | No | Effective start |
| `valid_to` | `TIMESTAMP` | No | Effective end |
| `is_current` | `BOOLEAN` | No | Current flag |
| `gold_loaded_at` | `TIMESTAMP` | No | Gold load time |

### Requirement 3 dummy data from current schemas

#### Existing Silver `users` rows

| user_id | entra_object_id | source_key | environment | user_principal_name | display_name | mail | user_type | account_enabled |
|---|---|---|---|---|---|---|---|---|
| `madev1_user-guid-alice` | `user-guid-alice` | `madev1` | source | `alice@xtlab1.onmicrosoft.com` | Alice Adams | `alice@xtlab1.onmicrosoft.com` | Member | true |
| `madev1_user-guid-bob-guest` | `user-guid-bob-guest` | `madev1` | source | `bob_ext#EXT#@xtlab1.onmicrosoft.com` | Bob Brown | `bob@fabrikam.example` | Guest | true |

These fields are present today. `home_tenant_id` is intentionally absent because the
current Silver output does not contain it.

#### Existing Gold `dim_user` rows

| user_silver_id | user_id | source_key | environment | user_principal_name | mail | user_type | account_enabled |
|---|---|---|---|---|---|---|---|
| `user_alice_hash` | `madev1_user-guid-alice` | `madev1` | source | `alice@xtlab1.onmicrosoft.com` | `alice@xtlab1.onmicrosoft.com` | Member | true |
| `user_bob_hash` | `madev1_user-guid-bob-guest` | `madev1` | source | `bob_ext#EXT#@xtlab1.onmicrosoft.com` | `bob@fabrikam.example` | Guest | true |

#### Modified Gold `dim_people` rows

| person_id | source_user_id | source_tenant_key | display_name_source | primary_email_source | person_category | home_tenant_id | identity_status |
|---|---|---|---|---|---|---|---|
| `person_alice_hash` | `madev1_user-guid-alice` | `madev1` | Alice Adams | `alice@xtlab1.onmicrosoft.com` | Internal | `2b1943ef-f207-4cd2-884f-3d1c420dbe58` | Resolved |
| `person_bob_hash` | `madev1_user-guid-bob-guest` | `madev1` | Bob Brown | `bob@fabrikam.example` | External | null | Resolved |

Alice can reuse the current `dim_people` derivation. Bob is a new population member
derived from the existing guest account, but his home tenant remains null.

#### New `bridge_person_account` rows

| person_id | user_silver_id | identity_role | match_method | match_confidence_pct | mapping_status |
|---|---|---|---|---:|---|
| `person_alice_hash` | `user_alice_hash` | SourceMember | ObjectId | 100.00 | Approved |
| `person_bob_hash` | `user_bob_hash` | Guest | NormalizedUpn | 100.00 | Approved |

#### New person fact rows

Rows for Finance Hub use `asset_key = asset_finance_hash`. Rows for Sales Team use
`asset_key = asset_sales_team_hash`. All rows use `source_key = madev1` and
`evidence_source_key = 1` for Audit.

| activity_date_key | person_id | user_silver_id | asset_key | operation_key | operation_class_key | activity_count | last_activity_at | is_cross_tenant | cross_tenant_basis |
|---:|---|---|---|---|---:|---:|---|---|---|
| 20260922 | `person_alice_hash` | `user_alice_hash` | `asset_finance_hash` | `op_access_hash` | 1 | 1 | 2026-09-22 09:00:00 | false | SameTenant |
| 20260922 | `person_alice_hash` | `user_alice_hash` | `asset_finance_hash` | `op_download_hash` | 1 | 1 | 2026-09-22 10:05:00 | false | SameTenant |
| 20260922 | `person_alice_hash` | `user_alice_hash` | `asset_finance_hash` | `op_modify_hash` | 2 | 1 | 2026-09-22 10:10:00 | false | SameTenant |
| 20260922 | `person_bob_hash` | `user_bob_hash` | `asset_finance_hash` | `op_access_hash` | 1 | 1 | 2026-09-22 10:15:00 | null | Unknown |
| 20260922 | `person_alice_hash` | `user_alice_hash` | `asset_sales_team_hash` | `op_teams_messageslisted_hash` | 1 | 1 | 2026-09-22 11:00:00 | false | SameTenant |
| 20260922 | `person_alice_hash` | `user_alice_hash` | `asset_sales_team_hash` | `op_teams_messagesent_hash` | 2 | 1 | 2026-09-22 11:05:00 | false | SameTenant |
| 20260922 | `person_bob_hash` | `user_bob_hash` | `asset_sales_team_hash` | `op_teams_messageslisted_hash` | 1 | 1 | 2026-09-22 11:10:00 | null | Unknown |
| 20260922 | `person_alice_hash` | `user_alice_hash` | `asset_sales_team_hash` | `op_teams_memberadded_hash` | 4 | 1 | 2026-09-22 11:15:00 | false | SameTenant |

Interpretation:

* Alice generated three Finance Hub activities and four Sales Team activities across
  both assets, all as an internal member.
* Bob generated one Read activity on each asset and used the same guest account both
  times, because a person keeps one `bridge_person_account` row per tenant account
  regardless of how many assets that account touches.
* Bob's cross-tenant usage is not counted in an authoritative percentage because his
  home tenant is unknown in the current schema.
* After a supported home-tenant source identifies Bob's tenant, his person and account
  rows receive that tenant, and the same fact rows can be reprocessed with
  `is_cross_tenant = true` and `cross_tenant_basis = 'HomeTenant'`.

## Requirement 4: Person Activity Profile

### Requirement explanation

Migration coordinators need a single, per-person answer to three questions: is this
person active, which tenants do they actually sign in to, and which applications and
devices do they use. This is a different signal from Requirement 1's asset activity.
Requirement 1 tells you *what happened to a SharePoint site or a Team*. Requirement 4
tells you *whether the human behind an account is still showing up at all*, even on
days when they touched no tracked asset. A person can sign in, check email on a phone,
and never once trigger a SharePoint or Teams audit event. Without a sign-in and device
signal, that person would look dormant in Requirement 1's facts even though they are
clearly present.

The user story is: *"As a migration coordinator, I want to know whether each person is
active, in which tenants they sign in, and which applications and devices they use so
that inactive accounts can be handled and batches contain people who will notice."*
That drives three sub-questions per person:

* **Active or not** — a single, explainable status (Active, AtRisk, Dormant, Disabled)
  a coordinator can filter a migration wave on.
* **Sign-in tenants** — which `dim_tenant` rows this person actually authenticated
  against, not just which rows a mailbox or SharePoint account exists in.
* **Applications and devices** — which client applications and which physical or
  virtual devices produced that sign-in activity.

### Dummy business example

Alice signs in every business day from her managed Windows laptop, using Outlook,
Teams, and SharePoint Online. She is unambiguously Active.

Bob, the guest, signs in only when he needs to review the Finance Hub site. His last
sign-in is one day old in this example, so today he still shows as Active, but he has
no Intune-managed device (guests use unmanaged, personal devices), so his device usage
is Unresolved rather than false.

Carol Chen is a new persona introduced only for this requirement. She is an internal
`madev1` member whose account is still enabled, but her last sign-in was 95 days ago and
she generated no asset activity in that window either. She is the coordinator's
"handle before the batch" case: Dormant, but not yet Disabled, so removing her from an
active migration wave is a judgment call, not an automatic exclusion.

### Existing metadata that supports the requirement

| Layer | Current table | Current columns used | How it supports DA05 | Status |
|---|---|---|---|---|
| Silver | `sign_in_logs` | `sign_in_id`, `user_silver_id`, `user_principal_name`, `app_id`, `app_display_name`, `client_app_used`, `is_interactive`, `risk_level_during_sign_in`, `created_at`, `source_key`, `environment` | Direct, person-attributed sign-in events per tenant and per application | Existing, reused |
| Silver | `devices` | `device_id`, `azure_ad_device_id`, `trust_type`, `is_compliant`, `is_managed`, `operating_system`, `approximate_last_sign_in_at`, `registration_at`, `source_key`, `environment` | Entra device identity and device-level last-sign-in signal | Existing, reused |
| Silver | `intune_devices` | `intune_device_id`, `azure_ad_device_id`, `user_entra_object_id`, `user_principal_name`, `compliance_state`, `last_sync_at`, `enrolled_at` | The only reliable device-to-person join key today, via `user_principal_name` | Existing, reused |
| Silver | `mde_devices` | Defender-managed device inventory (not detailed further here) | Optional third device-signal source for at-risk device posture | Existing, not modeled yet |
| Silver | `users` | `account_enabled`, `department`, `job_title`, `on_prem_sync_enabled` | Whether the account itself is still enabled, for the Disabled status | Existing, reused |
| Gold | `dim_user` | `user_silver_id`, `user_principal_name`, `account_enabled` | Account-grain identity the new facts key off | Existing, reused |
| Gold | `dim_people` | `person_id`, `source_user_id`, `person_category`, `identity_status` | Person-grain identity the new facts and the activity snapshot key off | Existing, modified |
| Gold | `bridge_person_account` | `person_id`, `user_silver_id`, `identity_role` | Rolls a person's sign-ins and devices up across every tenant account they hold | Existing, reused (created in Requirement 3) |

### Current source limitations

* **No user last-sign-in field on `users` itself.** The Silver `users` table has
  `last_password_change_at` but no `last_sign_in_at`. Every "when did this person last
  sign in" answer must come from aggregating `sign_in_logs`, which is a raw, append-only
  event table with no pre-aggregation.
* **`devices` has no owner foreign key.** The Silver `devices` transformation comment
  states that "owner correlation is done in gold via device deviceId to user
  relationships," but no `owner_upn` or `owner_entra_object_id` column exists on
  `devices` today. The only column that actually resolves a device to a person is
  `intune_devices.user_principal_name` (confirmed present, lowercased and trimmed), which
  is exactly the join the existing `users_with_active_devices` Gold table already uses.
  Devices that only exist in `devices` (Entra-registered, never Intune-enrolled) cannot
  be attributed to a person and must be reported as device-owner `Unresolved`, not
  silently dropped.
* **Guests rarely have a resolvable device.** Guest accounts such as Bob's are
  virtually never Intune-managed, so `device_usage_status = 'Unresolved'` is the
  expected, correct answer for most guests — it is not a data quality defect.
* **Sign-in logs are not one of the three governed `dim_evidence_source` rows.**
  `dim_evidence_source` (Requirement 1) governs Audit, Graph Usage, and Metadata
  Fallback specifically for *asset* activity volume and operation-class attribution.
  Entra sign-in logs are a different kind of telemetry: they are natively
  person-attributed (`user_silver_id` is populated on every row) and carry no asset or
  operation-class dimension at all. Reusing `evidence_source_key = 2` (Graph Usage,
  `supports_person_attribution = false`) would understate sign-in log quality and is
  avoided. The new sign-in and device facts intentionally do not carry an
  `evidence_source_key` column for this reason.

### Tables needed for Requirement 4

| Table | Type | Status | Requirement role |
|---|---|---|---|
| `dim_application` | Dimension | New | One row per Entra client/resource application seen in sign-in logs |
| `dim_device` | Dimension | New | One row per device, unioning Entra and Intune device identity |
| `dim_people` | Dimension | Existing, modified | Add current-state activity summary columns for fast Power BI slicing |
| `fact_person_signin_daily` | Fact | New | Daily, per-person, per-tenant, per-application sign-in aggregation |
| `fact_person_device_activity_daily` | Fact | New | Daily, per-person, per-device compliance and sync snapshot |
| `fact_person_activity_daily` | Fact | New | Daily, per-person periodic snapshot: Active/AtRisk/Dormant/Disabled status and dormancy score |

### Physical model for Requirement 4

#### `dim_application` (`New`)

| Column | Databricks type | Null | Source or rule |
|---|---|---:|---|
| `application_key` | `STRING` | No | PK, MD5 of `app_id` |
| `app_id` | `STRING` | No | Entra client application ID, from `sign_in_logs.app_id` |
| `app_display_name` | `STRING` | No | From `sign_in_logs.app_display_name` |
| `app_category` | `STRING` | No | Email, Collaboration, FileStorage, BusinessIntelligence, LowCodeAutomation, Portal, Other |
| `is_first_party_microsoft` | `BOOLEAN` | No | True for Microsoft-published first-party app IDs |
| `gold_loaded_at` | `TIMESTAMP` | No | Gold load time |

`app_id` values for first-party Microsoft applications are the same GUID in every
tenant, so `dim_application` is a conformed dimension shared across source and target
tenants without modification. SCD Type 1: a rename of an app's display name overwrites
the row, because the app category and identity do not change.

#### `dim_device` (`New`)

| Column | Databricks type | Null | Source or rule |
|---|---|---:|---|
| `device_key` | `STRING` | No | PK, MD5 of `source_key` and `azure_ad_device_id` |
| `azure_ad_device_id` | `STRING` | No | Shared join key between `devices` and `intune_devices` |
| `source_key` | `STRING` | No | FK to `dim_tenant` |
| `environment` | `STRING` | No | Source or target |
| `display_name` | `STRING` | Yes | From `devices.display_name` or `intune_devices.display_name` |
| `operating_system` | `STRING` | Yes | From either source table |
| `operating_system_version` | `STRING` | Yes | From either source table |
| `manufacturer` | `STRING` | Yes | From `devices.manufacturer` |
| `model` | `STRING` | Yes | From `devices.model` |
| `trust_type` | `STRING` | Yes | From `devices.trust_type` (Entra join, AzureAd, Hybrid) |
| `is_intune_managed` | `BOOLEAN` | No | True when a matching `intune_devices` row exists |
| `is_compliant` | `BOOLEAN` | Yes | From `intune_devices.compliance_state` |
| `is_encrypted` | `BOOLEAN` | Yes | From `intune_devices.is_encrypted` |
| `owner_user_silver_id` | `STRING` | Yes | FK to `dim_user`, resolved by lower/trim UPN match on `intune_devices.user_principal_name` |
| `owner_resolution_method` | `STRING` | No | `IntuneUpnMatch` or `Unresolved` |
| `registration_at` | `TIMESTAMP` | Yes | From `devices.registration_at` |
| `enrolled_at` | `TIMESTAMP` | Yes | From `intune_devices.enrolled_at` |
| `gold_loaded_at` | `TIMESTAMP` | No | Gold load time |

SCD Type 1. `owner_user_silver_id` is deliberately nullable: an Entra-registered,
never-Intune-enrolled device, and almost every guest's personal device, will carry
`owner_resolution_method = 'Unresolved'` and a null owner. Do not infer an owner from
`devices.display_name` or from proximity in the audit log.

#### `dim_people` (`Existing, modified`, additive columns for Requirement 4)

These columns are appended to the same `dim_people` row already modified in
Requirement 3. No existing column changes meaning.

| Column | Databricks type | Change | Description |
|---|---|---|---|
| `activity_status` | `STRING` | Add | Active, AtRisk, Dormant, Disabled, Unresolved — current-state copy of `fact_person_activity_daily.activity_status` for the latest snapshot date |
| `last_sign_in_at` | `TIMESTAMP` | Add | Current-state copy, latest across every bridged account |
| `last_asset_activity_at` | `TIMESTAMP` | Add | Current-state copy from `fact_person_asset_activity_daily` |
| `dormancy_score` | `DECIMAL(5,2)` | Add | Current-state copy of the latest daily dormancy score |
| `activity_status_calculated_at` | `TIMESTAMP` | Add | When the current-state columns were last refreshed |

These five columns exist purely so a Power BI report can filter and slice
`dim_people` directly without joining to the daily snapshot fact when only the
*current* status is needed. The daily snapshot fact remains the source of truth for
trend and history; these dimension columns are a denormalized, Type 1 convenience
copy of "today's row" from that fact, refreshed on every Gold load.

#### `fact_person_signin_daily` (`New`)

| Column | Databricks type | Null | Source or rule |
|---|---|---:|---|
| `activity_date_key` | `INT` | No | FK to `dim_date`, derived from `sign_in_logs.created_at` |
| `activity_month` | `DATE` | No | Partition column |
| `person_id` | `STRING` | No | FK to `dim_people`, resolved via `bridge_person_account` |
| `user_silver_id` | `STRING` | No | FK to `dim_user`, from `sign_in_logs.user_silver_id` |
| `source_key` | `STRING` | No | FK to `dim_tenant`, the tenant signed into |
| `application_key` | `STRING` | No | FK to `dim_application` |
| `sign_in_count` | `BIGINT` | No | Count of deduplicated `sign_in_id` values that day |
| `interactive_sign_in_count` | `BIGINT` | No | Count where `is_interactive = true` |
| `risky_sign_in_count` | `BIGINT` | No | Count where `risk_level_during_sign_in <> 'none'` |
| `first_sign_in_at` | `TIMESTAMP` | No | Minimum `created_at` that day |
| `last_sign_in_at` | `TIMESTAMP` | No | Maximum `created_at` that day |
| `gold_loaded_at` | `TIMESTAMP` | No | Gold load time |

Logical primary key: `(activity_date_key, person_id, user_silver_id, source_key, application_key)`

#### `fact_person_device_activity_daily` (`New`)

This is a periodic snapshot fact, not a transaction fact: `devices` and
`intune_devices` are SCD Type 1 state tables, not event logs, so there is one row per
device per day the device was seen as active, not one row per discrete usage event.

| Column | Databricks type | Null | Source or rule |
|---|---|---:|---|
| `snapshot_date_key` | `INT` | No | FK to `dim_date`, the Gold load date |
| `snapshot_month` | `DATE` | No | Partition column |
| `person_id` | `STRING` | Yes | FK to `dim_people`, null when `dim_device.owner_resolution_method = 'Unresolved'` |
| `device_key` | `STRING` | No | FK to `dim_device` |
| `source_key` | `STRING` | No | FK to `dim_tenant` |
| `is_compliant` | `BOOLEAN` | Yes | From `dim_device.is_compliant` as of this snapshot |
| `days_since_last_sync` | `INT` | Yes | `snapshot_date - intune_devices.last_sync_at` |
| `is_active_30d` | `BOOLEAN` | No | `days_since_last_sync <= 30` |
| `last_sync_at` | `TIMESTAMP` | Yes | From `intune_devices.last_sync_at` |
| `last_entra_sign_in_at` | `TIMESTAMP` | Yes | From `devices.approximate_last_sign_in_at` |
| `gold_loaded_at` | `TIMESTAMP` | No | Gold load time |

Logical primary key: `(snapshot_date_key, device_key)`

#### `fact_person_activity_daily` (`New`)

The consolidated, coordinator-facing periodic snapshot: one row per person per day,
rolling up both facts above and the existing `fact_person_asset_activity_daily` from
Requirement 1.

| Column | Databricks type | Null | Source or rule |
|---|---|---:|---|
| `snapshot_date_key` | `INT` | No | FK to `dim_date` |
| `snapshot_month` | `DATE` | No | Partition column |
| `person_id` | `STRING` | No | FK to `dim_people` |
| `account_enabled_any` | `BOOLEAN` | No | True if any bridged `dim_user.account_enabled = true` |
| `last_sign_in_at` | `TIMESTAMP` | Yes | Max across all bridged accounts, from `fact_person_signin_daily` |
| `last_sign_in_source_key` | `STRING` | Yes | Which `dim_tenant` produced the most recent sign-in |
| `last_asset_activity_at` | `TIMESTAMP` | Yes | Max `last_activity_at` from `fact_person_asset_activity_daily` |
| `last_device_activity_at` | `TIMESTAMP` | Yes | Max `last_sync_at`/`last_entra_sign_in_at` from `fact_person_device_activity_daily` |
| `distinct_tenants_signed_in_30d` | `SMALLINT` | No | Distinct `source_key` in `fact_person_signin_daily` over trailing 30 days |
| `distinct_applications_used_30d` | `SMALLINT` | No | Distinct `application_key` over trailing 30 days |
| `distinct_devices_used_30d` | `SMALLINT` | No | Distinct `device_key` over trailing 30 days |
| `days_since_last_signin` | `INT` | Yes | `snapshot_date - last_sign_in_at`, null when never observed |
| `days_since_last_asset_activity` | `INT` | Yes | `snapshot_date - last_asset_activity_at` |
| `dormancy_score` | `DECIMAL(5,2)` | No | See formula below, 0 (fully active) to 100 (fully dormant) |
| `activity_status` | `STRING` | No | Active, AtRisk, Dormant, Disabled, Unresolved |
| `gold_loaded_at` | `TIMESTAMP` | No | Gold load time |

Logical primary key: `(snapshot_date_key, person_id)`

Dormancy score and activity status formula:

```text
IF account_enabled_any = false THEN
    dormancy_score = 100
    activity_status = 'Disabled'
ELSE IF days_since_last_signin IS NULL AND days_since_last_asset_activity IS NULL THEN
    dormancy_score = 100
    activity_status = 'Unresolved'
ELSE
    signin_component = LEAST(COALESCE(days_since_last_signin, 90), 90) / 90.0 * 70
    asset_component  = LEAST(COALESCE(days_since_last_asset_activity, 90), 90) / 90.0 * 30
    dormancy_score   = signin_component + asset_component
    activity_status  = CASE
        WHEN dormancy_score <= 15 THEN 'Active'
        WHEN dormancy_score <= 50 THEN 'AtRisk'
        ELSE 'Dormant'
    END
```

The sign-in signal is weighted higher (70) than the asset-activity signal (30) because
a person can be fully engaged while only signing in and reading email, but a person who
neither signs in nor touches a tracked asset for 90 days is dormant regardless of which
signal is missing.

### Requirement 4 dummy data from current schemas

#### Existing Silver `sign_in_logs` rows

| sign_in_id | user_silver_id | user_principal_name | app_id | app_display_name | is_interactive | risk_level_during_sign_in | created_at | source_key | environment |
|---|---|---|---|---|---|---|---|---|---|
| `signin-alice-001` | `madev1_user-guid-alice` | `alice@xtlab1.onmicrosoft.com` | `00000003-0000-0ff1-ce00-000000000000` | SharePoint Online | true | none | 2026-09-22 07:55:00 | `madev1` | source |
| `signin-alice-002` | `madev1_user-guid-alice` | `alice@xtlab1.onmicrosoft.com` | `1fec8e78-bce4-4aaf-ab1b-5451cc387264` | Microsoft Teams | true | none | 2026-09-22 08:02:00 | `madev1` | source |
| `signin-bob-001` | `madev1_user-guid-bob-guest` | `bob_ext#EXT#@xtlab1.onmicrosoft.com` | `00000003-0000-0ff1-ce00-000000000000` | SharePoint Online | true | none | 2026-09-21 14:30:00 | `madev1` | source |
| `signin-carol-001` | `madev1_user-guid-carol` | `carol@xtlab1.onmicrosoft.com` | `00000002-0000-0ff1-ce00-000000000000` | Exchange Online | true | none | 2026-06-19 09:10:00 | `madev1` | source |

Carol's only row in this dummy set is 95 days before the 2026-09-22 snapshot date,
which is the whole point of her example: `sign_in_logs` contains no recent row for her
at all.

#### Existing Silver `devices` and `intune_devices` rows

| Table | device id | azure_ad_device_id | user_principal_name (Intune only) | last_sync_at / approximate_last_sign_in_at | is_compliant |
|---|---|---|---|---|---|
| `intune_devices` | `madev1_intune-dev-alice` | `aad-dev-alice-001` | `alice@xtlab1.onmicrosoft.com` | 2026-09-21 18:00:00 | true |
| `devices` | `madev1_aad-dev-alice-001` | `aad-dev-alice-001` | not present on this table | 2026-09-22 07:55:00 | n/a (Entra has no compliance column) |

Bob and Carol have no `intune_devices` row in this dummy set: Bob because guests are
not Intune-enrolled, and Carol because her corporate laptop has not synced within any
retained window (illustrating a device that ages out rather than one that never
existed).

#### New `dim_application` rows

| application_key | app_id | app_display_name | app_category | is_first_party_microsoft |
|---|---|---|---|---|
| `app_spo_hash` | `00000003-0000-0ff1-ce00-000000000000` | SharePoint Online | FileStorage | true |
| `app_teams_hash` | `1fec8e78-bce4-4aaf-ab1b-5451cc387264` | Microsoft Teams | Collaboration | true |
| `app_exo_hash` | `00000002-0000-0ff1-ce00-000000000000` | Exchange Online | Email | true |

#### New `dim_device` rows

| device_key | azure_ad_device_id | source_key | display_name | is_intune_managed | is_compliant | owner_user_silver_id | owner_resolution_method |
|---|---|---|---|---|---|---|---|
| `device_alice_hash` | `aad-dev-alice-001` | `madev1` | ALICE-LAPTOP-01 | true | true | `user_alice_hash` | IntuneUpnMatch |

No `dim_device` row exists for Bob or Carol in this dummy set: Bob has no Intune
enrollment to resolve, and Carol's old device has already fallen outside the pipeline's
retention window used for this example.

#### Modified `dim_people` rows (current-state columns added by this requirement)

| person_id | activity_status | last_sign_in_at | last_asset_activity_at | dormancy_score |
|---|---|---|---|---:|
| `person_alice_hash` | Active | 2026-09-22 08:02:00 | 2026-09-22 11:15:00 | 0.00 |
| `person_bob_hash` | Active | 2026-09-21 14:30:00 | 2026-09-22 11:10:00 | 0.78 |
| `person_carol_hash` | Dormant | 2026-06-19 09:10:00 | null | 100.00 |

#### New `fact_person_signin_daily` rows

| activity_date_key | person_id | user_silver_id | source_key | application_key | sign_in_count | interactive_sign_in_count | last_sign_in_at |
|---:|---|---|---|---|---:|---:|---|
| 20260922 | `person_alice_hash` | `user_alice_hash` | `madev1` | `app_spo_hash` | 1 | 1 | 2026-09-22 07:55:00 |
| 20260922 | `person_alice_hash` | `user_alice_hash` | `madev1` | `app_teams_hash` | 1 | 1 | 2026-09-22 08:02:00 |
| 20260921 | `person_bob_hash` | `user_bob_hash` | `madev1` | `app_spo_hash` | 1 | 1 | 2026-09-21 14:30:00 |
| 20260619 | `person_carol_hash` | `user_carol_hash` | `madev1` | `app_exo_hash` | 1 | 1 | 2026-06-19 09:10:00 |

#### New `fact_person_device_activity_daily` rows

| snapshot_date_key | person_id | device_key | is_compliant | days_since_last_sync | is_active_30d |
|---:|---|---|---|---:|---|
| 20260922 | `person_alice_hash` | `device_alice_hash` | true | 1 | true |

#### New `fact_person_activity_daily` rows (2026-09-22 snapshot)

| snapshot_date_key | person_id | account_enabled_any | days_since_last_signin | days_since_last_asset_activity | distinct_tenants_signed_in_30d | distinct_applications_used_30d | distinct_devices_used_30d | dormancy_score | activity_status |
|---:|---|---|---:|---:|---:|---:|---:|---:|---|
| 20260922 | `person_alice_hash` | true | 0 | 0 | 1 | 2 | 1 | 0.00 | Active |
| 20260922 | `person_bob_hash` | true | 1 | 0 | 1 | 1 | 0 | 0.78 | Active |
| 20260922 | `person_carol_hash` | true | 95 | null | 0 | 0 | 0 | 100.00 | Dormant |

Interpretation:

* Alice is unambiguously Active on both dimensions the coordinator cares about: recent
  sign-in and recent asset activity, on a compliant, Intune-managed device.
* Bob is Active by sign-in even though his cross-tenant status is still `Unknown` from
  Requirement 3 — activity status and cross-tenant resolution are independent findings.
  His device usage is `Unresolved`, not `false`, because a guest with no Intune
  enrollment is expected, not broken. His `dormancy_score` of 0.78 is
  `LEAST(1, 90) / 90 * 70 = 0.78` for the sign-in component plus `0.00` for the asset
  component, because his most recent asset activity was on the snapshot date itself.
* Carol has an enabled account with no sign-in and no asset activity for 95 days. Her
  `dormancy_score` of 100.00 is the sum of two capped components: the sign-in
  component is `LEAST(95, 90) / 90 * 70 = 70.00`, and the asset component is
  `LEAST(COALESCE(NULL, 90), 90) / 90 * 30 = 30.00` because she has no asset activity
  row at all, so the 90-day cap applies by default. `70.00 + 30.00 = 100.00`, the
  maximum possible score, which routes her to `Dormant` under the `> 50` rule.
  A migration coordinator reviewing the Dormant list would see Carol, confirm with her
  manager whether she is on leave, and decide whether to hold her out of the next
  migration wave rather than migrating an account nobody will notice failing.

## From audit rows to Gold fact rows, step by step

Every dummy row in this document already appears somewhere above. This section only
re-reads those rows in table order so the join path from Silver audit to Gold fact is
explicit, using one SharePoint event and one Teams event as worked examples.

### Trace 1: SharePoint `FileDownloaded` by Alice

**Step 1. Start from the raw Silver row.** This is `evt-002` from `audit_sharepoint`,
unchanged from the Requirement 2 dummy data:

| audit_id | user_id | operation | workload | object_id | created_at | source_key |
|---|---|---|---|---|---|---|
| `evt-002` | `alice@xtlab1.onmicrosoft.com` | FileDownloaded | SharePoint | `https://xtlab1.sharepoint.com/sites/finance/Shared Documents/Budget.xlsx` | 2026-09-22 10:05:00 | `madev1` |

**Step 2. Resolve the operation.** Look up `dim_operation` on the exact
`(workload, raw_operation)` pair:

```text
SELECT operation_key, operation_class_key
FROM dim_operation
WHERE workload = 'SharePoint' AND raw_operation = 'FileDownloaded'
-- returns: op_download_hash, 1 (Read)
```

**Step 3. Resolve the asset.** `object_id` is a full document URL. The Gold load
matches it against `dim_asset.asset_locator` as a URL prefix, because a site's asset
locator is the site URL and audit events fire on documents inside that site:

```text
SELECT asset_key
FROM dim_asset
WHERE workload = 'SharePoint'
  AND 'https://xtlab1.sharepoint.com/sites/finance/Shared Documents/Budget.xlsx'
      LIKE asset_locator || '%'
-- asset_locator = 'https://xtlab1.sharepoint.com/sites/finance'
-- returns: asset_finance_hash
```

**Step 4. Resolve the person.** `user_id` is the actor's UPN. First match the tenant
account, then the person who owns that account:

```text
SELECT user_silver_id FROM dim_user
WHERE user_principal_name = 'alice@xtlab1.onmicrosoft.com'
-- returns: user_alice_hash

SELECT person_id FROM bridge_person_account
WHERE user_silver_id = 'user_alice_hash' AND is_current = true
-- returns: person_alice_hash
```

**Step 5. Write the person-grain fact row.** Steps 2 through 4 supply every key needed
to insert or merge one row into `fact_person_asset_activity_daily`:

| activity_date_key | source_key | person_id | user_silver_id | asset_key | operation_key | operation_class_key | activity_count | last_activity_at |
|---:|---|---|---|---|---|---:|---:|---|
| 20260922 | `madev1` | `person_alice_hash` | `user_alice_hash` | `asset_finance_hash` | `op_download_hash` | 1 | 1 | 2026-09-22 10:05:00 |

**Step 6. Roll up to the asset-grain fact row.** A downstream aggregation groups every
`fact_person_asset_activity_daily` row that shares
`(activity_date_key, source_key, asset_key, operation_class_key, evidence_source_key)`.
For Finance Hub Read on 22 September that group also includes `evt-001` (Alice) and
`evt-004` (Bob), so `fact_asset_usage_daily` receives:

| activity_date_key | source_key | asset_key | operation_class_key | activity_volume | daily_distinct_people | last_activity_at |
|---:|---|---|---:|---:|---:|---|
| 20260922 | `madev1` | `asset_finance_hash` | 1 | 3 | 2 | 2026-09-22 10:15:00 |

This is the same Read row already shown in the Requirement 1 dummy result table.

### Trace 2: Microsoft Teams `MessageSent` by Alice

**Step 1. Start from the raw Silver row.** This is `evt-102` from `audit_general`,
unchanged from the Requirement 2 dummy data:

| audit_id | user_id | operation | workload | object_id | created_at | source_key |
|---|---|---|---|---|---|---|
| `evt-102` | `alice@xtlab1.onmicrosoft.com` | MessageSent | MicrosoftTeams | `grp-sales-001` | 2026-09-22 11:05:00 | `madev1` |

**Step 2. Resolve the operation.**

```text
SELECT operation_key, operation_class_key
FROM dim_operation
WHERE workload = 'MicrosoftTeams' AND raw_operation = 'MessageSent'
-- returns: op_teams_messagesent_hash, 2 (Write)
```

**Step 3. Resolve the asset.** Unlike SharePoint, `object_id` for this operation is the
Entra group ID rather than a URL, so the match is against `dim_asset.native_asset_id`,
not `asset_locator`:

```text
SELECT asset_key
FROM dim_asset
WHERE workload = 'MicrosoftTeams' AND native_asset_id = 'grp-sales-001'
-- returns: asset_sales_team_hash
```

This is a real limitation, not just a modeling convenience: the current Silver audit
envelope keeps only one generic `object_id` per event, and its meaning changes by
workload and even by operation. Membership operations like `MemberAdded` and
`MessagesListed` reliably carry the team's group ID, but some Teams operations do not
carry any asset-identifying value in `object_id` today, so they cannot be resolved to
`dim_asset` until a workload-specific field is retained in Silver.

**Step 4. Resolve the person.** Same two-step join as Trace 1:

```text
SELECT user_silver_id FROM dim_user
WHERE user_principal_name = 'alice@xtlab1.onmicrosoft.com'
-- returns: user_alice_hash

SELECT person_id FROM bridge_person_account
WHERE user_silver_id = 'user_alice_hash' AND is_current = true
-- returns: person_alice_hash
```

**Step 5. Write the person-grain fact row.**

| activity_date_key | source_key | person_id | user_silver_id | asset_key | operation_key | operation_class_key | activity_count | last_activity_at |
|---:|---|---|---|---|---|---:|---:|---|
| 20260922 | `madev1` | `person_alice_hash` | `user_alice_hash` | `asset_sales_team_hash` | `op_teams_messagesent_hash` | 2 | 1 | 2026-09-22 11:05:00 |

**Step 6. Roll up to the asset-grain fact row.** Alice's `MessageSent` event is the only
Write event on Sales Team that day, so the group has exactly one member and
`fact_asset_usage_daily` receives:

| activity_date_key | source_key | asset_key | operation_class_key | activity_volume | daily_distinct_people | last_activity_at |
|---:|---|---|---:|---:|---:|---|
| 20260922 | `madev1` | `asset_sales_team_hash` | 2 | 1 | 1 | 2026-09-22 11:05:00 |

This is the same Write row already shown in the Requirement 1 (Teams) dummy result
table. The `MessagesListed` (Read) and `MemberAdded` (Admin) events for Sales Team
follow the identical six-step path and produce the remaining two rows shown earlier in
Requirement 2.

### Trace 3: Alice's sign-in and device signals roll up to Person Activity Profile

Unlike Traces 1 and 2, this trace does not start from an audit event at all. It starts
from two different Silver system-of-record tables that have no asset dimension.

**Step 1. Start from the raw Silver rows.** These are two rows already shown in the
Requirement 4 dummy data:

| Table | Row |
|---|---|
| `sign_in_logs` | `signin-alice-002`: `user_silver_id = madev1_user-guid-alice`, `app_id = 1fec8e78-bce4-4aaf-ab1b-5451cc387264`, `created_at = 2026-09-22 08:02:00` |
| `intune_devices` | `madev1_intune-dev-alice`: `user_principal_name = alice@xtlab1.onmicrosoft.com`, `azure_ad_device_id = aad-dev-alice-001`, `last_sync_at = 2026-09-21 18:00:00` |

**Step 2. Resolve the application.** `sign_in_logs.app_id` is looked up directly, with
no workload-specific mapping needed because the Entra client ID is already the
conformed key:

```text
SELECT application_key FROM dim_application
WHERE app_id = '1fec8e78-bce4-4aaf-ab1b-5451cc387264'
-- returns: app_teams_hash
```

**Step 3. Resolve the person from the sign-in row.** `sign_in_logs.user_silver_id` is
the same compound Silver key `dim_user.user_silver_id` already exposes, so no
UPN-based matching is required here, unlike Traces 1 and 2:

```text
SELECT user_silver_id FROM dim_user WHERE user_silver_id = 'madev1_user-guid-alice'
-- returns: user_alice_hash (dim_user maps the Silver key to its own surrogate key)

SELECT person_id FROM bridge_person_account
WHERE user_silver_id = 'user_alice_hash' AND is_current = true
-- returns: person_alice_hash
```

**Step 4. Resolve the device owner.** This is the join the current `devices` table
cannot do on its own, and the reason `intune_devices.user_principal_name` is the
authoritative owner-resolution field:

```text
SELECT owner_user_silver_id, device_key FROM dim_device
WHERE azure_ad_device_id = 'aad-dev-alice-001'
-- populated because intune_devices.user_principal_name = 'alice@xtlab1.onmicrosoft.com'
--   matched dim_user.user_principal_name on the Gold load that built dim_device
-- returns: user_alice_hash, device_alice_hash
```

**Step 5. Write the two transaction/snapshot fact rows.**

| Fact | Row |
|---|---|
| `fact_person_signin_daily` | `20260922`, `person_alice_hash`, `user_alice_hash`, `madev1`, `app_teams_hash`, `sign_in_count = 1` |
| `fact_person_device_activity_daily` | `20260922`, `person_alice_hash`, `device_alice_hash`, `is_compliant = true`, `days_since_last_sync = 1` |

**Step 6. Roll up to the person-grain snapshot.** The nightly Gold load reads the
current day's rows from `fact_person_signin_daily`, `fact_person_device_activity_daily`,
and the already-built `fact_person_asset_activity_daily` (Trace 1 and Trace 2 both feed
this table), groups by `person_id`, and applies the dormancy-score formula from
Requirement 4's physical model:

```text
last_sign_in_at            = 2026-09-22 08:02:00   -- from Step 5
last_asset_activity_at     = 2026-09-22 11:15:00   -- from fact_person_asset_activity_daily
days_since_last_signin     = 0
days_since_last_asset_activity = 0
dormancy_score              = 0.00
activity_status              = 'Active'
```

This is the same `person_alice_hash` row already shown in the Requirement 4 dummy
result table, and it demonstrates why Requirement 4 needs its own fact tables rather
than reusing `fact_person_asset_activity_daily`: sign-in and device telemetry have no
`asset_key` to populate, so forcing them into the asset-grain fact would require a
synthetic placeholder asset that does not correspond to anything a coordinator can
report on.

### Why two facts instead of one

`fact_person_asset_activity_daily` must exist before `fact_asset_usage_daily` can be
built, because exact distinct-person counts (Active Users 7/30/90-day, daily distinct
people) cannot be derived from a fact that has already discarded `person_id`. The
six-step path above always writes the person-grain row first (Step 5) and only then
aggregates it (Step 6). This is why `fact_asset_usage_daily` is described as a rollup
rather than a second independent ingestion path.

## Why dim_evidence_source and dim_operation_class exist across workloads

Both dimensions exist because a specific line item in the DA05 business requirements
asks for exactly what they encode. Neither is internal ETL plumbing that could be
dropped without losing capability.

### dim_evidence_source: trust and lineage, not just a label

Requirement 7 (Metadata Usage Fallback) explicitly asks the model to flag every usage
record's source as **Audit / Metadata Fallback / Graph Usage**. Audit history is not
always complete — retention windows expire, a workload's audit logging may not have
been enabled from day one, or coverage is thinner for some workloads than others.
Without a trust flag, a blank `last_read_at` is ambiguous: it could mean "nobody used
this asset" or "somebody used it, but the audit trail already aged out." That
ambiguity is dangerous for a migration decision — an asset should not be marked safe
to retire just because its audit trail expired.

| Source | What it actually is | Names the person? | Real activity counts? | Classifies Read/Write? |
|---|---|---|---|---|
| Audit | Unified audit log event (`audit_sharepoint`, `audit_exchange`, `audit_general`) | Yes | Yes | Yes |
| Graph Usage | M365 Graph usage-report APIs (aggregated, often privacy-shielded) | No | Yes (counts only) | No |
| Metadata Fallback | A last-modified / last-logon timestamp on the object itself | No | No | No |

This is what powers `coverage_pct` on `fact_asset_usage_daily` — what fraction of an
asset's usage is backed by real audit evidence versus a guess — which feeds the
Migration Readiness Score. It also drives ETL source priority: prefer Audit, fall back
to Graph Usage, fall back to Metadata Fallback only when nothing else exists
(`source_priority` 1/2/3 already set in the Requirement 1 dummy data).

Feasibility today, per workload:

- **SharePoint** already has a real Metadata Fallback candidate: `spo_sites.source_modified_at`.
- **Power BI / Power Apps / Power Automate** already have one too: `last_updated_at` /
  `last_modified_at` on `powerbi_apps`, `powerplat_apps`, `powerplat_flows`.
- **Exchange/Mailbox** does **not** yet — Silver `mailboxes` has no last-logon field
  today (only a sync-time `last_updated_at`, which is not the same as a user's actual
  last sign-in). This is a genuine current data gap, not a modeling simplification —
  see [Section 9](./usage-telemetry-da05-logical-model.md) assumptions.

### dim_operation_class: the governed 5-category reporting taxonomy

Requirement 2 literally asks for this taxonomy: Read, Write, Share, Admin (plus
Unclassified as a safety valve), not the hundreds of raw operation strings each
workload emits. It is kept as its own tiny dimension, separate from the much larger
`dim_operation`, for two reasons:

1. **Stability for reporting.** Power BI slicers, the Read/Write Ratio, and the
   active-user trailing windows all group by these 5 rows. If the classification were
   computed ad hoc per report, a taxonomy correction would require re-touching every
   report instead of one dimension.
2. **Governance safety valve.** New or unrecognized raw operations default to
   `Unclassified` (key `0`) via `dim_operation`'s mapping, instead of blocking
   ingestion or breaking the fact grain. A steward can approve a real classification
   later without touching any fact table. `dim_operation` is the workload-aware bridge
   that performs the actual `(workload, raw_operation) -> operation_class_key`
   mapping; `dim_operation_class` itself almost never changes.

### The same 5 rows, reused across every workload

`dim_operation_class` is a genuinely conformed dimension (Kimball sense) — what
changes per workload is only which `dim_operation` rows point at it. The mappings
below are illustrative/proposed (`mapping_status = Proposed`), following the same
convention already used for the SharePoint and Teams rows earlier in this document —
each would need steward sign-off before being trusted in production:

| Workload | Raw operation examples | Maps to class | Notes |
|---|---|---|---|
| SharePoint (`audit_sharepoint`) | `FileAccessed`, `FileDownloaded` | Read | Content consumption |
| | `FileModified`, `FileUploaded` | Write | Content change |
| | `SharingSet`, `AnonymousLinkCreated` | Share | Grants access |
| | `PermissionLevelModified`, `SiteCollectionAdminAdded` | Admin | Site/permission configuration |
| Exchange (`audit_exchange`) | `MailItemsAccessed` | Read | Mailbox item access |
| | `Send`, `MoveToDeletedItems` | Write | Message creation/state change |
| | `Add-MailboxPermission` | Share | Grants mailbox access to another account |
| | `Set-Mailbox`, `New-InboxRule` | Admin | Mailbox configuration |
| General &rarr; Teams (`audit_general`, workload `MicrosoftTeams`) | `MessagesListed`, `ChatRetrieved` | Read | Used in the Requirement 1/2 dummy data above |
| | `MessageSent` | Write | Used in the Requirement 1/2 dummy data above |
| | `MemberAdded`, `TeamSettingChanged` | Admin | Membership/config change |
| General &rarr; Power BI (`audit_general`, workload `PowerBI`) | `ViewReport`, `ViewDashboard` | Read | Report/dashboard consumption |
| | `CreateReport`, `UpdateReport` | Write | Content authored/changed |
| | `ShareReport` | Share | Grants access to a report |
| | `UpdateWorkspaceUsers` | Admin | Workspace-level access change |
| General &rarr; Power Apps (`audit_general`, workload `PowerApps`) | `AppOpened` / `LaunchApp` | Read | App usage |
| | `SaveAppVersion`, `UpdateApp` | Write | App authored/changed |
| | `ShareApp` | Share | Grants access to an app |
| General &rarr; Power Automate (`audit_general`, workload `MicrosoftFlow`) | `FlowRunStarted` | Write | A run executes an automation, treated as a content-changing action, not passive consumption |
| | `CreateFlow`, `UpdateFlow` | Write | Flow authored/changed |
| | `ShareFlow` | Share | Grants access to a flow |

SharePoint's `FileDownloaded` and Teams' `MessagesListed` are two different
`dim_operation` rows (different `workload`, different `raw_operation`), but both
resolve to the identical `operation_class_key = 1` (Read). That identical foreign key
across unrelated workloads is exactly what makes `dim_operation_class` a conformed
dimension: `fact_asset_usage_daily` can compute one Read/Write Ratio, one Usage Trend,
and one set of active-user windows across SharePoint, Exchange, Teams, Power BI,
Power Apps, and Power Automate without any workload-specific branching logic.

## Why dim_user and dim_people are both needed

They sit at two different grains on purpose:

| | `dim_user` | `dim_people` |
|---|---|---|
| Grain | One row per tenant account (one row per Entra object, per tenant, per environment) | One row per real human being |
| Primary key | `user_silver_id` | `person_id` |
| Can one human have more than one row? | Yes, a source account, a target account, and a guest account are three different `dim_user` rows | No, exactly one row per person regardless of how many accounts they hold |
| Answers | Which specific Entra identity generated this event | Which unique individual should this activity count toward |
| Used for | Login and account attributes: UPN, mail, `user_type`, `account_enabled`, home tenant | Cross-tenant person resolution, migration status, deduplicated Active Users counts |

If a fact only carried `user_silver_id`, a migrated user who is active both before and
after cutover would be counted as two different users. `person_id` is what collapses
those account rows back into one human for counting.

### Worked example: Alice before and after migration

Alice starts with one source-tenant account:

```text
dim_user:   user_alice_hash        -> alice@xtlab1.onmicrosoft.com   (source, Member)
dim_people: person_alice_hash      -> Alice Adams, category = Internal
bridge_person_account: person_alice_hash <-> user_alice_hash  (role = SourceMember)
```

After her mailbox migrates, she gains a second Entra account in the target tenant.
`bridge_person_account` links both accounts to the same person:

```text
dim_user:   user_alice_target_hash -> alice@contoso.onmicrosoft.com  (target, Member)
bridge_person_account: person_alice_hash <-> user_alice_target_hash (role = TargetMember)
```

`fact_person_asset_activity_daily` now has two rows for Alice, one per account:

| activity_date_key | person_id | user_silver_id | asset_key | activity |
|---:|---|---|---|---|
| 20260922 | `person_alice_hash` | `user_alice_hash` | `asset_finance_hash` | Read (source) |
| 20261005 | `person_alice_hash` | `user_alice_target_hash` | `asset_finance_hash_target` | Read (target) |

Counting distinct `user_silver_id` returns 2, which looks like two different people
touched the asset. Counting distinct `person_id` returns 1, correctly recognizing the
same human active on both sides of the migration. This is why Active Users 7/30/90-day
and cross-tenant usage percent are computed off `person_id`, while `user_silver_id` is
still kept on the fact so a query can tell whether a given activity happened in the
source tenant or the target tenant for post-migration usage verification.

Bob (the guest example used throughout this document) is currently a simple 1:1 case:
one `dim_user` row, one `dim_people` row, linked by a single `bridge_person_account`
row with `identity_role = Guest`. The bridge table is what makes the model ready for
the day a home-tenant source resolves Bob to a second, home-tenant account without any
change to the fact grain.

## Consolidated relationships

```text
Existing DimDate ------------------------------+
Existing DimTenant ----------------------------+
Modified DimPeople ----------------------------+---------------------------+
Modified DimUser ---- New BridgePersonAccount  |                           |
New DimAsset ----------------------------------+                           |
New DimOperation ------------------------------+                           |
New DimOperationClass -------------------------+                           |
New DimEvidenceSource -------------------------+                           |
                                                 \                          |
                                     New FactPersonAssetActivityDaily      |
                                                 |                          |
                                                 v                          |
                                      New FactAssetUsageDaily              |
                                                 ^                          |
                                                 |                          |
New DimApplication -----------+                 |                          |
New DimDevice -----------------+---> New FactPersonDeviceActivityDaily     |
Existing DimUser (sign-in) ----+---> New FactPersonSigninDaily             |
                                                 \                          |
                                                  +------------------------>+
                                                                 |
                                                                 v
                                              New FactPersonActivityDaily
                                                                 |
                                                                 v
                                          Modified DimPeople (current-state refresh)
```

`dim_asset.source_dimension_key` points to the applicable existing workload dimension.
For the dummy SharePoint site, it points to `dim_spo_site.site_silver_id`. For the
dummy Team, it points to `shared_data_sets.shared_data_set_id`.

`fact_person_activity_daily` is a rollup fact: it does not read from Silver directly.
It reads the current day's rows from `fact_person_signin_daily`,
`fact_person_device_activity_daily`, and `fact_person_asset_activity_daily`
(the same fact that already feeds `fact_asset_usage_daily` for Requirement 1), groups
by `person_id`, and computes `dormancy_score` and `activity_status`. The result both
lands in the fact table and refreshes the five additive columns on `dim_people`, so
Power BI can filter on current activity status without joining to a fact table.

## Requirement-to-table summary

| Requirement | Existing, reused | Existing, modified | New |
|---|---|---|---|
| 1. Asset Usage Facts | `dim_date`, `dim_tenant`, `dim_spo_site`, `shared_data_sets` (SharePoint and Teams), Silver workload metadata (`spo_sites`, `teams`, `teams_team_details`) | None required for the SharePoint and Teams slices | `dim_asset`, `dim_evidence_source`, both daily facts |
| 2. Read versus Write | Silver `audit_sharepoint`, `audit_general` operation and workload columns | None | `dim_operation`, `dim_operation_class`; class keys on both facts |
| 3. Usage by Person | Silver `users`, `user_mapping_candidates`; Gold `user_map` | `dim_tenant`, `dim_user`, `dim_people` | `bridge_person_account`; person and account keys on the person fact |
| 4. Person Activity Profile | Silver `sign_in_logs`, `devices`, `intune_devices`, `users`; `dim_user`, `bridge_person_account`, `fact_person_asset_activity_daily` (Requirement 1) | `dim_people` (second, additive modification: `activity_status`, `last_sign_in_at`, `last_asset_activity_at`, `dormancy_score`, `activity_status_calculated_at`) | `dim_application`, `dim_device`, `fact_person_signin_daily`, `fact_person_device_activity_daily`, `fact_person_activity_daily` |

## Implementation sequence

1. Implement SharePoint as the first vertical slice because the current `object_id` URL
   can be matched to existing `dim_spo_site.web_url`.
2. Create and seed `dim_operation_class`, `dim_operation`, and
   `dim_evidence_source`.
3. Create `dim_asset` and populate SharePoint members from existing `dim_spo_site`, and
   Team members from existing `shared_data_sets` (type `team`).
4. Extend `dim_people` population and add `bridge_person_account`.
5. Build the person-level daily fact from deduplicated Silver audit rows.
6. Build the asset daily fact and exact trailing active-person windows.
7. Implement Teams as the second vertical slice, matching `native_asset_id` against the
   Entra group ID in `object_id` for membership and messaging operations. Add Exchange,
   Power BI, Power Apps, and Power Automate afterward, each of which needs its own
   workload-specific audit asset identifier retained in Silver before it can be matched
   as reliably as SharePoint's URL or Teams' group ID.
8. Add authoritative guest home-tenant ingestion before publishing cross-tenant usage
   as a certified metric.
9. Seed `dim_application` from the first-party Entra app catalog (Teams, Exchange
   Online, Power BI, SharePoint Online client IDs are stable and can be hard-seeded;
   Power Apps and Power Automate client IDs should be added as they are observed in
   `sign_in_logs.app_id`).
10. Build `dim_device` from `intune_devices` joined to `dim_user` on
    `user_principal_name`, then enriched with `devices` on `azure_ad_device_id` for
    Entra-only attributes. Rows with no UPN match load with `owner_resolution_method =
    'Unresolved'` rather than being dropped, so device counts stay reconcilable to
    `intune_devices` row counts.
11. Build `fact_person_signin_daily` directly from `sign_in_logs`, joining
    `user_silver_id` to `dim_user` and `app_id` to `dim_application`.
12. Build `fact_person_device_activity_daily` as a daily snapshot over `dim_device`,
    carrying `days_since_last_sync` forward from `intune_devices.last_sync_at`.
13. Build `fact_person_activity_daily` last, once Requirement 1's
    `fact_person_asset_activity_daily` is already populated for the day, and use it to
    refresh the five current-state columns added to `dim_people` in step 4.
