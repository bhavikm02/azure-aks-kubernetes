# Deployments with YAML

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "From ReplicaSet to Deployment"
        RSTemplate[ReplicaSet Template<br/>replicaset-definition.yml]
        RSTemplate --> ChangeKind[Change kind:<br/>ReplicaSet → Deployment]
        ChangeKind --> ChangeAPI[apiVersion: apps/v1<br/>stays same]
        ChangeAPI --> UpdateImage[Update image:<br/>kubenginx:2.0.0 → 3.0.0]
        UpdateImage --> UpdateLabels[Update labels/names:<br/>myapp2 → myapp3]
        UpdateLabels --> DeployYAML[deployment-definition.yml]
    end
    
    subgraph "Deployment YAML Structure"
        DeployYAML --> DeploySpec[apiVersion: apps/v1<br/>kind: Deployment]
        DeployYAML --> DeployMeta[metadata:<br/>  name: myapp3-deployment]
        DeployYAML --> SpecSection[spec:]
        
        SpecSection --> Replicas[replicas: 3]
        SpecSection --> Selector[selector:<br/>  matchLabels: app=myapp3]
        SpecSection --> Template[template: # Pod template<br/>  metadata & spec]
    end
    
    subgraph "Deployment Creation Flow"
        Apply[kubectl apply -f<br/>deployment-definition.yml]
        Apply --> CreateDeploy[Create Deployment Object]
        CreateDeploy --> DeployController[Deployment Controller]
        
        DeployController --> AutoCreateRS[Automatically create<br/>ReplicaSet]
        AutoCreateRS --> RSName[ReplicaSet<br/>myapp3-deployment-abc1234]
        RSName --> CreatePods[ReplicaSet creates 3 Pods]
        
        CreatePods --> Pod1[myapp3-deployment-abc1234-x1y2z]
        CreatePods --> Pod2[myapp3-deployment-abc1234-a3b4c]
        CreatePods --> Pod3[myapp3-deployment-abc1234-d5e6f]
    end
    
    subgraph "Hierarchy Visualization"
        Hierarchy[Resource Hierarchy]
        Hierarchy --> Level1[Deployment: myapp3-deployment<br/>Manages rollouts & history]
        Level1 --> Level2[ReplicaSet: myapp3-deployment-abc1234<br/>Maintains desired replicas]
        Level2 --> Level3[Pods: 3 instances<br/>Run actual containers]
    end
    
    subgraph "Service for Deployment"
        ServiceYAML[service-definition.yml<br/>type: LoadBalancer]
        ServiceYAML --> ServiceMatch[selector:<br/>  app: myapp3]
        
        ServiceMatch --> ServicePods[Matches all Deployment Pods]
        ServicePods --> Pod1
        ServicePods --> Pod2
        ServicePods --> Pod3
        
        ServiceYAML --> AzureLBDeploy[Azure Load Balancer<br/>Public IP for access]
    end
    
    subgraph "Key Differences"
        Differences[Deployment vs ReplicaSet]
        Differences --> D1[Deployment:<br/>✓ Rolling updates<br/>✓ Rollback support<br/>✓ Version history<br/>✓ Manages ReplicaSets]
        Differences --> D2[ReplicaSet:<br/>✓ Maintains replicas<br/>✗ No update strategy<br/>✗ No rollback<br/>✗ No history]
    end
    
    style CreateDeploy fill:#326ce5
    style DeployController fill:#326ce5
    style AzureLBDeploy fill:#0078d4
    style D1 fill:#28a745
```

### Understanding the Diagram

- **Easy Migration**: Converting **ReplicaSet to Deployment** requires changing only the **kind** field and optionally updating image versions and names
- **apiVersion Consistency**: Both ReplicaSets and Deployments use **apps/v1** API version, making conversion straightforward
- **Deployment Controller**: Manages the entire **application lifecycle** including creating ReplicaSets, handling updates, and maintaining revision history
- **Automatic ReplicaSet**: Deployment automatically creates a **ReplicaSet** with a generated name (deployment-name + hash) to manage Pods
- **Resource Hierarchy**: **Deployment → ReplicaSet → Pods** - each level manages the layer below it with specific responsibilities
- **Pod Naming Convention**: Pods created by Deployments have names like **deployment-name-replicaset-hash-pod-hash** for clear traceability
- **Service Label Matching**: LoadBalancer Service uses **app: myapp3** selector to route traffic to all Pods, regardless of the managing ReplicaSet
- **Rolling Update Capability**: Deployments support **zero-downtime updates** by creating new ReplicaSets and gradually shifting traffic
- **Rollback Support**: Deployments maintain **revision history** allowing instant rollback to previous versions using old ReplicaSets
- **Production Recommendation**: Always use **Deployments** over ReplicaSets in production for better update management and rollback capabilities

---

## Step-01: Copy templates from ReplicaSet
- Copy templates from ReplicaSet and change the `kind: Deployment` 
- Update Container Image version to `3.0.0`
- Change all names to Deployment
- Change all labels and selectors to `myapp3`

```
# Create Deployment
kubectl apply -f 02-deployment-definition.yml
kubectl get deploy
kubectl get rs
kubectl get po

# Create LoadBalancer Service
kubectl apply -f 03-deployment-LoadBalancer-service.yml

# List Service
kubectl get svc

# Get Public IP
kubectl get nodes -o wide

# Access Application
http://<Load-Balancer-Service-IP>
```
## API References
- [Deployment](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#deployment-v1-apps)
