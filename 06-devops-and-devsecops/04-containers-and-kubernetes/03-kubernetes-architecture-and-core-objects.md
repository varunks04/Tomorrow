# Kubernetes Architecture, Control Plane Internals, and Core Objects

## 1. Topic & Definition
**Kubernetes (K8s)** is an open-source container orchestration platform designed to automate the deployment, scaling, management, and self-healing of containerized applications across distributed clusters of physical or virtual machines.

Originally designed by Google based on its internal Borg system, Kubernetes operates on a **declarative reconciliation model**: administrators declare the "desired state" of the infrastructure in YAML manifests, and internal control loops continuously drive the "actual state" of the cluster to match that desired specification.

---

## 2. How It Works: Control Plane and Worker Node Architecture

```
+---------------------------------------------------------------------------------------------------+
|                                  Kubernetes Cluster Architecture                                  |
+---------------------------------------------------------------------------------------------------+

   +-------------------------------- CONTROL PLANE (Master Nodes) -------------------------------+
   |                                                                                             |
   |   [ kubectl / API Clients ]                                                                 |
   |              |                                                                              |
   |              v                                                                              |
   |   +--------------------+  Watches / Updates  +--------------------------+                   |
   |   |   kube-apiserver   |<===================>|        etcd Store        | (Port 2379)       |
   |   | (AuthN/AuthZ/Valid)|                     | (Raft Consensus Key-Val) |                   |
   |   +--------------------+                     +--------------------------+                   |
   |         |         ^                                                                         |
   |         |         +--------+----------------------------+                                   |
   |         v                  |                            |                                   |
   |   +-------------------+    |                  +-------------------+                         |
   |   |  kube-scheduler   |----+                  |  kube-controller  |                         |
   |   | (Node Placement)  |                       |      manager      |                         |
   |   +-------------------+                       +-------------------+                         |
   +---------------------------------------------------------------------------------------------+
                                     |
                (TLS gRPC Communications over Port 10250)
                                     v
   +--------------------------------- WORKER NODES (Data Plane) ---------------------------------+
   |                                                                                             |
   |   +------------------------------------+    +------------------------------------+          |
   |   | Worker Node 01                     |    | Worker Node 02                     |          |
   |   |                                    |    |                                    |          |
   |   |  +------------+   +------------+   |    |  +------------+   +------------+   |          |
   |   |  |  kubelet   |   | kube-proxy |   |    |  |  kubelet   |   | kube-proxy |   |          |
   |   |  +------------+   +------------+   |    |  +------------+   +------------+   |          |
   |   |         |               |          |    |         |               |          |          |
   |   |         v (CRI)         v (iptables|    |         v (CRI)         v (iptables|          |
   |   |  +---------------------------+     |    |  +---------------------------+     |          |
   |   |  | containerd / CRI-O Runtime|     |    |  | containerd / CRI-O Runtime|     |          |
   |   |  |  [ Pod A ]    [ Pod B ]   |     |    |  |  [ Pod C ]    [ Pod D ]   |     |          |
   |   |  +---------------------------+     |    |  +---------------------------+     |          |
   |   +------------------------------------+    +------------------------------------+          |
   +---------------------------------------------------------------------------------------------+
```

### A. The Control Plane Components
1. **`kube-apiserver` (The Front Door):** The central REST API server. All cluster communication—from internal controllers, worker nodes, and administrators—must pass through the apiserver. It executes Authentication, Authorization (RBAC), and Admission Control.
2. **`etcd` (The Source of Truth):** A highly consistent, distributed key-value store using the Raft consensus algorithm (Port 2379). It stores the entire cluster state, specifications, and secrets. **If `etcd` is compromised or lost, the entire cluster is lost**.
3. **`kube-scheduler`:** Evaluates unscheduled Pods and assigns them to optimal worker nodes based on resource requests/limits, node affinity/anti-affinity, taints, and tolerations.
4. **`kube-controller-manager`:** A daemon that runs core control reconciliation loops (NodeController, DeploymentController, EndpointSliceController, ServiceAccountController).

### B. The Worker Node Components
1. **`kubelet`:** The primary node agent running on every worker node. It registers the node with the apiserver, monitors node health, and instructs the container runtime to pull images and start/stop containers to match assigned `PodSpec` declarations (listens on Port 10250).
2. **`kube-proxy`:** Manages network routing rules (via Linux `iptables` or `IPVS`) on every node, enabling load-balanced routing to backend pods across Kubernetes Services.
3. **Container Runtime (CRI):** The low-level runtime (`containerd`, `CRI-O`) that executes OCI container lifecycles via `runc`.

---

## 3. Core Kubernetes Objects and Abstractions

```
+------------------------------------+---------------------------------------------------------------+
| Kubernetes Object                  | Architectural Definition & Responsibility                     |
+------------------------------------+---------------------------------------------------------------+
| **Pod**                            | The smallest deployable unit in K8s. A group of one or more   |
|                                    | containers sharing network namespace (IP/localhost) & storage.|
| **Deployment**                     | Declarative controller managing Pod rollout, self-healing,   |
|                                    | replicas, and zero-downtime rolling updates via ReplicaSets.  |
| **Service (ClusterIP / NodePort)** | Stable virtual IP and internal DNS name load-balancing        |
|                                    | traffic across ephemeral, dynamically changing pod IPs.       |
| **ConfigMap & Secret**             | Injects non-sensitive configs and Base64 secrets into pods    |
|                                    | as environment variables or mounted volume files.             |
| **Namespace**                      | Logical partitioning mechanism for naming and quota isolation |
|                                    | (NOTE: Namespaces provide NO network isolation by default!).  |
+------------------------------------+---------------------------------------------------------------+
```

### The Kubernetes Networking Model
Kubernetes enforces fundamental networking requirements:
1. **Every Pod receives its own unique IP address** within the cluster pod CIDR.
2. **All Pods can communicate with all other Pods** across any node directly without Network Address Translation (NAT).
3. Agents on a node (e.g., `kubelet`) can communicate with all Pods on that node.

---

## 4. Key Differences Matrix: Service Types

| Service Type | Scope & Routing | Assigned Address | Primary Use Case |
| :--- | :--- | :--- | :--- |
| **ClusterIP** | **Internal Only** (Default) | Virtual internal IP from Service CIDR | Internal microservice-to-microservice traffic |
| **NodePort** | External access via Node IP | High static port ($30000 - 32767$) on each node | Development / Direct node routing |
| **LoadBalancer** | **Public External** | Provisions cloud provider Load Balancer (AWS NLB/ALB) | Production internet-facing applications |
| **ExternalName** | Internal DNS Alias | CNAME record mapping to external hostname | Connecting pods to external third-party DBs |

---

## 5. Cybersecurity Relevance, Threats & Misconfigurations

### 1. The Namespace Isolation Illusion
- **The Misconception:** Many engineering teams believe creating separate Namespaces (e.g., `production` vs `development`) isolates workloads.
- **The Security Reality:** **Namespaces provide ZERO security or network isolation by default**. A compromised pod in the `development` namespace can freely ping, port scan, and query the REST APIs of payment pods in the `production` namespace across the flat cluster network unless **NetworkPolicies** are explicitly created!

### 2. Exposed Kubelet API (Port 10250)
- **Vulnerability:** Leaving `kubelet` API Port 10250 unauthenticated (`anonymous-auth: true`).
- **Exploitation:** An attacker on the local network executes commands directly inside running pods without touching the `kube-apiserver`:
  ```bash
  curl -k -X POST https://<node-ip>:10250/run/<namespace>/<pod>/<container> -d "cmd=id"
  ```
- **Defense:** Set `--anonymous-auth=false` and enforce webhook authorization on all kubelets.

### 3. Compromise of the `etcd` Store (Port 2379)
- **Vulnerability:** Unauthenticated access or cleartext storage in `etcd`.
- **Exploitation:** Anyone who queries `etcd` directly can read every Secret, API token, and service account key in the entire cluster.
- **Defense:** Restrict `etcd` access via mutual TLS (mTLS) to `kube-apiserver` only; enable KMS encryption at rest.

---

## 6. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Kubernetes is a distributed container orchestration platform based on a declarative control plane and distributed worker nodes. The Control Plane consists of `kube-apiserver` as the central authenticated gateway, `etcd` as the single source of truth storing cluster state, `kube-scheduler` for intelligent pod-to-node placement, and `kube-controller-manager` driving reconciliation loops. Worker nodes run `kubelet` to manage container lifecycles via the Container Runtime Interface (CRI), and `kube-proxy` for service routing. In cybersecurity, understanding the K8s flat network model is crucial: by default, all pods can communicate across nodes and namespaces with zero restrictions. Consequently, securing a cluster requires implementing explicit NetworkPolicies, restricting kubelet port 10250, and enforcing RBAC at the API server."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Assuming Kubernetes Namespaces block network traffic between pods. *Correction:* Namespaces provide logical naming and resource quota partitioning; they do not restrict network routing without NetworkPolicies.
- **Trap 2:** Confusing Pods with Containers. *Correction:* A Pod is an abstraction that wraps one or more containers that share the exact same network namespace (IP, localhost), IPC namespace, and storage volumes.
- **Trap 3:** Storing cleartext secrets in etcd. *Correction:* Kubernetes Secrets are only Base64-encoded by default, not encrypted; `etcd` must have EncryptionConfiguration enabled with KMS.

### Expected Follow-Up Questions
1. *What is the role of `kube-proxy`?*
   - `kube-proxy` watches the apiserver for Service and EndpointSlice changes and writes corresponding Linux `iptables` or `IPVS` packet filtering rules on the worker node to route traffic destined for a virtual Service ClusterIP to actual backend pod IPs.
2. *Why is `etcd` considered the ultimate prize for a Kubernetes attacker?*
   - Because `etcd` stores the entire database of the cluster in plaintext (unless KMS encryption is configured), including all passwords, service account tokens, and cluster configurations; compromising `etcd` equals full cluster administrative takeover.
