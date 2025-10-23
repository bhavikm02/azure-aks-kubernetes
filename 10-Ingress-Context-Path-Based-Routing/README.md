# Ingress - Context Path Based Routing

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Multiple Applications"
        App1[NginxApp1 Deployment<br/>ClusterIP Service<br/>app1-nginx-clusterip-service]
        App2[NginxApp2 Deployment<br/>ClusterIP Service<br/>app2-nginx-clusterip-service]
        App3[UserMgmt Web App<br/>ClusterIP Service<br/>usermgmt-webapp-service]
        App3 --> MySQL[(MySQL Database<br/>Azure Disk storage)]
    end
    
    subgraph "Ingress Resource - Path Based Routing"
        IngressResource[Ingress Resource<br/>ingress-pathbased.yml]
        IngressResource --> Rule1[Path: /app1/*<br/>→ app1-nginx-clusterip-service:80]
        IngressResource --> Rule2[Path: /app2/*<br/>→ app2-nginx-clusterip-service:80]
        IngressResource --> Rule3[Path: /<br/>→ usermgmt-webapp-service:8080]
        
        Rule1 --> App1
        Rule2 --> App2
        Rule3 --> App3
    end
    
    subgraph "Ingress Controller"
        IngressController[NGINX Ingress Controller<br/>ingress-basic namespace<br/>2 replicas]
        IngressController --> LoadBalancer[Azure Load Balancer<br/>Static Public IP]
    end
    
    subgraph "Traffic Flow"
        User[Internet User]
        User --> Request1[http://Public-IP/app1/index.html]
        User --> Request2[http://Public-IP/app2/index.html]
        User --> Request3[http://Public-IP/]
        
        Request1 --> LoadBalancer
        Request2 --> LoadBalancer
        Request3 --> LoadBalancer
        
        LoadBalancer --> IngressController
        IngressController --> RouteDecision{Path<br/>Matching}
        
        RouteDecision -->|/app1/*| RouteApp1[Route to App1 Service]
        RouteDecision -->|/app2/*| RouteApp2[Route to App2 Service]
        RouteDecision -->|/| RouteApp3[Route to UserMgmt Service]
        
        RouteApp1 --> App1
        RouteApp2 --> App2
        RouteApp3 --> App3
    end
    
    subgraph "Path Types & Annotations"
        PathTypes[Path Types]
        PathTypes --> Prefix[Prefix: Matches /app1/*<br/>Most common]
        PathTypes --> Exact[Exact: Matches /app1 only<br/>Strict matching]
        PathTypes --> ImplementationSpecific[ImplementationSpecific:<br/>Depends on Ingress Controller]
        
        Annotations[Ingress Annotations]
        Annotations --> Rewrite[nginx.ingress.kubernetes.io/rewrite-target<br/>URL rewriting]
        Annotations --> SSL[nginx.ingress.kubernetes.io/ssl-redirect<br/>HTTPS redirect]
    end
    
    style IngressController fill:#326ce5
    style LoadBalancer fill:#0078d4
    style RouteDecision fill:#ffa500
```

### Understanding the Diagram

- **Path-Based Routing**: Single **Ingress resource** routes traffic to **multiple backend services** based on **URL path** (e.g., /app1, /app2, /)
- **Ingress Rules**: Each **path** in the Ingress spec defines a **routing rule** mapping URL paths to specific **ClusterIP Services** and ports
- **NGINX Ingress Controller**: Watches Ingress resources and **dynamically configures NGINX** to route traffic according to defined rules
- **Single Load Balancer**: All applications share **one Azure Load Balancer** with **one public IP**, reducing costs and simplifying DNS management
- **Path Matching**: Ingress Controller examines the **request path** and routes to the matching service (/app1/* → App1, /app2/* → App2, / → UserMgmt)
- **ClusterIP Services**: Backend applications use **ClusterIP Services** (internal only) since external access is handled by **Ingress**
- **URL Rewrite**: Use **rewrite-target annotation** to modify URL before forwarding to backend (e.g., /app1/index.html → /index.html)
- **Prefix Path Type**: Most common type - matches **all paths with the specified prefix** (/app1/anything matches /app1/*)
- **Cost Optimization**: Multiple apps behind **single Load Balancer** vs separate LoadBalancer Service per app saves **Azure Load Balancer costs**
- **Namespace Default**: Ingress resource must be in the **same namespace** as the backend Services it references (typically `default`)

---

## Step-01: Introduction
- We are going to implement context path based routing using Ingress

[![Image](https://www.stacksimplify.com/course-images/azure-aks-ingress-path-based-routing.png "Azure AKS Kubernetes - Masterclass")](https://www.udemy.com/course/aws-eks-kubernetes-masterclass-devops-microservices/?referralCode=257C9AD5B5AF8D12D1E1)

## Step-02: Review k8s Application Manifests
- 01-NginxApp1-Manifests
- 02-NginxApp2-Manifests
- 03-UserMgmtmWebApp-Manifests

## Step-03: Review Ingress Service Manifests
- 04-IngressService-Manifests

## Step-04: Deploy and Verify
```t
# Deploy Apps
kubectl apply -R -f kube-manifests/

# List Pods
kubectl get pods

# List Services
kubectl get svc

# List Ingress
kubectl get ingress

# Verify Ingress Controller Logs
kubectl get pods -n ingress-basic
kubectl logs -f <pod-name> -n ingress-basic
```

## Step-05: Access Applications
```t
# Access App1
http://<Public-IP-created-for-Ingress>/app1/index.html

# Access App2
http://<Public-IP-created-for-Ingress>/app2/index.html

# Access Usermgmt Web App
http://<Public-IP-created-for-Ingress>
Username: admin101
Password: password101
```

## Step-06: Clean-Up Applications
```t
# Delete Apps
kubectl delete -f kube-manifests/

# Delete Azure Disk created for Usermgmt Web App
Go to All Services -> Azure Disks -> Delete disk
```

## Ingress Annotation Reference
- https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/
- [Ingress Path Types](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types)

