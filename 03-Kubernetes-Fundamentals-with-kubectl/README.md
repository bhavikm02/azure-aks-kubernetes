# Kubernetes Fundaments using Imperative Approach using kubectl

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Kubernetes Imperative Approach with kubectl"
        kubectl[kubectl CLI] --> Commands{Command Type}
        
        Commands --> Create[kubectl create/run]
        Commands --> Read[kubectl get/describe]
        Commands --> Update[kubectl edit/scale]
        Commands --> Delete[kubectl delete]
        
        Create --> Pod[Pod Creation<br/>kubectl run my-pod --image=nginx]
        Create --> RS[ReplicaSet Creation]
        Create --> Deploy[Deployment Creation]
        
        Pod --> SinglePod[Single Container Instance]
        
        RS --> RSObj[ReplicaSet Object]
        RSObj --> MultiplePods[Multiple Pod Replicas<br/>High Availability]
        
        Deploy --> DeployObj[Deployment Object]
        DeployObj --> ManagesRS[Manages ReplicaSets]
        ManagesRS --> RollingUpdate[Rolling Updates<br/>Rollback Support]
        
        Pod --> ExposeCmd[kubectl expose]
        MultiplePods --> ExposeCmd
        RollingUpdate --> ExposeCmd
        
        ExposeCmd --> ServiceType{Service Type}
        ServiceType --> ClusterIP[ClusterIP<br/>Internal Access]
        ServiceType --> NodePort[NodePort<br/>Node-level Access]
        ServiceType --> LoadBalancer[LoadBalancer<br/>External Access]
        
        LoadBalancer --> AzureLB[Azure Load Balancer<br/>Public IP]
        AzureLB --> Traffic[External Traffic Routing]
        
        Read --> Inspect[Inspect Resources<br/>kubectl get pods -o wide]
        Update --> Scale[Scale Workloads<br/>kubectl scale deploy nginx --replicas=5]
        Delete --> Cleanup[Remove Resources<br/>kubectl delete pod my-pod]
    end
    
    style kubectl fill:#326ce5
    style Deploy fill:#28a745
    style AzureLB fill:#0078d4
    style Traffic fill:#ffd700
```

### Understanding the Diagram

- **Imperative Approach**: Execute **direct commands** using **kubectl CLI** to create, modify, and delete Kubernetes resources **without YAML files**
- **kubectl CLI**: The **command-line interface** that communicates with the **Kubernetes API server** to manage cluster resources imperatively
- **Pod Management**: Create standalone **Pods** using **kubectl run**, the most basic deployable unit containing **one or more containers**
- **ReplicaSets**: Ensure **high availability** by maintaining a specified number of **identical Pod replicas** running at all times
- **Deployments**: Highest-level abstraction that manages **ReplicaSets** and provides **declarative updates**, **rolling deployments**, and **rollback capabilities**
- **CRUD Operations**: Perform **Create, Read, Update, Delete** operations using kubectl subcommands like **get, describe, edit, scale, delete**
- **Service Types**: Expose applications using **ClusterIP** (internal), **NodePort** (node-level), or **LoadBalancer** (external with cloud integration)
- **Azure Integration**: **LoadBalancer Services** automatically provision **Azure Load Balancers** with **public IP addresses** for external access
- **Scaling Operations**: Dynamically adjust **replica counts** using **kubectl scale** to handle varying traffic loads
- **Inspection Commands**: Use **kubectl get** with flags like **-o wide** or **-o yaml** to inspect resource details, status, and configuration

---

| S.No  | Topic Name |
| ------| ------------- |
| 01.   | Pods   |
| 02.   | ReplicaSets  |
| 03.   | Deployments  |
| 04.   | Services  |