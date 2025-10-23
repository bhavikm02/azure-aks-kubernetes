# Pause & Resume Deployments

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Current Deployment State"
        InitialState[Deployment Running<br/>Image: kubenginx:3.0.0<br/>Revision 5<br/>10 Pods Active]
    end
    
    subgraph "Pause Deployment Workflow"
        InitialState --> PauseCmd[kubectl rollout pause<br/>deployment/my-first-deployment]
        
        PauseCmd --> PausedState[Deployment PAUSED<br/>Updates blocked]
        
        PausedState --> Change1[Change 1: Update Image<br/>kubectl set image<br/>kubenginx:3.0.0 → 4.0.0]
        
        Change1 --> NoRollout1[No Rollout Triggered<br/>Still Paused]
        
        NoRollout1 --> Change2[Change 2: Set Resources<br/>kubectl set resources<br/>cpu=20m, memory=30Mi]
        
        Change2 --> NoRollout2[No Rollout Triggered<br/>Changes Queued]
        
        NoRollout2 --> MoreChanges[Can make more changes<br/>env vars, labels, etc.]
    end
    
    subgraph "Resume Deployment Workflow"
        MoreChanges --> ResumeCmd[kubectl rollout resume<br/>deployment/my-first-deployment]
        
        ResumeCmd --> ApplyAll[Apply ALL Changes Together<br/>Single Rolling Update]
        
        ApplyAll --> NewRS[Create New ReplicaSet<br/>Image: 4.0.0<br/>Resources: cpu=20m, memory=30Mi]
        
        NewRS --> RollingUpdate[Rolling Update Process]
        
        RollingUpdate --> GradualShift[Gradually replace<br/>old Pods with new Pods]
        
        GradualShift --> FinalState[Deployment Running<br/>Image: kubenginx:4.0.0<br/>New Revision Created<br/>10 Pods with new config]
    end
    
    subgraph "Benefits of Pause/Resume"
        Benefits[Why Pause/Resume?]
        Benefits --> B1[Batch Multiple Changes<br/>Single rollout]
        Benefits --> B2[Reduce Rollout Count<br/>Fewer revisions]
        Benefits --> B3[Test Configuration<br/>Before applying]
        Benefits --> B4[Atomic Updates<br/>All or nothing]
    end
    
    subgraph "Comparison: With vs Without Pause"
        WithoutPause[Without Pause] --> WP1[Change 1 → Rollout 1]
        WP1 --> WP2[Change 2 → Rollout 2]
        WP2 --> WP3[Change 3 → Rollout 3]
        WP3 --> WP4[3 Separate Rollouts<br/>3 Revisions Created]
        
        WithPause[With Pause] --> P1[Pause]
        P1 --> P2[Changes 1, 2, 3 queued]
        P2 --> P3[Resume]
        P3 --> P4[1 Single Rollout<br/>1 Revision Created]
    end
    
    style PauseCmd fill:#ffa500
    style PausedState fill:#ff6b6b
    style ResumeCmd fill:#326ce5
    style FinalState fill:#28a745
    style P4 fill:#28a745
```

### Understanding the Diagram

- **Pause Command**: Use **kubectl rollout pause** to temporarily **freeze deployment updates**, allowing you to make multiple configuration changes
- **Changes While Paused**: All **kubectl** commands (set image, set resources, etc.) are **accepted** but no **rolling update** is triggered while paused
- **Queued Changes**: Multiple changes are **queued** in the deployment specification, waiting for the **resume** command to apply them all
- **Resume Command**: **kubectl rollout resume** applies **all queued changes together** in a **single rolling update**, creating only one new revision
- **Atomic Updates**: All changes are applied **atomically** - either all succeed together or all fail together, ensuring consistency
- **Reduced Rollout Count**: Pausing prevents **multiple consecutive rollouts**, reducing cluster churn and creating cleaner revision history
- **Single Revision**: Instead of creating **3 separate revisions** for 3 changes, pause/resume creates **1 revision** with all changes combined
- **Resource Efficiency**: Fewer rollouts mean less **Pod churn**, reducing resource consumption and minimizing potential disruption
- **Testing Configuration**: Review all **queued changes** in the deployment spec before resuming to catch configuration errors early
- **Version 3 to 4 Example**: Diagram shows updating from **v3.0.0 to v4.0.0** along with resource limits, all in a single coordinated update

---

## Step-00: Introduction
- Why do we need Pausing & Resuming Deployments?
  - If we want to make multiple changes to our Deployment, we can pause the deployment make all changes and resume it. 
- We are going to update our Application Version from **V3 to V4** as part of learning "Pause and Resume Deployments"  

## Step-01: Pausing & Resuming Deployments
### Check current State of Deployment & Application
 ```
# Check the Rollout History of a Deployment
kubectl rollout history deployment/my-first-deployment  
Observation: Make a note of last version number

# Get list of ReplicaSets
kubectl get rs
Observation: Make a note of number of replicaSets present.

# Access the Application 
http://<External-IP-from-get-service-output>
Observation: Make a note of application version
```

### Pause Deployment and Two Changes
```
# Pause the Deployment
kubectl rollout pause deployment/<Deployment-Name>
kubectl rollout pause deployment/my-first-deployment

# Update Deployment - Application Version from V3 to V4
kubectl set image deployment/my-first-deployment kubenginx=stacksimplify/kubenginx:4.0.0 --record=true

# Check the Rollout History of a Deployment
kubectl rollout history deployment/my-first-deployment  
Observation: No new rollout should start, we should see same number of versions as we check earlier with last version number matches which we have noted earlier.

# Get list of ReplicaSets
kubectl get rs
Observation: No new replicaSet created. We should have same number of replicaSets as earlier when we took note. 

# Make one more change: set limits to our container
kubectl set resources deployment/my-first-deployment -c=kubenginx --limits=cpu=20m,memory=30Mi
```
### Resume Deployment 
```
# Resume the Deployment
kubectl rollout resume deployment/my-first-deployment

# Check the Rollout History of a Deployment
kubectl rollout history deployment/my-first-deployment  
Observation: You should see a new version got created

# Get list of ReplicaSets
kubectl get rs
Observation: You should see new ReplicaSet.

# Get Load Balancer IP
kubectl get svc
```
### Access Application
```
# Access the Application 
http://<External-IP-from-get-service-output>
Observation: You should see Application V4 version
```


## Step-02: Clean-Up
```
# Delete Deployment
kubectl delete deployment my-first-deployment

# Delete Service
kubectl delete svc my-first-deployment-service

# Get all Objects from Kubernetes default namespace
kubectl get all
```