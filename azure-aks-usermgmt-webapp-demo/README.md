# Azure AKS User Management WebApp Demo

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Application Architecture"
        App[User Management Web Application<br/>Spring Boot Java Application]
        App --> Frontend[Frontend: JSP Pages<br/>User Interface]
        App --> Backend[Backend: Spring Boot Controllers<br/>REST APIs]
        Backend --> Service[Service Layer<br/>Business Logic]
        Service --> DAO[DAO Layer<br/>Data Access Objects]
    end
    
    subgraph "Database Layer"
        DAO --> MySQL[(MySQL Database<br/>usermgmt schema)]
        MySQL --> Tables[Tables:<br/>- users table<br/>- roles table<br/>- Stores user credentials]
    end
    
    subgraph "Docker Containerization"
        Dockerfile[Dockerfile]
        Dockerfile --> BaseImage[Base: openjdk:8-jdk-alpine]
        Dockerfile --> CopyJAR[Copy Application JAR]
        Dockerfile --> ExposePort[Expose Port 8080]
        Dockerfile --> Entrypoint[Run: java -jar app.jar]
        
        DockerBuild[docker build -t usermgmt-webapp .]
        DockerBuild --> DockerImage[Docker Image Created]
    end
    
    subgraph "Kubernetes Deployment"
        K8sManifests[Kubernetes Manifests]
        K8sManifests --> MySQLDeployment[MySQL Deployment<br/>+ PVC for storage]
        K8sManifests --> MySQLService[MySQL ClusterIP Service]
        K8sManifests --> AppDeployment[WebApp Deployment<br/>Environment variables]
        K8sManifests --> AppService[WebApp LoadBalancer Service]
        
        AppDeployment --> EnvVars[Environment Variables:<br/>DB_HOSTNAME: mysql<br/>DB_USERNAME: root<br/>DB_PASSWORD: dbpassword11<br/>DB_NAME: usermgmt]
    end
    
    subgraph "Application Flow"
        User[End User] --> Browser[Web Browser]
        Browser --> LB[Azure Load Balancer<br/>Public IP]
        LB --> AppPod[Web App Pod<br/>Port 8080]
        AppPod --> DatabaseConnection[Connect to MySQL<br/>via ClusterIP]
        DatabaseConnection --> MySQLPod[MySQL Pod]
        MySQLPod --> PersistentStorage[Azure Disk<br/>Persistent storage]
    end
    
    subgraph "Application Features"
        Features[Application Capabilities]
        Features --> F1[User Registration<br/>Create new users]
        Features --> F2[User Login<br/>Authentication]
        Features --> F3[List Users<br/>View all registered users]
        Features --> F4[Update User<br/>Edit user details]
        Features --> F5[Delete User<br/>Remove users]
    end
    
    DockerImage --> AppDeployment
    MySQLPod --> DatabaseConnection
    EnvVars --> DatabaseConnection
    
    style App fill:#28a745
    style MySQL fill:#0078d4
    style DockerImage fill:#2496ed
    style AppPod fill:#326ce5
    style Features fill:#ffd700
```

### Understanding the Diagram

- **Spring Boot Application**: Full-stack **Java web application** built with **Spring Boot framework** featuring user registration, login, and CRUD operations for user management
- **Three-Tier Architecture**: Frontend (**JSP pages**) → Backend (**Spring Boot controllers**) → Database (**MySQL**), following standard enterprise application architecture patterns
- **MySQL Database**: Backend stores user data in **MySQL database** with **usermgmt schema** containing users and roles tables for persistent storage
- **Docker Containerization**: Application packaged as **Docker image** using **openjdk:8-jdk-alpine** base image, making it portable and deployable to any container platform
- **Kubernetes Deployment**: Complete **K8s manifests** for deploying MySQL (with persistent volume), ClusterIP service, web app deployment, and LoadBalancer service
- **Environment Variables**: App connects to database using **environment variables** (DB_HOSTNAME, DB_USERNAME, DB_PASSWORD, DB_NAME) injected into pods via deployment manifest
- **Internal Communication**: Web app connects to MySQL via **ClusterIP service** named "mysql", leveraging Kubernetes **internal DNS** for service discovery
- **Persistent Storage**: MySQL data stored on **Azure Disk** via **Persistent Volume Claim**, ensuring data survives pod restarts and rescheduling
- **External Access**: Users access application via **LoadBalancer service** with **public IP**, while MySQL remains internal (ClusterIP) for security
- **Demo Purpose**: This application demonstrates **end-to-end containerized application deployment** on AKS with database connectivity and persistent storage

---