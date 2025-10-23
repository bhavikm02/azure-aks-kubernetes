# Azure Virtual Nodes for Azure Kubernetes Service

## 📊 Architecture & Workflow Diagram

```mermaid
graph TB
    subgraph "AKS Cluster with Virtual Nodes"
        AKS[AKS Cluster<br/>with Azure CNI]
        
        AKS --> NodePool[Regular Node Pool<br/>Azure VMs<br/>System + User Pods]
        AKS --> VirtualNode[Virtual Node<br/>virtual-node-aci-linux<br/>Serverless]
        
        VirtualNode --> VirtualKubelet[Virtual Kubelet<br/>ACI Connector Linux]
        VirtualKubelet --> ACI[Azure Container Instances<br/>Serverless Containers<br/>Pay-per-second]
    end
    
    subgraph "Pod Scheduling Logic"
        CreatePod[Create Pod]
        CreatePod --> HasNodeSelector{Has nodeSelector<br/>+ tolerations?}
        
        HasNodeSelector -->|No| ScheduleVM[Schedule to<br/>Regular Node Pool<br/>Azure VMs]
        
        HasNodeSelector -->|Yes| CheckVirtualNode{Virtual Node<br/>selector?}
        CheckVirtualNode -->|Yes| ScheduleACI[Schedule to<br/>Virtual Node<br/>Azure ACI]
        CheckVirtualNode -->|No| ScheduleVM
        
        ScheduleVM --> VMPod[Pod runs on VM<br/>Persistent node<br/>Shared resources]
        ScheduleACI --> ACIPod[Pod runs on ACI<br/>Isolated container<br/>Dedicated resources]
    end
    
    subgraph "Virtual Node Configuration"
        PodSpec[Pod Specification]
        PodSpec --> NodeSel[nodeSelector:<br/>  type: virtual-kubelet<br/>  kubernetes.io/role: agent<br/>  beta.kubernetes.io/os: linux]
        PodSpec --> Tolerations[tolerations:<br/>  - key: virtual-kubelet.io/provider<br/>  - key: azure.com/aci]
    end
    
    subgraph "Virtual Kubelet Architecture"
        VK[Virtual Kubelet]
        VK --> Provider[ACI Provider<br/>Translates K8s API to ACI API]
        Provider --> CreateContainer[Create ACI Container Group]
        CreateContainer --> Monitor[Monitor Container Status]
        Monitor --> ReportBack[Report back to K8s API]
    end
    
    subgraph "Use Cases"
        UseCases[Virtual Nodes Use Cases]
        UseCases --> UC1[Burst Workloads:<br/>Handle traffic spikes<br/>without pre-provisioning VMs]
        UseCases --> UC2[CI/CD Pipelines:<br/>Ephemeral build agents<br/>Pay-per-job]
        UseCases --> UC3[Batch Processing:<br/>Short-lived jobs<br/>Fast startup]
        UseCases --> UC4[Cost Optimization:<br/>No idle VM costs<br/>Pay only when running]
        UseCases --> UC5[Fast Scaling:<br/>Start Pods in seconds<br/>No node provisioning wait]
    end
    
    subgraph "Comparison"
        Comparison[Regular Nodes vs Virtual Nodes]
        Comparison --> RegularNodes[Regular Nodes:<br/>✓ Full K8s features<br/>✓ Persistent storage<br/>✓ Network policies<br/>❌ Fixed capacity<br/>❌ Pay for idle VMs]
        Comparison --> VirtualNodes[Virtual Nodes:<br/>✓ Unlimited scale<br/>✓ Fast startup<br/>✓ Pay-per-second<br/>❌ Limited K8s features<br/>❌ No persistent volumes]
    end
    
    style VirtualKubelet fill:#326ce5
    style ACI fill:#00d1b2
    style ScheduleACI fill:#28a745
```

### Understanding the Diagram

- **Virtual Nodes**: Enable **serverless Kubernetes** by scheduling Pods on **Azure Container Instances (ACI)** instead of traditional VMs
- **Virtual Kubelet**: Acts as a **bridge** between Kubernetes API and Azure Container Instances, translating K8s Pod specs to ACI containers
- **Azure CNI Required**: Virtual Nodes require **Azure CNI networking** to integrate ACI containers into the same virtual network as AKS nodes
- **NodeSelector + Tolerations**: Pods must specify **virtual-kubelet nodeSelector and tolerations** to be scheduled on Virtual Nodes instead of regular nodes
- **Burst Scaling**: Virtual Nodes enable **unlimited, instant scale-out** for workloads without waiting for VM provisioning or paying for idle capacity
- **Pay-per-Second Billing**: With ACI, you pay only for **actual container runtime** (per-second billing), not for idle VM capacity
- **Fast Startup**: Pods on Virtual Nodes start in **seconds**, not minutes, as they don't require VM provisioning
- **Mixed Mode**: Can run **regular Pods on VMs** for stateful workloads and **burst Pods on ACI** for ephemeral/short-lived tasks
- **Limitations**: Virtual Nodes don't support **persistent volumes, DaemonSets, privileged containers**, or some advanced K8s features
- **Ideal for CI/CD**: Perfect for **ephemeral build agents, batch jobs, and burst workloads** where fast provisioning and cost efficiency matter

---

1. What is [Virtual Kubelet](https://github.com/virtual-kubelet/virtual-kubelet)?
2.  What is [Azure Container Instances - ACI](https://docs.microsoft.com/en-us/azure/container-instances/)?
3. What are [AKS Virtual Nodes](https://docs.microsoft.com/en-us/azure/aks/virtual-nodes-portal)?
- **Important Note:** Virtual nodes require AKS clusters with [Azure CNI networking](https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni)
4. Understand and implement basic usecase using Azure Virtual Nodes
5. **Advanced Implementation:** Implement Mixed Mode Deployments with Azure Virtual Nodes and Azure AKS Nodepools
