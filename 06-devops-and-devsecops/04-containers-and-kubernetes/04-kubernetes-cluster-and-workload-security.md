# Kubernetes Cluster, Workload Security, RBAC, and Network Policies

## 1. Topic & Definition
Kubernetes workload and cluster security requires defense-in-depth across the **4C's of Cloud Native Security: Cloud, Cluster, Container, and Code**.

Securing a Kubernetes cluster spans two operational boundaries:
1. **Control Plane Security:** Authenticating and authorizing requests to the `kube-apiserver` via **Role-Based Access Control (RBAC)**, enforcing admission policies, and encrypting `etcd`.
2. **Data Plane / Workload Security:** Restricting what pods can do at runtime via **Pod Security Standards (PSS)** and controlling inter-pod communication via **Network Policies**.

---

## 2. How It Works: The API Server Request Processing Pipeline

```
+---------------------------------------------------------------------------------------------------+
|                         The Kubernetes API Server Request Lifecycle                               |
+---------------------------------------------------------------------------------------------------+
  API Request (User / Pod ServiceAccount)
        |
        v
  [ 1. AUTHENTICATION (AuthN) ]     --> X.509 Client Certs, OIDC Bearer Tokens, Webhooks
        | (Identity verified: e.g., "system:serviceaccount:prod:billing-sa")
        v
  [ 2. AUTHORIZATION (AuthZ) ]      --> RBAC (Roles, ClusterRoles) & Node Authorizer
        | (Permitted to perform: "create pods in namespace prod"?)
        v
  [ 3. MUTATING ADMISSION WEBHOOKS ]--> Injects sidecars, defaults configs (Kyverno / Gatekeeper)
        |
        v
  [ 4. SCHEMA VALIDATION ]          --> Validates against OpenAPI / PodSpec schema
        |
        v
  [ 5. VALIDATING ADMISSION WEBHOOKS]--> Enforces Pod Security Standards (PSA: Blocks root/privileged)
        |
        v
  [ 6. PERSIST TO ETCD ]            --> KMS-encrypted write to etcd database (Port 2379)
```

---

## 3. Kubernetes RBAC Architecture: Roles vs ClusterRoles

Kubernetes RBAC regulates access to API resources based on four core primitives:

```
+---------------------------------------------------------------------------------------------------+
|                                   Kubernetes RBAC Relationship Graph                              |
+---------------------------------------------------------------------------------------------------+
  SCOPED TO SINGLE NAMESPACE:
  [ Role: "pod-reader" ]  <==== (RoleBinding) ====>  [ Subject: User "Alice" / ServiceAccount ]
  - apiGroups: [""]
  - resources: ["pods"]
  - verbs: ["get", "list", "watch"]

  CLUSTER-WIDE SCOPE (All Namespaces + Non-namespaced resources like Nodes):
  [ ClusterRole: "node-admin" ] <== (ClusterRoleBinding) ==> [ Subject: Group "platform-engineers" ]
  - resources: ["nodes", "persistentvolumes"]
  - verbs: ["*"]
```

### Least Privilege Service Account Hardening
By default, every Pod mounts a Kubernetes JWT ServiceAccount token at `/var/run/secrets/kubernetes.io/serviceaccount/token`.
- **Attack Vector:** An attacker who achieves RCE inside a web pod reads this token file and uses it to query the `kube-apiserver`. If the service account has broad permissions, the attacker compromises the cluster.
- **Defensive Rule:** Always disable automatic token mounting unless the pod explicitly interacts with the API server:
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: hardened-web-pod
  spec:
    automountServiceAccountToken: false # PREVENTS API TOKEN INJECTION!
  ```

---

## 4. Pod Security Standards (PSS) and Admission (PSA)

Replacing deprecated PodSecurityPolicies (PSP), the built-in **Pod Security Admission (PSA)** evaluates pods against three official security levels defined by the **Pod Security Standards (PSS)**:

```
+------------------------------------+---------------------------------------------------------------+
| PSS Level                          | Security Controls Enforced                                    |
+------------------------------------+---------------------------------------------------------------+
| **Privileged**                     | Unrestricted: Allows `--privileged`, hostPID, hostNetwork, root|
| (For CNI, CSI, infra daemons only) | No security restrictions.                                     |
+------------------------------------+---------------------------------------------------------------+
| **Baseline**                       | Minimally restrictive: Prevents known privilege escalations.  |
| (Default standard for apps)        | Blocks hostNetwork, hostPID, hostIPC, capabilities like SYS_ADMIN.|
+------------------------------------+---------------------------------------------------------------+
| **Restricted**                     | **Hardened Gold Standard:** Mandates non-root execution (`USER`),|
| (Critical production apps)         | drops ALL capabilities, enforces read-only root filesystems,  |
|                                    | and restricts volume types to safe configMaps/secrets/PVCs.   |
+------------------------------------+---------------------------------------------------------------+
```

### Enforcing PSS via Namespace Labels
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production-workloads
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
```

---

## 5. Kubernetes Network Policies: The Default-Deny Isolation Model

Because the default Kubernetes network is flat and permits all inter-pod traffic, clusters must enforce **Network Policies** (requires a CNI plugin supporting policies like **Calico** or **Cilium / eBPF**).

### The Foundation: Namespace-Wide Default Deny All Policy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {} # Matches ALL pods in the namespace
  policyTypes:
  - Ingress
  - Egress
```
*Impact:* Instantly isolates all pods in the namespace. No pod can receive incoming connections, and no pod can initiate outbound egress (preventing reverse shells and C2 beaconing) until explicit allow rules are created!

### Explicit Microservice Allow Rule (Frontend to Backend API)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend-api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend-web
    ports:
    - protocol: TCP
      port: 8080
```

---

## 6. Key Differences Matrix: Kyverno vs OPA Gatekeeper

| Feature | Kyverno | OPA Gatekeeper |
| :--- | :--- | :--- |
| **Policy Language** | **Native Kubernetes YAML** | **Rego** (Declarative query language) |
| **Learning Curve** | Low (Familiar to K8s engineers) | High (Requires learning Rego) |
| **Mutation Capabilities**| Native YAML mutation (injects labels/certs)| Requires separate Mutating Webhook configs |
| **Generation Capabilities**| Can generate default NetworkPolicies per NS| Validation only; does not generate resources |
| **Ecosystem Reach** | Kubernetes-only | Broad (Kubernetes, Terraform, Envoy, Cloud) |

---

## 7. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Securing Kubernetes workloads requires defense-in-depth across the API control plane and container data plane. Access to the `kube-apiserver` must be strictly bounded by Role-Based Access Control (RBAC) following the principle of least privilege, while disabling `automountServiceAccountToken` on pods that do not interact with the API. At the workload level, Pod Security Admission (PSA) enforces the 'Restricted' Pod Security Standard to block root execution, mandate read-only root filesystems, and drop dangerous Linux capabilities. Because Kubernetes defaults to an unrestricted flat network, implementing CNI NetworkPolicies (via Calico or Cilium) with a 'Default-Deny-All' baseline is mandatory to prevent lateral movement and block outbound attacker C2 beaconing."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Assuming Kubernetes NetworkPolicies work without a compatible CNI. *Correction:* Standard basic kubenet or AWS VPC CNI without NetworkPolicy controllers silently ignores NetworkPolicy YAML manifests; a policy-enforcing CNI like Calico or Cilium is required.
- **Trap 2:** Granting wildcards (`*`) in RBAC roles. *Correction:* Granting `verbs: ["*"]` on `resources: ["*"]` or granting access to `secrets` or `pods/exec` amounts to cluster admin takeover.
- **Trap 3:** Confusing RoleBinding with ClusterRoleBinding. *Correction:* RoleBinding binds permissions strictly within a specific namespace; ClusterRoleBinding applies permissions globally across every namespace in the cluster.

### Expected Follow-Up Questions
1. *What is the security danger of granting `pods/exec` create permissions in RBAC?*
   - Anyone with `create` permissions on `pods/exec` can run `kubectl exec` to spawn an interactive root shell inside running application pods, bypassing pod code restrictions to extract in-memory secrets and pivot through the network.
2. *Why is `etcd` encryption at rest critical?*
   - Kubernetes Secrets are stored in `etcd` as plain Base64-encoded strings by default. Enabling `EncryptionConfiguration` with a cloud KMS key ensures that even if physical backup snapshots of `etcd` are leaked, all secrets remain cryptographically encrypted.
