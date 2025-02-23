# Migration guide

This document is meant to help you migrate your Terraform config to the new newest version. In migration guides, we will only
describe deprecations or breaking changes and help you to change your configuration to keep the same (or similar) behavior
across different versions.

## v0.93.0 ➞ v0.94.0

### *(new feature)* new snowflake_account_role resource

Already existing `snowflake_role` was deprecated in favor of the new `snowflake_account_role`. The old resource got upgraded to
have the same features as the new one. The only difference is the deprecation message on the old resource.

New fields:
- added `show_output` field that holds the response from SHOW ROLES. Remember that the field will be only recomputed if one of the fields (`name` or `comment`) are changed.

### *(breaking change)* refactored snowflake_roles data source

Changes:
- New `in_class` filtering option to filter out roles by class name, e.g. `in_class = "SNOWFLAKE.CORE.BUDGET"`
- `pattern` was renamed to `like`
- output of SHOW is enclosed in `show_output`, so before, e.g. `roles.0.comment` is now `roles.0.show_output.0.comment`

## v0.92.0 ➞ v0.93.0

### general changes

With this change we introduce the first resources redesigned for the V1. We have made a few design choices that will be reflected in these and in the further reworked resources. This includes:
- Handling the [default values](./v1-preparations/CHANGES_BEFORE_V1.md#default-values).
- Handling the ["empty" values](./v1-preparations/CHANGES_BEFORE_V1.md#empty-values).
- Handling the [Snowflake parameters](./v1-preparations/CHANGES_BEFORE_V1.md#snowflake-parameters).
- Saving the [config values in the state](./v1-preparations/CHANGES_BEFORE_V1.md#config-values-in-the-state).
- Providing a ["raw Snowflake output"](./v1-preparations/CHANGES_BEFORE_V1.md#empty-values) for the managed resources.

They are all described in short in the [changes before v1 doc](./v1-preparations/CHANGES_BEFORE_V1.md). Please familiarize yourself with these changes before the upgrade.

### old grant resources removal
Following the [announcement](https://github.com/Snowflake-Labs/terraform-provider-snowflake/discussions/2736) we have removed the old grant resources. The two resources [snowflake_role_ownership_grant](https://registry.terraform.io/providers/Snowflake-Labs/snowflake/latest/docs/resources/role_ownership_grant) and [snowflake_user_ownership_grant](https://registry.terraform.io/providers/Snowflake-Labs/snowflake/latest/docs/resources/user_ownership_grant) were not listed in the announcement, but they were also marked as deprecated ones. We are removing them too to conclude the grants redesign saga.

### *(new feature)* Api authentication resources
Added new api authentication resources, i.e.:
- `snowflake_api_authentication_integration_with_authorization_code_grant`
- `snowflake_api_authentication_integration_with_client_credentials`
- `snowflake_api_authentication_integration_with_jwt_bearer`

See reference [doc](https://docs.snowflake.com/en/sql-reference/sql/create-security-integration-api-auth).

### *(new feature)* snowflake_oauth_integration_for_custom_clients and snowflake_oauth_integration_for_partner_applications resources

To enhance clarity and functionality, the new resources `snowflake_oauth_integration_for_custom_clients` and `snowflake_oauth_integration_for_partner_applications` have been introduced 
to replace the previous `snowflake_oauth_integration`. Recognizing that the old resource carried multiple responsibilities within a single entity, we opted to divide it into two more specialized resources.
The newly introduced resources are aligned with the latest Snowflake documentation at the time of implementation, and adhere to our [new conventions](#general-changes). 
This segregation was based on the `oauth_client` attribute, where `CUSTOM` corresponds to `snowflake_oauth_integration_for_custom_clients`, 
while other attributes align with `snowflake_oauth_integration_for_partner_applications`.

### *(new feature)* snowflake_security_integrations datasource
Added a new datasource enabling querying and filtering all types of security integrations. Notes:
- all results are stored in `security_integrations` field.
- `like` field enables security integrations filtering.
- SHOW SECURITY INTEGRATIONS output is enclosed in `show_output` field inside `security_integrations`.
- Output from **DESC SECURITY INTEGRATION** (which can be turned off by declaring `with_describe = false`, **it's turned on by default**) is enclosed in `describe_output` field inside `security_integrations`.
  **DESC SECURITY INTEGRATION** returns different properties based on the integration type. Consult the documentation to check which ones will be filled for which integration.
  The additional parameters call **DESC SECURITY INTEGRATION** (with `with_describe` turned on) **per security integration** returned by **SHOW SECURITY INTEGRATIONS**.
  It's important to limit the records and calls to Snowflake to the minimum. That's why we recommend assessing which information you need from the data source and then providing strong filters and turning off additional fields for better plan performance.

### snowflake_external_oauth_integration resource changes

#### *(behavior change)* Renamed fields
Renamed fields:
- `type` to `external_oauth_type`
- `issuer` to `external_oauth_issuer`
- `token_user_mapping_claims` to `external_oauth_token_user_mapping_claim`
- `snowflake_user_mapping_attribute` to `external_oauth_snowflake_user_mapping_attribute`
- `scope_mapping_attribute` to `external_oauth_scope_mapping_attribute`
- `jws_keys_urls` to `external_oauth_jws_keys_url`
- `rsa_public_key` to `external_oauth_rsa_public_key`
- `rsa_public_key_2` to `external_oauth_rsa_public_key_2`
- `blocked_roles` to `external_oauth_blocked_roles_list`
- `allowed_roles` to `external_oauth_allowed_roles_list`
- `audience_urls` to `external_oauth_audience_list`
- `any_role_mode` to `external_oauth_any_role_mode`
- `scope_delimiter` to `external_oauth_scope_delimiter`
to align with Snowflake docs. Please rename this field in your configuration files. State will be migrated automatically.

#### *(behavior change)* Force new for multiple attributes after removing from config
Conditional force new was added for the following attributes when they are removed from config. There are no alter statements supporting UNSET on these fields.
- `external_oauth_rsa_public_key`
- `external_oauth_rsa_public_key_2`
- `external_oauth_scope_mapping_attribute`
- `external_oauth_jws_keys_url`
- `external_oauth_token_user_mapping_claim`

#### *(behavior change)* Conflicting fields
Fields listed below can not be set at the same time in Snowflake. They are marked as conflicting fields.
- `external_oauth_jws_keys_url` <-> `external_oauth_rsa_public_key`
- `external_oauth_jws_keys_url` <-> `external_oauth_rsa_public_key_2`
- `external_oauth_allowed_roles_list` <-> `external_oauth_blocked_roles_list`

#### *(behavior change)* Changed diff suppress for some fields
The fields listed below had diff suppress which removed '-' from strings. Now, this behavior is removed, so if you had '-' in these strings, please remove them. Note that '-' in these values is not allowed by Snowflake.
- `external_oauth_snowflake_user_mapping_attribute`
- `external_oauth_type`
- `external_oauth_any_role_mode`

### *(new feature)* snowflake_saml2_integration resource

The new `snowflake_saml2_integration` is introduced and deprecates `snowflake_saml_integration`. It contains new fields
and follows our new conventions making it more stable. The old SAML integration wasn't changed, so no migration needed, 
but we recommend to eventually migrate to the newer counterpart.

### snowflake_scim_integration resource changes
#### *(behavior change)* Changed behavior of `sync_password`

Now, the `sync_password` field will set the state value to `default` whenever the value is not set in the config. This indicates that the value on the Snowflake side is set to the Snowflake default.

#### *(behavior change)* Renamed fields

Renamed field `provisioner_role` to `run_as_role` to align with Snowflake docs. Please rename this field in your configuration files. State will be migrated automatically.

#### *(new feature)* New fields
Fields added to the resource:
- `enabled`
- `sync_password`
- `comment`

#### *(behavior change)* Changed behavior of `enabled`
New field `enabled` is required. Previously the default value during create in Snowflake was `true`. If you created a resource with Terraform, please add `enabled = true` to have the same value.

#### *(behavior change)* Force new for multiple attributes
ForceNew was added for the following attributes (because there are no usable SQL alter statements for them):
- `scim_client`
- `run_as_role`

### snowflake_warehouse resource changes

Because of the multiple changes in the resource, the easiest migration way is to follow our [migration guide](https://github.com/Snowflake-Labs/terraform-provider-snowflake/blob/main/docs/technical-documentation/resource_migration.md) to perform zero downtime migration. Alternatively, it is possible to follow some pointers below. Either way, familiarize yourself with the resource changes before version bumping. Also, check the [design decisions](./v1-preparations/CHANGES_BEFORE_V1.md).

#### *(potential behavior change)* Default values removed
As part of the [redesign](https://github.com/Snowflake-Labs/terraform-provider-snowflake/blob/main/ROADMAP.md#preparing-essential-ga-objects-for-the-provider-v1) we are removing the default values for attributes having their defaults on Snowflake side to reduce coupling with the provider (read more in [default values](./v1-preparations/CHANGES_BEFORE_V1.md#default-values)). Because of that the following defaults were removed:
- `comment` (previously `""`)
- `enable_query_acceleration` (previously `false`)
- `query_acceleration_max_scale_factor` (previously `8`)
- `warehouse_type` (previously `"STANDARD"`)
- `max_concurrency_level` (previously `8`)
- `statement_queued_timeout_in_seconds` (previously `0`)
- `statement_timeout_in_seconds` (previously `172800`)

**Beware!** For attributes being Snowflake parameters (in case of warehouse: `max_concurrency_level`, `statement_queued_timeout_in_seconds`, and `statement_timeout_in_seconds`), this is a breaking change (read more in [Snowflake parameters](./v1-preparations/CHANGES_BEFORE_V1.md#snowflake-parameters)). Previously, not setting a value for them was treated as a fallback to values hardcoded on the provider side. This caused warehouse creation with these parameters set on the warehouse level (and not using the Snowflake default from hierarchy; read more in the [parameters documentation](https://docs.snowflake.com/en/sql-reference/parameters)). To keep the previous values, fill in your configs to the default values listed above.

All previous defaults were aligned with the current Snowflake ones, however it's not possible to distinguish between filled out value and no value in the automatic state upgrader. Therefore, if the given attribute is not filled out in your configuration, terraform will try to perform update after the change (to UNSET the given attribute to the Snowflake default); it should result in no changes on Snowflake object side, but it is required to make Terraform state aligned with your config. **All** other optional fields that were not set inside the config at all (because of the change in handling state logic on our provider side) will follow the same logic. To avoid the need for the changes, fill out the default fields in your config. Alternatively, run `terraform apply`; no further changes should be shown as a part of the plan.

#### *(note)* Automatic state migrations
There are three migrations that should happen automatically with the version bump:
- incorrect `2XLARGE`, `3XLARGE`, `4XLARGE`, `5XLARGE`, `6XLARGE` values for warehouse size are changed to the proper ones
- deprecated `wait_for_provisioning` attribute is removed from the state
- old empty resource monitor attribute is cleaned (earlier it was set to `"null"` string)

#### *(fix)* Warehouse size UNSET

Before the changes, removing warehouse size from the config was not handled properly. Because UNSET is not supported for warehouse size (check the [docs](https://docs.snowflake.com/en/sql-reference/sql/alter-warehouse#properties-parameters) - usage notes for unset) and there are multiple defaults possible, removing the size from config will result in the resource recreation.

#### *(behavior change)* Validation changes
As part of the [redesign](https://github.com/Snowflake-Labs/terraform-provider-snowflake/blob/main/ROADMAP.md#preparing-essential-ga-objects-for-the-provider-v1) we are adjusting validations or removing them to reduce coupling between Snowflake and the provider. Because of that the following validations were removed/adjusted/added:
- `max_cluster_count` - adjusted: added higher bound (10) according to Snowflake docs
- `min_cluster_count` - adjusted: added higher bound (10) according to Snowflake docs
- `auto_suspend` - adjusted: added `0` as valid value
- `warehouse_size` - adjusted: removed incorrect `2XLARGE`, `3XLARGE`, `4XLARGE`, `5XLARGE`, `6XLARGE` values
- `resource_monitor` - added: validation for a valid identifier (still subject to change during [identifiers rework](https://github.com/Snowflake-Labs/terraform-provider-snowflake/blob/main/ROADMAP.md#identifiers-rework))
- `max_concurrency_level` - added: validation according to MAX_CONCURRENCY_LEVEL parameter docs
- `statement_queued_timeout_in_seconds` - added: validation according to STATEMENT_QUEUED_TIMEOUT_IN_SECONDS parameter docs
- `statement_timeout_in_seconds` - added: validation according to STATEMENT_TIMEOUT_IN_SECONDS parameter docs

#### *(behavior change)* Deprecated `wait_for_provisioning` field removed
`wait_for_provisioning` field was deprecated a long time ago. It's high time it was removed from the schema.

#### *(behavior change)* `query_acceleration_max_scale_factor` conditional logic removed
Previously, the `query_acceleration_max_scale_factor` was depending on `enable_query_acceleration` parameter, but it is not required on Snowflake side. After migration, `terraform plan` should suggest changes if `enable_query_acceleration` was earlier set to false (manually or from default) and if `query_acceleration_max_scale_factor` was set in config.

#### *(behavior change)* `initially_suspended` forceNew removed
Previously, the `initially_suspended` attribute change caused the resource recreation. This attribute is used only during creation (to create suspended warehouse). There is no reason to recreate the whole object just to have initial state changed.

#### *(behavior change)* Boolean type changes
To easily handle three-value logic (true, false, unknown) in provider's configs, type of `auto_resume` and `enable_query_acceleration` was changed from boolean to string. This should not require updating existing configs (boolean/int value should be accepted and state will be migrated to string automatically), however we recommend changing config values to strings. Terraform should perform an action for configs lacking `auto_resume` or `enable_query_acceleration` (`ALTER WAREHOUSE UNSET AUTO_RESUME` and/or `ALTER WAREHOUSE UNSET ENABLE_QUERY_ACCELERATION` will be run underneath which should not affect the Snowflake object, because `auto_resume` and `enable_query_acceleration` are false by default).

#### *(note)* `resource_monitor` validation and diff suppression
`resource_monitor` is an identifier and handling logic may be still slightly changed as part of https://github.com/Snowflake-Labs/terraform-provider-snowflake/blob/main/ROADMAP.md#identifiers-rework. It should be handled automatically (without needed manual actions on user side), though, but it is not guaranteed.

#### *(behavior change)* snowflake_warehouses datasource
- Added `like` field to enable warehouse filtering
- Added missing fields returned by SHOW WAREHOUSES and enclosed its output in `show_output` field.
- Added outputs from **DESC WAREHOUSE** and **SHOW PARAMETERS IN WAREHOUSE** (they can be turned off by declaring `with_describe = false` and `with_parameters = false`, **they're turned on by default**).
  The additional parameters call **DESC WAREHOUSE** (with `with_describe` turned on) and **SHOW PARAMETERS IN WAREHOUSE** (with `with_parameters` turned on) **per warehouse** returned by **SHOW WAREHOUSES**.
  The outputs of both commands are held in `warehouses` entry, where **DESC WAREHOUSE** is saved in the `describe_output` field, and **SHOW PARAMETERS IN WAREHOUSE** in the `parameters` field.
  It's important to limit the records and calls to Snowflake to the minimum. That's why we recommend assessing which information you need from the data source and then providing strong filters and turning off additional fields for better plan performance.

You can read more in ["raw Snowflake output"](./v1-preparations/CHANGES_BEFORE_V1.md#empty-values).

### *(new feature)* new database resources
As part of the [preparation for v1](https://github.com/Snowflake-Labs/terraform-provider-snowflake/blob/main/ROADMAP.md#preparing-essential-ga-objects-for-the-provider-v1), we split up the database resource into multiple ones:
- Standard database - can be used as `snowflake_database` (replaces the old one and is used to create databases with optional ability to become a primary database ready for replication)
- Shared database - can be used as `snowflake_shared_database` (used to create databases from externally defined shares)
- Secondary database - can be used as `snowflake_secondary_database` (used to create replicas of databases from external sources)

All the field changes in comparison to the previous database resource are:
- `is_transient`
    - in `snowflake_shared_database`
        - removed: the field is removed from `snowflake_shared_database` as it doesn't have any effect on shared databases.
- `from_database` - database cloning was entirely removed and is not possible by any of the new database resources.
- `from_share` - the parameter was moved to the dedicated resource for databases created from shares `snowflake_shared_database`. Right now, it's a text field instead of a map. Additionally, instead of legacy account identifier format we're expecting the new one that with share looks like this: `<organization_name>.<account_name>.<share_name>`. For more information on account identifiers, visit the [official documentation](https://docs.snowflake.com/en/user-guide/admin-account-identifier).
- `from_replication` - the parameter was moved to the dedicated resource for databases created from primary databases `snowflake_secondary_database`
- `replication_configuration` - renamed: was renamed to `configuration` and is only available in the `snowflake_database`. Its internal schema changed that instead of list of accounts, we expect a list of nested objects with accounts for which replication (and optionally failover) should be enabled. More information about converting between both versions [here](#resource-renamed-snowflake_database---snowflake_database_old). Additionally, instead of legacy account identifier format we're expecting the new one that looks like this: `<organization_name>.<account_name>` (it will be automatically migrated to the recommended format by the state upgrader). For more information on account identifiers, visit the [official documentation](https://docs.snowflake.com/en/user-guide/admin-account-identifier).
- `data_retention_time_in_days`
  - in `snowflake_shared_database`
      - removed: the field is removed from `snowflake_shared_database` as it doesn't have any effect on shared databases.
  - in `snowflake_database` and `snowflake_secondary_database`
    - adjusted: now, it uses different approach that won't set it to -1 as a default value, but rather fills the field with the current value from Snowflake (this still can change).
- added: The following set of [parameters](https://docs.snowflake.com/en/sql-reference/parameters) was added to every database type:
    - `max_data_extension_time_in_days`
    - `external_volume`
    - `catalog`
    - `replace_invalid_characters`
    - `default_ddl_collation`
    - `storage_serialization_policy`
    - `log_level`
    - `trace_level`
    - `suspend_task_after_num_failures`
    - `task_auto_retry_attempts`
    - `user_task_managed_initial_warehouse_size`
    - `user_task_timeout_ms`
    - `user_task_minimum_trigger_interval_in_seconds`
    - `quoted_identifiers_ignore_case`
    - `enable_console_output`

The split was done (and will be done for several objects during the refactor) to simplify the resource on maintainability and usage level.
Its purpose was also to divide the resources by their specific purpose rather than cramping every use case of an object into one resource.

### *(behavior change)* Resource renamed snowflake_database -> snowflake_database_old
We made a decision to use the existing `snowflake_database` resource for redesigning it into a standard database.
The previous `snowflake_database` was renamed to `snowflake_database_old` and the current `snowflake_database`
contains completely new implementation that follows our guidelines we set for V1.
When upgrading to the 0.93.0 version, the automatic state upgrader should cover the migration for databases that didn't have the following fields set:
- `from_share` (now, the new `snowflake_shared_database` should be used instead)
- `from_replica` (now, the new `snowflake_secondary_database` should be used instead)
- `replication_configuration`

For configurations containing `replication_configuraiton` like this one:
```terraform
resource "snowflake_database" "test" {
  name = "<name>"
  replication_configuration {
    accounts = ["<account_locator>", "<account_locator_2>"]
    ignore_edition_check = true
  }
}
```

You have to transform the configuration into the following format (notice the change from account locator into the new account identifier format):
```terraform
resource "snowflake_database" "test" {
  name = "%s"
  replication {
    enable_to_account {
      account_identifier = "<organization_name>.<account_name>"
      with_failover      = false
    }
    enable_to_account {
      account_identifier = "<organization_name_2>.<account_name_2>"
      with_failover      = false
    }
  }
  ignore_edition_check = true
}
```

If you had `from_database` set, it should migrate automatically.
For now, we're dropping the possibility to create a clone database from other databases.
The only way will be to clone a database manually and import it as `snowflake_database`, but if
cloned databases diverge in behavior from standard databases, it may cause issues.

For databases with one of the fields mentioned above, manual migration will be needed.
Please refer to our [migration guide](https://github.com/Snowflake-Labs/terraform-provider-snowflake/blob/main/docs/technical-documentation/resource_migration.md) to perform zero downtime migration.

If you would like to upgrade to the latest version and postpone the upgrade, you still have to perform the manual migration
to the `snowflake_database_old` resource by following the [zero downtime migrations document](https://github.com/Snowflake-Labs/terraform-provider-snowflake/blob/main/docs/technical-documentation/resource_migration.md).
The only difference would be that instead of writing/generating new configurations you have to just rename the existing ones to contain `_old` suffix.

### *(behavior change)* snowflake_databases datasource
- `terse` and `history` fields were removed.
- `replication_configuration` field was removed from `databases`.
- `pattern` was replaced by `like` field.
- Additional filtering options added (`limit`).
- Added missing fields returned by SHOW DATABASES and enclosed its output in `show_output` field.
- Added outputs from **DESC DATABASE** and **SHOW PARAMETERS IN DATABASE** (they can be turned off by declaring `with_describe = false` and `with_parameters = false`, **they're turned on by default**).
The additional parameters call **DESC DATABASE** (with `with_describe` turned on) and **SHOW PARAMETERS IN DATABASE** (with `with_parameters` turned on) **per database** returned by **SHOW DATABASES**.
The outputs of both commands are held in `databases` entry, where **DESC DATABASE** is saved in the `describe_output` field, and **SHOW PARAMETERS IN DATABASE** in the `parameters` field.
It's important to limit the records and calls to Snowflake to the minimum. That's why we recommend assessing which information you need from the data source and then providing strong filters and turning off additional fields for better plan performance.

## v0.89.0 ➞ v0.90.0
### snowflake_table resource changes
#### *(behavior change)* Validation to column type added
While solving issue [#2733](https://github.com/Snowflake-Labs/terraform-provider-snowflake/issues/2733) we have introduced diff suppression for `column.type`. To make it work correctly we have also added a validation to it. It should not cause any problems, but it's worth noting in case of any data types used that the provider is not aware of.

### snowflake_procedure resource changes
#### *(behavior change)* Validation to arguments type added
Diff suppression for `arguments.type` is needed for the same reason as above for `snowflake_table` resource.

### tag_masking_policy_association resource changes
Now the `tag_masking_policy_association` resource will only accept fully qualified names separated by dot `.` instead of pipe `|`.

Before
```terraform
resource "snowflake_tag_masking_policy_association" "name" {
    tag_id            = snowflake_tag.this.id
    masking_policy_id = snowflake_masking_policy.example_masking_policy.id
}
```

After
```terraform
resource "snowflake_tag_masking_policy_association" "name" {
    tag_id            = "\"${snowflake_tag.this.database}\".\"${snowflake_tag.this.schema}\".\"${snowflake_tag.this.name}\""
    masking_policy_id = "\"${snowflake_masking_policy.example_masking_policy.database}\".\"${snowflake_masking_policy.example_masking_policy.schema}\".\"${snowflake_masking_policy.example_masking_policy.name}\""
}
```

It's more verbose now, but after identifier rework it should be similar to the previous form.

## v0.88.0 ➞ v0.89.0
#### *(behavior change)* ForceNew removed
The `ForceNew` field was removed in favor of in-place Update for `name` parameter in:
- `snowflake_file_format`
- `snowflake_masking_policy`
So from now, these objects won't be re-created when the `name` changes, but instead only the name will be updated with `ALTER .. RENAME TO` statements.

## v0.87.0 ➞ v0.88.0
### snowflake_procedure resource changes
#### *(behavior change)* Execute as validation added
From now on, the `snowflake_procedure`'s `execute_as` parameter allows only two values: OWNER and CALLER (case-insensitive). Setting other values earlier resulted in falling back to the Snowflake default (currently OWNER) and creating a permadiff.

### snowflake_grants datasource changes
`snowflake_grants` datasource was refreshed as part of the ongoing [Grants Redesign](https://github.com/Snowflake-Labs/terraform-provider-snowflake/blob/main/ROADMAP.md#redesigning-grants).

#### *(behavior change)* role fields renames
To be aligned with the convention in other grant resources, `role` was renamed to `account_role` for the following fields:
- `grants_to.role`
- `grants_of.role`
- `future_grants_to.role`.

To migrate simply change `role` to `account_role` in the aforementioned fields.

#### *(behavior change)* grants_to.share type change
`grants_to.share` was a text field. Because Snowflake introduced new syntax `SHOW GRANTS TO SHARE <share_name> IN APPLICATION PACKAGE <app_package_name>` (check more in the [docs](https://docs.snowflake.com/en/sql-reference/sql/show-grants#variants)) the type was changed to object. To migrate simply change:
```terraform
data "snowflake_grants" "example_to_share" {
  grants_to {
    share = "some_share"
  }
}
```
to
```terraform
data "snowflake_grants" "example_to_share" {
  grants_to {
    share {
      share_name = "some_share"
    }
  }
}
```
Note: `in_application_package` is not yet supported.

#### *(behavior change)* future_grants_in.schema type change
`future_grants_in.schema` was an object field allowing to set required `schema_name` and optional `database_name`. Our strategy is to be explicit, so the schema field was changed to string and fully qualified name is expected. To migrate change:
```terraform
data "snowflake_grants" "example_future_in_schema" {
  future_grants_in {
    schema {
      database_name = "some_database"
      schema_name   = "some_schema"
    }
  }
}
```
to
```terraform
data "snowflake_grants" "example_future_in_schema" {
  future_grants_in {
    schema = "\"some_database\".\"some_schema\""
  }
}
```
#### *(new feature)* grants_to new options
`grants_to` was enriched with three new options:
- `application`
- `application_role`
- `database_role`

No migration work is needed here.

#### *(new feature)* grants_of new options
`grants_to` was enriched with two new options:
- `database_role`
- `application_role`

No migration work is needed here.

#### *(new feature)* future_grants_to new options
`future_grants_to` was enriched with one new option:
- `database_role`

No migration work is needed here.

#### *(documentation)* improvements
Descriptions of attributes were altered. More examples were added (both for old and new features).

## v0.86.0 ➞ v0.87.0
### snowflake_database resource changes
#### *(behavior change)* External object identifier changes

Previously, in `snowflake_database` when creating a database form share, it was possible to provide `from_share.provider`
in the format of `<org_name>.<account_name>`. It worked even though we expected account locator because our "external" identifier wasn't quoting its string representation.
To be consistent with other identifier types, we quoted the output of "external" identifiers which makes such configurations break
(previously, they were working "by accident"). To fix it, the previous format of `<org_name>.<account_name>` has to be changed
to account locator format `<account_locator>` (mind that it's now case-sensitive). The account locator can be retrieved by calling `select current_account();` on the sharing account.
In the future we would like to eventually come back to the `<org_name>.<account_name>` format as it's recommended by Snowflake.

### Provider configuration changes

#### **IMPORTANT** *(bug fix)* Configuration hierarchy
There were several issues reported about the configuration hierarchy, e.g. [#2294](https://github.com/Snowflake-Labs/terraform-provider-snowflake/issues/2294) and [#2242](https://github.com/Snowflake-Labs/terraform-provider-snowflake/issues/2242).
In fact, the order of precedence described in the docs was not followed. This have led to the incorrect behavior.

After migrating to this version, the hierarchy from the docs should be followed:
```text
The Snowflake provider will use the following order of precedence when determining which credentials to use:
1) Provider Configuration
2) Environment Variables
3) Config File
```

**BEWARE**: your configurations will be affected with that change because they may have been leveraging the incorrect configurations precedence. Please be sure to check all the configurations before running terraform.

### snowflake_failover_group resource changes
#### *(bug fix)* ACCOUNT PARAMETERS is returned as PARAMETERS from SHOW FAILOVER GROUPS
Longer context in [#2517](https://github.com/Snowflake-Labs/terraform-provider-snowflake/issues/2517).
After this change, one apply may be required to update the state correctly for failover group resources using `ACCOUNT PARAMETERS`.

### snowflake_database, snowflake_schema, and snowflake_table resource changes
#### *(behavior change)* Database `data_retention_time_in_days` + Schema `data_retention_days` + Table `data_retention_time_in_days`
For context [#2356](https://github.com/Snowflake-Labs/terraform-provider-snowflake/issues/2356).
To make data retention fields truly optional (previously they were producing plan every time when no value was set),
we added `-1` as a possible value, and it is set as default. That got rid of the unexpected plans when no value is set and added possibility to use default value assigned by Snowflake (see [the data retention period](https://docs.snowflake.com/en/user-guide/data-time-travel#data-retention-period)).

### snowflake_table resource changes
#### *(behavior change)* Table `data_retention_days` field removed in favor of `data_retention_time_in_days`
For context [#2356](https://github.com/Snowflake-Labs/terraform-provider-snowflake/issues/2356).
To define data retention days for table `data_retention_time_in_days` should be used as deprecated `data_retention_days` field is being removed.

## v0.85.0 ➞ v0.86.0
### snowflake_table_constraint resource changes

#### *(behavior change)* NOT NULL removed from possible types
The `type` of the constraint was limited back to `UNIQUE`, `PRIMARY KEY`, and `FOREIGN KEY`.
The reason for that is, that syntax for Out-of-Line constraint ([docs](https://docs.snowflake.com/en/sql-reference/sql/create-table-constraint#out-of-line-unique-primary-foreign-key)) does not contain `NOT NULL`.
It is noted as a behavior change but in some way it is not; with the previous implementation it did not work at all with `type` set to `NOT NULL` because the generated statement was not a valid Snowflake statement.

We will consider adding `NOT NULL` back because it can be set by `ALTER COLUMN columnX SET NOT NULL`, but first we want to revisit the whole resource design.

#### *(behavior change)* table_id reference
The docs were inconsistent. Example prior to 0.86.0 version showed using the `table.id` as the `table_id` reference. The description of the `table_id` parameter never allowed such a value (`table.id` is a `|`-delimited identifier representation and only the `.`-separated values were listed in the docs: https://registry.terraform.io/providers/Snowflake-Labs/snowflake/0.85.0/docs/resources/table_constraint#required. The misuse of `table.id` parameter will result in error after migrating to 0.86.0. To make the config work, please remove and reimport the constraint resource from the state as described in [resource migration doc](./docs/technical-documentation/resource_migration.md).

After discussions in [#2535](https://github.com/Snowflake-Labs/terraform-provider-snowflake/issues/2535) we decided to provide a temporary workaround in 0.87.0 version, so that the manual migration is not necessary. It allows skipping the migration and jumping straight to 0.87.0 version. However, the temporary workaround will be gone in one of the future versions. Please adjust to the newly suggested reference with the new resources you create.

### snowflake_external_function resource changes

#### *(behavior change)* return_null_allowed default is now true
The `return_null_allowed` attribute default value is now `true`. This is a behavior change because it was `false` before. The reason it was changed is to match the expected default value in the [documentation](https://docs.snowflake.com/en/sql-reference/sql/create-external-function#optional-parameters) `Default: The default is NULL (i.e. the function can return NULL values).`

#### *(behavior change)* comment is no longer required
The `comment` attribute is now optional. It was required before, but it is not required in Snowflake API.

### snowflake_external_functions data source changes

#### *(behavior change)* schema is now required with database
The `schema` attribute is now required with `database` attribute to match old implementation `SHOW EXTERNAL FUNCTIONS IN SCHEMA "<database>"."<schema>"`. In the future this may change to make schema optional.

## vX.XX.X -> v0.85.0

### Migration from old (grant) resources to new ones

In recent changes, we introduced a new grant resources to replace the old ones.
To aid with the migration, we wrote a guide to show one of the possible ways to migrate deprecated resources to their new counter-parts.
As the guide is more general and applies to every version (and provider), we moved it [here](./docs/technical-documentation/resource_migration.md).

### snowflake_procedure resource changes
#### *(deprecation)* return_behavior
`return_behavior` parameter is deprecated because it is also deprecated in the Snowflake API.

### snowflake_function resource changes
#### *(behavior change)* return_type
`return_type` has become force new because there is no way to alter it without dropping and recreating the function.

## v0.84.0 ➞ v0.85.0

### snowflake_stage resource changes

#### *(behavior change/regression)* copy_options
Setting `copy_options` to `ON_ERROR = 'CONTINUE'` would result in a permadiff. Use `ON_ERROR = CONTINUE` (without single quotes) or bump to v0.89.0 in which the behavior was fixed.

### snowflake_notification_integration resource changes
#### *(behavior change)* notification_provider
`notification_provider` becomes required and has three possible values `AZURE_STORAGE_QUEUE`, `AWS_SNS`, and `GCP_PUBSUB`.
It is still possible to set it to `AWS_SQS` but because there is no underlying SQL, so it will result in an error.
Attributes `aws_sqs_arn` and `aws_sqs_role_arn` will be ignored.
Computed attributes `aws_sqs_external_id` and `aws_sqs_iam_user_arn` won't be updated.

#### *(behavior change)* force new for multiple attributes
Force new was added for the following attributes (because no usable SQL alter statements for them):
- `azure_storage_queue_primary_uri`
- `azure_tenant_id`
- `gcp_pubsub_subscription_name`
- `gcp_pubsub_topic_name`

#### *(deprecation)* direction
`direction` parameter is deprecated because it is added automatically on the SDK level.

#### *(deprecation)* type
`type` parameter is deprecated because it is added automatically on the SDK level (and basically it's always `QUEUE`).

## v0.73.0 ➞ v0.74.0
### Provider configuration changes

In this change we have done a provider refactor to make it more complete and customizable by supporting more options that
were already available in Golang Snowflake driver. This lead to several attributes being added and a few deprecated.
We will focus on the deprecated ones and show you how to adapt your current configuration to the new changes.

#### *(rename)* username ➞ user

```terraform
provider "snowflake" {
  # before
  username = "username"

  # after
  user = "username"
}
```

#### *(structural change)* OAuth API

```terraform
provider "snowflake" {
  # before
  browser_auth        = false
  oauth_access_token  = "<access_token>"
  oauth_refresh_token = "<refresh_token>"
  oauth_client_id     = "<client_id>"
  oauth_client_secret = "<client_secret>"
  oauth_endpoint      = "<endpoint>"
  oauth_redirect_url  = "<redirect_uri>"

  # after
  authenticator = "ExternalBrowser"
  token         = "<access_token>"
  token_accessor {
    refresh_token   = "<refresh_token>"
    client_id       = "<client_id>"
    client_secret   = "<client_secret>"
    token_endpoint  = "<endpoint>"
    redirect_uri    = "<redirect_uri>"
  }
}
```

#### *(remove redundant information)* region

Specifying a region is a legacy thing and according to https://docs.snowflake.com/en/user-guide/admin-account-identifier
you can specify a region as a part of account parameter. Specifying account parameter with the region is also considered legacy,
but with this approach it will be easier to convert only your account identifier to the new preferred way of specifying account identifier.

```terraform
provider "snowflake" {
  # before
  region = "<cloud_region_id>"

  # after
  account = "<account_locator>.<cloud_region_id>"
}
```

#### *(todo)* private key path

```terraform
provider "snowflake" {
  # before
  private_key_path = "<filepath>"

  # after
  private_key = file("<filepath>")
}
```

#### *(rename)* session_params ➞ params

```terraform
provider "snowflake" {
  # before
  session_params = {}

  # after
  params = {}
}
```

#### *(behavior change)* authenticator (JWT)

Before the change `authenticator` parameter did not have to be set for private key authentication and was deduced by the provider. The change is a result of the introduced configuration alignment with an underlying [gosnowflake driver](https://github.com/snowflakedb/gosnowflake). The authentication type is required there, and it defaults to user+password one. From this version, set `authenticator` to `JWT` explicitly.
