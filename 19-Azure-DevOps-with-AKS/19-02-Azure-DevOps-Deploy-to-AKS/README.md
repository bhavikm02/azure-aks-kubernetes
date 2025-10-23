---
title: Azure DevOps Build, Push to ACR and Deploy to AKS
description: Create Azure Pipeline to Build and Push Docker Image to Azure Container Registry and Deploy to AKS Kubernetes Cluster  
---
# Azure DevOps - Build, Push to ACR and Deploy to AKS

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Source Repository"
        GitHub[GitHub Repository]
        GitHub --> AppCode[Application Code:<br/>index.html<br/>Dockerfile]
        GitHub --> K8sManifests[Kubernetes Manifests:<br/>deployment.yml<br/>service.yml]
        GitHub --> PipelineYAML[Pipeline YAML:<br/>02-pipeline.yml]
    end
    
    subgraph "Azure DevOps Pipeline - Build Stage"
        Trigger[Git Push to master branch]
        Trigger --> BuildStage[Stage 1: Build]
        
        BuildStage --> BuildAgent[Ubuntu Build Agent<br/>Microsoft-hosted]
        BuildAgent --> Checkout[Checkout Code<br/>from GitHub]
        Checkout --> DockerTask[Docker@2 Task<br/>buildAndPush command]
        
        DockerTask --> BuildImage[Build Docker Image<br/>using Dockerfile]
        BuildImage --> TagImage[Tag Image:<br/>aksdevopsacr.azurecr.io/app1nginxaks:BuildID]
        TagImage --> PushImage[Push to ACR<br/>via Service Connection]
        
        PushImage --> UploadArtifact[Upload Artifact:<br/>manifests/ folder<br/>deployment.yml, service.yml]
    end
    
    subgraph "Azure Container Registry"
        ACR[Azure Container Registry<br/>aksdevopsacr.azurecr.io]
        ACR --> StoredImage[Stored Image:<br/>app1nginxaks:123]
        ACR --> ImagePullSecret[imagePullSecret<br/>For AKS authentication]
    end
    
    subgraph "Azure DevOps Pipeline - Deploy Stage"
        DeployStage[Stage 2: Deploy<br/>dependsOn: Build]
        DeployStage --> DeployAgent[Ubuntu Deploy Agent]
        DeployAgent --> Environment[Environment:<br/>AKS Cluster Connection<br/>default namespace]
        
        Environment --> DownloadArtifact[Download Artifact:<br/>manifests/]
        DownloadArtifact --> CreateSecret[KubernetesManifest@0 Task:<br/>Create imagePullSecret]
        CreateSecret --> DeployManifests[KubernetesManifest@0 Task:<br/>Deploy manifests]
        
        DeployManifests --> ApplyDeployment[Apply: deployment.yml<br/>Apply: service.yml]
    end
    
    subgraph "Azure Kubernetes Service"
        AKS[AKS Cluster: aksdemo3<br/>default namespace]
        
        ApplyDeployment --> Deployment[Deployment: app1nginxaks<br/>Replicas: 1]
        Deployment --> Pods[Pod: app1nginxaks-xxx<br/>Pulls image from ACR]
        
        ApplyDeployment --> Service[Service: app1nginxaks<br/>Type: LoadBalancer]
        Service --> AzureLB[Azure Load Balancer<br/>Public IP]
        
        Pods --> Service
    end
    
    subgraph "Service Connections"
        SvcConn[Service Connections]
        SvcConn --> ACRConnection[ACR Service Connection<br/>dockerRegistryServiceConnection]
        SvcConn --> AKSConnection[AKS Service Connection<br/>kubectl context]
        
        ACRConnection --> PushImage
        ACRConnection --> CreateSecret
        AKSConnection --> DeployManifests
    end
    
    subgraph "End User Access"
        User[End User]
        User --> PublicIP[http://Public-IP]
        PublicIP --> AzureLB
        AzureLB --> Pods
        Pods --> Response[Application Response<br/>index.html V3]
    end
    
    PushImage --> ACR
    StoredImage --> Pods
    
    style GitHub fill:#28a745
    style BuildStage fill:#326ce5
    style DeployStage fill:#9370db
    style ACR fill:#0078d4
    style AKS fill:#0078d4
    style Response fill:#ffd700
```

### Understanding the Diagram

- **Two-Stage Pipeline**: Pipeline consists of **Build stage** (create and push Docker image) and **Deploy stage** (deploy to AKS cluster), with Deploy stage **depending on** successful completion of Build stage
- **Automated Trigger**: Every **git push to master branch** automatically triggers the entire pipeline, providing **continuous integration** and **continuous deployment** without manual intervention
- **Build Stage Tasks**: Executes **Docker@2 task** with **buildAndPush command** to build the image using Dockerfile and push directly to **Azure Container Registry** using authenticated service connection
- **Artifact Upload**: Build stage uploads **Kubernetes manifests** (deployment.yml, service.yml) as **pipeline artifacts**, making them available for the Deploy stage to consume
- **Deploy Stage Dependency**: Deploy stage has **dependsOn: Build** ensuring it only runs after Build stage succeeds, preventing deployment of failed builds to production
- **AKS Environment**: Pipeline uses an **Environment** resource that represents the **AKS cluster** and **namespace**, enabling deployment tracking, approvals, and environment-specific configurations
- **imagePullSecret Creation**: Deploy stage automatically creates a **Kubernetes secret** for ACR authentication, allowing AKS pods to pull private container images from ACR
- **Manifest Deployment**: **KubernetesManifest@0 task** applies deployment.yml (creates pods) and service.yml (creates LoadBalancer), orchestrating the complete application deployment
- **Service Connection Security**: Two service connections authenticate the pipeline: **ACR connection** for pushing images and **AKS connection** for deploying manifests via kubectl
- **Complete CI/CD Flow**: Code commit → Build image → Push to ACR → Download manifests → Create pull secret → Deploy to AKS → LoadBalancer exposes app → Users access via public IP

---

## Step-00: Pre-requisites
- We should have Azure AKS Cluster Up and Running.
```
# Configure Command Line Credentials
az aks get-credentials --name aksdemo2 --resource-group aks-rg2

# Verify Nodes
kubectl get nodes 
kubectl get nodes -o wide
```

## Step-01: Introduction
- Add a Deployment Pipeline in Azure Pipelines to Deploy newly built docker image from ACR to Azure AKS

[![Image](https://www.stacksimplify.com/course-images/azure-devops-pipelines-deploy-to-aks.png "Azure AKS Kubernetes - Masterclass")](https://www.stacksimplify.com/course-images/azure-devops-pipelines-deploy-to-aks.png)

## Step-02: Create Pipeline for Deploy to AKS
- Go to Pipleines -> Create new Pipleine
- Where is your code?: Github
- Select a Repository: "select your repo" (stacksimplify/azure-devops-github-acr-aks-app1)
- Configure your pipeline: Deploy to Azure Kubernetes Service
- Select Subscription: stacksimplify-paid-subscription (select your subscription)
- Provide username and password (Azure cloud admin user)
- Deploy to Azure Kubernetes Service
  - Cluster: aksdemo3
  - Namespace: existing (default)
  - Container Registry: aksdevopsacr
  - Image Name: app1nginxaks
  - Service Port: 80
- Click on **Validate and Configure**
- Review your pipeline YAML
  -  Change Pipeline Name: 02-docker-build-push-to-acs-deploy-to-aks-pipeline.yml
- Click on **Save and Run**
- Commit Message: Docker, Build, Push and Deploy to AKS
- Commit directly to master branch: check
- Click on  **Save and Run**

 ## Step-03: Verify Build and Deploy logs
 - Build stage should pass. Verify logs
 - Deploy stage should pass. Verify logs


## Step-04: Verify Build and Deploy pipeline logs
- Go to Pipeline -> Verify logs
```
# Verify Pods
kubectl get pods

# Get Public IP
kubectl get svc

# Access Application
http://<Public-IP-from-Get-Service-Output>
```

 ## Step-05: Rename Pipeline Name
- Go to pipeline -> Rename / Move
- Name: 02-Docker-BuildPushToACR-DeployToAKSCluster
- Folder: App1-Pipelines
- Refresh till changes reflect
- Verify -> Pipelines -> Click on **All** tab

## Step-06: Make Changes to index.html and Verify
```
 # Pull
 git pull

# Make changes to index.html
Change version to V3

# Commit and Push
git commit -am "V3 commit index.html"
git push

# Verify Build and Deploy logs
- Build stage logs
- Deploy stage logs
- Verify ACR Repository

# List Pods (Verify Age of Pod)
kubectl get pods 

# Get Public IP
kubectl get svc

# Access Application
http://<Public-IP-from-Get-Service-Output>

``` 

## Step-07: Disable Pipeline
- Go to Pipeline -> 02-Docker-BuildPushToACR-DeployToAKSCluster -> Settings -> Disable


## Step-08: Review Pipeline code
- Click on Pipeline -> Edit Pipeline
- Review pipeline code
- Review Service Connections
 ```yaml
 # Deploy to Azure Kubernetes Service
# Build and push image to Azure Container Registry; Deploy to Azure Kubernetes Service
# https://docs.microsoft.com/azure/devops/pipelines/languages/docker

trigger:
- master

resources:
- repo: self

variables:

  # Container registry service connection established during pipeline creation
  dockerRegistryServiceConnection: '8e06f498-fd9e-481c-8453-12d8c2da0245'
  imageRepository: 'app1nginxaks'
  containerRegistry: 'aksdevopsacr.azurecr.io'
  dockerfilePath: '**/Dockerfile'
  tag: '$(Build.BuildId)'
  imagePullSecret: 'aksdevopsacr1755e8d5-auth'

  # Agent VM image name
  vmImageName: 'ubuntu-latest'
  

stages:
- stage: Build
  displayName: Build stage
  jobs:  
  - job: Build
    displayName: Build
    pool:
      vmImage: $(vmImageName)
    steps:
    - task: Docker@2
      displayName: Build and push an image to container registry
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: $(dockerfilePath)
        containerRegistry: $(dockerRegistryServiceConnection)
        tags: |
          $(tag)
          
    - upload: manifests
      artifact: manifests

- stage: Deploy
  displayName: Deploy stage
  dependsOn: Build

  jobs:
  - deployment: Deploy
    displayName: Deploy
    pool:
      vmImage: $(vmImageName)
    environment: 'stacksimplifyazuredevopsgithubacraksapp1internal-1561.default'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            displayName: Create imagePullSecret
            inputs:
              action: createSecret
              secretName: $(imagePullSecret)
              dockerRegistryEndpoint: $(dockerRegistryServiceConnection)
              
          - task: KubernetesManifest@0
            displayName: Deploy to Kubernetes cluster
            inputs:
              action: deploy
              manifests: |
                $(Pipeline.Workspace)/manifests/deployment.yml
                $(Pipeline.Workspace)/manifests/service.yml
              imagePullSecrets: |
                $(imagePullSecret)
              containers: |
                $(containerRegistry)/$(imageRepository):$(tag)
 ``` 

 ## Step-09: Clean-Up Apps in AKS Cluster
 ```
 # Delete Deployment
 kubectl get deploy
 kubectl delete deploy app1nginxaks

 # Delete Service
 kubectl get svc
 kubectl delete svc app1nginxaks
 ```
