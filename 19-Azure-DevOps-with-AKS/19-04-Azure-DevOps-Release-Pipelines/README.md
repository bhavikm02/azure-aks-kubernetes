---
title: Azure DevOps Release Pipelines for AKS Kubernetes
description: Create Azure Release Pipeline to Deploy Kubernetes Workloads to Dev, QA, Staging and Prod Environments
---
# Azure DevOps Release Pipelines

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Artifact Source"
        BuildPipeline[Build Pipeline:<br/>04-custom2-BuildPushToACR-Publish-k8s-manifests]
        BuildPipeline --> Artifact[Published Artifact:<br/>kube-manifests<br/>deployment.yml, service.yml]
        Artifact --> AutoTrigger[Continuous Deployment Trigger<br/>New build = New release]
    end
    
    subgraph "AKS Cluster Namespaces"
        AKS[AKS Cluster: aksdemo2]
        AKS --> DevNS[Namespace: dev]
        AKS --> QANS[Namespace: qa]
        AKS --> StagingNS[Namespace: staging]
        AKS --> ProdNS[Namespace: prod]
    end
    
    subgraph "Service Connections"
        SvcConnections[Service Connections]
        SvcConnections --> DevSvc[dev-ns-k8s-aks-svc-conn<br/>Namespace: dev]
        SvcConnections --> QASvc[qa-ns-k8s-aks-svc-conn<br/>Namespace: qa]
        SvcConnections --> StagingSvc[staging-ns-k8s-aks-svc-conn<br/>Namespace: staging]
        SvcConnections --> ProdSvc[prod-ns-k8s-aks-svc-conn<br/>Namespace: prod]
    end
    
    subgraph "Stage 1: Dev Environment"
        DevStage[Stage 1: Dev]
        AutoTrigger --> DevStage
        DevStage --> DevAgent[Agent: ubuntu-latest]
        DevAgent --> DevDownload[Download Artifact:<br/>kube-manifests]
        DevDownload --> DevDeploy[Deploy Task:<br/>kubectl apply -f<br/>deployment.yml, service.yml]
        DevDeploy --> DevNS
        DevSvc --> DevDeploy
    end
    
    subgraph "Stage 2: QA Environment"
        QAStage[Stage 2: QA<br/>Manual trigger or auto after Dev]
        QAStage --> QAApproval[Pre-Deployment Approval<br/>Email notification<br/>Approve/Reject]
        QAApproval -->|Approved| QAAgent[Agent: ubuntu-latest]
        QAAgent --> QADownload[Download Artifact:<br/>kube-manifests]
        QADownload --> QADeploy[Deploy Task:<br/>kubectl apply to qa namespace]
        QADeploy --> QANS
        QASvc --> QADeploy
    end
    
    subgraph "Stage 3: Staging Environment"
        StagingStage[Stage 3: Staging<br/>Manual trigger or auto after QA]
        StagingStage --> StagingApproval[Pre-Deployment Approval<br/>Email notification<br/>Approve/Reject]
        StagingApproval -->|Approved| StagingAgent[Agent: ubuntu-latest]
        StagingAgent --> StagingDownload[Download Artifact:<br/>kube-manifests]
        StagingDownload --> StagingDeploy[Deploy Task:<br/>kubectl apply to staging namespace]
        StagingDeploy --> StagingNS
        StagingSvc --> StagingDeploy
    end
    
    subgraph "Stage 4: Production Environment"
        ProdStage[Stage 4: Production<br/>Manual trigger only]
        ProdStage --> ProdApproval[Pre-Deployment Approval<br/>Email notification<br/>Approve/Reject]
        ProdApproval -->|Approved| ProdAgent[Agent: ubuntu-latest]
        ProdAgent --> ProdDownload[Download Artifact:<br/>kube-manifests]
        ProdDownload --> ProdDeploy[Deploy Task:<br/>kubectl apply to prod namespace]
        ProdDeploy --> ProdNS
        ProdSvc --> ProdDeploy
    end
    
    subgraph "Release Pipeline Workflow"
        DevStage --> QAStage
        QAStage --> StagingStage
        StagingStage --> ProdStage
        
        ProgressiveDeployment[Progressive Deployment:<br/>Dev → QA → Staging → Prod]
    end
    
    subgraph "Approval Gates"
        ApprovalGates[Approval Configuration]
        ApprovalGates --> AG1[Dev: No approval needed<br/>Auto-deploy on artifact]
        ApprovalGates --> AG2[QA: Pre-deployment approval<br/>Email to approvers]
        ApprovalGates --> AG3[Staging: Pre-deployment approval<br/>Email to approvers]
        ApprovalGates --> AG4[Prod: Pre-deployment approval<br/>Email to approvers]
    end
    
    DevNS --> DevLB[LoadBalancer Service<br/>Dev Public IP]
    QANS --> QALB[LoadBalancer Service<br/>QA Public IP]
    StagingNS --> StagingLB[LoadBalancer Service<br/>Staging Public IP]
    ProdNS --> ProdLB[LoadBalancer Service<br/>Prod Public IP]
    
    style DevStage fill:#28a745
    style QAStage fill:#326ce5
    style StagingStage fill:#9370db
    style ProdStage fill:#dc3545
    style QAApproval fill:#ffd700
    style StagingApproval fill:#ffd700
    style ProdApproval fill:#ffd700
```

### Understanding the Diagram

- **Release Pipeline Concept**: Separates **build** (compile, test, package) from **deployment** (release to environments), using **artifacts** from build pipelines to deploy the same tested version across multiple environments
- **Multi-Environment Strategy**: Deploys the **same artifact** progressively through **four environments** (Dev, QA, Staging, Prod), each in its own **Kubernetes namespace**, ensuring environment isolation
- **Namespace Isolation**: Each environment uses a **dedicated namespace** in the same AKS cluster, providing **logical separation** of resources, configs, and secrets while sharing cluster infrastructure
- **Service Connection per Environment**: Each environment has its own **service connection** scoped to a specific namespace, providing **namespace-level RBAC** and preventing accidental cross-environment deployments
- **Continuous Deployment to Dev**: Dev stage has **automatic trigger** enabled - every new build artifact automatically deploys to Dev namespace **without approval**, enabling rapid iteration
- **Approval Gates**: QA, Staging, and Prod stages require **pre-deployment approvals** via email notifications, ensuring human verification before promoting to higher environments
- **Progressive Deployment**: Application moves through environments in sequence (**Dev → QA → Staging → Prod**), with each stage validating the release before proceeding to the next
- **Artifact Reuse**: The **same built artifact** is deployed to all environments, eliminating "works on my machine" issues and ensuring what's tested in QA is exactly what goes to Prod
- **Manual vs Automatic Triggers**: Dev auto-deploys, QA/Staging can be auto or manual after previous stage success, Production typically **manual trigger only** for controlled releases
- **LoadBalancer per Environment**: Each namespace gets its own **LoadBalancer service** with a **unique public IP**, allowing independent testing and access to each environment

---

## Step-01: Introduction
- Understand Release Pipelines concept
- Create Release Pipelines to Deploy to Kubernetes Dev, QA, Staging and Prod namespaces
- Add Pre-Deployment email approval for QA, Staging and Prod environment deployments

[![Image](https://stacksimplify.com/course-images/azure-devops-release-pipelines-for-azure-aks.png "Azure AKS Kubernetes - Masterclass")](https://stacksimplify.com/course-images/azure-devops-release-pipelines-for-azure-aks.png)

[![Image](https://stacksimplify.com/course-images/azure-devops-release-pipelines-demo-for-azure-aks.png "Azure AKS Kubernetes - Masterclass")](https://stacksimplify.com/course-images/azure-devops-release-pipelines-demo-for-azure-aks.png)

[![Image](https://stacksimplify.com/course-images/azure-devops-release-pipelines-releases-demo-for-azzure-aks.png "Azure AKS Kubernetes - Masterclass")](https://stacksimplify.com/course-images/azure-devops-release-pipelines-releases-demo-for-azzure-aks.png)

## Step-02: Create Namespaces
```
# List Namespaces
kubectl get ns

# Create Namespaces
kubectl create ns dev
kubectl create ns qa
kubectl create ns staging
kubectl create ns prod

# List Namespaces
kubectl get ns
```

## Step-03: Create Service Connections for Dev, QA, Staging and Prod Namespaces in Kubernetes Cluster
### Dev Service Connection
- Go to Project -> azure-devops-github-acr-aks-app1 -> Project Settings -> Pipelines -> Service Connections
- Click on **New Service Connection**
- Choose a service or connection type: kubernetes
- Authentication Method: Azure Subscription
- Username: Azure Cloud Administrator
- Password: Azure Cloud Admin Password
- Cluster: aksdemo2
- Namespace: dev
- Service connection name: dev-ns-k8s-aks-svc-conn
- Description: Dev Namespace AKS Cluster Service Connection
- Security: Grant access permission to all pipelines (default Checked)
- Click on **SAVE** 

### QA Service Connection
- Go to Project -> azure-devops-github-acr-aks-app1 -> Project Settings -> Pipelines -> Service Connections
- Click on **New Service Connection**
- Choose a service or connection type: kubernetes
- Authentication Method: Azure Subscription
- Username: Azure Cloud Administrator
- Password: Azure Cloud Admin Password
- Cluster: aksdemo2
- Namespace: qa
- Service connection name: qa-ns-k8s-aks-svc-conn
- Description: QA Namespace AKS Cluster Service Connection
- Security: Grant access permission to all pipelines (default Checked)
- Click on **SAVE** 

### Staging Service Connection
- Go to Project -> azure-devops-github-acr-aks-app1 -> Project Settings -> Pipelines -> Service Connections
- Click on **New Service Connection**
- Choose a service or connection type: kubernetes
- Authentication Method: Azure Subscription
- Username: Azure Cloud Administrator
- Password: Azure Cloud Admin Password
- Cluster: aksdemo2
- Namespace: staging
- Service connection name: staging-ns-k8s-aks-svc-conn
- Description: Staging Namespace AKS Cluster Service Connection
- Security: Grant access permission to all pipelines (default Checked)
- Click on **SAVE** 


### Production Service Connection
- Go to Project -> azure-devops-github-acr-aks-app1 -> Project Settings -> Pipelines -> Service Connections
- Click on **New Service Connection**
- Choose a service or connection type: kubernetes
- Authentication Method: Azure Subscription
- Username: Azure Cloud Administrator
- Password: Azure Cloud Admin Password
- Cluster: aksdemo2
- Namespace: prod
- Service connection name: prod-ns-k8s-aks-svc-conn
- Description: Production Namespace AKS Cluster Service Connection
- Security: Grant access permission to all pipelines (default Checked)
- Click on **SAVE** 

## Step-04: Create Release Pipeline - Add Artifacts
- Release Pipeline Name: 01-app1-release-pipeline

### Add Artifact
- Source Type: Build
- Project: leave to default (azure-aks-app1-github-acr)
- Source (Build Pipeline): App1-Pipelines\04-custom2-BuildPushToACR-Publish-k8s-manifests-to-AzurePipelines
- Default Version: Latest (auto-populated)
- Source Alias: leave to default (auto-populated)
- Click on **Add**

### Continuous Deployment Trigger
- Continuous deployment trigger: Enabled


## Step-05: Release Pipeline - Create Dev Stage
- Go to Pipelines -> Releases
- Create new **Release Pipeline**
### Create Dev Stage and Test
- Stage Name: Dev
- Create Task 
- Agent Job: Change to Ubunut Linux (latest)
### Add Task: Create Secret
- Display Name: Create Secret to allow image pull from ACR
- Action: create secret
- Kubernetes service connection: dev-ns-k8s-aks-svc-conn
- Namespace: dev
- Type of secret: dockerRegistry
- Secret name: dev-aksdevopsacr-secret
- Docker registry service connection: manual-aksdevopsacr-svc
- Rest all leave to defaults
- Click on **SAVE** to save release
- Comment: Dev k8s Create Secret task added

### Add Task: Deploy to Kubernetes
- Display Name: Deploy to AKS
- Action: deploy
- Kubernetes Service Connection: dev-ns-k8s-aks-svc-conn
- Namespace: dev
- Strategy: None
- Manifest: Select 01-Deployment-and-LoadBalancer-Service.yml  from build artifacts
```
# Sample Value for Manifest after adding it
Manifest: $(System.DefaultWorkingDirectory)/_04-custom2-BuildPushToACR-Publish-k8s-manifests-to-AzurePipelines/kube-manifests/01-Deployment-and-LoadBalancer-Service.yml
```
- Container: aksdevopsacr.azurecr.io/custom2aksnginxapp1:$(Build.BuildId)
- ImagePullSecrets: dev-aksdevopsacr-secret
- Rest all leave to defaults
- Click on **SAVE** to save release
- Comment: Dev k8s Deploy task added

## Step-06: Verify k8s Deployment Manifest Image 
- Review the **image** value and update it from Container registry if required
- File: kube-manifests/01-Deployment-and-LoadBalancer-Service.yml
```yaml
    spec:
      containers:
        - name: app1-nginx
          image: aksdevopsacr.azurecr.io/custom2aksnginxapp1
          ports:
            - containerPort: 80
```

## Step-07: Check-In Code and Test
- Update index.html
```
# Commit and Push
git commit -am "V11 Commit"
git push
```
- View Build Logs
- View Dev Release logs
- Access App after successful deployment
```
# Get Public IP
kubectl get svc -n dev

# Access Application
http://<Public-IP-from-Get-Service-Output>
```

## Step-08: Update Deploy to AKS Task with Build.SourceVersion in Release Pipelines
- Go to Release Pipelines -> 01-app1-release-pipeline -> Edit -> Dev Tasks
- Go to **Deploy to AKS** Task
- Replace
```
#Before
Containers: aksdevopsacr.azurecr.io/custom2aksnginxapp1:$(Build.BuildId)

# After
Containers: aksdevopsacr.azurecr.io/custom2aksnginxapp1:$(Build.SourceVersion)
```
- Click on **SAVE** to save release
- Comment: Dev Container Tag changed from Build Id to Build Source Version


## Step-09: Check-In Code and Test
- Update index.html
```
# Commit and Push
git commit -am "V12 Commit"
git push
```
- View Build Logs
- View Dev Release logs
- Access App after successful deployment
```
# Get Public IP
kubectl get svc -n dev

# Access Application
http://<Public-IP-from-Get-Service-Output>
```
- Verify Github Commit Id on Github Repository and Container Registry

## Step-10: Create QA, Staging and Prod Release Stages
- Create QA, Staging and Prod Stages
- Add Email Approvals
- Click on **SAVE** to save release

### Clone Dev Stage to Create QA Stage
- Go to Releases -> 01-app1-release-pipeline -> Edit
- Select **Dev Stage** -> Add -> **Clone Stage**
- Stage Name: QA
#### Task-1: Create Secret
- Kubernetes service connection: qa-ns-k8s-aks-svc-conn
- Namespace: qa
- Secret name: qa-aksdevopsacr-secret
- Click SAVE
- Commit Message: QA Create Secret task updated

#### Task-2: Deploy to AKS
- Kubernetes service connection: qa-ns-k8s-aks-svc-conn
- Namespace: qa
- ImagePullSecrets: qa-aksdevopsacr-secret
- Click SAVE
- Commit Message: QA Deploy to AKS task updated



## Step-11: Check-In Code and Test
- Update index.html
```
# Commit and Push
git commit -am "V13 Commit"
git push
```
- View Build Logs
- View Dev Release logs
- Access App after successful deployment
- Approve deployment at qa, staging and prod stages
```
# Get Public IP
kubectl get svc -n dev
kubectl get svc -n qa
kubectl get svc -n staging
kubectl get svc -n prod
kubect get svc --all-namespaces

# Access Application
http://<Public-IP-from-Get-Service-Output>
```

## Step-12: Clean-Up Apps
```
# Before Clean-Up: List all Pods and Services
kubectl get pod,svc --all-namespaces

# Clean-Up all Apps in Kubernetes
kubectl delete ns dev
kubectl delete ns qa
kubectl delete ns staging
kubectl delete ns prod

# After Clean-Up: List all Pods and Services
kubectl get pod,svc --all-namespaces
```

## References
- [Release Pipelines Task - Deploy to Kubernetes](https://docs.microsoft.com/en-us/azure/devops/pipelines/ecosystems/kubernetes/deploy?view=azure-devops)
