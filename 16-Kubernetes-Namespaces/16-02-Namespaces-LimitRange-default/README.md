---
title: Azure AKS Kubernetes Namespaces Limit Range
description: Understand Kubernetes Namespaces Limit Range Concept Azure Kubernetes Service 
---
# Kubernetes Namespaces - LimitRange - Declarative using YAML

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Namespace without LimitRange"
        NS1[Namespace: dev-no-limits]
        NS1 --> Pod1[Pod without resources:<br/>Can request any amount<br/>❌ May request too little<br/>❌ May request too much]
        Pod1 --> Problem1[Problems:<br/>Resource exhaustion<br/>Unfair resource distribution<br/>Inconsistent pod specs]
    end
    
    subgraph "Namespace with LimitRange"
        NS2[Namespace: dev-with-limits]
        NS2 --> LimitRange[LimitRange Resource<br/>Defines constraints]
        
        LimitRange --> DefaultRequest[default:<br/>  cpu: 100m<br/>  memory: 128Mi]
        LimitRange --> DefaultLimit[defaultRequest:<br/>  cpu: 100m<br/>  memory: 128Mi]
        LimitRange --> MaxLimit[max:<br/>  cpu: 500m<br/>  memory: 1Gi]
        LimitRange --> MinLimit[min:<br/>  cpu: 10m<br/>  memory: 16Mi]
    end
    
    subgraph "Pod Creation with LimitRange"
        CreatePod[Create Pod in namespace]
        CreatePod --> HasResources{Pod has<br/>resources<br/>defined?}
        
        HasResources -->|No| ApplyDefaults[Apply default requests/limits<br/>from LimitRange]
        HasResources -->|Yes| ValidateResources{Within<br/>min/max<br/>constraints?}
        
        ValidateResources -->|Yes| AllowCreation[Pod Created Successfully]
        ValidateResources -->|No| RejectPod[Pod Rejected<br/>Admission error]
        
        ApplyDefaults --> AllowCreation
    end
    
    subgraph "LimitRange Enforcement Examples"
        Example1[Example 1: Pod with no resources]
        Example1 --> Auto1[Automatically gets:<br/>requests: 100m CPU, 128Mi Memory<br/>limits: 100m CPU, 128Mi Memory]
        
        Example2[Example 2: Pod requests 600m CPU]
        Example2 --> Reject2[Rejected:<br/>Exceeds max: 500m CPU]
        
        Example3[Example 3: Pod requests 5m CPU]
        Example3 --> Reject3[Rejected:<br/>Below min: 10m CPU]
        
        Example4[Example 4: Pod requests 200m CPU]
        Example4 --> Accept4[Accepted:<br/>Within range 10m-500m]
    end
    
    subgraph "LimitRange Scope"
        Scope[LimitRange Applies to:]
        Scope --> Containers[Container level<br/>Per container in Pod]
        Scope --> Pods[Pod level<br/>Sum of all containers]
        Scope --> PVC[PersistentVolumeClaim<br/>Storage requests]
    end
    
    subgraph "Benefits"
        Benefits[LimitRange Benefits]
        Benefits --> B1[Prevents resource abuse<br/>No unlimited requests]
        Benefits --> B2[Sets sensible defaults<br/>Developers don't need to specify]
        Benefits --> B3[Ensures minimum quality<br/>Prevents too-small allocations]
        Benefits --> B4[Namespace-level control<br/>Different limits per environment]
        Benefits --> B5[Admission control<br/>Validates before creation]
    end
    
    style LimitRange fill:#326ce5
    style AllowCreation fill:#28a745
    style RejectPod fill:#ff6b6b
    style Benefits fill:#28a745
```

### Understanding the Diagram

- **LimitRange**: Namespace-scoped resource that sets **default, minimum, and maximum** resource constraints for containers, Pods, and PVCs
- **Default Values**: When Pods don't specify resources, LimitRange automatically applies **default requests and limits**, ensuring consistency
- **Minimum Constraints**: Prevents creating Pods with **too few resources**, which could lead to poor performance or crashes
- **Maximum Constraints**: Prevents resource hogging by limiting the **maximum CPU and memory** a single Pod can request
- **Admission Control**: LimitRange is enforced at **Pod creation time** by the admission controller, rejecting invalid Pod specs
- **Namespace Isolation**: Each namespace can have its own LimitRange with **different constraints** (dev: lower limits, prod: higher limits)
- **Container-Level Enforcement**: LimitRange applies to **individual containers** within Pods, not just the Pod as a whole
- **Automatic Resource Assignment**: Developers can omit resource specs, and **LimitRange fills them in automatically** based on defaults
- **Validation Before Creation**: Invalid resource requests are **rejected immediately** with clear error messages, preventing deployment issues
- **Best Practice**: Apply LimitRange to **all namespaces** to ensure consistent resource management and prevent runaway resource consumption

---

[![Image](https://stacksimplify.com/course-images/azure-kubernetes-service-namespaces-limit-range.png "Azure Kubernetes Service - Masterclass")](https://stacksimplify.com/course-images/azure-kubernetes-service-namespaces-limit-range.png){:target="_blank"}  


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


## Step-01: Create Namespace manifest
- **Important Note:** File name starts with `00-`  so that when creating k8s objects namespace will get created first so it don't throw an error.
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev3
```

## Step-02: Create LimitRange manifest
- Instead of specifying `resources like cpu and memory` in every container spec of a pod defintion, we can provide the default CPU & Memory for all containers in a namespace using `LimitRange`
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ns-resource-quota
  namespace: dev3
spec:
  limits:
    - default:
        memory: "512Mi" # If not specified the Container's memory limit is set to 512Mi, which is the default memory limit for the namespace.
        cpu: "500m"  # If not specified default limit is 1 vCPU per container 
      defaultRequest:
        memory: "256Mi" # If not specified default it will take from whatever specified in limits.default.memory
        cpu: "300m" # If not specified default it will take from whatever specified in limits.default.cpu
      type: Container                        
```

## Step-03: Update all k8s manifest with namespace
- Update all files from with `namespace: dev3` in top metadata section in folder `kube-manifests/` 
- **Example**
```yaml
# Deployment Manifest metadata section
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-nginx-deployment
  labels:
    app: app1-nginx
  namespace: dev3    # Added namespace
spec:

# Service Manifest metadata section
apiVersion: v1
kind: Service
metadata:
  name: app1-nginx-clusterip-service
  labels:
    app: app1-nginx
  namespace: dev3   # Added namespace
spec: 
```

## Step-04: Create k8s objects & Test
```
# Create All Objects
kubectl apply -f kube-manifests/

# List Pods
kubectl get pods -n dev3 

# View Pod Specification (CPU & Memory)
kubectl get pod <pod-name> -o yaml -n dev3

# Get & Describe Limits
kubectl get limits -n dev3
kubectl describe limits default-cpu-mem-limit-range -n dev3

# List Services
kubectl get svc -n dev3

# Access Application
http://<Public-IP-from-List-Services-Output>/app1/index.html

```
## Step-05: Clean-Up
- Delete all k8s objects created as part of this section
```
# Delete All
kubectl delete -f kube-manifests/
```


## References:
- https://kubernetes.io/docs/tasks/administer-cluster/namespaces-walkthrough/
- https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/
- https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/
