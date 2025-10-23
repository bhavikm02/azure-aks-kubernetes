# Azure AKS Storage - Azure Disks

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "Storage Components"
        SC[Storage Class<br/>Defines disk type & parameters]
        PVC[Persistent Volume Claim<br/>Request for storage]
        PV[Persistent Volume<br/>Actual Azure Disk]
        AzureDisk[(Azure Managed Disk<br/>Premium/Standard SSD)]
    end
    
    subgraph "MySQL Deployment Architecture"
        ConfigMap[ConfigMap<br/>usermgmt schema SQL]
        
        MySQLDeploy[MySQL Deployment]
        MySQLDeploy --> MySQLPod[MySQL Pod]
        
        MySQLPod --> EnvVars[Environment Variables<br/>MYSQL_ROOT_PASSWORD<br/>from secrets]
        MySQLPod --> VolumeMount[Volume Mount<br/>/var/lib/mysql]
        MySQLPod --> InitScript[Init Container<br/>Execute SQL from ConfigMap]
        
        VolumeMount --> Volume[Volume: mysql-persistent-storage]
        Volume --> PVC
        
        MySQLPod --> ClusterIPSvc[ClusterIP Service<br/>mysql:3306<br/>Internal access only]
    end
    
    subgraph "User Management Web App"
        WebAppDeploy[Web App Deployment<br/>Spring Boot]
        WebAppDeploy --> WebAppPod[Web App Pods]
        
        WebAppPod --> WebAppEnv[Environment Variables<br/>DB_HOSTNAME: mysql<br/>DB_USERNAME: root<br/>DB_PASSWORD<br/>DB_NAME: usermgmt]
        
        WebAppPod --> ConnectMySQL[Connect to MySQL<br/>via ClusterIP Service]
        ConnectMySQL --> ClusterIPSvc
        
        WebAppDeploy --> LBSvc[LoadBalancer Service<br/>External Access]
    end
    
    subgraph "Storage Lifecycle"
        CreateSC[1. Create StorageClass<br/>Define Azure Disk parameters]
        CreateSC --> CreatePVC[2. Create PVC<br/>Request 5Gi storage]
        CreatePVC --> ProvisionPV[3. Kubernetes provisions PV<br/>Creates Azure Disk]
        ProvisionPV --> AttachPod[4. Pod starts<br/>Disk attached to Node]
        AttachPod --> DataPersists[5. Data persists<br/>even if Pod restarts]
    end
    
    subgraph "Key Concepts"
        Concepts[Storage Concepts]
        Concepts --> C1[Storage Class: Template<br/>for dynamic provisioning]
        Concepts --> C2[PVC: Storage request<br/>by application]
        Concepts --> C3[PV: Actual storage<br/>bound to PVC]
        Concepts --> C4[Volume: Mount point<br/>in Pod spec]
        Concepts --> C5[VolumeMount: Path<br/>in container filesystem]
    end
    
    style SC fill:#0078d4
    style AzureDisk fill:#0078d4
    style PV fill:#28a745
    style DataPersists fill:#28a745
```

### Understanding the Diagram

- **Storage Class**: Defines the **type of Azure Disk** (Premium SSD, Standard SSD) and provisioning parameters like **reclaim policy** and **volume binding mode**
- **Persistent Volume Claim (PVC)**: Application's **request for storage** specifying size (5Gi) and access mode (ReadWriteOnce for Azure Disks)
- **Persistent Volume (PV)**: Actual **Azure Managed Disk** automatically provisioned by Kubernetes based on PVC and Storage Class specifications
- **ConfigMap**: Stores **initialization SQL scripts** to create the `usermgmt` database schema when MySQL Pod starts
- **Init Containers**: Execute **one-time setup tasks** like running SQL scripts from ConfigMap before the main MySQL container starts
- **Volume Mount**: Mounts the **Azure Disk** at `/var/lib/mysql` inside the MySQL container to persist database files
- **ClusterIP Service**: Provides **stable internal endpoint** (mysql:3306) for the Web App to connect to MySQL without knowing Pod IPs
- **Environment Variables**: Pass **database connection details** to the Web App, enabling it to connect to MySQL using the ClusterIP Service name
- **Data Persistence**: Azure Disk ensures **data survives Pod restarts**, node failures, and redeployments - critical for stateful applications
- **Storage Lifecycle**: Shows the complete flow from **Storage Class definition** to **data persistence**, demonstrating Kubernetes dynamic provisioning

---

## Topics
1. Understand about Azure Disks
2. How we are going to use Azure Disks for Applications deployed on AKS for persistent Storage?
3. Understand best possible options available that we can configure in Storage Classess to persist our data, save cost and performance etc.

## Concepts
| Kubernetes Object  | YAML File |
| ------------- | ------------- |
| Storage Class  | 01-storage-class.yml |
| Persistent Volume Claim | 02-persistent-volume-claim.yml   |
| Config Map  | 03-UserManagement-ConfigMap.yml  |
| Deployment | 04-mysql-deployment.yml  |
| Environment Variables | 04-mysql-deployment.yml  |
| Volumes  | 04-mysql-deployment.yml  |
| VolumeMounts  | 04-mysql-deployment.yml  |
| ClusterIP Service  | 05-mysql-clusterip-service.yml  |
| Deployment  | 06-UserMgmtWebApp-Deployment.yml  |
| Environment Variables| 06-UserMgmtWebApp-Deployment.yml |
| Init Containers  | 06-UserMgmtWebApp-Deployment.yml  |
| Load Balancer Service  | 07-UserMgmtWebApp-Service.yml  |


