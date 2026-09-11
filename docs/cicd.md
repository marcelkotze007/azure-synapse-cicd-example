# CI/CD flow — technical reference

The detailed breakdown of how this template packages and deploys an Azure Synapse
workspace. For the plain-language overview, see the [README](../README.md).

## Source layout

- `synapse/` — the Synapse Studio-exported workspace JSON, one folder per artifact
  type (pipelines, datasets, linked services, notebooks, credentials, integration
  runtimes, managed virtual networks). This folder is the Git integration *root
  folder* configured in Synapse Studio.
- `synapse/template-parameters-definition.json` — decides which properties become
  ARM template parameters. See [Parameterisation](#parameterisation) below.
- `synapse/publish_config.json` — written by Synapse Studio; records the
  publish branch used by the workspace's Git integration.
- `.azure/pipelines/` — the Azure DevOps pipeline: one entry point, one packaging
  stage, and one reusable deployment stage used once per environment.

## Pipeline flow

There are two kinds of stage: a single `Validate` stage that builds the deployable
artifact **once**, and a `deploy_synapse_*` stage per environment that deploys
**that same artifact** with environment-specific values.

### 1. Validate — generate the ARM template and parameters

`.azure/pipelines/stages/validate.yml` checks out the repo and runs the
`Synapse workspace deployment@2` task with `operation: validate` against the
`./synapse` folder. That converts the raw workspace JSON into an ARM template pair:

- `TemplateForWorkspace.json` — every Synapse artifact, as ARM resources.
- `TemplateParametersForWorkspace.json` — one parameter for each property that
  `template-parameters-definition.json` flags.

`TargetWorkspaceName` is set to the literal `synapseArtifact` rather than a real
workspace name: for the `validate` operation this value names the **published
pipeline artifact**, which is what the deploy stages ask for by name. No
deployment has happened at the end of this stage.

### 2. Deploy — download the artifact, override the parameters, apply

`.azure/pipelines/stages/deploy-synapse.yml` is a single parameterised template,
invoked once per environment from `pipeline.yml`. Each invocation:

1. Downloads the `synapseArtifact` produced by Validate.
2. Loads that environment's variable group (`pear_synapse_dev_variables`,
   `pear_synapse_prod_variables`, ...) for its subscription, resource group,
   workspace, pool, server, database and secrets.
3. Runs `Synapse workspace deployment@2` with `operation: deploy` against that
   environment's service connection and workspace.
4. Supplies a value for every generated parameter through `OverrideArmParameters`.
   This is the step that turns one generic artifact into a correctly configured
   deployment for that specific environment.

Each stage targets its own Azure DevOps `environment:`, so approvals and checks
configured there gate the deployment.

Adding an environment is a new block in `pipeline.yml` — the deployment steps
themselves are never copied, because all environments share the one stage template.

## Parameterisation

`synapse/template-parameters-definition.json` lists, per artifact type, the
properties that should become ARM parameters. The markers mean:

| Marker | Meaning |
| --- | --- |
| `"="` | Make this an ARM parameter, **keeping the current value as the default**. If an environment supplies no override, the value committed in the repo is used. |
| `"-"` | Make this an ARM parameter **with no default value**. Every environment must supply an override or the deployment fails. |
| `"\|"` | Resolve the value from an Azure Key Vault secret reference. |

`"-"` is the safer marker for anything that must never silently carry a
development value into another environment — SQL pool names, servers and
databases. That is exactly how this template uses it.

### Parameter naming rules

The generated parameter name is the artifact name followed by the path to the
property, with `_` between segments. The `activities` array contributes its
**index** rather than its name:

| Property in the artifact | Generated parameter name |
| --- | --- |
| Pipeline, top-level activity *n* | `<pipeline>_properties_<n>_sqlPool_referenceName` |
| Pipeline, activity *n*, nested activity *m* | `<pipeline>_properties_<n>_typeProperties_<m>_sqlPool_referenceName` |
| Dataset property | `<dataset>_properties_sqlPool_referenceName` |
| Linked service property | `<linkedService>_properties_typeProperties_<property>` |
| Notebook property | `<notebook>_properties_bigDataPool_referenceName` |

Because activity **position** is part of the name, reordering activities in a
pipeline changes its parameter names. Update `OverrideArmParameters` in the same
change, otherwise the deployment fails on an unmatched parameter.

### What this template's artifacts generate

| Parameter | Source artifact | Marker |
| --- | --- | --- |
| `Refresh Sales Aggregates_properties_0_sqlPool_referenceName` | `Refresh Sales Aggregates`, activity 0 (`Refresh Sales Summary`) | `-` |
| `Refresh Sales Aggregates_properties_2_typeProperties_0_sqlPool_referenceName` | `Refresh Sales Aggregates`, activity 2 (`For Each Region`), nested activity 0 | `-` |
| `SalesRegionPoolTable_properties_sqlPool_referenceName` | dataset `SalesRegionPoolTable` | `-` |
| `PearAzureSqlDatabase_properties_typeProperties_server` | linked service `PearAzureSqlDatabase` | `-` |
| `PearAzureSqlDatabase_properties_typeProperties_database` | linked service `PearAzureSqlDatabase` | `-` |
| `PearAzureSqlDatabase_properties_typeProperties_password` | linked service `PearAzureSqlDatabase` | `=` |
| `PearDataLakeStorage_properties_typeProperties_connectionString` | linked service `PearDataLakeStorage` | `=` |
| `PearSalesExploration_properties_bigDataPool_referenceName` | notebook `PearSalesExploration` | `=` |

`Load Customer Orders` and `Daily Sales Orchestration` generate nothing: the first
reaches its data through a linked service (which carries the environment-specific
values on its own), and the second only orchestrates other pipelines.

### Entries in the definition file with no example here

`template-parameters-definition.json` is kept intact, so it also covers
connectors this minimal example does not include:

- `typeProperties.accountName` and `typeProperties.secretAccessKey` — used by
  storage and S3-style connectors. Add such a linked service and the parameters
  appear automatically; add the matching `OverrideArmParameters` line.
- `typeProperties.username` — note the casing. Synapse writes `userName` for Azure
  SQL linked services, and matching is case-sensitive, so this entry does **not**
  match `PearAzureSqlDatabase`. Change the key to `userName` if you want the user
  name parameterised too. The template instead deploys a fixed user name and
  overrides only the password.
- `datasets` → `typeProperties.folderPath` / `fileName` — these match the older
  flat dataset shape. `DelimitedText` datasets like `OrdersLakeCsv` nest the same
  properties under `typeProperties.location`, so the entries produce no parameter
  for it. Move them under a `location` block in the definition file if you need
  per-environment paths for modern datasets.

## Environment settings

Each environment needs a variable group with these keys:

| Variable | Example (Dev) | Secret |
| --- | --- | --- |
| `subscription_id` | `00000000-0000-0000-0000-000000000000` | no |
| `resource_group_name` | `rg-pear-analytics-dev` | no |
| `synapse_workspace_name` | `synw-pear-dev-01` | no |
| `sql_pool_name` | `pear_dw` | no |
| `spark_pool_name` | `sparkpeardev` | no |
| `orders_sql_server` | `sql-pear-dev.database.windows.net,1433` | no |
| `orders_sql_database` | `sqldb-pear-orders-dev` | no |
| `orders_sql_password` | — | yes |
| `lake_connection_string` | — | yes |

Approvals, service connections and secrets are configured in Azure DevOps
(environments, variable groups, service connections) and are never committed here.
Secret variables are best backed by Azure Key Vault through a linked variable group.
