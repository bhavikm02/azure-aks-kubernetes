# Create Azure AKS Cluster Linux and Windows Node Pools

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Existing AKS Cluster via Terraform"
        ExistingCluster[AKS Cluster<br/>Created in previous module]
        ExistingCluster --> SystemPool[System Node Pool<br/>In AKS cluster resource]
    end
    
    subgraph "Linux Node Pool Resource"
        LinuxNP[azurerm_kubernetes_cluster_node_pool]
        LinuxNP --> LinuxConfig[name: linux101<br/>kubernetes_cluster_id: AKS ID<br/>vm_size: Standard_DS2_v2<br/>node_count: 1]
        LinuxConfig --> LinuxAS[enable_auto_scaling: true<br/>min_count: 1<br/>max_count: 3]
        LinuxConfig --> LinuxLabel[node_labels:<br/>app = "java-apps"<br/>nodepoolos = "linux"]
        LinuxConfig --> LinuxTags[tags:<br/>environment = "dev"]
    end
    
    subgraph "Windows Node Pool Resource"
        WindowsNP[azurerm_kubernetes_cluster_node_pool]
        WindowsNP --> WinConfig[name: win101<br/>os_type: Windows<br/>vm_size: Standard_DS2_v2<br/>node_count: 1]
        WinConfig --> WinAS[enable_auto_scaling: true<br/>min_count: 1<br/>max_count: 3]
        WinConfig --> WinLabel[node_labels:<br/>app = "dotnet-apps"<br/>nodepoolos = "windows"]
        WinConfig --> WinTags[tags:<br/>environment = "dev"]
    end
    
    subgraph "Terraform Resource Dependencies"
        Depends[Resource Dependencies]
        Depends --> D1[Node pools depend on:<br/>kubernetes_cluster_id]
        Depends --> D2[Reference cluster resource:<br/>azurerm_kubernetes_cluster.aks.id]
        Depends --> D3[Terraform creates cluster first<br/>then node pools]
    end
    
    subgraph "Output Values"
        Outputs[Node Pool Outputs]
        Outputs --> LinuxOut[linux_node_pool_id<br/>linux_node_pool_name]
        Outputs --> WinOut[windows_node_pool_id<br/>windows_node_pool_name]
    end
    
    subgraph "Terraform Workflow"
        Workflow[Provisioning Steps]
        Workflow --> TFPlan[terraform plan<br/>Shows 2 new node pools]
        TFPlan --> TFApply[terraform apply<br/>Creates Linux + Windows pools]
        TFApply --> Verify[kubectl get nodes<br/>See 3 pools: system, linux101, win101]
        Verify --> Deploy[Deploy apps with nodeSelector<br/>Target specific pools]
    end
    
    subgraph "Node Pool Features"
        Features[Node Pool Capabilities]
        Features --> F1[Independent Scaling<br/>Each pool scales separately]
        Features --> F2[Custom Node Labels<br/>For pod scheduling]
        Features --> F3[Different VM Sizes<br/>Per workload requirements]
        Features --> F4[OS-Specific Pools<br/>Linux + Windows support]
    end
    
    ExistingCluster --> LinuxNP
    ExistingCluster --> WindowsNP
    LinuxNP --> Outputs
    WindowsNP --> Outputs
    
    style ExistingCluster fill:#326ce5
    style LinuxNP fill:#28a745
    style WindowsNP fill:#0078d4
    style Workflow fill:#9370db
    style Outputs fill:#ffd700
```

### Understanding the Diagram

- **Separate Node Pool Resources**: Unlike the default node pool (defined within AKS cluster resource), additional pools use **azurerm_kubernetes_cluster_node_pool** as separate Terraform resources for **modular management**
- **Resource Dependency**: Node pools require **kubernetes_cluster_id** referencing the AKS cluster, ensuring Terraform creates the **cluster first** before attempting to create **node pools**
- **Linux User Pool**: Creates **linux101** node pool with **Standard_DS2_v2** VMs, auto-scaling (1-3 nodes), and custom **node labels** (app=java-apps) for targeted pod scheduling
- **Windows User Pool**: Creates **win101** pool with **os_type: Windows**, enabling **Windows Server 2019** nodes for **.NET Framework** applications, with identical auto-scaling configuration
- **Node Labels**: Each pool gets **custom node labels** added automatically to all nodes, enabling **nodeSelector** in pod specs to schedule workloads on specific pools
- **Independent Auto-Scaling**: Each node pool has **separate auto-scaling config** (min/max counts), allowing different scaling profiles based on workload characteristics
- **Terraform Plan Output**: Running **terraform plan** shows **2 new resources to add** (Linux and Windows pools), previewing infrastructure changes before applying
- **Verification Workflow**: After **terraform apply**, use **kubectl get nodes** to verify all **three pools** appear (systempool, linux101, win101) with appropriate OS labels
- **Workload Targeting**: Deploy applications with **nodeSelector** referencing the **custom labels** (app=java-apps, app=dotnet-apps) to route pods to appropriate node pools
- **Infrastructure as Code Benefits**: Node pools defined in Terraform are **version-controlled**, **reproducible**, and can be **destroyed/recreated** easily for environment management

---

## Step-01: Introduction
- Create Windows and Linux Nodepools

## Step-02: Create Azure AKS Linux User Node Pool using Terraform
- Understand about Terraform resource [azurerm_kubernetes_cluster_node_pool](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/kubernetes_cluster_node_pool)
- Create Azure AKS Linux User Node pool
- Create a file named **09-aks-cluster-linux-user-nodepools.tf**
```
resource "azurerm_kubernetes_cluster_node_pool" "linux101" {
  availability_zones    = [1, 2, 3]
  enable_auto_scaling   = true
  kubernetes_cluster_id = azurerm_kubernetes_cluster.aks.id
  max_count             = 3
  min_count             = 1
  mode                  = "User"
  name                  = "linux101"
  orchestrator_version  = data.azurerm_kubernetes_service_versions.current.latest_version
  os_disk_size_gb       = 30
  os_type               = "Linux" # Default is Linux, we can change to Windows
  vm_size               = "Standard_DS2_v2"
  priority              = "Regular"  # Default is Regular, we can change to Spot with additional settings like eviction_policy, spot_max_price, node_labels and node_taints
  node_labels = {
    "nodepool-type" = "user"
    "environment"   = "production"
    "nodepoolos"    = "linux"
    "app"           = "java-apps"
  }
  tags = {
    "nodepool-type" = "user"
    "environment"   = "production"
    "nodepoolos"    = "linux"
    "app"           = "java-apps"
  }
}
```

## Step-03: Create Azure AKS Windows User Node Pool using Terraform
- Create Azure AKS Windows User Node pool to run Windows workloads
- Create a file named **10-aks-cluster-windows-user-nodepools.tf**
```
resource "azurerm_kubernetes_cluster_node_pool" "win101" {
  availability_zones    = [1, 2, 3]
  enable_auto_scaling   = true
  kubernetes_cluster_id = azurerm_kubernetes_cluster.aks.id
  max_count             = 3
  min_count             = 1
  mode                  = "User"
  name                  = "win101"
  orchestrator_version  = data.azurerm_kubernetes_service_versions.current.latest_version
  os_disk_size_gb       = 30
  os_type               = "Windows" # Default is Linux, we can change to Windows
  vm_size               = "Standard_DS2_v2"
  priority              = "Regular"  # Default is Regular, we can change to Spot with additional settings like eviction_policy, spot_max_price, node_labels and node_taints
  #vnet_subnet_id        = azurerm_subnet.aks-default.id 
  node_labels = {
    "nodepool-type" = "user"
    "environment"   = "production"
    "nodepoolos"    = "windows"
    "app"           = "dotnet-apps"
  }
  tags = {
    "nodepool-type" = "user"
    "environment"   = "production"
    "nodepoolos"    = "windows"
    "app"           = "dotnet-apps"
  }
}
```

## Step-04: Deploy Terraform Manifests with nodepool additions (Linux & Windows)
```
# Change Directory 
cd 24-04-Create-AKS-NodePools-using-Terraform/terraform-manifests-aks

# Initialize Terraform
terraform init

# Validate Terraform manifests
terraform validate

# Review the Terraform Plan
terraform plan 

# Deploy Terraform manifests
terraform apply 
```

## Step-05: Verify if Nodepools added successfully
```
# List Node Pools
az aks nodepool list --resource-group terraform-aks-dev --cluster-name  terraform-aks-dev-cluster --output table

# List Nodes using Labels
kubectl get nodes -o wide
kubectl get nodes -o wide -l nodepoolos=linux
kubectl get nodes -o wide -l nodepoolos=windows
kubectl get nodes -o wide -l environment=dev
```


## Step-06: Deploy Sample Applications for all 3 node pools
- Webserver App to System Nodepool
- Sample Java App to Linux Nodepool
- Dotnet App to Windows Nodepool
```
# Change Directory 
cd 24-04-Create-AKS-NodePools-using-Terraform/

# Deploy All Apps
kubectl apply -R -f kube-manifests/

# List Pods
kubectl get pods -o wide
```

## Step-07: Access Applications
```
# List Services to get Public IP for each service we deployed 
kubectl get svc

# Access Webserver App (Running on System Nodepool)
http://<public-ip-of-webserver-app>/app1/index.html

# Access Java-App (Running on linux101 nodepool)
http://<public-ip-of-java-app>
Username: admin101
Password: password101

# Access Windows App (Running on win101 nodepool)
http://<public-ip-of-windows-app>
```

## Step-08: Destroy our Terraform Cluster
```
# Clean-Up Applications deployed
kubectl delete -R -f kube-manifests

# Change Directory 
cd 24-04-Create-AKS-NodePools-using-Terraform/terraform-manifests-aks

# Destroy all our Terraform Resources
terraform destroy
```
