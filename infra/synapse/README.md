# Synapse infrastructure

Use this folder for the Azure infrastructure that hosts the workspace this repo
deploys into: the Synapse workspace itself, its dedicated SQL pool, Spark pool,
storage account and networking.

Recommended layout:

```text
infra/synapse/
  main.bicep
  parameters/
    dev.parameters.json
    prod.parameters.json
```

Keep the split clear: this folder creates the workspace, the pipeline under
`.azure/` fills it with artifacts. Keep secrets out of source control and inject
them from Azure DevOps variable groups or Azure Key Vault.
