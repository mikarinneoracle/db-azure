# db-azure

Terraform project for deploying Oracle Autonomous Database on Azure with an Azure DevOps pipeline.

## What This Project Does

- Provisions Azure resources (resource group, VNet/subnet) and Oracle Autonomous Database on Azure.
- Uses Terraform with an `azurerm` remote backend for persisted state.
- Runs plan/apply (and optional destroy) through `pipeline.yaml`.

## Azure DevOps Pipeline

Pipeline file: [`pipeline.yaml`](/Users/MRINNE/projects/db-azure/pipeline.yaml)

The pipeline runs these Terraform stages:

1. `TerraformInstaller@1` to install Terraform `1.9.2`
2. `TerraformTaskV4@4` `init` with Azure backend config
3. `TerraformTaskV4@4` `plan` using `dev.tfvars`
4. `TerraformTaskV4@4` `apply` using the saved `tfplan`

Optional (commented out by default):

5. `plan -destroy` to create `tfdestroyplan`
6. `apply tfdestroyplan` to destroy resources

## Required Pipeline Variables

Set these as Azure DevOps pipeline variables:

- `AZURE_SERVICE_CONNECTION_NAME`
  - ARM service connection name used by Terraform tasks for authentication.
  - Referenced in:
    - `backendServiceArm`
    - `environmentServiceNameAzureRM`
- `AZURE_RM_GROUP_NAME`
  - Azure Resource Group name used for Terraform backend state storage.
  - Referenced in:
    - `backendAzureRmResourceGroupName`
    - `backendAzureRmStorageAccountName` (as currently configured in this repo)

## Terraform State Persistence (Remote tfstate)

Terraform backend is configured as `azurerm` in [`terraform.tf`](/Users/MRINNE/projects/db-azure/terraform.tf).

During pipeline `init`, state is stored remotely in Azure Storage:

- Resource Group: `$(AZURE_RM_GROUP_NAME)`
- Storage Account: `$(AZURE_RM_GROUP_NAME)` (current pipeline mapping)
- Container (bucket-equivalent): `tfstate`
- State file key: `terraform.tfstate`

This keeps state persistent between runs and shared across pipeline executions.

## How Destroy Works

Destroy steps are intentionally commented in [`pipeline.yaml`](/Users/MRINNE/projects/db-azure/pipeline.yaml):

- `Create Destroy Plan`
- `Apply Destroy Plan`

To run Terraform destroy:

1. Uncomment both destroy tasks in `pipeline.yaml`.
2. Commit/push the change (or run pipeline from branch with those lines uncommented).
3. Run pipeline.

The pipeline will:

1. Create `tfdestroyplan` using `terraform plan -destroy`.
2. Apply `tfdestroyplan` to remove managed resources.

After destroy is complete, re-comment these steps to avoid accidental deletion in future runs.
