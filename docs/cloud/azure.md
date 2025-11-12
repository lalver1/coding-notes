# Azure

The [Microsoft Cloud Adoption Framework for Azure](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/overview) provides a good overview of Azure from the point of view of setting up a cloud environment. Specifically, the [Ready (Prepare for Cloud adoption)](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/) section explains the following concepts:

- [Tenant](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/azure-ad-define): The top-level way to manage identities and access in [Microsoft Entra](https://learn.microsoft.com/en-us/entra/fundamentals/what-is-entra).
- [Account](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-setup-guide/identity): A secure and well-managed identity in [Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/fundamentals/what-is-entra). Entra ID is a product of Entra and it provides identity and access management in Azure.
- [Subscription](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/resource-org-subscriptions): A unit of management, billing, and scale within Azure.
- [Landing zone](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/): An environment for hosting your Azure resources

In addition, the [naming convention section](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming) provides useful information on how to name resources.

## Azure CLI

### Configuration

In your home folder, `~/.azure/azureProfile.json`, you can see `tenant`, `user`, and `subscription` information.

### Commands

To start an Azure CLI working session, log in to the Azure CLI using:

```bash
az login
```

You will typically be asked to select a subscription.

## Set up an Azure Container App using Terraform

Terraform is an infrastructure as code (IaC) tool that lets you build, change, and version infrastructure. At a high level, it 
consists of `provider`s to establish the connection to a cloud provider's APIs, `resource`s that define the components
of the infrastructure you want to manage, and `data` sources that allow you to query information about existing
infrastructure or external data for use in Terraform configurations.

For this guide we will use the following Terraform configuration:

```tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
  backend "azurerm" {
        resource_group_name  = "rg-aca-web"
        storage_account_name = "salalver1al"
        container_name       = "tfstate"
        key                  = "terraform.tfstate"
      }
}

provider "azurerm" {
  features {}
}

# Get information about the current Azure client (logged-in identity)
data "azurerm_client_config" "current" {}

# 1. Resource group
# Initially created manually to hold the tfstate storage account
# Import once before the first terraform apply
resource "azurerm_resource_group" "main" {
  name     = "rg-aca-web"
  location = "West US"
}

# 2. Storage Account
# Initially created manually to hold the tfstate file
# Import once before the first terraform apply
resource "azurerm_storage_account" "main" {
  name                          = "salalver1al"
  resource_group_name           = azurerm_resource_group.main.name
  location                      = azurerm_resource_group.main.location
  account_tier                  = "Standard"
  account_replication_type      = "LRS"
  public_network_access_enabled = true

  blob_properties {
    last_access_time_enabled = true
    versioning_enabled       = true

    container_delete_retention_policy {
      days = 7
    }

    delete_retention_policy {
      days = 7
    }
  }

  network_rules {
    default_action = "Deny"
    bypass         = ["AzureServices"]
  }
}

# 3. Container App Environment
resource "azurerm_container_app_environment" "main" {
  name                = "web-aca-env"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
}

# 4. Container App - Web
resource "azurerm_container_app" "web" {
  name                         = "aca-web"
  container_app_environment_id = azurerm_container_app_environment.main.id
  resource_group_name          = azurerm_resource_group.main.name
  revision_mode                = "Single"

  ingress {
    external_enabled = true
    target_port      = 50505 # match the port the Flask app listens on
    traffic_weight {
      latest_revision = true
      percentage=100
      }
    transport        = "auto"
  }

  template {
    container {
      name   = "web"
      image  = "ghcr.io/lalver1/azure-learning:${var.container_tag}"
      cpu    = 0.5
      memory = "1.0Gi"
    
    env {
        name  = "APPLICATIONINSIGHTS_CONNECTION_STRING"
        value = azurerm_application_insights.main.connection_string
      }
    }
  }
}

# 5. Log Analytics Workspace (needed for App Insights)
resource "azurerm_log_analytics_workspace" "main" {
  name                = "law-web"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

# 6. Application Insights
resource "azurerm_application_insights" "main" {
  name                = "appi-web"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  application_type    = "web"
  workspace_id        = azurerm_log_analytics_workspace.main.id
}

# 7. Key Vault
resource "azurerm_key_vault" "main" {
  name                        = "kv-al"
  location                    = azurerm_resource_group.main.location
  resource_group_name         = azurerm_resource_group.main.name
  tenant_id                   = data.azurerm_client_config.current.tenant_id
  sku_name                    = "standard"

  access_policy {
    tenant_id = data.azurerm_client_config.current.tenant_id
    object_id = data.azurerm_client_config.current.object_id

    secret_permissions = [
      "Get",
      "List",
      "Set",
      "Delete",
      "Purge",
      "Recover",
      "Backup",
      "Restore"
    ]

    key_permissions = [
    "Get", "List", "Create", "Delete", "Purge", "Recover", "Backup", "Restore"
  ]
  }
}

# 8. Key Vault Secret
data "azurerm_key_vault_secret" "slack_webhook_url" {
  name         = "slack-webhook-url"
  key_vault_id = azurerm_key_vault.main.id
}

# 9. A secure, random key for the Azure Functions' webhook URL
resource "random_string" "function_key" {
  length  = 32
  special = false
}

# 10. Container App - Azure Functions
resource "azurerm_container_app" "funcs" {
  name                         = "aca-funcs"
  container_app_environment_id = azurerm_container_app_environment.main.id
  resource_group_name          = azurerm_resource_group.main.name
  revision_mode                = "Single"

  # Securely store secrets that will be injected as environment variables
  secret {
    name  = "slack-webhook-url-secret"
    value = data.azurerm_key_vault_secret.slack_webhook_url.value
  }
  secret {
    name  = "function-key-secret"
    value = random_string.function_key.result
  }
  secret {
    name  = "appinsights-api-key-secret"
    value = azurerm_application_insights_api_key.main.api_key
  }

  template {
    container {
      name   = "functions"
      image  = "ghcr.io/lalver1/azure-functions:${var.container_tag}"
      cpu    = 0.25
      memory = "0.5Gi"

      env {
        name  = "APPLICATIONINSIGHTS_CONNECTION_STRING"
        value = azurerm_application_insights.main.connection_string
      }
      env {
        name  = "AzureWebJobsStorage"
        value = azurerm_storage_account.main.primary_connection_string
      }
      env {
        name        = "SLACK_WEBHOOK_URL"
        secret_name = "slack-webhook-url-secret"
      }
      env {
        name        = "AZURE_FUNCTION_KEY"
        secret_name = "function-key-secret"
      }
      env {
        name        = "APPINSIGHTS_API_KEY"
        secret_name = "appinsights-api-key-secret"
      }
    }
    
    min_replicas = 1
    max_replicas = 1
  }

  ingress {
    external_enabled = true
    target_port      = 80
    transport        = "auto"

    traffic_weight {
      percentage      = 100
      latest_revision = true
    }
  }

  identity {
  type = "SystemAssigned"
}
}

# 11. Action Group that posts to the Functions App
resource "azurerm_monitor_action_group" "main" {
  name                = "ag-funcapp-webhook"
  resource_group_name = azurerm_resource_group.main.name
  short_name          = "funcag"

  webhook_receiver {
    name        = "funcapp-webhook"
    service_uri = "https://${azurerm_container_app.funcs.ingress[0].fqdn}/api/alert_to_slack?CODE=${random_string.function_key.result}"
  }
}

# 12. Log Search alert rule
resource "azurerm_monitor_scheduled_query_rules_alert_v2" "main" {
  name                = "qr-error"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  scopes              = [azurerm_application_insights.main.id]

  description  = "Alerts when any exception is logged in Application Insights."
  display_name = "Web Flask App Error Alert"
  enabled      = true
  severity     = 2

  evaluation_frequency = "PT5M"
  window_duration      = "PT5M"

  criteria {
    query     = <<-QUERY
      union (exceptions | where type !has "ServiceResponseError"), (traces | where severityLevel >= 3)
    QUERY
    operator  = "GreaterThan"
    threshold = 0
    time_aggregation_method = "Count"

    failing_periods {
      minimum_failing_periods_to_trigger_alert = 1
      number_of_evaluation_periods             = 1
    }
  }

  action {
    action_groups = [azurerm_monitor_action_group.main.id]
    custom_properties = {
      subject = "🚨 Application Error"
    }
  }

  lifecycle {
    ignore_changes = [tags]
  }
}

# 13. Create an API key for querying Application Insights data
resource "azurerm_application_insights_api_key" "main" {
  name                    = "api-query-key-funcs"
  application_insights_id = azurerm_application_insights.main.id

  read_permissions = [
    "search",
  ]
}
```

### Initial Configuration

Using the Azure console (or CLI), create a resource group (called `rg-aca-web`), a storage account (called `salalver1al` with
`Public network access` set to `Enable` and `Public network access scope` set to `Enable from selected networks`) in the resource group, and a blob container (called `tfstate`) in the storage account. This is where Terraform's state file will be saved. Make sure to add your
current IP address to the list of allowed IPs of the storage account.

!!! note "Terraform state"
    Terraform uses a state file (`terraform.tfstate`) to keep track of the infrastructure it manages. It records resources you’ve created, their IDs, properties, and relationships.
    
    When you run `terraform plan` or `terraform apply`, Terraform compares your config (`.tf` files) with the current state to determine what needs to be added, changed, or destroyed. Without the state, Terraform wouldn’t know what’s already deployed.

    By default, Terraform keeps state locally (a `JSON` file in your project directory). This works if you’re the only one managing the infrastructure. But for real projects, using a local state is risky. It can cause collaboration issues where multiple engineers could overwrite each other’s state. If the state file is lost, Terraform loses track of your resources. In addition, it can cause security issues since the state file may contain secrets.

    Keeping the state file in Azure Blob storage is ideal since all team members can use the same state. It's safe, since the state is persisted in Azure, not just locally. You can take advanrtage of versioning, so with blob versioning enabled, you can roll back to an older state if needed.

#### Variables

Variables defined with the `variable` keyword in a `variables.tf` file without a default value defined are required to be passed to `terraform` via the `-var` option. Note that terraform state tracks resource values, and variables track user input. Terraform will not “reuse” your last variable input. That’s why you must supply it every run unless you save it in:

- a `.tfvars` file
- environment variable
- default value in `variables.tf`

### Deploying resources

To deploy the resources that make up the web app:

- Log in to the Azure CLI.

- From the folder containing the `.tf` file, initialize Terraform using:

    ```bash
    terraform init
    ```

    Terraform downloads the AzureRM provider (`hashicorp/azurerm`) and connects to your existing storage account backend (`lalver1azurelearning`) and checks the state container (`tfstate`). It also creates a version lock file
    if one doesn't already exist. Note that if you have made changes to the terraform configuration (such as changing
    the version of the `azurerm` provider), `terraform init` will **not** make any changes to the state file, changes
    are only made to the state file when you run `apply`. 

- Import the resource group and storage account using:

    ```bash
    terraform import azurerm_resource_group.main /subscriptions/<sub_id>/resourceGroups/rg-aca-web
    ```

    ```bash
    terraform import azurerm_storage_account.main /subscriptions/<sub_id>/resourceGroups/rg-aca-web/providers/Microsoft.Storage/storageAccounts/salalver1al
    ```

    This tells Terraform that the resources already exist, so add them to state instead of creating them. Terraform then can start managing them.

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
| `.terraform.lock.hcl` | A dependency lock file. Records the exact provider versions (e.g. hashicorp/azurerm v3.117.1) and their checksums. Ensures reproducible deployments — so when you or a teammate run terraform init again, you all get the same provider version. | Yes (or store in Azure Blob Storage) |
| `terraform.tfstate` | The local copy of Terraform state. By default, Terraform creates it locally. | No |
| `LICENSE.txt` | License file that comes bundled with the Terraform provider plugin (terraform-provider-azurerm). | No |
| `terraform-provider-azurerm_v3.117.1_x5` | The actual compiled provider binary that Terraform downloaded. Platform-specific (that _x5 suffix refers to your OS/arch build). Lives in the working dir so Terraform can run. Auto-managed by terraform init. | No |

### Chicken/egg problem

In the [Initial Configuration](#initial-configuration) we solved the problem of referencing the Terraform state in code by manually creating the resource group and storage account before running a `plan` or `apply`.
Another situation where this comes up is when reading a secret. For example, the webhook URL is a secret whose value is not defined in code, only the resource. For these situations we follow this approach:

1. On the first run, clear out all other resources and define the key vault.

    ```
    # 7. Key Vault
    resource "azurerm_key_vault" "main" {
      name                        = "kv-al"
      location                    = azurerm_resource_group.main.location
      resource_group_name         = azurerm_resource_group.main.name
      tenant_id                   = data.azurerm_client_config.current.tenant_id
      sku_name                    = "standard"

      access_policy {
        tenant_id = data.azurerm_client_config.current.tenant_id
        object_id = data.azurerm_client_config.current.object_id

        secret_permissions = [
          "Get",
          "List",
          "Set",
          "Delete",
          "Purge",
          "Recover",
          "Backup",
          "Restore"
        ]

        key_permissions = [
        "Get", "List", "Create", "Delete", "Purge", "Recover", "Backup", "Restore"
      ]
      }
    }
    ```

    and run `terraform apply` to create the KV.

2. After the key vault exists, create the secret using the portal or CLI.

3. Once the secret has been created, add this to the terraform definition (make sure that
you use the exact same name for the secret as what you entered in the previous step)

    ```
    # Read the existing secret
    data "azurerm_key_vault_secret" "slack_webhook_url" {
      name         = "slack-webhook-url"
      key_vault_id = azurerm_key_vault.main.id
    }
    ```

    and anywhere that you need to use the secret, use `data.azurerm_key_vault_secret.slack_webhook_url.value`

3. Run Terraform again. Now, when you run `terraform plan` or `terraform apply`, Terraform will reads the secret from the Key Vault 

This is a nice approach because Terraform knows how to use the secret without exposing it in the Terraform state or `.tfvars`. Once the infra CI/CD is running and you need to add a new secret, for example, you can

1. add the secret manually to the KV
2. modify the `.tf` file to define the new secret using `data`
3. deploy the infra (merge the PR, for example; or redeploy manually)

## Updating infrastructure

To test out new infrastructure, sometimes it is handy to just add the new resources in the Azure Portal. Doing this **does not directly impact your Terraform state file.** The state file is a record of what **Terraform believes it manages**, not a live log of your Azure subscription. When you add resources manually, they are simply "unmanaged" and invisible to Terraform.

The interaction between the state file and the currently deployed infrastructure happens the next time you run `terraform plan`.

### Scenarios: IaC vs. manual resources

This ["out-of-band" change](https://www.hashicorp.com/en/resources/terraform-config-drift-how-to-handle-out-of-band-infrastructure-changes) (working outside Terraform) creates three common scenarios.

#### Scenario 1: "Unmanaged" resource

You create a new, separate resource in the portal. For example, your Terraform manages `Resource-A`, and you manually create `Resource-B`.

  * **Interaction:** When you run `terraform plan`, Terraform reads its state file, sees it's supposed to manage `Resource-A`, and checks that `Resource-A` matches the code. It **does not know or care** that `Resource-B` exists.
  * **Result:** The plan will show **"No changes."** Your manually-created resource is safe and completely ignored by Terraform.

#### Scenario 2: "Configuration drift"

You go into the portal and **modify a resource that Terraform *does* manage.** For example, your code defines a database SKU as "Standard", and you manually change it to "Premium" in the portal.

  * **Interaction:** When you run `terraform plan`, Terraform reads its state (which says "Standard") and compares it to the *actual* resource in Azure (which is now "Premium"). Terraform identifies this difference.
  * **Result:** The plan will show one resource "to be updated." If you run `terraform apply`, Terraform will **"fix" the drift** by changing the SKU *back to "Standard"*, wiping out your manual change since Terraform's code is always the source of truth.

#### Scenario 3: "Name collision"

You manually create a resource, (e.g., a storage account named `myteststorage`). Later, you (or a teammate) add a *new* resource to your Terraform code with that *exact same name*.

  * **Interaction:** You run `terraform plan`. The plan will show one resource "to be created."
  * **Result:** When you run `terraform apply`, it will **fail**. Azure's API will return a `ResourceAlreadyExists` error. Terraform will not automatically "take over" the resource; it only knows how to create one, and it can't. This scenario is the primary reason for Terraform's "Import" workflow, similar to the storage account creation step described in the [Deploying Resources](#deploying-resources) section.

### Workflow: Deleting your manual resources

Ater you're done testing, you can start the "cleanup" phase.

1.  **Identify the Resources:** You must manually track what you created. The easiest way is to put all your test resources into a single, separate **Resource Group** (e.g., `rg-my-temp-tests`).
2.  **Delete the Resources:**
      * **If you used a separate resource group:** Simply delete that one resource group in the Azure Portal. Everything inside it will be deleted. This is the cleanest method.
      * **If you scattered resources:** You must go to the portal and delete each resource one by one.
3.  **Run `terraform plan`:** After deleting, run `plan` again. It should still show **"No changes,"** proving that your cleanup didn't affect your Terraform-managed infrastructure.

### Workflow: Keep importing your manual resources

You created a resource, liked it, and now want Terraform to manage it.

#### Step 1: Write the Terraform code

Go to your `.tf` files and write a new `resource` block that **exactly matches** the resource you created in the portal. You must match the `name`, `location`, `sku`, and all other relevant arguments.

```hcl
# In, for example, your storage.tf file
resource "azurerm_storage_account" "my_new_sa" {
  name                     = "myteststorage" # Must match portal
  resource_group_name      = "rg-aca-web"
  location                 = "West US"
  account_tier             = "Standard"    # Must match portal
  account_replication_type = "LRS"       # Must match portal
  # ... and so on
}
```

#### Step 2: Find the resource ID

Go to the Azure Portal, find your resource, and copy its full **resource ID**. It will look like this:
`/subscriptions/your-sub-id/resourceGroups/rg-aca-web/providers/Microsoft.Storage/storageAccounts/myteststorage`

#### Step 3: Run `terraform import`

In your terminal, run the `import` command using your **Terraform address** (from Step 1) and the **Azure ID** (from Step 2).

```sh
terraform import azurerm_storage_account.my_new_sa /subscriptions/your-sub-id/resourceGroups/rg-aca-web/providers/Microsoft.Storage/storageAccounts/myteststorage
```

Terraform will connect to Azure, read the resource, and write it into your `terraform.tfstate` file.

#### Step 4: Verify with `terraform plan`

This is the most important step. Run `terraform plan`.

  * **If it shows "No changes."**: You are done. Your code from Step 1 exactly matched the imported resource. It is now fully managed.
  * **If it shows "Changes to be applied."**: This is very common. It means your code from Step 1 *doesn't* exactly match. For example, the plan might want to change `account_tier` from "Standard" to "Premium". **Do not run `apply`**. Instead, go back to your `.tf` file and *fix your code* to match what was imported (e.g., change the tier in your code to "Standard").
  * Run `terraform plan` again. Repeat until it says "No changes."

### Monitoring a Container App

Azure Monitor collects and aggregates the data from every layer and component of your system across multiple Azure and non-Azure subscriptions and tenants. It stores it in a common data platform for consumption by a common set of tools that can correlate, analyze, visualize, and/or respond to the data.

We will focus on the following setup:

- Azure Monitor
  - Opentelemetry to instrument the web app
    - details
  - Application Insights