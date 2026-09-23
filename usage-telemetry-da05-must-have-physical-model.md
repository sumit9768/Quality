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

### Why two facts instead of one

`fact_person_asset_activity_daily` must exist before `fact_asset_usage_daily` can be
built, because exact distinct-person counts (Active Users 7/30/90-day, daily distinct
people) cannot be derived from a fact that has already discarded `person_id`. The
six-step path above always writes the person-grain row first (Step 5) and only then
aggregates it (Step 6). This is why `fact_asset_usage_daily` is described as a rollup
rather than a second independent ingestion path.

## Consolidated relationships

```text
Existing DimDate ------------------------------+
Existing DimTenant ----------------------------+
Modified DimPeople ----------------------------+
Modified DimUser ---- New BridgePersonAccount  |
New DimAsset ----------------------------------+
New DimOperation ------------------------------+
New DimOperationClass -------------------------+
New DimEvidenceSource -------------------------+
                                                 \
                                     New FactPersonAssetActivityDaily
                                                 |
                                                 v
                                      New FactAssetUsageDaily
```

`dim_asset.source_dimension_key` points to the applicable existing workload dimension.
For the dummy SharePoint site, it points to `dim_spo_site.site_silver_id`. For the
dummy Team, it points to `shared_data_sets.shared_data_set_id`.

## Requirement-to-table summary

| Requirement | Existing, reused | Existing, modified | New |
|---|---|---|---|
| 1. Asset Usage Facts | `dim_date`, `dim_tenant`, `dim_spo_site`, `shared_data_sets` (SharePoint and Teams), Silver workload metadata (`spo_sites`, `teams`, `teams_team_details`) | None required for the SharePoint and Teams slices | `dim_asset`, `dim_evidence_source`, both daily facts |
| 2. Read versus Write | Silver `audit_sharepoint`, `audit_general` operation and workload columns | None | `dim_operation`, `dim_operation_class`; class keys on both facts |
| 3. Usage by Person | Silver `users`, `user_mapping_candidates`; Gold `user_map` | `dim_tenant`, `dim_user`, `dim_people` | `bridge_person_account`; person and account keys on the person fact |

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
