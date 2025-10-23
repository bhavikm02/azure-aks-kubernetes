# Services with YAML

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Backend Tier - ClusterIP Service"
        BackendYAML[01-backend-deployment.yml<br/>Spring Boot REST API]
        BackendYAML --> BackendDeploy[Deployment: my-backend<br/>replicas: 3<br/>image: kube-helloworld:1.0.0<br/>port: 8080]
        
        BackendDeploy --> BackendPods[3 Backend Pods<br/>IP: 10.244.x.x]
        
        BackendSvcYAML[02-backend-clusterip-service.yml]
        BackendSvcYAML --> ClusterIPService[kind: Service<br/>type: ClusterIP<br/>name: my-backend-service]
        
        ClusterIPService --> BackendSelector[selector:<br/>  app: backend<br/>port: 8080<br/>targetPort: 8080]
        BackendSelector --> BackendPods
        ClusterIPService --> InternalIP[ClusterIP: 10.100.x.x<br/>Internal only]
    end
    
    subgraph "Frontend Tier - LoadBalancer Service"
        FrontendYAML[03-frontend-deployment.yml<br/>Nginx Reverse Proxy]
        FrontendYAML --> FrontendDeploy[Deployment: my-frontend<br/>replicas: 3<br/>image: kube-frontend-nginx:1.0.0<br/>port: 80]
        
        FrontendDeploy --> FrontendPods[3 Frontend Pods<br/>Nginx config:<br/>proxy_pass to<br/>my-backend-service:8080]
        
        FrontendPods --> DNSResolution[Kubernetes DNS<br/>my-backend-service<br/>→ 10.100.x.x]
        DNSResolution --> ClusterIPService
        
        FrontendSvcYAML[04-frontend-LoadBalancer-service.yml]
        FrontendSvcYAML --> LBService[kind: Service<br/>type: LoadBalancer<br/>name: my-frontend-service]
        
        LBService --> FrontendSelector[selector:<br/>  app: frontend<br/>port: 80<br/>targetPort: 80]
        FrontendSelector --> FrontendPods
        LBService --> AzureLBSvc[Azure Load Balancer<br/>Public IP: 20.x.x.x]
    end
    
    subgraph "End-to-End Traffic Flow"
        User[Internet User] --> HTTPRequest[http://20.x.x.x/hello]
        HTTPRequest --> AzureLBSvc
        AzureLBSvc --> RouteFrontend[Route to Frontend Pod]
        RouteFrontend --> FrontendPods
        FrontendPods --> NginxReverseProxy[Nginx Reverse Proxy<br/>proxy_pass]
        NginxReverseProxy --> InternalCall[Internal call:<br/>my-backend-service:8080/hello]
        InternalCall --> DNSResolution
        ClusterIPService --> RouteBackend[Route to Backend Pod]
        RouteBackend --> BackendPods
        BackendPods --> JSONResponse[JSON Response<br/>Backend hostname]
        JSONResponse --> ReturnResponse[Response path reversed]
    end
    
    subgraph "YAML-Based Management"
        YAMLFolder[kube-manifests/ folder<br/>All YAML files]
        
        YAMLFolder --> ApplyAll[kubectl apply -f kube-manifests/]
        ApplyAll --> CreateAll[Creates all resources:<br/>2 Deployments<br/>2 Services]
        
        YAMLFolder --> DeleteAll[kubectl delete -f kube-manifests/]
        DeleteAll --> RemoveAll[Deletes all resources]
        
        CreateAll --> GitOps[Version control benefits:<br/>✓ Track changes<br/>✓ Code review<br/>✓ Rollback<br/>✓ Reproducible]
    end
    
    style ClusterIPService fill:#9370db
    style LBService fill:#0078d4
    style AzureLBSvc fill:#0078d4
    style GitOps fill:#28a745
```

### Understanding the Diagram

- **Two-Tier Architecture**: **Frontend tier** (Nginx reverse proxy) exposed externally via **LoadBalancer**, **backend tier** (REST API) internal via **ClusterIP**
- **ClusterIP Service**: Default service type providing **internal-only** access with stable **virtual IP** for backend-to-backend communication
- **Service Name Importance**: Nginx configuration must use **exact service name** (my-backend-service) as configured in the ClusterIP Service
- **Kubernetes DNS**: Automatically resolves **service names to ClusterIPs**, enabling service discovery without hardcoding IP addresses
- **LoadBalancer Service**: Exposes frontend to **internet** by provisioning **Azure Load Balancer** with **public IP address**
- **Label Selectors**: Backend service selects Pods with **app: backend**, frontend service selects Pods with **app: frontend** for traffic routing
- **Reverse Proxy Pattern**: Frontend Nginx **forwards requests** to backend service, hiding backend complexity and providing single entry point
- **Traffic Flow**: Request flows **Internet → Azure LB → Frontend Pod → ClusterIP Service → Backend Pod**, response follows reverse path
- **Folder-Based Deployment**: Use **kubectl apply -f kube-manifests/** to deploy all YAML files in a folder with single command
- **Version Control**: Store all YAML manifests in **Git repository** for **change tracking**, **collaboration**, **code review**, and **reproducible deployments**

---

## Step-01: Introduction to Services
- We are going to look in to below two services in detail with a frotnend and backend example
  - LoadBalancer Service
  - ClusterIP Service

## Step-02: Create Backend Deployment & Cluster IP Service
- Write the Deployment template for backend REST application.
- Write the Cluster IP service template for backend REST application.
- **Important Notes:** 
  - Name of Cluster IP service should be `name: my-backend-service` because  same is configured in frontend nginx reverse proxy `default.conf`. 
  - Test with different name and understand the issue we face
  - We have also discussed about in our section [03-04-Services-with-kubectl](https://github.com/stacksimplify/azure-aks-kubernetes-masterclass/tree/master/03-Kubernetes-Fundamentals-with-kubectl/03-04-Services-with-kubectl)
```
cd 04-05-Services-with-YAML/kube-manifests
kubectl get all
kubectl apply -f 01-backend-deployment.yml -f 02-backend-clusterip-service.yml
kubectl get all
```


## Step-03: Create Frontend Deployment & LoadBalancer Service
- Write the Deployment template for frontend Nginx Application
- Write the LoadBalancer service template for frontend Nginx Application
```
cd 04-05-Services-with-YAML/kube-manifests
kubectl get all
kubectl apply -f 03-frontend-deployment.yml -f 04-frontend-LoadBalancer-service.yml
kubectl get all
```
- **Access REST Application**
```
# Get Service IP
kubectl get svc

# Access REST Application 
http://<Load-Balancer-Service-IP>/hello
```

## Step-04: Delete & Recreate Objects using kubectl apply
### Delete Objects (file by file)
```
kubectl delete -f 01-backend-deployment.yml -f 02-backend-clusterip-service.yml -f 03-frontend-deployment.yml -f 04-frontend-LoadBalancer-service.yml
kubectl get all
```
### Recreate Objects using YAML files in a folder
```
cd 04-05-Services-with-YAML/
kubectl apply -f kube-manifests/
kubectl get all
```
### Delete Objects using YAML files in folder
```
cd 04-05-Services-with-YAML/
kubectl delete -f kube-manifests/
kubectl get all
```


## Additional References - Use Label Selectors for get and delete
- [Labels](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#using-labels-effectively)
- [Labels-Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors)