# Operator Specification: DPF-HCP Bridge Operator

The main purpose of this operator is to serve as the linkage between the DPF and Hypershift.
It abstracts the complexity of Hypershift management from the end-user, treating the hosted cluster as a "black box".

## 1. CRD Definition
The operator will manage a single CR: DPUHCPBridge.

### 1.1 Spec Fields (Input)
These fields are provided by the user to define the desired HCP environment:

| Field Name (YAML/JSON) | Type    | Description                                                                                                                   |
|------------------------|---------|-------------------------------------------------------------------------------------------------------------------------------|
| `baseDomain` | string  | The base domain for the hosted cluster.                                                                                       |
| `sshKey` | string  | File path to the public SSH key.                                                                               |
| `pullSecret` | string  | File path to a pull secret.                                                                                                   |
| `ocpReleaseImage` | string  | The full pull-spec URL for the OCP release image to be used for the hosted cluster.                                           |
| `etcdStorageClass` | string  | The persistent volume storage class name for etcd data volumes.                                                               |
| `controlPlaneAvailabilityPolicy` | string  | Availability policy for hosted control plane components. Must be `SingleReplica` or `HighlyAvailable`.                        |
| `infrastructureAvailabilityPolicy` | string  | Availability policy for infrastructure services (data plane) in the guest cluster. Must be `SingleReplica` or `HighlyAvailable`. |
| `exposeThroughLoadBalancer` | boolean | If `true`, services are exposed via LoadBalancer. If `false` (or unset), NodePorts will be used.                              |
| `clusterType` | string  | DPUCluster type identifier.                                                                                                   |
| `metalLBVirtualIP` | string  | The virtual IP address that MetalLB will use for service exposure.                                                            |


### 1.2 Status Fields (Output)
The operator reports its progress, the state of the managed Hypershift resources, and its internal health.

#### External/Hypershift Status Fields
These fields are primarily mirrors of the underlying Hypershift HostedCluster resource.

| Field Name | Type    | Description                                                                                                           |
|-----------------------|---------|-----------------------------------------------------------------------------------------------------------------------|
| `HostedClusterAvailable` | string  | indicates whether the HostedCluster has a healthy control plane.                                                      |
| `HostedClusterProgressing` | string  | indicates whether the HostedCluster is attempting an initial deployment or upgrade.                                   |
| `HostedClusterDegraded` | string  | indicates whether the HostedCluster is encountering an error that may require user intervention to resolve.           |
| `InfrastructureReady` | string  | signals if the infrastructure for a control plane to be operational                                                   |
| `KubeAPIServerAvailable` | string  | signals if the kube API server is available                                                                           |
| `EtcdAvailable` | string  | signals if etcd is available                                                                                          |
| `ValidReleaseInfo` | string  | indicates if the release contains all the images used by hypershift and reports missing images if any.                |
| `IgnitionEndpointAvailable` | boolean | ndicates whether the ignition server for the HostedCluster is available to handle ignition requests.                  |
| `IgnitionServerValidReleaseInfo` | string  | indicates if the release contains all the images used by the local ignition provider and reports missing images if any. |
| `ValidReleaseImage` | string  | indicates if the release image set in the spec is valid for the HostedCluster                                         |
| `EtcdRecoveryActive` | string  | indicates that the Etcd cluster is failing and the recovery job was triggered.                                        |
| `HostedClusterRestoredFromBackup` | string  | indicates that the HostedCluster was restored from backup.                                          |
| `DataPlaneConnectionAvailable` | string  | indicates whether the control plane has a successful network connection to the data plane components.                 |

#### Operator-Specific Status Fields

| Field Name                         | Type    | Description                                                                                                                                                        |
|------------------------------------|---------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `BlueFieldContainerImageAvailable` | string  | indicates whether the equivalent BlueField container image for the given release image was found.                                                                  |
| `HostedClusterCleanup`             | string  | indicates the status of the associated HostedCluster deletion following the removal of the DPUHCPBridge CR. Values could include Succeeded, InProgress, or Failed. |

## 2. DPF-HCP Linkage Controller (The Reconciler)
This will be the primary controller responsible for watching the DPUHCPBridge CR and managing the lifecycle of the dependent Hypershift objects.

* **Controller Name:** DPFHCPBridgeController 
* **Watches:** The DPUHCPBridge Custom Resource. 
* **Key Responsibilities (Reconciliation Logic):**
  * **HostedCluster Management:** Manages the full lifecycle (creation, updates, deletion) of the Hypershift resources, driven by changes to the DPUHCPBridge CR.
  * **Post-Creation Kubeconfig Injection:** Once the HostedCluster is successfully created:
    * The controller extracts the HostedCluster's kubeconfig. 
    * This kubeconfig is used to create a Kubernetes Secret in the same namespace as the related DPUCluster object. 
    * The controller then modifies the related DPUCluster CR to add the reference to this new Secret in its .spec.
  * **CSR Approval Logic** (Automated Joining):
    * The controller actively monitors the hosted cluster for new CSRs from worker nodes joining with the same username as the DPU name. 
    * Once the worker node reaches the Installed state, the controller automatically approves its CSRs, ensuring it can successfully join the cluster. 
    * **Minimal Security Mechanism:** Upon receiving additional CSRs from an already-known hostname, the controller must validate that the previously approved DPU remains active and has joined the hosted cluster. Subsequent CSRs from that hostname must not be approved if the DPU is already known and active.
  * **MetalLB Deployment:** Deploys and manages the necessary MetalLB CRs (like IPAddressPools and L2Advertisements) to provide software load balancing.
  * **Ignition Override:** Override the default OS image in the Hypershift ignition configuration using MachineConfig.

## 3. Relationships and Constraints
### 3.1 Resource Relationship
The relationship between DPUCluster, DPUHCPBridge, and HostedCluster is strictly 1:1:1. The controller must enforce this relationship.

### 3.2 Deletion Behavior (Critical Decision Point)
* If the DPUHCPBridge object is deleted, the controller MUST attempt to delete the associated HostedCluster.
* If the DPUCluster is deleted, the controller should (need to decide how it's best to handle this)
* If the associated HostedCluster fails to be removed, the DPUHCPBridge resource status should be updated to reflect a HostedClusterCleanup condition.

## 4. Validations
### 4.1 Validating Admission Webhook
The operator requires a Validating Admission Webhook to perform pre-persistence checks.
* **Requirement 1 (Required Fields):** Validate that all required fields in the DPUHCPBridge CR are correct and present before the object is persisted. 
* **Requirement 2 (Image Availability):** Validate that the equivalent BlueField container image is available within the OCP-BlueField ConfigMap before allowing HostedCluster creation. If the image is missing, the admission webhook MUST block the resource creation and return an error (condition BlueFieldContainerImageAvailable).

## 5. Prerequisites and Deployment
### 5.1 Deployment Prerequisites (MGMT Cluster)
The following must be deployed and running on the Management Cluster before deploying this operator:
* **MCE Operator:** Required to deploy and manage the Hypershift Operator. 
* **DPF Operator:** Required for DPU management. 
* **Hypershift Operator:** Required for Hosted Cluster provisioning. 
* **ODF Operator:** Necessary to provide persistent storage for the Hosted Control Plane’s etcd. 
* **MetalLB Operator:** Required to provide the stable VIP for the HA Hypershift control plane.

### 5.2 Deployment Method
The operator is required to develop a Helm chart for deployment on the Management Cluster.

## 6. Future Scope (Next Steps)
* This controller, or a separate component, should manage the worker configurations to minimize manual user actions during the DPU installation process. This may necessitate a name change for the operator.
* RHCOS Ignition/BFB.cfg Generator