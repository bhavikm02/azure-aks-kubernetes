# Provision Azure AKS Cluster using Terraform

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Terraform Files Structure"
        Files[Terraform Configuration Files]
        Files --> Variables[01-variables.tf:<br/>SSH key path<br/>Windows credentials<br/>Environment, location]
        Files --> Datasource[02-datasource.tf:<br/>Get latest AKS version<br/>azurerm_kubernetes_service_versions]
        Files --> RG[03-resource-group.tf:<br/>Resource group for AKS]
        Files --> LogAnalytics[04-log-analytics.tf:<br/>Monitor workspace]
        Files --> AKSG[05-aks-aad-group.tf:<br/>Azure AD group for admins]
        Files --> AKSCluster[06-aks-cluster.tf:<br/>AKS cluster resource]
        Files --> Outputs[07-outputs.tf:<br/>Export cluster details]
    end
    
    subgraph "Data Sources"
        GetVersion[Data Source: AKS Versions]
        GetVersion --> QueryAzure[Query Azure for available K8s versions]
        QueryAzure --> LatestVersion[Get latest stable version<br/>latest_version attribute<br/>Exclude preview versions]
    end
    
    subgraph "Resource: Log Analytics Workspace"
        LAW[azurerm_log_analytics_workspace]
        LAW --> LAWName[Name: ${var.environment}-logs-${random_id}<br/>Dynamic naming]
        LAW --> LAWSKU[SKU: PerGB2018<br/>Pay per GB ingested]
        LAW --> LAWRetention[retention_in_days: 30<br/>Log retention period]
    end
    
    subgraph "Resource: Azure AD Group"
        AADGroup[azuread_group]
        AADGroup --> GroupName[Name: ${var.environment}-aks-admins<br/>Environment-specific]
        AADGroup --> GroupDesc[Description: AKS Admins<br/>For RBAC]
        AADGroup --> GroupMembers[Add AD Users<br/>Grant cluster admin access]
    end
    
    subgraph "Resource: AKS Cluster"
        AKS[azurerm_kubernetes_cluster]
        AKS --> AKSName[name: ${var.environment}-aks<br/>dns_prefix: Environment-based]
        AKS --> K8sVersion[kubernetes_version:<br/>data.azurerm_kubernetes_service_versions.current.latest_version]
        
        AKS --> DefaultPool[default_node_pool:<br/>name: systempool<br/>vm_size: Standard_DS2_v2<br/>enable_auto_scaling: true<br/>min_count: 1, max_count: 5]
        
        AKS --> LinuxProfile[linux_profile:<br/>admin_username: ubuntu<br/>ssh_key: var.ssh_public_key]
        
        AKS --> WindowsProfile[windows_profile:<br/>admin_username: var.windows_admin_username<br/>admin_password: var.windows_admin_password]
        
        AKS --> ServicePrincipal[identity:<br/>type: SystemAssigned<br/>Managed Identity]
        
        AKS --> AADProfile[azure_active_directory_role_based_access_control:<br/>managed: true<br/>azure_rbac_enabled: true<br/>admin_group_object_ids]
        
        AKS --> NetworkProfile[network_profile:<br/>network_plugin: azure<br/>service_cidr, dns_service_ip]
        
        AKS --> Addon[addon_profile:<br/>oms_agent: enabled<br/>log_analytics_workspace_id]
    end
    
    subgraph "Terraform Provisioning Workflow"
        Workflow[Provisioning Steps]
        Workflow --> W1[1. terraform init<br/>Download providers]
        W1 --> W2[2. terraform plan<br/>Preview resources]
        W2 --> W3[3. terraform apply<br/>Create cluster takes 10-15 min]
        W3 --> W4[4. az aks get-credentials<br/>Configure kubectl]
        W4 --> W5[5. kubectl get nodes<br/>Verify cluster]
    end
    
    subgraph "Cluster Access Methods"
        Access[Access AKS Cluster]
        Access --> AdminAccess[Admin Access:<br/>az aks get-credentials --admin<br/>Bypass Azure AD]
        Access --> AADAccess[Azure AD Access:<br/>az aks get-credentials<br/>Authenticate as AD user]
        AADAccess --> AADLogin[az login<br/>AD user credentials]
        AADLogin --> KubectlCmd[kubectl commands<br/>RBAC enforced]
    end
    
    subgraph "Output Values"
        OutputValues[Terraform Outputs]
        OutputValues --> O1[cluster_id<br/>resource_group_name]
        OutputValues --> O2[cluster_name<br/>cluster_fqdn]
        OutputValues --> O3[kube_config<br/>Full kubeconfig]
        OutputValues --> O4[client_certificate<br/>client_key<br/>cluster_ca_certificate]
    end
    
    Datasource --> K8sVersion
    LAW --> Addon
    AADGroup --> AADProfile
    Variables --> LinuxProfile
    Variables --> WindowsProfile
    
    style Files fill:#5c4ee5
    style AKS fill:#326ce5
    style LAW fill:#ff8c00
    style AADGroup fill:#00bcf2
    style Workflow fill:#28a745
    style OutputValues fill:#ffd700
```

### Understanding the Diagram

- **Modular Terraform Files**: Configuration split into separate **.tf files** - variables, datasources, resources (RG, Log Analytics, AAD group, AKS cluster), and outputs for **better organization** and **maintainability**
- **Data Source for Versions**: Use **azurerm_kubernetes_service_versions** datasource to **dynamically get latest stable K8s version**, eliminating hardcoded versions and ensuring up-to-date deployments
- **Log Analytics Integration**: Creates **Log Analytics workspace** with **random suffix** for unique naming, integrated with AKS for **container insights**, **metrics**, and **log aggregation**
- **Azure AD RBAC**: Creates **Azure AD group** for AKS admins, integrated with cluster via **azure_active_directory_role_based_access_control** block, enabling **AD-based authentication** and **Kubernetes RBAC**
- **System-Assigned Identity**: Cluster uses **System-assigned Managed Identity** instead of service principal, automatically managing credentials for accessing **ACR**, **Load Balancers**, and **Azure Disks**
- **SSH Keys for Linux**: Define **SSH public key** path in variables for Linux worker node access, enabling **secure shell access** for troubleshooting and debugging
- **Windows Node Pool Support**: Setting **windows_profile** during cluster creation enables future **Windows node pools**, required for running **.NET Framework** and **Windows containers**
- **Default Node Pool**: Initial **system node pool** with **auto-scaling** (1-5 nodes) for running **system pods** (CoreDNS, metrics-server), with **Standard_DS2_v2** VMs for cost-effective learning
- **Azure CNI Networking**: **network_plugin: azure** provides pods with IPs from VNet, enabling **direct communication** with Azure services and **network policy support**
- **Dual Access Modes**: Access cluster with **--admin flag** (bypass AD) for emergencies or via **Azure AD** authentication for normal operations with RBAC enforcement

---

## Step-01: Introduction
- Create SSH Keys for AKS Linux VMs
- Declare Windows Username, Passwords for Windows nodepools. This needs to be done during the creation of cluster for 1st time itself if you have plans for Windows workloads on your cluster
- Understand about Datasources and Create Datasource for Azure AKS latest Version
- Create Azure Log Analytics Workspace Resource in Terraform
- Create Azure AD AKS Admins Group Resource in Terraform
- Create AKS Cluster with default nodepool
- Create AKS Cluster Output Values
- Provision Azure AKS Cluster using Terraform
- Access and Test using Azure AKS default admin `--admin`
- Access and Test using Azure AD User as AKS Admin

## Step-02: Create SSH Public Key for Linux VMs
```
# Create Folder
mkdir $HOME/.ssh/aks-prod-sshkeys-terraform

# Create SSH Key
ssh-keygen \
    -m PEM \
    -t rsa \
    -b 4096 \
    -C "azureuser@myserver" \
    -f ~/.ssh/aks-prod-sshkeys-terraform/aksprodsshkey \
    -N mypassphrase

# List Files
ls -lrt $HOME/.ssh/aks-prod-sshkeys-terraform
```

## Step-03: Create 3 more Terraform Input Vairables to variables.tf
- SSH Public Key for Linux VMs
- Windows Admin username
- Windows Admin Password
```
# V2 Changes
# SSH Public Key for Linux VMs
variable "ssh_public_key" {
  default = "~/.ssh/aks-prod-sshkeys-terraform/aksprodsshkey.pub"
  description = "This variable defines the SSH Public Key for Linux k8s Worker nodes"  
}

# Windows Admin Username for k8s worker nodes
variable "windows_admin_username" {
  type = string
  default = "azureuser"
  description = "This variable defines the Windows admin username k8s Worker nodes"  
}

# Windows Admin Password for k8s worker nodes
variable "windows_admin_password" {
  type = string
  default = "P@ssw0rd1234"
  description = "This variable defines the Windows admin password k8s Worker nodes"  
}
```

## Step-04: Create a Terraform Datasource for getting latest Azure AKS Versions 
- Understand [Terraform Datasources](https://www.terraform.io/docs/configuration/data-sources.html) concept as part of this step
- Data sources allow data to be fetched or computed for use elsewhere in Terraform configuration. 
- Use of data sources allows a Terraform configuration to make use of information defined outside of Terraform, or defined by another separate Terraform configuration.
- Use Azure AKS versions datasource API to get the latest version and use it
```
# Call get-versions API via command line
az aks get-versions --location centralus -o table
```
- Create **04-aks-versions-datasource.tf**
- **Important Note:**
  - `include_preview` defaults to true which means we get preview version as latest version which we should not use in production.
  - So we need to enable this flag in datasource and make it to false to use latest version which is not in preview for our production grade clusters
```
# Datasource to get Latest Azure AKS latest Version
data "azurerm_kubernetes_service_versions" "current" {
  location = azurerm_resource_group.aks_rg.location
  include_preview = false  
}
```
- [Data Source: azurerm_kubernetes_service_versions](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/data-sources/kubernetes_service_versions)

## Step-05: Create Azure Log Analytics Workspace Terraform Resource
- The Azure Monitor for Containers (also known as Container Insights) feature provides performance monitoring for workloads running in the Azure Kubernetes cluster.
- We need to create [Log Analytics workspace](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/log_analytics_workspace) and reference its id in AKS Cluster when enabling the monitoring feature.
- Create a file **05-log-analytics-workspace.tf**
```
# Create Log Analytics Workspace
resource "azurerm_log_analytics_workspace" "insights" {
  name                = "logs-${random_pet.aksrandom.id}"
  location            = azurerm_resource_group.aks_rg.location
  resource_group_name = azurerm_resource_group.aks_rg.name
  retention_in_days   = 30
}
```

## Step-06: Create Azure AD Group for AKS Admins Terraform Resource
- To enable AKS AAD Integration, we need to provide Azure AD group object id. 
- We wil create a [Azure Active Directory group](https://registry.terraform.io/providers/hashicorp/azuread/latest/docs/resources/group) for AKS Admins
```
# Create Azure AD Group in Active Directory for AKS Admins
resource "azuread_group" "aks_administrators" {
  name        = "${azurerm_resource_group.aks_rg.name}-cluster-administrators"
  description = "Azure AKS Kubernetes administrators for the ${azurerm_resource_group.aks_rg.name}-cluster."
}
```

## Step-07: Create AKS Cluster Terraform Resource
- Create a file named  **07-aks-cluster.tf**
- Understand and discuss about the terraform resource named  [azurerm_kubernetes_cluster](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/kubernetes_cluster)
- This is going to be a very big terraform template when compared to what we created so far  we will do it slowly step by step.

```
# Provision AKS Cluster
/*
1. Add Basic Cluster Settings
  - Get Latest Kubernetes Version from datasource (kubernetes_version)
  - Add Node Resource Group (node_resource_group)
2. Add Default Node Pool Settings
  - orchestrator_version (latest kubernetes version using datasource)
  - availability_zones
  - enable_auto_scaling
  - max_count, min_count
  - os_disk_size_gb
  - type
  - node_labels
  - tags
3. Enable MSI
4. Add On Profiles 
  - Azure Policy
  - Azure Monitor (Reference Log Analytics Workspace id)
5. RBAC & Azure AD Integration
6. Admin Profiles
  - Windows Admin Profile
  - Linux Profile
7. Network Profile
8. Cluster Tags  
*/

resource "azurerm_kubernetes_cluster" "aks_cluster" {
  name                = "${azurerm_resource_group.aks_rg.name}-cluster"
  location            = azurerm_resource_group.aks_rg.location
  resource_group_name = azurerm_resource_group.aks_rg.name
  dns_prefix          = "${azurerm_resource_group.aks_rg.name}-cluster"
  kubernetes_version  = data.azurerm_kubernetes_service_versions.current.latest_version
  node_resource_group = "${azurerm_resource_group.aks_rg.name}-nrg"

  default_node_pool {
    name                 = "systempool"
    vm_size              = "Standard_DS2_v2"
    orchestrator_version = data.azurerm_kubernetes_service_versions.current.latest_version
    #availability_zones   = [1, 2, 3]
    # Added June2023
    zones = [1, 2, 3]
    #enable_auto_scaling  = true # COMMENTED OCT2024
    auto_scaling_enabled = true  # ADDED OCT2024
    max_count            = 3
    min_count            = 1
    os_disk_size_gb      = 30
    type                 = "VirtualMachineScaleSets"
    node_labels = {
      "nodepool-type"    = "system"
      "environment"      = "dev"
      "nodepoolos"       = "linux"
      "app"              = "system-apps" 
    } 
   tags = {
      "nodepool-type"    = "system"
      "environment"      = "dev"
      "nodepoolos"       = "linux"
      "app"              = "system-apps" 
   } 
  }

# Identity (System Assigned or Service Principal)
  identity {
    type = "SystemAssigned"
  }

# Added June 2023
oms_agent {
  log_analytics_workspace_id = azurerm_log_analytics_workspace.insights.id
}
# Add On Profiles
#  addon_profile {
#    azure_policy {enabled =  true}
#    oms_agent {
#      enabled =  true
#      log_analytics_workspace_id = azurerm_log_analytics_workspace.insights.id
#    }
#  }

# RBAC and Azure AD Integration Block
#  role_based_access_control {
#    enabled = true
#    azure_active_directory {
#      managed = true
#      admin_group_object_ids = [azuread_group.aks_administrators.id]
#    }
#  }
# Added June 2023
azure_active_directory_role_based_access_control {
  #managed = true # COMMENTED OCT2024
  #admin_group_object_ids = [azuread_group.aks_administrators.id] # COMMENTED OCT2024
  admin_group_object_ids = [azuread_group.aks_administrators.object_id] # ADDED OCT2024
}

# Windows Profile
  windows_profile {
    admin_username = var.windows_admin_username
    admin_password = var.windows_admin_password
  }

# Linux Profile
  linux_profile {
    admin_username = "ubuntu"
    ssh_key {
      key_data = file(var.ssh_public_key)
    }
  }

# Network Profile
  network_profile {
    network_plugin = "azure"
    load_balancer_sku = "standard"
  }

  tags = {
    Environment = "dev"
  }
}


```

## Step-08: Create Terraform Output Values for AKS Cluster
- Create a file named **08-outputs.tf**
```
# Create Outputs
# 1. Resource Group Location
# 2. Resource Group Id
# 3. Resource Group Name

# Resource Group Outputs
output "location" {
  value = azurerm_resource_group.aks_rg.location
}

output "resource_group_id" {
  value = azurerm_resource_group.aks_rg.id
}

output "resource_group_name" {
  value = azurerm_resource_group.aks_rg.name
}

# Azure AKS Versions Datasource
output "versions" {
  value = data.azurerm_kubernetes_service_versions.current.versions
}

output "latest_version" {
  value = data.azurerm_kubernetes_service_versions.current.latest_version
}

# Azure AD Group Object Id
output "azure_ad_group_id" {
  value = azuread_group.aks_administrators.id
}
output "azure_ad_group_objectid" {
  value = azuread_group.aks_administrators.object_id
}


# Azure AKS Outputs

output "aks_cluster_id" {
  value = azurerm_kubernetes_cluster.aks_cluster.id
}

output "aks_cluster_name" {
  value = azurerm_kubernetes_cluster.aks_cluster.name
}

output "aks_cluster_kubernetes_version" {
  value = azurerm_kubernetes_cluster.aks_cluster.kubernetes_version
}

```


## Step-09: Deploy Terraform Resources
```
# Change Directory 
cd 24-03-Create-AKS-Cluster/terraform-manifests-aks

# Initialize Terraform from this new folder
# Anyway our state storage is from Azure Storage we are good from any folder
terraform init

# Validate Terraform manifests
terraform validate

# Review the Terraform Plan
terraform plan

# Deploy Terraform manifests
terraform apply 
```

## Step-10: Access Terraform created AKS cluster using AKS default admin
```
# Azure AKS Get Credentials with --admin
az aks get-credentials --resource-group terraform-aks-dev --name terraform-aks-dev-cluster --admin

# Get Full Cluster Information
az aks show --resource-group terraform-aks-dev --name terraform-aks-dev-cluster
az aks show --resource-group terraform-aks-dev --name terraform-aks-dev-cluster -o table

# Get AKS Cluster Information using kubectl
kubectl cluster-info

# List Kubernetes Nodes
kubectl get nodes
```

## Step-11: Verify Resources using Azure Management Console
- Resource Group
  - terraform-aks-dev
  - terraform-aks-dev-nrg
- AKS Cluster & Node Pool
  - Cluster: terraform-aks-dev-cluster
  - AKS System Pool
- Log Analytics Workspace
- Azure AD Group
  - terraform-aks-dev-cluster-administrators


## Step-12: Create a User in Azure AD and Associate User to AKS Admin Group in Azure AD
- Create a user in Azure Active Directory
  - User Name: taksadmin1
  - Name: taksadmin1
  - First Name: taks
  - Last Name: admin1
  - Password: @AKSadmin11
  - Groups: terraform-aks-prod-administrators
  - Click on Create
- Login and change password 
  - URL: https://portal.azure.com
  - Username: taksadmin1@stacksimplifygmail.onmicrosoft.com  (Change your domain name)
  - Old Password: @AKSadmin11
  - New Password: @AKSadmin22
  - Confirm Password: @AKSadmin22

## Step-13: Access Terraform created AKS Cluster 
```
# Azure AKS Get Credentials with --admin
az aks get-credentials --resource-group terraform-aks-dev --name terraform-aks-dev-cluster --overwrite-existing

# List Kubernetes Nodes
kubectl get nodes
URL: https://microsoft.com/devicelogin
Code: GUKJ3T9AC (sample)
Username: taksadmin1@stacksimplifygmail.onmicrosoft.com  (Change your domain name)
Password: @AKSadmin22
```