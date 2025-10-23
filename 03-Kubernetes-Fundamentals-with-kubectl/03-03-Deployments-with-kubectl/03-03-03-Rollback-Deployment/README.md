# Rollback Deployment

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Deployment Revision History"
        History[Deployment History] --> Rev1[Revision 1<br/>Image: kubenginx:1.0.0<br/>Created: Day 1]
        History --> Rev2[Revision 2<br/>Image: kubenginx:2.0.0<br/>Created: Day 2]
        History --> Rev3[Revision 3<br/>Image: kubenginx:3.0.0<br/>Created: Day 3 - Current]
    end
    
    subgraph "Rollback to Previous Version"
        CurrentV3[Current State<br/>Revision 3: v3.0.0<br/>ReplicaSet-3: 10 Pods] --> Issue[Issue Detected<br/>Application Error]
        
        Issue --> UndoCmd[kubectl rollout undo<br/>deployment/my-first-deployment]
        
        UndoCmd --> RollbackProcess[Rollback Initiated]
        RollbackProcess --> ScaleUpRS2[Scale up ReplicaSet-2<br/>v2.0.0]
        ScaleUpRS2 --> ScaleDownRS3[Scale down ReplicaSet-3<br/>v3.0.0]
        
        ScaleDownRS3 --> NewState[New State<br/>Revision 4 points to v2.0.0<br/>ReplicaSet-2: 10 Pods<br/>ReplicaSet-3: 0 Pods]
    end
    
    subgraph "Rollback to Specific Revision"
        NewState --> SpecificIssue[Need to rollback<br/>to v3.0.0]
        
        SpecificIssue --> SpecificUndo[kubectl rollout undo<br/>--to-revision=3]
        
        SpecificUndo --> CheckHistory[kubectl rollout history<br/>Review available revisions]
        
        CheckHistory --> TargetRev[Target Revision 3<br/>Image: kubenginx:3.0.0]
        
        TargetRev --> ScaleUpRS3[Scale up ReplicaSet-3<br/>v3.0.0 again]
        ScaleUpRS3 --> ScaleDownRS2[Scale down ReplicaSet-2<br/>v2.0.0]
        
        ScaleDownRS2 --> FinalState[New State<br/>Revision 5 points to v3.0.0<br/>ReplicaSet-3: 10 Pods]
    end
    
    subgraph "Rolling Restart"
        FinalState --> RestartNeed[Need to restart Pods<br/>without version change]
        
        RestartNeed --> RestartCmd[kubectl rollout restart<br/>deployment/my-first-deployment]
        
        RestartCmd --> RecreatePods[Recreate all Pods<br/>Same image version<br/>Rolling fashion]
        
        RecreatePods --> NewPods[All new Pods<br/>Fresh container state<br/>Same ReplicaSet]
    end
    
    subgraph "ReplicaSet Architecture"
        RSArchitecture[ReplicaSet Architecture] --> RS1[ReplicaSet-1: 0 Pods<br/>Kept for history]
        RSArchitecture --> RS2[ReplicaSet-2: 0 Pods<br/>Kept for history]
        RSArchitecture --> RS3[ReplicaSet-3: 10 Pods<br/>Active]
    end
    
    style UndoCmd fill:#ff6b6b
    style SpecificUndo fill:#ff6b6b
    style NewState fill:#28a745
    style FinalState fill:#28a745
    style RestartCmd fill:#ffa500
```

### Understanding the Diagram

- **Revision History**: Kubernetes maintains a **complete history** of all deployment revisions with details about **images, configurations, and timestamps**
- **Previous Version Rollback**: Use **kubectl rollout undo** without arguments to instantly rollback to the **immediately previous version** (revision N-1)
- **ReplicaSet Scaling**: Rollback works by **scaling up the old ReplicaSet** and **scaling down the current ReplicaSet** in a rolling fashion
- **New Revision Number**: Each rollback creates a **new revision number** that points to the old configuration (e.g., rollback from rev-3 creates rev-4 pointing to rev-2 config)
- **Specific Revision Rollback**: Use **--to-revision=N** to rollback to any **specific previous revision**, not just the immediate previous one
- **Inspect Before Rollback**: Use **kubectl rollout history --revision=N** to inspect the **exact configuration** of each revision before rolling back
- **Zero Downtime Rollback**: Rollback follows the same **rolling update strategy**, ensuring no downtime during the rollback process
- **ReplicaSet Retention**: All old ReplicaSets are **preserved with 0 Pods**, providing instant rollback capability without recreating resources
- **Rolling Restart**: Use **kubectl rollout restart** to recreate all Pods with the **same image version**, useful for picking up config changes or clearing memory leaks
- **Quick Recovery**: Rollback is **faster than forward deployment** since the old ReplicaSet already exists, only requiring scaling operations

---

## Step-00: Introduction
- We can rollback a deployment in two ways.
  - Previous Version
  - Specific Version

## Step-01: Rollback a Deployment to previous version

### Check the Rollout History of a Deployment
```
# List Deployment Rollout History
kubectl rollout history deployment/<Deployment-Name>
kubectl rollout history deployment/my-first-deployment  
```

### Verify changes in each revision
- **Observation:** Review the "Annotations" and "Image" tags for clear understanding about changes.
```
# List Deployment History with revision information
kubectl rollout history deployment/my-first-deployment --revision=1
kubectl rollout history deployment/my-first-deployment --revision=2
kubectl rollout history deployment/my-first-deployment --revision=3
```


### Rollback to previous version
- **Observation:** If we rollback, it will go back to revision-2 and its number increases to revision-4
```
# Undo Deployment
kubectl rollout undo deployment/my-first-deployment

# List Deployment Rollout History
kubectl rollout history deployment/my-first-deployment  
```

### Verify Deployment, Pods, ReplicaSets
```
kubectl get deploy
kubectl get rs
kubectl get po
kubectl describe deploy my-first-deployment
```

### Access the Application using Public IP
- We should see `Application Version:V2` whenever we access the application in browser
```
# Get Load Balancer IP
kubectl get svc

# Application URL
http://<External-IP-from-get-service-output>
```


## Step-02: Rollback to specific revision
### Check the Rollout History of a Deployment
```
# List Deployment Rollout History
kubectl rollout history deployment/<Deployment-Name>
kubectl rollout history deployment/my-first-deployment 
```
### Rollback to specific revision
```
# Rollback Deployment to Specific Revision
kubectl rollout undo deployment/my-first-deployment --to-revision=3
```

### List Deployment History
- **Observation:** If we rollback to revision 3, it will go back to revision-3 and its number increases to revision-5 in rollout history
```
# List Deployment Rollout History
kubectl rollout history deployment/my-first-deployment  
```


### Access the Application using Public IP
- We should see `Application Version:V3` whenever we access the application in browser
```
# Get Load Balancer IP
kubectl get svc

# Application URL
http://<Load-Balancer-IP>
```

## Step-03: Rolling Restarts of Application
- Rolling restarts will kill the existing pods and recreate new pods in a rolling fashion. 
```
# Rolling Restarts
kubectl rollout restart deployment/<Deployment-Name>
kubectl rollout restart deployment/my-first-deployment

# Get list of Pods
kubectl get po
```