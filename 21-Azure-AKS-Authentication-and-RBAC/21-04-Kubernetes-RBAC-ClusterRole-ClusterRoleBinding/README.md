---
title: K8S RBAC Cluster Role & Role Binding with AD on AKS
description: Restrict Access to k8s resources using Kubernetes RBAC Cluster Role and Role Binding with Azure AD
---
# K8S RBAC Cluster Role & Role Binding with AD on AKS

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Azure Active Directory"
        ReadOnlyGroup[Azure AD Group:<br/>aksreadonly]
        ReadOnlyGroup --> ReadUser[Azure AD User:<br/>aksread1@<tenant>.com]
    end
    
    subgraph "Azure RBAC"
        ClusterUserRole[Azure Role:<br/>"Azure Kubernetes Service Cluster User Role"]
        ClusterUserRole --> RoleAssignment[Role Assignment:<br/>Group: aksreadonly<br/>Scope: AKS Cluster]
    end
    
    subgraph "Kubernetes Cluster Resources"
        AllNamespaces[All Namespaces:<br/>default dev qa kube-system]
        ClusterScoped[Cluster-Scoped Resources:<br/>Nodes<br/>PersistentVolumes<br/>Namespaces<br/>StorageClasses]
    end
    
    subgraph "Kubernetes RBAC - Cluster Level"
        ClusterRole[ClusterRole:<br/>aks-cluster-readonly-role<br/>Cluster-scoped]
        
        ClusterRole --> Rules[rules:<br/>- apiGroups: "" extensions apps<br/>  resources: "*"<br/>  verbs: get list watch<br/>- apiGroups: batch<br/>  resources: jobs cronjobs<br/>  verbs: get list watch]
        
        ClusterRoleBinding[ClusterRoleBinding:<br/>aks-cluster-readonly-rolebinding<br/>Cluster-scoped]
        
        ClusterRoleBinding --> BindCR[roleRef:<br/>  kind: ClusterRole<br/>  name: aks-cluster-readonly-role]
        
        ClusterRoleBinding --> BindGroup[subjects:<br/>  kind: Group<br/>  name: <aksreadonly-objectId><br/>  Azure AD Group]
        
        ReadOnlyGroup --> BindGroup
    end
    
    subgraph "Read-Only User Access Flow"
        UserLogin[aksread1 logs in]
        UserLogin --> GetToken[Azure AD OAuth Token<br/>Contains group: aksreadonly]
        
        GetToken --> ReadCommand[kubectl get pods --all-namespaces]
        ReadCommand --> K8sAPI[Kubernetes API Server]
        
        K8sAPI --> ValidateToken[Validate token with Azure AD<br/>Extract groups]
        ValidateToken --> CheckCRB{Is user in group<br/>bound to ClusterRole?}
        
        CheckCRB -->|Yes, aksreadonly| CheckVerb{Requested action<br/>allowed by ClusterRole?}
        CheckCRB -->|No| DenyAccess[Access Denied]
        
        CheckVerb -->|get list watch| AllowReadAccess[Grant read access<br/>Across all namespaces]
        CheckVerb -->|create update delete| DenyWriteAccess[Access Denied:<br/>ClusterRole only allows<br/>get list watch]
        
        AllowReadAccess --> ReturnPods[Return Pods from all namespaces]
    end
    
    subgraph "Attempted Write Operations"
        CreatePodCmd[kubectl create deployment nginx<br/>--image=nginx -n dev]
        CreatePodCmd --> K8sAPI
        K8sAPI --> CheckCreateVerb{Is "create" verb<br/>allowed?}
        CheckCreateVerb -->|No| DenyCreate[Access Denied:<br/>ClusterRole doesn't include<br/>create verb]
        
        DeletePodCmd[kubectl delete pod <pod-name> -n dev]
        DeletePodCmd --> K8sAPI
        K8sAPI --> CheckDeleteVerb{Is "delete" verb<br/>allowed?}
        CheckDeleteVerb -->|No| DenyDelete[Access Denied:<br/>ClusterRole doesn't include<br/>delete verb]
    end
    
    subgraph "ClusterRole vs Role"
        Comparison[Key Differences]
        Comparison --> ClusterRoleChar[ClusterRole:<br/>✓ Cluster-wide scope<br/>✓ Access across all namespaces<br/>✓ Can access cluster-scoped resources<br/>Nodes PVs Namespaces<br/>✓ Use for read-only auditors]
        Comparison --> RoleChar[Role:<br/>✓ Namespace-scoped<br/>✓ Access within single namespace<br/>✓ Cannot access cluster resources<br/>✓ Use for team-based isolation]
    end
    
    subgraph "Read-Only Permission Pattern"
        ReadOnlyVerbs[Read-Only Verbs]
        ReadOnlyVerbs --> GetVerb[get:<br/>Read single resource by name<br/>kubectl get pod nginx]
        ReadOnlyVerbs --> ListVerb[list:<br/>List all resources of a type<br/>kubectl get pods]
        ReadOnlyVerbs --> WatchVerb[watch:<br/>Monitor resource changes<br/>kubectl get pods --watch]
    end
    
    subgraph "Common ClusterRole Use Cases"
        UseCases[ClusterRole Use Cases]
        UseCases --> UC1[✓ Read-only cluster auditors<br/>View all resources<br/>No modification ability]
        UseCases --> UC2[✓ Monitoring teams<br/>Access metrics across namespaces<br/>Read-only for alerts]
        UseCases --> UC3[✓ Security teams<br/>Audit permissions<br/>Review configurations]
        UseCases --> UC4[✓ Support teams<br/>Troubleshoot issues<br/>View logs cluster-wide]
    end
    
    style AllowReadAccess fill:#28a745
    style DenyWriteAccess fill:#ff6b6b
    style DenyCreate fill:#ff6b6b
    style DenyDelete fill:#ff6b6b
    style ClusterRole fill:#9370db
```

### Understanding the Diagram

- **ClusterRole**: Cluster-wide resource defining **permissions that apply across all namespaces** and to cluster-scoped resources (Nodes, PVs, Namespaces)
- **ClusterRoleBinding**: Cluster-wide resource **linking ClusterRole to subjects** (Azure AD users/groups) - grants cluster-wide permissions
- **Read-Only Access**: ClusterRole with verbs `get list watch` provides **read-only access** to all resources across all namespaces
- **No Write Operations**: User can **view resources but cannot create, update, or delete** - requests with write verbs are denied by RBAC
- **Cluster-Wide Scope**: Unlike namespace-scoped Roles, ClusterRole allows access to **all namespaces and cluster-scoped resources**
- **Azure AD Group Integration**: ClusterRoleBinding uses **Azure AD Group Object ID** as subject - all group members get cluster-wide read access
- **Azure Cluster User Role**: Azure RBAC role assignment **required for kubectl access** - ClusterRole/ClusterRoleBinding controls permissions within cluster
- **Read-Only Pattern**: Verbs `get` (single resource), `list` (all resources), `watch` (monitor changes) provide **comprehensive read-only access**
- **Access Denial**: Write operations (create, update, delete, patch) are **denied with permission error** as ClusterRole doesn't grant those verbs
- **Use Cases**: Ideal for **auditors, monitoring teams, security teams, support staff** who need read-only cluster-wide visibility without modification ability

---

## Step-01: Introduction
- AKS can be configured to use Azure AD for Authentication which we have seen in our previous section
- In addition, we can also configure Kubernetes role-based access control (RBAC) to limit access to cluster resources based a user's identity or group membership.
- Understand about Kubernetes RBAC **Cluster Role & Cluster Role Binding**

[![Image](https://stacksimplify.com/course-images/azure-kubernetes-service-RBAC-CR-CRB-1.png "Azure AKS Kubernetes - Masterclass")](https://stacksimplify.com/course-images/azure-kubernetes-service-RBAC-CR-CRB-1.png)

[![Image](https://stacksimplify.com/course-images/azure-kubernetes-service-RBAC-CR-CRB-2.png "Azure AKS Kubernetes - Masterclass")](https://stacksimplify.com/course-images/azure-kubernetes-service-RBAC-CR-CRB-2.png)


## Step-02: Create AD Group, Role Assignment and User
```
# Get Azure AKS Cluster Id
AKS_CLUSTER_ID=$(az aks show --resource-group aks-rg3 --name aksdemo3 --query id -o tsv)
echo $AKS_CLUSTER_ID

# Create Azure AD Group
AKS_READONLY_GROUP_ID=$(az ad group create --display-name aksreadonly --mail-nickname aksreadonly --query objectId -o tsv)    
echo $AKS_READONLY_GROUP_ID

# Create Role Assignment 
az role assignment create \
  --assignee $AKS_READONLY_GROUP_ID \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope $AKS_CLUSTER_ID

# Create AKS ReadOnly User in Azure AD
AKS_READONLY_USER_OBJECT_ID=$(az ad user create \
  --display-name "AKS READ1" \
  --user-principal-name aksread1@stacksimplifygmail.onmicrosoft.com \
  --password @AKSDemo123 \
  --query objectId -o tsv)
echo $AKS_READONLY_USER_OBJECT_ID

# Associate aksread1 User to aksreadonly Group in Azure AD
az ad group member add --group aksreadonly --member-id $AKS_READONLY_USER_OBJECT_ID
```

## Step-03: Test aksreadonly User Authentication to Portal
- URL: https://portal.azure.com
- Username: aksread1@stacksimplifygmail.onmicrosoft.com
- Password: @AKSDemo123


## Step-04: Review Kubernetes RBAC ClusterRole & ClusterRoleBinding
### Kubernetes RBAC Role for aksreadonly User Access
- **File Name:** ClusterRole-ReadOnlyAccess.yaml
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: aks-cluster-readonly-role
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources:
  - jobs
  - cronjobs
  verbs: ["get", "list", "watch"]
```
### Get Object Id for aksreadonly AD Group
```
# Get Object ID for AD Group aksreadonly
az ad group show --group aksreadonly --query objectId -o tsv

# Output
e808215d-d159-49ba-8bb6-9661ba478842
```

[![Image](https://stacksimplify.com/course-images/azure-kubernetes-service-RBAC-ClusterRole.png "Azure AKS Kubernetes - Masterclass")](https://stacksimplify.com/course-images/azure-kubernetes-service-RBAC-ClusterRole.png)

### Review & Update Kubernetes RBAC ClusterRoleBinding with Azure AD Group ID
- Update Azure AD Group **aksreadonly** Object ID in Cluster Role Binding k8s manifest
- **File Name:** ClusterRoleBinding-ReadOnlyAccess.yaml
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: aks-cluster-readonly-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: aks-cluster-readonly-role
subjects:
- kind: Group
  #name: groupObjectId
  name: "e808215d-d159-49ba-8bb6-9661ba478842"   
```

[![Image](https://stacksimplify.com/course-images/azure-kubernetes-service-RBAC-ClusterRoleBinding.png "Azure AKS Kubernetes - Masterclass")](https://stacksimplify.com/course-images/azure-kubernetes-service-RBAC-ClusterRoleBinding.png)

## Step-05: Create Kubernetes RBAC ClusterRole & ClusterRoleBinding 
```
# As AKS Cluster Admin (--admin)
az aks get-credentials --resource-group aks-rg3 --name aksdemo3 --admin

# Create Kubernetes Role and Role Binding
kubectl apply -f kube-manifests/

# Verify ClusterRole & ClusterRoleBinding 
kubectl get clusterrole
kubectl get clusterrolebinding
```

## Step-06: Access AKS Cluster
```
# Overwrite kubectl credentials
az aks get-credentials --resource-group aks-rg3 --name aksdemo3 --overwrite-existing

# List Pods 
kubectl get pods --all-namespaces
- URL: https://microsoft.com/devicelogin
- Code: GCHL8J45R (Sample)(View on terminal)
- Username: aksread1@stacksimplifygmail.onmicrosoft.com
- Password: @AKSDemo123

# List Nodes
kubectl get nodes
```


## Step-07: Create any resource on k8s and observe message
- Create a namespace and see what happems
- We should see forbidder error as this user (aksread1) has only read access to cluster. This use cannot create k8s resources
```
# Create Namespaces dev and qa
kubectl create namespace dev
kubectl create namespace qa

# Error Message
Kalyans-Mac-mini:21-04-Kubernetes-RBAC-ClusterRole-ClusterRoleBinding kalyanreddy$ kubectl create namespace dev
Error from server (Forbidden): namespaces is forbidden: User "aksread1@stacksimplifygmail.onmicrosoft.com" cannot create resource "namespaces" in API group "" at the cluster scope
Kalyans-Mac-mini:21-04-Kubernetes-RBAC-ClusterRole-ClusterRoleBinding kalyanreddy$ 
```


## Step-08: Clean-Up
```
# Clean-Up Clusters Delete Clusters aksdemo3, aksdemo4
Go to All Services -> Resource Groups -> Delete Resource group  aks-rg3
Go to All Services -> Resource Groups -> Delete Resource group  aks-rg4

# Delete Azure AD Users & Groups
# Users
  - user1aksadmin@stacksimplifygmail.onmicrosoft.com 
  - aksdev1@stacksimplifygmail.onmicrosoft.com
  - aksread1@stacksimplifygmail.onmicrosoft.com
# Groups
  - k8sadmins
  - devaksteam
  - aksreadonly
```

