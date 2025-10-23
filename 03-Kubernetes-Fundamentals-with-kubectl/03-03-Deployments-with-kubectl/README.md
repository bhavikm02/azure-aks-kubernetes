# Kubernetes - Deployment

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Deployment Architecture"
        Deployment[Deployment<br/>my-app-deployment] --> ManagesRS[Manages ReplicaSets]
        ManagesRS --> RS1[ReplicaSet v1<br/>Image: app:1.0]
        ManagesRS --> RS2[ReplicaSet v2<br/>Image: app:2.0]
        
        RS1 --> Pods1[3 Pods Running<br/>Version 1.0]
        RS2 --> Pods2[3 Pods Running<br/>Version 2.0]
    end
    
    subgraph "Deployment Operations"
        Create[kubectl create deployment<br/>--image=app:1.0] --> Running1[Deployment v1 Running]
        
        Running1 --> Scale[kubectl scale deployment<br/>--replicas=5]
        Scale --> ScaledUp[5 Pods Running]
        
        ScaledUp --> Update[kubectl set image<br/>deployment/my-app<br/>app:1.0 → app:2.0]
        Update --> RollingUpdate[Rolling Update Strategy]
        
        RollingUpdate --> NewRS[Create New ReplicaSet v2]
        NewRS --> GradualShift[Gradually shift traffic<br/>v1 → v2]
        GradualShift --> OldRSScaleDown[Scale down old ReplicaSet]
        OldRSScaleDown --> Complete[Update Complete]
        
        Complete --> Rollback{Issue<br/>Detected?}
        Rollback -->|Yes| RollbackCmd[kubectl rollout undo]
        RollbackCmd --> PreviousVersion[Revert to v1]
        Rollback -->|No| Stable[Deployment Stable]
    end
    
    subgraph "Advanced Operations"
        Pause[kubectl rollout pause] --> PausedState[Paused Deployment<br/>Make multiple changes]
        PausedState --> Resume[kubectl rollout resume]
        Resume --> ApplyAll[Apply all changes together]
        
        Restart[kubectl rollout restart] --> RollingRestart[Rolling Restart<br/>No version change]
    end
    
    subgraph "Service Exposure"
        Stable --> Expose[kubectl expose deployment<br/>--type=LoadBalancer]
        Expose --> Service[Service routes to<br/>current active Pods]
        Service --> AzureLB[Azure Load Balancer<br/>Public IP]
    end
    
    style Deployment fill:#326ce5
    style Complete fill:#28a745
    style RollingUpdate fill:#ffa500
    style AzureLB fill:#0078d4
```

### Understanding the Diagram

- **Deployment Hierarchy**: Deployments are the **highest-level abstraction** that manages **ReplicaSets**, which in turn manage **Pods**
- **Version Management**: Each deployment update creates a **new ReplicaSet** while keeping previous ReplicaSets for **easy rollback**
- **Creation Command**: Use **kubectl create deployment** to imperatively create a Deployment with specified **image** and **replica count**
- **Scaling Operations**: **kubectl scale** adjusts the number of **Pod replicas** without changing application version or configuration
- **Rolling Update Strategy**: Updates happen **gradually** by creating new Pods while terminating old ones, ensuring **zero downtime**
- **Automatic Rollback**: If issues are detected, use **kubectl rollout undo** to instantly revert to the **previous working version**
- **Pause and Resume**: **Pause deployments** to make **multiple changes** (image, env vars, resources) then **resume** to apply all changes together
- **Rolling Restart**: Perform **kubectl rollout restart** to restart all Pods with a **rolling strategy** without changing the application version
- **Service Integration**: Expose Deployments via **LoadBalancer Service** which automatically routes traffic to the **current active Pods**
- **Azure Load Balancer**: Service creates an **Azure Load Balancer** with **public IP** that distributes traffic across healthy Pod replicas

---

## Topics
1. Create Deployment
2. Scale the Deployment
3. Expose Deployment as a Service
4. Update Deployment
5. Rollback Deployment
6. Rolling Restarts
7. Pause & Resume Deployments
8. Canary Deployments (Will be covered at Declarative section of Deployments)





