## Terraform project for deploying Oracle Autonomous Database on Azure with an Azure DevOps pipeline.

## Credit

This project is based on the Oracle reference template:

- https://github.com/oci-landing-zones/terraform-oci-multicloud-azure/tree/main/templates/azurerm-oci-adbs-quickstart

## What This Project Does

- Provisions Azure resources (VNet/subnet) and Oracle Autonomous Database on Azure.
- Uses Terraform with an `azurerm` remote backend for persisted state.
- Runs plan/apply (and optional destroy) through `pipeline.yaml`.
- Database settings are defined in [`dev.tfvars`](./dev.tfvars):
  - https://github.com/mikarinneoracle/db-azure/blob/main/dev.tfvars

## Naming Pattern Update

VNET and DB names were modified in [`main.az.tf`](./main.az.tf) to follow the same suffix pattern logic.

- VNET uses: `<base-name>000<random-suffix>`
- DB uses: `<base-name>000<random-suffix>`

For DB naming, `000` is used as the separator because DB name constraints do not allow a hyphen.

## Azure DevOps Pipeline

Pipeline file: [`pipeline.yaml`](./pipeline.yaml)

The pipeline runs these Terraform stages:

1. `TerraformInstaller@1` to install Terraform `1.9.2`
2. `TerraformTaskV4@4` `init` with Azure backend config
3. `TerraformTaskV4@4` `plan` and `apply` for `apply` mode
4. `TerraformTaskV4@4` `plan -destroy` and `apply` for `destroy` mode

The pipeline now has a runtime parameter:

- `operation` (default: `apply`)
- Allowed values:
  - `apply`: create/update ADB resources
  - `destroy`: delete ADB resources

When running manually from Azure DevOps (**Run pipeline**), users can select the `operation` value from a dropdown before execution.

## Prerequisites

Before running the pipeline, create the remote backend storage resources in Azure in advance.

- The Azure DevOps ARM service connection must already exist, and must match `AZURE_SERVICE_CONNECTION_NAME`.
- The backend Resource Group must already exist, and must match `AZURE_RM_GROUP_NAME`.
- The storage account used for backend state must already exist.
- The `tfstate` container must already exist in that storage account.

This is required because Terraform backend initialization (`terraform init`) happens before Terraform can create any infrastructure. In other words, Terraform cannot bootstrap its own backend resource group/storage/container during the same run.

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

Terraform backend is configured as `azurerm` in [`terraform.tf`](./terraform.tf).

During pipeline `init`, state is stored remotely in Azure Storage:

- Resource Group: `$(AZURE_RM_GROUP_NAME)`
- Storage Account: `$(AZURE_RM_GROUP_NAME)` (current pipeline mapping)
- Container (bucket-equivalent): `tfstate`
- State file key: `terraform.tfstate`

This keeps state persistent between runs and shared across pipeline executions.

## How `operation` Works

Use `operation=apply` for normal deployments and `operation=destroy` to remove managed resources.

Behavior by trigger type:

1. Manual run (`Build.Reason = Manual`):
   - User can choose `apply` or `destroy`.
   - `destroy` is allowed and runs:
     - `Terraform Plan (Destroy)`
     - `Terraform Destroy`
2. Auto-triggered run from `main`:
   - No runtime prompt is shown.
   - Pipeline uses default `operation=apply`.
   - If `operation=destroy` is somehow set for a non-manual run, guard step `Guard: Block destroy on non-manual runs` fails the job intentionally.
