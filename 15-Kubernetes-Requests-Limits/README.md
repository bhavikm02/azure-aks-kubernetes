---
title: Azure AKS Kubernetes Requests & Limits
description: Understand Kubernetes Resources Requests & Limits on Azure Kubernetes Service AKS Cluster
---
# Kubernetes - Requests and Limits

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Pod without Requests/Limits"
        PodNoLimits[Pod Specification<br/>NO resource specs]
        PodNoLimits --> UnlimitedCPU[Can consume unlimited CPU<br/>on the node]
        PodNoLimits --> UnlimitedMemory[Can consume unlimited Memory<br/>on the node]
        PodNoLimits --> NoGuarantee[❌ No scheduling guarantee<br/>❌ Can be evicted<br/>❌ May starve other Pods]
    end
    
    subgraph "Pod with Requests & Limits"
        PodWithLimits[Pod Specification<br/>WITH resource specs]
        PodWithLimits --> RequestsSpec[resources.requests:<br/>  cpu: 100m<br/>  memory: 128Mi]
        PodWithLimits --> LimitsSpec[resources.limits:<br/>  cpu: 200m<br/>  memory: 256Mi]
    end
    
    subgraph "Requests - Scheduling Guarantee"
        RequestsSpec --> SchedulerCheck[Kubernetes Scheduler]
        SchedulerCheck --> FindNode{Find Node with<br/>Available Resources?}
        
        FindNode -->|Yes| Node1[Worker Node 1<br/>CPU Available: 500m<br/>Memory Available: 1Gi]
        FindNode -->|No| Pending[Pod Status: Pending<br/>Waiting for resources]
        
        Node1 --> ReserveResources[Reserve Resources:<br/>CPU: 100m<br/>Memory: 128Mi]
        ReserveResources --> GuaranteedResources[✓ Pod guaranteed<br/>minimum resources]
    end
    
    subgraph "Limits - Resource Enforcement"
        LimitsSpec --> Kubelet[Kubelet enforces limits<br/>on worker node]
        
        Kubelet --> CPUThrottle{CPU Usage<br/>> 200m?}
        CPUThrottle -->|Yes| ThrottleCPU[Throttle CPU<br/>Slow down process]
        CPUThrottle -->|No| AllowCPU[Allow normal execution]
        
        Kubelet --> MemoryLimit{Memory Usage<br/>> 256Mi?}
        MemoryLimit -->|Yes| OOMKill[Out Of Memory Kill<br/>Pod terminates/restarts]
        MemoryLimit -->|No| AllowMemory[Allow normal execution]
    end
    
    subgraph "Resource Units"
        Units[Resource Units]
        Units --> CPUUnits[CPU Units:<br/>1000m = 1 CPU core<br/>100m = 0.1 CPU<br/>500m = 0.5 CPU]
        Units --> MemoryUnits[Memory Units:<br/>128Mi = 128 Mebibytes<br/>1Gi = 1 Gibibyte<br/>256M = 256 Megabytes]
    end
    
    subgraph "QoS Classes"
        QoS[Quality of Service Classes]
        QoS --> Guaranteed[Guaranteed QoS:<br/>requests = limits<br/>Highest priority<br/>Last to be evicted]
        QoS --> Burstable[Burstable QoS:<br/>requests < limits<br/>Medium priority<br/>Can use more than requested]
        QoS --> BestEffort[BestEffort QoS:<br/>No requests/limits<br/>Lowest priority<br/>First to be evicted]
    end
    
    subgraph "Node Resource Capacity"
        NodeCapacity[Worker Node<br/>Total: 2 CPU, 4Gi Memory]
        NodeCapacity --> Allocatable[Allocatable:<br/>~1.8 CPU, ~3.5Gi<br/>after system reserves]
        
        Allocatable --> Pod1Req[Pod 1: 100m CPU, 128Mi]
        Allocatable --> Pod2Req[Pod 2: 200m CPU, 256Mi]
        Allocatable --> Pod3Req[Pod 3: 300m CPU, 512Mi]
        
        Pod1Req --> Remaining[Remaining:<br/>~1.2 CPU, ~2.6Gi]
    end
    
    style RequestsSpec fill:#326ce5
    style LimitsSpec fill:#ffa500
    style Guaranteed fill:#28a745
    style OOMKill fill:#ff6b6b
```

### Understanding the Diagram

- **Requests**: Minimum resources **guaranteed** to a container; Scheduler only places Pod on nodes with **available requested resources**
- **Limits**: Maximum resources a container can use; Kubelet **enforces limits** by throttling CPU or killing container if memory limit exceeded
- **CPU Units**: Measured in **millicores (m)** where 1000m = 1 CPU core; 100m means 10% of one CPU core
- **Memory Units**: Use **Ki, Mi, Gi** (binary) or **K, M, G** (decimal); Mi = Mebibyte (1024²), M = Megabyte (1000²)
- **Scheduling Decision**: Scheduler sums all **requests** on a node; if node lacks requested resources, Pod stays **Pending** until resources available
- **CPU Throttling**: When container exceeds CPU limit, it's **throttled** (slowed down) but not killed - less severe than memory limit
- **OOM Kill**: When container exceeds memory limit, kernel performs **Out Of Memory (OOM) kill**, terminating the container and restarting it
- **QoS Guaranteed**: When requests = limits for all containers, Pod gets **Guaranteed QoS** - highest priority, last to be evicted
- **QoS Burstable**: When requests < limits, Pod can **burst** above requests up to limits - medium priority, moderate eviction risk
- **Node Capacity**: Total node resources minus **system reserves** (kubelet, OS) = **allocatable resources** for Pods

---

## Step-01: Introduction
- We can specify how much each container in a pod needs the resources like CPU & Memory. 
- When we provide this information in our pod, the scheduler uses this information to decide which node to place the Pod on based on availability of k8s worker Node CPU and Memory Resources. 
- When you specify a resource limit for a Container, the kubelet enforces those `limits` so that the running container is not allowed to use more of that resource than the limit you set. 
-  The kubelet also reserves at least the `request` amount of that system resource specifically for that container to use.

[![Image](https://stacksimplify.com/course-images/azure-kubernetes-service-resources-requests-limits-1.png "Azure Kubernetes Service - Masterclass")](https://stacksimplify.com/course-images/azure-kubernetes-service-resources-requests-limits-1.png){:target="_blank"}  

[![Image](https://stacksimplify.com/course-images/azure-kubernetes-service-resources-requests-limits-2.png "Azure Kubernetes Service - Masterclass")](https://stacksimplify.com/course-images/azure-kubernetes-service-resources-requests-limits-2.png){:target="_blank"}  

## Pre-requisite Check (Optional)
- We should already have our AKS Cluster UP and Running. 
- We should have configured our AKS Cluster credentials in command line to execute `kubectl` commands
```
# Configure AKS Cluster Credentials from command line
az aks get-credentials --name aksdemo1 --resource-group aks-rg1

# List Worker Nodes
kubectl get nodes
kubectl get nodes -o wide
```


## Step-02: Add Requests & Limits & Review k8s Manifests
- **Folder:** kube-manifests-v1
```yaml
          # Requests & Limits    
          resources:
            requests:
              cpu: "100m" 
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"                                                         
```

## Step-03: Create k8s objects & Test
```
# Create All Objects
kubectl apply -f kube-manifests-v1/

# List Pods
kubectl get pods

# List Services
kubectl get svc

# Access Application 
http://<Public-IP-from-List-Services-Output>/app1/index.html
```
## Step-04: Clean-Up
- Delete all k8s objects created as part of this section
```
# Delete All
kubectl delete -f kube-manifests-v1/
```

## Step-05: Assignment
- You can deploy and test `kube-manifests-v2`
- Verify the `Resources` section in file `05-UserMgmtWebApp-Deployment.yml` before deploying
```
# Create All Objects
kubectl apply -f kube-manifests-v2/

# List Pods
kubectl get pods

# List Services
kubectl get svc

# Access Application 
http://<Public-IP-from-List-Services-Output>
Username: admin101
Password: password101

# Clean-Up
kubectl delete -f kube-manifests-v2/
```


## References:
- https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/