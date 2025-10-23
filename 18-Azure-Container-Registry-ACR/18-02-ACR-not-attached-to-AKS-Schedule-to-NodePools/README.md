---
title: Azure AKS Pull ACR using Service Principal
description: Pull Docker Images from Azure Container Registry using Service Principal to Azure AKS Node pools
---

# Azure AKS Pull Docker Images from ACR using Service Principal

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Azure Container Registry"
        ACR[ACR: acrdemo2ss.azurecr.io<br/>Image: acr-app2:v1]
        ACR --> BuildPush[Build & Push Image:<br/>docker build -t acr-app2:v1 .<br/>docker tag + docker push]
        BuildPush --> ImageInACR[Image stored in ACR]
    end
    
    subgraph "Create Service Principal"
        CreateSP[az ad sp create-for-rbac<br/>--name acr-sp-demo<br/>--skip-assignment]
        CreateSP --> SPOutput[Service Principal Created:<br/>appId: <CLIENT_ID><br/>password: <CLIENT_SECRET><br/>tenant: <TENANT_ID>]
        
        SPOutput --> SaveCredentials[Save credentials for next step]
    end
    
    subgraph "Assign AcrPull Role"
        GetACRID[az acr show --name acrdemo2ss<br/>--query id --output tsv]
        GetACRID --> ACRID[ACR Resource ID]
        
        ACRID --> AssignRole[az role assignment create<br/>--assignee <SP_CLIENT_ID><br/>--scope <ACR_RESOURCE_ID><br/>--role AcrPull]
        
        AssignRole --> RoleAssigned[Service Principal has<br/>AcrPull permission on ACR]
    end
    
    subgraph "Create Kubernetes Secret"
        SaveCredentials --> CreateSecret[kubectl create secret docker-registry acr-secret<br/>--docker-server=acrdemo2ss.azurecr.io<br/>--docker-username=<SP_CLIENT_ID><br/>--docker-password=<SP_CLIENT_SECRET><br/>--docker-email=<your-email>]
        
        CreateSecret --> K8sSecret[Secret: acr-secret<br/>Type: kubernetes.io/dockerconfigjson<br/>Namespace: default]
    end
    
    subgraph "Deploy Application to Node Pools"
        Deployment[Deployment YAML:<br/>acr-app2-deployment.yml]
        Deployment --> ImageSpecSP[spec.template.spec:<br/>  containers:<br/>    image: acrdemo2ss.azurecr.io/acr-app2:v1<br/>  imagePullSecrets:<br/>    - name: acr-secret]
        
        ImageSpecSP --> NoNodeSelector[No nodeSelector<br/>Schedules to regular AKS Node Pool]
        
        NoNodeSelector --> KubectlApply[kubectl apply -f deployment.yml]
        KubectlApply --> CreatePod[Kubelet creates Pod]
        
        CreatePod --> PullWithSecret[Kubelet pulls image using<br/>imagePullSecrets: acr-secret<br/>SP credentials for authentication]
        
        K8sSecret --> PullWithSecret
        ImageInACR --> PullWithSecret
        
        PullWithSecret --> PodOnNodePool[Pod Running on<br/>Regular Node Pool<br/>aks-agentpool-xxx-vmss000000]
    end
    
    subgraph "Service Exposure"
        PodOnNodePool --> Service[Service Type: LoadBalancer<br/>acr-app2-lb-service]
        Service --> AzureLB[Azure Load Balancer<br/>Public IP]
        AzureLB --> ExternalAccess[External Access:<br/>http://<PUBLIC-IP>]
    end
    
    subgraph "Verification Commands"
        Verify[Verification]
        Verify --> GetSecret[kubectl get secret acr-secret<br/>Shows: kubernetes.io/dockerconfigjson]
        Verify --> GetPods[kubectl get pods -o wide<br/>NODE: aks-agentpool-xxx]
        Verify --> DescribePod[kubectl describe pod <pod-name><br/>Events: Pulled image from acrdemo2ss.azurecr.io]
        Verify --> GetSvc[kubectl get svc<br/>EXTERNAL-IP for LoadBalancer]
    end
    
    subgraph "Service Principal vs Managed Identity"
        Comparison[Authentication Methods]
        Comparison --> SPMethod[Service Principal:<br/>✓ Works with any K8s cluster<br/>✓ Manual secret management<br/>❌ Credential rotation needed<br/>❌ More complex setup]
        Comparison --> MSIMethod[Managed Identity (ACR Attached):<br/>✓ No secrets in cluster<br/>✓ Automatic credential rotation<br/>✓ Simpler setup<br/>✓ Best for AKS + ACR]
    end
    
    style ACR fill:#0078d4
    style RoleAssigned fill:#9370db
    style K8sSecret fill:#ffa500
    style PodOnNodePool fill:#28a745
```

### Understanding the Diagram

- **ACR Not Attached**: When ACR is **not attached to AKS**, must use **Service Principal credentials** to authenticate and pull images from ACR
- **Service Principal Creation**: Create SP using `az ad sp create-for-rbac` and save the **appId (CLIENT_ID) and password (CLIENT_SECRET)** for authentication
- **AcrPull Role Assignment**: Assign **AcrPull role** to Service Principal on the ACR resource, granting read-only access to pull images
- **Kubernetes Secret**: Create **docker-registry Secret** containing SP credentials - Kubelet uses this to authenticate with ACR when pulling images
- **imagePullSecrets**: Deployment **must reference the Secret** in `spec.template.spec.imagePullSecrets` for Kubelet to use SP credentials
- **Scheduled to Node Pools**: Without nodeSelector, Pods are **scheduled to regular AKS Node Pools** (Azure VMs), not Virtual Nodes
- **Image Pull Process**: Kubelet reads **imagePullSecrets**, extracts SP credentials from Secret, and authenticates with ACR to pull the image
- **LoadBalancer Service**: Expose application with **LoadBalancer Service**, creating Azure Load Balancer with Public IP for external access
- **Credential Management**: Service Principal credentials require **manual rotation** and management, unlike Managed Identity (ACR attached) which auto-rotates
- **Use Case**: Service Principal method is useful for **cross-cluster scenarios** or when ACR is in a different subscription/tenant than AKS

---

## Step-00: Pre-requisites
- We should have Azure AKS Cluster Up and Running.
- We have created a new aksdemo2 cluster as part of Azure Virtual Nodes demo in previous section.
- We are going to leverage the same cluster for all 3 demos planned for Azure Container Registry and AKS.
```
# Configure Command Line Credentials
az aks get-credentials --name aksdemo2 --resource-group aks-rg2

# Verify Nodes
kubectl get nodes 
kubectl get nodes -o wide

# Verify aci-connector-linux
kubectl get pods -n kube-system

# Verify logs of ACI Connector Linux
kubectl logs -f $(kubectl get po -n kube-system | egrep -o 'aci-connector-linux-[A-Za-z0-9-]+') -n kube-system
```

## Step-01: Introduction
- We are going to pull Images from Azure Container Registry which is not attached to AKS Cluster. 
- We are going to do that using Azure Service Principals.
- Build a Docker Image from our Local Docker on our Desktop
- Push to Azure Container Registry
- Create Service Principal and using that create Kubernetes Secret. 
- Using Kubernetes Secret associated to Pod Specificaiton, pull the docker image from Azure Container Registry and Schedule on Azure AKS NodePools


[![Image](https://stacksimplify.com/course-images/azure-kubernetes-service-and-acr-nodepools.png "Azure AKS Kubernetes - Masterclass")](https://stacksimplify.com/course-images/azure-kubernetes-service-and-acr-nodepools.png)


## Step-02: Create Azure Container Registry
- Go to Services -> Container Registries
- Click on **Add**
- Subscription: StackSimplify-Paid-Subsciption
- Resource Group: acr-rg2
- Registry Name: acrdemo2ss   (NAME should be unique across Azure Cloud)
- Location: Central US
- SKU: Basic  (Pricing Note: $0.167 per day)
- Click on **Review + Create**
- Click on **Create**

## Step-02: Build Docker Image Locally
```
# Change Directory
cd docker-manifests
 
# Docker Build
docker build -t acr-app2:v1 .

# List Docker Images
docker images
docker images acr-app2:v1
```

## Step-03: Run locally and test
```
# Run locally and Test
docker run --name acr-app2 --rm -p 80:80 -d acr-app2:v1

# Access Application locally
http://localhost

# Stop Docker Image
docker stop acr-app2
```

## Step-04: Enable Docker Login for ACR Repository 
- Go to Services -> Container Registries -> acrdemo2ss
- Go to **Access Keys**
- Click on **Enable Admin User**
- Make a note of Username and password

## Step-05: Push Docker Image to Azure Container Registry

### Build, Test Locally, Tag and Push to ACR
```
# Export Command
export ACR_REGISTRY=acrdemo2ss.azurecr.io
export ACR_NAMESPACE=app2
export ACR_IMAGE_NAME=acr-app2
export ACR_IMAGE_TAG=v1
echo $ACR_REGISTRY, $ACR_NAMESPACE, $ACR_IMAGE_NAME, $ACR_IMAGE_TAG

# Login to ACR
docker login $ACR_REGISTRY

# Tag
docker tag acr-app2:v1  $ACR_REGISTRY/$ACR_NAMESPACE/$ACR_IMAGE_NAME:$ACR_IMAGE_TAG
It replaces as below
docker tag acr-app2:v1 acrdemo2ss.azurecr.io/app2/acr-app2:v1

# List Docker Images to verify
docker images acr-app2:v1
docker images $ACR_REGISTRY/$ACR_NAMESPACE/$ACR_IMAGE_NAME:$ACR_IMAGE_TAG

# Push Docker Images
docker push $ACR_REGISTRY/$ACR_NAMESPACE/$ACR_IMAGE_NAME:$ACR_IMAGE_TAG
```
### Verify Docker Image in ACR Repository
- Go to Services -> Container Registries -> acrdemo2ss
- Go to **Repositories** -> **app2/acr-app2**

## Step-05: Create Service Principal to access Azure Container Registry
- Review file: shell-script/generate-service-principal.sh
- Update ACR_NAME with your container registry name
- Update SERVICE_PRINCIPAL_NAME as desired
### NEW SCRIPT - UPDATED ON 22-MAY-2024 - Updated SP_PASSWD with SUBSCRIPTION_ID
```sh
#!/bin/bash
# This script requires Azure CLI version 2.25.0 or later. Check version with `az --version`.

# Modify for your environment.
# ACR_NAME: The name of your Azure Container Registry
# SERVICE_PRINCIPAL_NAME: Must be unique within your AD tenant
ACR_NAME=acrdemo2ss
SERVICE_PRINCIPAL_NAME=acr-sp-demo

# Obtain the full registry ID for subsequent command args
ACR_REGISTRY_ID=$(az acr show --name $ACR_NAME --query id --output tsv)

# Get subscription ID
SUBSCRIPTION_ID=$(az account show --query id -o tsv)


# Create the service principal with rights scoped to the registry.
# Default permissions are for docker pull access. Modify the '--role'
# argument value as desired:
# acrpull:     pull only
# acrpush:     push and pull
# owner:       push, pull, and assign roles
## IMPORTANT NOTE: REPLACE SUBSCRIPTION_ID with your subscription ID
SP_PASSWD=$(az ad sp create-for-rbac --name $SERVICE_PRINCIPAL_NAME --scopes $ACR_REGISTRY_ID --scope subscriptions/$SUBSCRIPTION_ID --role acrpull --query "password" --output tsv)

SP_APP_ID=$(az ad sp list --display-name $SERVICE_PRINCIPAL_NAME --query [].appId --output tsv)

# Output the service principal's credentials; use these in your services and
# applications to authenticate to the container registry.
echo "Service principal ID: $SP_APP_ID"
echo "Service principal password: $SP_PASSWD"
```

### OLD SCRIPT V2 (BEFORE MAY2024) - NOT VALID - JUST FOR REFERENCE
```sh
#!/bin/bash
# This script requires Azure CLI version 2.25.0 or later. Check version with `az --version`.

# Modify for your environment.
# ACR_NAME: The name of your Azure Container Registry
# SERVICE_PRINCIPAL_NAME: Must be unique within your AD tenant
ACR_NAME=acrdemo2ss
SERVICE_PRINCIPAL_NAME=acr-sp-demo

# Obtain the full registry ID for subsequent command args
ACR_REGISTRY_ID=$(az acr show --name $ACR_NAME --query id --output tsv)

# Create the service principal with rights scoped to the registry.
# Default permissions are for docker pull access. Modify the '--role'
# argument value as desired:
# acrpull:     pull only
# acrpush:     push and pull
# owner:       push, pull, and assign roles
## IMPORTANT NOTE: REPLACE SUBSCRIPTION_ID with your subscription ID
SP_PASSWD=$(az ad sp create-for-rbac --name $SERVICE_PRINCIPAL_NAME --scopes $ACR_REGISTRY_ID --role acrpull --query password --output tsv)

SP_APP_ID=$(az ad sp list --display-name $SERVICE_PRINCIPAL_NAME --query [].appId --output tsv)

# Output the service principal's credentials; use these in your services and
# applications to authenticate to the container registry.
echo "Service principal ID: $SP_APP_ID"
echo "Service principal password: $SP_PASSWD"
```

### OLD SCRIPT V1 - NOT VALID - JUST FOR REFERENCE
```sh
#!/bin/bash

# Modify for your environment.
# ACR_NAME: The name of your Azure Container Registry
# SERVICE_PRINCIPAL_NAME: Must be unique within your AD tenant
#ACR_NAME=<container-registry-name>
ACR_NAME=acrdemo2ss
SERVICE_PRINCIPAL_NAME=acr-sp-demo

# Obtain the full registry ID for subsequent command args
ACR_REGISTRY_ID=$(az acr show --name $ACR_NAME --query id --output tsv)

# Create the service principal with rights scoped to the registry.
# Default permissions are for docker pull access. Modify the '--role'
# argument value as desired:
# acrpull:     pull only
# acrpush:     push and pull
# owner:       push, pull, and assign roles
SP_PASSWD=$(az ad sp create-for-rbac --name http://$SERVICE_PRINCIPAL_NAME --scopes $ACR_REGISTRY_ID --role acrpull --query password --output tsv)
SP_APP_ID=$(az ad sp show --id http://$SERVICE_PRINCIPAL_NAME --query appId --output tsv)

# Output the service principal's credentials; use these in your services and
# applications to authenticate to the container registry.
echo "Service principal ID: $SP_APP_ID"
echo "Service principal password: $SP_PASSWD"
```
### Using Windows
```sh
$ACR_NAME='aks2021'
$SERVICE_PRINCIPAL_NAME='acr-sp-demo'
$ACR_REGISTRY_ID=$(az acr show --name $ACR_NAME --query id --output tsv)
 
$SP_PASSWD=$(az ad sp create-for-rbac --name http://$SERVICE_PRINCIPAL_NAME --scopes $ACR_REGISTRY_ID --role acrpull --query password --output tsv)
$SP_APP_ID=$(az ad app list --display-name http://$SERVICE_PRINCIPAL_NAME --query [].appId --output tsv)
```

## Step-06: Create Image Pull Secret
```
# Template
kubectl create secret docker-registry <secret-name> \
    --namespace <namespace> \
    --docker-server=<container-registry-name>.azurecr.io \
    --docker-username=<service-principal-ID> \
    --docker-password=<service-principal-password>

# Replace
kubectl create secret docker-registry acrdemo2ss-secret \
    --namespace default \
    --docker-server=acrdemo2ss.azurecr.io \
    --docker-username=80beacfe-7176-4ff5-ad22-dbb15528a9a8 \
    --docker-password=0zjUzGzSx3_.xi1SC40VcWkdVyl8Ml8QNj    

# List Secrets
kubectl get secrets    
```


## Step-07: Review, Update & Deploy to AKS & Test
### Update Deployment Manifest with Image Name, ImagePullSecrets
```yaml
    spec:
      containers:
        - name: acrdemo-localdocker
          image: acrdemo2ss.azurecr.io/app2/acr-app2:v1
          imagePullPolicy: Always
          ports:
            - containerPort: 80
      imagePullSecrets:
        - name: acrdemo2ss-secret           
```

### Deploy to AKS and Test
```
# Deploy
kubectl apply -f kube-manifests/

# List Pods
kubectl get pods

# Describe Pod
kubectl describe pod <pod-name>

# Get Load Balancer IP
kubectl get svc

# Access Application
http://<External-IP-from-get-service-output>
```

## Step-07: Clean-Up
```
# Delete Applications
kubectl delete -f kube-manifests/
```


## References
- [Azure Container Registry Authentication - Options](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-authentication)
- [Pull images from an Azure container registry to a Kubernetes cluster](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-kubernetes)
