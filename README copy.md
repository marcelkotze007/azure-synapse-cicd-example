# Azure Synapse workspace CI/CD template

A small, complete example of keeping an Azure Synapse workspace in Git and
deploying it through Azure DevOps — the same artifact packaged once and released
to each environment with its own connection details.

The example workspace belongs to a fictional **Pear Company** and has three
pipelines, deliberately covering the three cases you meet in practice:

| Pipeline | What it shows |
| --- | --- |
| `Load Customer Orders` | Works through a **linked service** (copy from Azure SQL to blob storage, then a stored procedure over that linked service). |
| `Refresh Sales Aggregates` | Works against a **dedicated SQL pool**, both on a top-level activity and on an activity nested inside a `ForEach`. |
| `Daily Sales Orchestration` | Uses **neither** — it only sequences the other two, so it deploys unchanged everywhere. |

Between them, plus one notebook, two linked services and three datasets, every
property flagged in `synapse/template-parameters-definition.json` that this
example can demonstrate has a working example behind it.

## What's in here

```text
synapse/       The Synapse workspace content (pipelines, datasets, linked services, notebook, ...)
.azure/        The Azure DevOps deployment process
docs/          Technical reference for the CI/CD flow and parameterisation
infra/         Placeholder for the Azure infrastructure that hosts the workspace
```

## How a change gets deployed

A change merged into `main` and rolled out in two steps:

1. **Package** — the whole workspace is packaged into one deployable ARM bundle.
   Anything environment-specific (server, database, SQL pool, secrets) is left as
   a placeholder, because the same bundle is reused everywhere.
2. **Deploy** — that bundle is deployed one environment at a time (**Dev → Prod**
   in this template). Before each deployment the placeholders are filled in with
   that environment's real settings, so a single release lands correctly wherever
   it goes.

Each environment can require a sign-off before its deployment proceeds, so nothing
reaches production without approval.

The mechanics — how placeholders are chosen, how their names are derived, and what
each environment must supply — are in [`docs/cicd.md`](docs/cicd.md).

## Using this template

1. **Copy the repo** and replace the Pear Company naming with your own: artifact
   names (`Pear*`), the SQL pool `pear_dw`, the Spark pool `sparkpeardev`, the
   folder `Pear Analytics`, and the Azure DevOps names in
   `.azure/pipelines/pipeline.yml` (variable groups `pear_synapse_*_variables`,
   service connections `PEAR_DEVOPS_*`, environments `pear-synapse-*`).
2. **Point Synapse Studio at the repo** with `synapse` as the Git integration root
   folder and `workspace_publish` as the publish branch.
3. **Create one variable group per environment** with the keys listed in
   [`docs/cicd.md`](docs/cicd.md#environment-settings), and a service connection
   and Azure DevOps environment for each.
4. **Create the pipeline** in Azure DevOps from `.azure/pipelines/pipeline.yml`.
5. **Keep `OverrideArmParameters` in step** with the workspace: adding a
   parameterised artifact means adding a line to
   `.azure/pipelines/stages/deploy-synapse.yml`. There is only one copy of that
   block, shared by every environment.

## Replacing the example content

Delete the example artifacts under `synapse/` and export your own from Synapse
Studio. Keep `template-parameters-definition.json`, `publish_config.json` and the
`.azure/` folder, then rebuild `OverrideArmParameters` from the parameter names in
the generated `TemplateParametersForWorkspace.json` (the Validate stage's artifact
is the quickest way to see them).

## Notes

- Approvals, secrets and service connections live in Azure DevOps, never in this
  repo. The `password` and `connectionString` values committed here are the masked
  placeholders Synapse Studio exports; the real values arrive at deploy time.
- `PearAzureSqlDatabase` uses SQL authentication to demonstrate a secret being
  overridden per environment. Prefer a system-assigned managed identity
  (`"authenticationType": "SystemAssignedManagedIdentity"`) where the target
  supports it — then there is no secret to manage at all.
- `DeleteArtifactsNotInTemplate` is `false`, so artifacts removed from the repo
  are left in place in the target workspace. Switch it on once the repo is the
  only way anyone changes the workspace.

## Licence

MIT — see [LICENSE](LICENSE).
