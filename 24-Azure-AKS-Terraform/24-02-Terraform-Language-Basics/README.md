# Understand Terraform Language Basics

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Terraform Language Syntax"
        Syntax[Terraform Configuration Language HCL]
        Syntax --> Blocks[Blocks<br/>resource, variable, output, etc.]
        Syntax --> Arguments[Arguments<br/>name = value pairs]
        Syntax --> Expressions[Expressions<br/>Variables, functions, conditionals]
        Syntax --> Comments[Comments<br/># Single line<br/>/* Multi-line */]
    end
    
    subgraph "Terraform Settings Block"
        Settings[terraform Block]
        Settings --> TFVersion[required_version<br/>>= 0.13<br/>Terraform version constraint]
        Settings --> Providers[required_providers<br/>Define provider sources & versions]
        Settings --> Backend[backend Configuration<br/>Remote state storage]
        
        Providers --> AzureRM[azurerm Provider<br/>source: hashicorp/azurerm<br/>version: ~> 2.0]
        Providers --> AzureAD[azuread Provider<br/>source: hashicorp/azuread<br/>version: ~> 1.0]
        Providers --> Random[random Provider<br/>source: hashicorp/random<br/>version: ~> 3.0]
    end
    
    subgraph "Input Variables"
        Variables[variables.tf]
        Variables --> VarDef[Variable Definition<br/>variable "region" type = string]
        VarDef --> VarDefault[default = "Central US"<br/>Optional default value]
        VarDef --> VarDesc[description = "Azure Region"<br/>Documentation]
        
        VarUsage[Variable Usage in Config]
        VarUsage --> RefVar[Reference: var.region<br/>Use in resource blocks]
    end
    
    subgraph "Output Values"
        Outputs[outputs.tf]
        Outputs --> OutputDef[Output Definition<br/>output "resource_group_id"]
        OutputDef --> OutputValue[value = azurerm_resource_group.rg.id<br/>Resource attribute reference]
        OutputDef --> OutputDesc[description = "RG ID"<br/>Documentation]
        
        OutputUsage[Output Usage]
        OutputUsage --> TerminalOutput[terraform output<br/>Display values in terminal]
        OutputUsage --> ModuleOutput[Module outputs<br/>Pass to other modules]
    end
    
    subgraph "Local State vs Remote State"
        LocalState[Local State]
        LocalState --> LocalFile[terraform.tfstate<br/>Stored on local disk<br/>❌ No collaboration<br/>❌ No locking]
        
        RemoteState[Remote State: Azure Storage]
        RemoteState --> StorageAccount[Azure Storage Account<br/>terraformstatexlrwdrzs]
        StorageAccount --> Container[Blob Container<br/>prodtfstate]
        Container --> StateBlob[terraform.tfstate<br/>✓ Team collaboration<br/>✓ State locking<br/>✓ Encryption]
    end
    
    subgraph "State Migration Process"
        Migration[Migrate Local → Remote]
        Migration --> Step1[1. Create Azure Storage<br/>Resource Group + Storage Account]
        Step1 --> Step2[2. Configure backend in terraform block<br/>Add azurerm backend config]
        Step2 --> Step3[3. terraform init -migrate-state<br/>Copy local state to remote]
        Step3 --> Step4[4. Verify state in Azure Portal<br/>Check blob storage]
    end
    
    subgraph "Resource Blocks Example"
        ResourceBlock[Resource Block Structure]
        ResourceBlock --> BlockType[Block Type: resource]
        ResourceBlock --> BlockLabel1[Block Label 1: "azurerm_resource_group"]
        ResourceBlock --> BlockLabel2[Block Label 2: "aksdev"]
        ResourceBlock --> BlockBody[Block Body:<br/>name = "aks-rg2-tf"<br/>location = var.region<br/>tags = ]
    end
    
    subgraph "Terraform Workflow with Variables"
        Workflow[Development Workflow]
        Workflow --> W1[1. Define variables.tf<br/>Input parameters]
        Workflow --> W2[2. Define main.tf<br/>Use var.* references]
        Workflow --> W3[3. Define outputs.tf<br/>Export values]
        Workflow --> W4[4. terraform plan<br/>Preview with variable values]
        Workflow --> W5[5. terraform apply<br/>Create resources]
        Workflow --> W6[6. terraform output<br/>View exported values]
    end
    
    VarDef --> RefVar
    OutputDef --> TerminalOutput
    Backend --> RemoteState
    
    style Settings fill:#5c4ee5
    style Variables fill:#326ce5
    style Outputs fill:#00bcf2
    style RemoteState fill:#28a745
    style Migration fill:#ff8c00
    style ResourceBlock fill:#9370db
    style StateBlob fill:#ffd700
```

### Understanding the Diagram

- **Terraform Language HCL**: Terraform uses **HashiCorp Configuration Language** (HCL) with syntax consisting of **blocks** (resource, variable), **arguments** (name=value), **expressions** (var.region), and **comments**
- **Terraform Settings Block**: The **terraform** block defines **required Terraform version**, **provider sources** (azurerm, azuread, random), and **backend configuration** for state storage
- **Provider Configuration**: Specify provider **source** (hashicorp/azurerm) and **version constraints** (~> 2.0 means >= 2.0 and < 3.0), ensuring consistent provider versions across team
- **Input Variables**: Define in **variables.tf** with **type**, **default value**, and **description**, then reference in configs using **var.variable_name** syntax for parameterization
- **Output Values**: Define in **outputs.tf** to expose **resource attributes** (IDs, IPs, names) after apply, viewable via **terraform output** command or consumed by other modules
- **Local vs Remote State**: **Local state** (terraform.tfstate on disk) lacks collaboration features; **remote state** in Azure Storage enables **team collaboration**, **state locking**, and **encryption**
- **State Migration**: Move from local to remote state by creating **Azure Storage Account**, configuring **backend block**, running **terraform init -migrate-state**, and verifying state uploaded to blob
- **Resource Block Structure**: Consists of **block type** (resource), **two labels** (provider_type and name), and **block body** with arguments defining resource configuration
- **Variable Benefits**: Variables enable **reusable configurations**, **environment-specific values** (dev/prod), **CI/CD integration**, and **reduced hardcoding** in Terraform files
- **State Locking**: Remote backend supports **state locking** preventing concurrent modifications, **critical for team environments** where multiple users run Terraform simultaneously

---

## Step-01: Introduction
- Understand Terraform language basics 
  - Understand Resources
  - Understand Blocks
  - Understand Arguments
  - Understand Identifiers
  - Understand Comments
- Understand Input Variables in Terraform
- Understand Output Values in Terraform
- Migrate Terraform State from local to Remote Storage (Azure Storage)

## Step-02: Terraform Configuration Language Syntax
- Understand Resources
- Understand Blocks
- Understand Arguments
- Understand Identifiers
- Understand Comments
- [Terraform Configuration](https://www.terraform.io/docs/configuration/index.html)
- [Terraform Configuration Syntax](https://www.terraform.io/docs/configuration/syntax.html)
```
# Template
<BLOCK TYPE> "<BLOCK LABEL>" "<BLOCK LABEL>"   {
  # Block body
  <IDENTIFIER> = <EXPRESSION> # Argument
}

# Example
resource "azurerm_resource_group" "aksdev" {   # BLOCK
  name     = "aks-rg2-tf" # Argument
  location = var.region   # Argument with value as expression (Variable value replaced from varibales.tf )

  tags = {  #BLOCK
    "environment" = "k8sdev"
  }
}
```

## Step-03: Define Terraform Providers
- [VS Code Terraform Plugin -  Syntax highlighting and autocompletion for Terraform](https://marketplace.visualstudio.com/items?itemName=HashiCorp.terraform)
- Understand about [Terraform Settings Block](https://www.terraform.io/docs/configuration/terraform.html)
- We primarily define the below 3 items in Terraform Settings Block
  - Terraform Version
  - Terraform Providers
    - [Azure RM Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest)
    - [Azure AD Provider](https://registry.terraform.io/providers/hashicorp/azuread/latest)
    - [Random Provider](https://registry.terraform.io/providers/hashicorp/random/latest)
    - Append with **/docs** for above 3 links to get their equivalent documentation
  - Terraform State Storage Backend
- Create a file **01-main.tf** and create terraform providers
```
# Terraform Settings Block (https://www.terraform.io/docs/configuration/terraform.html)
terraform {
  # Use a recent version of Terraform
  required_version = ">= 0.13"

  # Map providers to thier sources, required in Terraform 13+
  required_providers {
    # Azure Resource Manager 2.x (Base Azure RM Module)
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 2.0"
    }
    # Azure Active Directory 1.x (required for AKS and Azure AD Integration)
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 1.0"
    }
    # Random 3.x (Required to generate random names for Log Analytics Workspace)
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
# Configure Terraform State Storage
    backend "azurerm" {
    resource_group_name   = "terraform-storage-rg"
    storage_account_name  = "terraformstatexlrwdrzs"
    container_name        = "prodtfstate"
    key                   = "terraform.tfstate"
  }
}

```

## Step-04: Define Azure Resource Manager Features Block
- Add **azurerm features block** to **01-main.tf**
```
# 2. Terraform Provider Block for AzureRM
provider "azurerm" {
  subscription_id = "XXXXXXXX"
  features {
  }
}
```



## Step-05: Create Random Pet Resource
- Create Random pet resource in **01-main.tf**
```
# Create Random pet resource
resource "random_pet" "aksrandom" {}
```
- Initialize the terraform and understand what happened
```
# Terraform Initialize
terraform init
```
## Step-06: Create a Resource Group Resource
- Create a file named **03-resource-group.tf** 
```
resource "azurerm_resource_group" "aks_rg" {
  location = "Central US"
  name     = "terraform-aks"
}
```

## Step-07: Understand and Create Terraform Input Variables
### Three types of Terraform Variables
- [Input Variables](https://www.terraform.io/docs/configuration/variables.html)
- [Output Values](https://www.terraform.io/docs/configuration/outputs.html)
- [Local Values](https://www.terraform.io/docs/configuration/locals.html)

### Input Variables
- We can parameterize our deployments using Terraform Input Variables.
- This is the right way to build a Terraform project that can be reused to deploy multiple environments like dev, qa, staging and production
- Implement input variables in terraform for AKS Cluster
- Understand different options available to pass input variables
  - variables.tf
  - arguments during runtime (-var)
  - arguments during runtime (-var-file terraform.tfvars) with a file containing variables
#### Using 02-variables.tf
- Define few input variables in `02-variables.tf`
- Reference them in `03-resource-group.tf`
- Variables we are going to play with
  - Resource Group Name
  - Location or Region
  - AKS Environment Name

### Create Variables in a file and Reference them
- Create a variables file **02-variables.tf**
```
# https://www.terraform.io/docs/configuration/variables.html
# Input Variables
# Output Values
# Local Values

# Define Input Variables
# 1. Azure Location (CentralUS)
# 2. Azure Resource Group Name 
# 3. Azure AKS Environment Name (Dev, QA, Prod)
# 4. Azure AKS Cluster Name

# Azure Location
variable "location" {
  type = string
  description = "Azure Region where all these resources will be provisioned"
  default = "Central US"
}

# Azure Resource Group Name
variable "resource_group_name" {
  type = string
  description = "This variable defines the Resource Group"
  default = "terraform-aks"
}

# Azure AKS Environment Name
variable "environment" {
  type = string  
  description = "This variable defines the Environment"  
  default = "prod"
}
```
- Update a file named **03-resource-group.tf** with Variable references
```
# Terraform Resource to Create Azure Resource Group with Input Variables defined in variables.tf
resource "azurerm_resource_group" "aks_rg" {
  location = var.location
  name     = var.resource_group_name
}
```

## Step-08: Terraform Input Variables - Multiple Options
### Option-1: With -var
- `-var` flag enables a single input variable value to be passed in at the command-line per each `-var`.
```
# Execute Terraform plan with -var and observe
terraform plan -var "location=eastus"
Observation: Value to should be picked from runtime -var what ever provided which is eastus.
```

### Option-2: With -var-file
- `-var-file` flag enables multiple input variable values to be passed in by referencing a file that contains the values.
- Create file `terraform.tfvars`
```
terraform.tfvars with content in it as 
location = "westus"

# Run plan and observe 
terraform plan 
Observation: No need to give -var-file when name terraform.tfvars and values pickup directly from terraform.tfvars
```
- Rename file name `terraform.tfvars` to `dev.tfvars`
```
# Run plan and observe 
terraform plan 
Observation: dev.tfvars will not be picked and value comes from variables.tf default attribute

# Run plan with -var-file 'dev.tfvars'
terraform plan -var-file 'dev.tfvars'
Observation: value to should be picked from dev.tfvars now
```

### Option-3: With filename.auto.tfvars
- We already seen the auto loading of `.tfvars` when the file name is `terraform.tfvars`
- Now we can also do that when we have the file names with `filname.auto.tfvars`
- Lets try out
```
# Rename file 
mv dev.tfvars dev.auto.tfvars

# Execute Terraform plan (No runtime arguments like -var-file)
terraform plan
Observation: You can see that variable present in dev.auto.tfvars autoloaded now. 
```

- **Clean-Up:** Move `dev.auto.tfvars` to backup folder for reference and move back to original `02-variables.tf` and move to next step.


### Option-4: With Environment Variables
- When running Terraform commands, we can also use Environment Variables to define the values for Input Variables 
- **Important Note:** Be sure to keep in mind that if the Operating System is case-sensitive, then Terraform will match variable names exactly as given during configuration. 
```
# Template
TF_VAR_<VARIABLE-NAME>  - case-sensitive
# Set Environment Variable using Bash
export TF_VAR_location="westus"
```

## Step-09: Final Look of Resource Group
- Combine Two variables to make it one for resource group
- Two have 1 resource group per environment (dev, qa), we are going to use this approach
```
resource "azurerm_resource_group" "aks_rg" {
  location = var.location
  name     = "${var.resource_group_name}-${var.environment}"
}
```

## Step-10: Define Output Values
- Understand about [Terraform Output Values](https://www.terraform.io/docs/configuration/outputs.html)
- Output values are like the return values of a Terraform module
- Output values are a way to expose some of that information to the user of your module.
- A child module can use outputs to expose a subset of its resource attributes to a parent module
- A root module can use outputs to print certain values in the CLI output after running terraform apply
```
# Create Outputs
# 1. Resource Group Location
# 2. Resource Group Id
# 3. Resource Group Name

output "location" {
  value = azurerm_resource_group.aks_rg.location
}

output "resource_group_id" {
  value = azurerm_resource_group.aks_rg.id
}

output "resource_group_name" {
  value = azurerm_resource_group.aks_rg.name
}
```


## Step-11: Create or Deploy Terraform Resources & Verify
```
# Initialize Terraform 
terraform init

# Validate Terraform Templates
terraform validate

# Execute Terraform Plan
terraform plan
terraform plan -out v1out.plan

# Create / Deploy Terraform Resources
terrafrom apply 
terraform apply v1out.plan 

# Verify current infrastructure state
terraform show

# Format Terraform files
terraform fmt
```

## Step-12: Verify the same in Azure Portal Mgmt Console
- Verify if Resource Group got created in Azure Mgmt Console
- Understand about terraform state file named **terraform.tfstate**
- Migrate this state file to Azure Storage Account

## Step-13: Migrate Terraform State Storage to Azure Storage Account

### Create Azure Storage Account in new Resource Group
- Why should be we create terraform state storage in different resource group? 
  - State storage is key for all terraform resources and it should be deleted at any point of time even accidentally.
- **Create New Resource Group:** terraform-storage-rg
- **Create Storage Account:** terraformstatexlrwdrzs  (Note: Name should be unique across Azure)
- **Create Container Name:** tfstatefiles
- Upload the file **terraform.tfstate** to storage account container

### Update main.tf with Terraform State Storage
```
# Configure Terraform State Storage
terraform {
  backend "azurerm" {
    resource_group_name   = "terraform-storage-rg"
    storage_account_name  = "terraformstatexlrwdrzs"
    container_name        = "tfstatefiles"
    key                   = "terraform.tfstate"
  }
}
```

### Migrate terraform backend by re-initializing
- First backup local terraform.tfstate
```
# Backup existing terraform.tfstate present locally
mkdir BACKUP-LOCAL-TFSTATE
mv terraform.tfstate BACKUP-LOCAL-TFSTATE

# Try terraform validate
terraform validate

# Try terraform plan (Should fail telling us to re-initialize backed)
terraform plan

# Re-Initialize Terraform Backend
terraform init 

# Verify if any local state file
ls -lrta
```
- This completes successful migration of **terraform.tfstate** from local to Azure Storage Container
- No local dependency now. Straight away initialize your terraform files from any folder and start working 

## References
- [Terraform Syntax](https://www.terraform.io/docs/configuration/syntax.html)
- [Terraform Azure Get Started](https://learn.hashicorp.com/collections/terraform/azure-get-started)
- [Store Terraform state in Azure Storage](https://docs.microsoft.com/en-us/azure/developer/terraform/store-state-in-azure-storage)



