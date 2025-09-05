# Azure

The [Microsoft Cloud Adoption Framework for Azure](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/overview) provides a good overview of Azure from the point fo view of setting up a cloud environment. Specifically, the [Ready (Prepare for Cloud adoption)](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/) section explains the following concepts:

- [Tenant](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/azure-ad-define): The top-level way to manage identities and access in [Microsoft Entra](https://learn.microsoft.com/en-us/entra/fundamentals/what-is-entra).
- [Account](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-setup-guide/identity): A secure and well-managed identity in [Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/fundamentals/what-is-entra). Entra ID is a product of Entra and it provides identity and access management in Azure.
- [Subscription](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/resource-org-subscriptions): A unit of management, billing, and scale within Azure.
- [Landing zone](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/): An environment for hosting your Azure resources

## Azure CLI

### Configuration

In your home folder, `~/.azure/azureProfile.json`, you can see `tenant`, `user`, and `subscription` information.

### Commands

To start an Azure CLI working session, log in to the Azure CLI using:

```bash
az login
```

You will typically be asked to select a subscription.

## Setting up an Azure Container (Web) App using Terraform

### Initial Configuration

Using the Azure console (or CLI), create a resource group (called `web-flask-aca-rg`), a storage account (called `lalver1azurelearning`) in the resource group, and a blob container (called `tfstate`) in the storage account. This is where Terraform's state file will be saved.

!!! note "Terraform state"
    Terraform uses a state file (`terraform.tfstate`) to keep track of the infrastructure it manages. It records resources you’ve created, their IDs, properties, and relationships.
    
    When you run `terraform plan` or `terraform apply`, Terraform compares your config (`.tf` files) with the current state to determine what needs to be added, changed, or destroyed. Without the state, Terraform wouldn’t know what’s already deployed.

    By default, Terraform keeps state locally (a `JSON` file in your project directory). This works if you’re the only one managing the infrastructure. But for real projects, using a local state is risky. It can cause collaboration issues where multiple engineers could overwrite each other’s state. If the state file is lost, Terraform loses track of your resources. In addition, it can cause security issues since the state file may contain secrets.

    Keeping the state file in Azure Blob storage is ideal since all team members can use the same state. It's safe, since the state is persisted in Azure, not just locally. You can take advanrtage of versioning, so with blob versioning enabled, you can roll back to an older state if needed.

### Deploying resources

To deploy the resources that make up the web app:

- Log in to the Azure CLI.

- From the folder containing the `.tf` file, initialize Terraform using:

    ```bash
    terraform init
    ```

    Terraform downloads the AzureRM provider (hashicorp/azurerm) and connects to your existing storage account backend (`lalver1azurelearning`) and checks the state container (`tfstate`).

- Import the resource group using:

    ```bash
    terraform import azurerm_resource_group.rg /subscriptions/<sub_id>/resourceGroups/web-flask-aca-rg
    ```

    This tells Terraform that the resource already exists, so add it to state instead of creating it. Terraform then can start managing it.

- Preview the deployment (plan) by using

    ```bash
    terraform plan -out=tfplan
    ```

    This shows you exactly what resources Terraform intends to create/modify.

- Apply the plan

    ```bash
    terraform apply tfplan
    ```

    This actually provisions the resources in Azure.

### Version control

After running `terraform init` ensure that you set up your version control for:

| File | Description | Version control? |
| -----|-------------|------------------|
| `.terraform.lock.hcl` | A dependency lock file. Records the exact provider versions (e.g. hashicorp/azurerm v3.117.1) and their checksums. Ensures reproducible deployments — so when you or a teammate run terraform init again, you all get the same provider version. | Yes |
| `terraform.tfstate` | The local copy of Terraform state. By default, Terraform creates it locally. | No |
| `LICENSE.txt` | License file that comes bundled with the Terraform provider plugin (terraform-provider-azurerm). | No |
| `terraform-provider-azurerm_v3.117.1_x5` | The actual compiled provider binary that Terraform downloaded. Platform-specific (that _x5 suffix refers to your OS/arch build). Lives in the working dir so Terraform can run. Auto-managed by terraform init. | No |

