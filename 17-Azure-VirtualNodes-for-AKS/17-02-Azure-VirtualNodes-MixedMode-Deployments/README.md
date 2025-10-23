---
title: Azure AKS Virtual Nodes Mixed Mode Deployments
description: Deploy Applications in mixed mode to Virtual Nodes and AKS Nodepools
---

# Azure AKS Virtual Nodes Mixed Mode Deployments

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Mixed Mode Architecture"
        AKSCluster[AKS Cluster: aksdemo2]
        
        AKSCluster --> RegularNodePool[Regular Node Pool<br/>System Node Pool<br/>Azure VM<br/>For stateful workloads]
        
        AKSCluster --> VirtualNode[Virtual Node<br/>virtual-node-aci-linux<br/>For stateless workloads]
    end
    
    subgraph "MySQL on Regular Node Pool"
        MySQLDeploy[MySQL Deployment<br/>Stateful Application]
        MySQLDeploy --> NoNodeSelector[No nodeSelector specified<br/>Defaults to regular nodes]
        
        NoNodeSelector --> MySQLPod[MySQL Pod<br/>With Persistent Volume]
        MySQLPod --> ScheduleRegular[Scheduled to:<br/>aks-agentpool-xxx-vmss000000]
        
        MySQLDeploy --> PVC[PVC: mysql-pv-claim<br/>Azure Disk 5Gi<br/>ReadWriteOnce]
        PVC --> PV[Persistent Volume<br/>Azure Managed Disk]
        PV --> MySQLPod
        
        MySQLDeploy --> ConfigMap[ConfigMap: usermgmt-dbcreation-script<br/>Database init SQL]
        ConfigMap --> InitContainer[Init Container:<br/>Runs DB schema setup]
        InitContainer --> MySQLPod
    end
    
    subgraph "Web App on Virtual Node"
        WebAppDeploy[User Management Web App Deployment<br/>Stateless Application]
        WebAppDeploy --> VirtualNodeSelector[nodeSelector:<br/>  kubernetes.io/role: agent<br/>  beta.kubernetes.io/os: linux<br/>  type: virtual-kubelet]
        
        WebAppDeploy --> VirtualTolerations[tolerations:<br/>  - key: virtual-kubelet.io/provider<br/>  - key: azure.com/aci]
        
        VirtualNodeSelector --> WebAppPod[Web App Pod<br/>No persistent storage<br/>Connects to MySQL via Service]
        WebAppPod --> ScheduleVirtual[Scheduled to:<br/>virtual-node-aci-linux]
        ScheduleVirtual --> ACIContainer[ACI Container Instance<br/>Serverless, Pay-per-second]
    end
    
    subgraph "Service Communication"
        WebAppPod --> MySQLService[MySQL Service<br/>Type: ClusterIP<br/>Port: 3306]
        MySQLService --> MySQLPod
        
        ExternalUser[External Users] --> LBService[Service: LoadBalancer<br/>Web App Public IP]
        LBService --> WebAppPod
    end
    
    subgraph "Environment Variables in Web App"
        WebAppPod --> EnvVars[Environment Variables:<br/>DB_HOSTNAME: mysql<br/>DB_PORT: 3306<br/>DB_NAME: webappdb]
        EnvVars --> DNSResolution[Kubernetes DNS resolves<br/>mysql -> ClusterIP]
        DNSResolution --> MySQLService
    end
    
    subgraph "Verification Commands"
        Verify[Verification]
        Verify --> GetPods[kubectl get pods -o wide<br/>Shows node assignment]
        Verify --> MySQLOnVM[MySQL Pod:<br/>NODE: aks-agentpool-xxx]
        Verify --> WebAppOnVN[Web App Pod:<br/>NODE: virtual-node-aci-linux]
        Verify --> GetNodes[kubectl get nodes<br/>Shows both node types]
        Verify --> GetPVC[kubectl get pvc<br/>Shows MySQL PVC Bound]
    end
    
    subgraph "Why Mixed Mode?"
        Reasons[Mixed Mode Benefits]
        Reasons --> StatefulVM[Stateful workloads on VMs:<br/>✓ Persistent storage support<br/>✓ Full Kubernetes features<br/>✓ Databases, caches]
        Reasons --> StatelessACI[Stateless workloads on ACI:<br/>✓ No storage needed<br/>✓ Fast scaling<br/>✓ Pay-per-second<br/>✓ Web frontends, APIs]
        Reasons --> CostOptimized[Cost Optimization:<br/>✓ VMs for data tier<br/>✓ ACI for burst capacity<br/>✓ No idle VM costs for web tier]
    end
    
    style VirtualNode fill:#00d1b2
    style ACIContainer fill:#00d1b2
    style RegularNodePool fill:#326ce5
    style MySQLPod fill:#326ce5
    style WebAppPod fill:#00d1b2
```

### Understanding the Diagram

- **Mixed Mode Deployment**: Run **stateful workloads (MySQL) on regular VM nodes** for persistent storage, and **stateless workloads (Web App) on Virtual Nodes (ACI)** for cost efficiency
- **MySQL on Regular Nodes**: MySQL Pod scheduled on **regular node pool** because it needs **Persistent Volume** support (Azure Disk), which Virtual Nodes don't support
- **Web App on Virtual Nodes**: User Management Web App scheduled on **virtual-node-aci-linux** using nodeSelector and tolerations for **serverless ACI execution**
- **NodeSelector for Virtual Nodes**: Web App Deployment specifies `type: virtual-kubelet` nodeSelector to target Virtual Nodes instead of regular VMs
- **Tolerations Required**: Must add tolerations for `virtual-kubelet.io/provider` and `azure.com/aci` to allow scheduling on Virtual Nodes
- **Service Communication**: Web App connects to MySQL via **ClusterIP Service** using Kubernetes DNS (service name: `mysql` resolves to ClusterIP)
- **Persistent Volume**: MySQL uses **PVC with Azure Disk** for data persistence - only works on regular VM nodes, not Virtual Nodes
- **Init Container**: MySQL Pod uses **ConfigMap** to run database initialization script via Init Container before main container starts
- **Cost Efficiency**: Virtual Nodes provide **pay-per-second billing** for web app, while VMs run database tier - no need for extra VMs for web tier burst capacity
- **Verification**: Use `kubectl get pods -o wide` to see which node each Pod is scheduled on - MySQL on `aks-agentpool-xxx`, Web App on `virtual-node-aci-linux`

---

## Step-01: Introduction
- We are going to deploy MySQL on regular AKS nodepools (default system nodepool)
- We are going to deploy **User Management Web Application** on Azure Virtual Nodes
- All this we are going to do using NodeSelectors concept in Kubernetes

[![Image](https://stacksimplify.com/course-images/azure-kubernetes-service-virtual-nodes-mixed-mode-deployments.png "Azure AKS Kubernetes - Masterclass")](https://stacksimplify.com/course-images/azure-kubernetes-service-virtual-nodes-mixed-mode-deployments.png)

## Step-02: Review Kubernetes Manifests
### MySQL Deployment 
- **File Name:** 04-mysql-deployment.yml
- No changes in it, MySQL pod will get scheduled on default AKS nodepool

### User Management Web Application Deployment
- **File Name:** 06-UserMgmtWebApp-Deployment.yml
- User Management web app pod will schedule on Azure Virtual Node
```yaml
# To schedule pods on Azure Virtual Nodes            
      nodeSelector:
        kubernetes.io/role: agent
        beta.kubernetes.io/os: linux
        type: virtual-kubelet
      tolerations:
      - key: virtual-kubelet.io/provider
        operator: Exists
      - key: azure.com/aci
        effect: NoSchedule    
```

## Step-03: Deploy App & Test
```
# Deploy
kubectl apply -f kube-manifests/

# Verify Pods
kubectl get pods

# Verify Pods scheduled on which Nodes
kubectl get pods -o wide

# List Kubernetes Nodes
kubectl get nodes 
kubectl get nodes -o wide

# List Node Pools
az aks nodepool list --cluster-name aksdemo2 --resource-group aks-rg2 --output table

# Access Application
kubectl get svc
http://<Public-IP-from-Get-Service-Output>
Username: admin101
Password: password101
```


## Step-04: Clean-Up Apps
```
# Delete App
kubectl delete -f kube-manifests/

# Delete this new cluster created for Virtual Nodes (if you want to)
az aks delete --name aksdemo2 --resource-group aks-rg2
```

