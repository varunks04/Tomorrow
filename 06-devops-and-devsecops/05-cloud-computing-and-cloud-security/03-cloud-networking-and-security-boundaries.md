# Cloud Networking, VPC Architecture, and Security Boundaries

## 1. Topic & Definition
**Cloud Networking** defines the virtualized Software-Defined Network (SDN) topology that encapsulates, segments, and protects cloud workloads. In Amazon Web Services (AWS), this boundary is the **Virtual Private Cloud (VPC)** (in Azure, a **Virtual Network / VNet**).

Securing cloud networking requires implementing a defense-in-depth perimeter: segregating workloads across distinct **Subnet Tiers** (Public, Private, Isolated), enforcing micro-segmentation via **Security Groups and Network ACLs**, eliminating public IP exposure via **PrivateLink / VPC Endpoints**, and maintaining forensic visibility via **VPC Flow Logs**.

---

## 2. How It Works: Multi-Tier VPC Network Architecture

```
+---------------------------------------------------------------------------------------------------+
|                              Multi-Tier Secure VPC Architecture                                   |
+---------------------------------------------------------------------------------------------------+
  INTERNET
     |
  [ Internet Gateway (IGW) ]
     |
  +--+----------------------------------- AWS VPC (10.0.0.0/16) -----------------------------------+
  |  |                                                                                              |
  |  +---> [ PUBLIC SUBNET: 10.0.1.0/24 ]                                                           |
  |        - Public Application Load Balancer (ALB)                                                 |
  |        - Managed NAT Gateway (Allocated Elastic IP)                                             |
  |        (Route Table: 0.0.0.0/0 ---> Target: IGW)                                                |
  |              |                                                                                  |
  |              v  (Internal Traffic via Private IP)                                               |
  |        [ PRIVATE SUBNET: 10.0.2.0/24 ]                                                          |
  |        - Backend Microservice API (EC2 / ECS / EKS Pods)                                        |
  |        - NO Public IP addresses!                                                                |
  |        (Route Table: 0.0.0.0/0 ---> Target: NAT Gateway)                                        |
  |              |                                                                                  |
  |              v  (Strictly Internal DB Traffic: Port 5432)                                       |
  |        [ ISOLATED DATABASE SUBNET: 10.0.3.0/24 ]                                                |
  |        - Amazon Aurora / RDS PostgreSQL Database Cluster                                        |
  |        - NO Internet Gateway route! NO NAT Gateway route!                                       |
  |        (Route Table: 10.0.0.0/16 ---> Local only; COMPLETELY AIR-GAPPED FROM INTERNET)          |
  +-------------------------------------------------------------------------------------------------+
```

### The Three Subnet Tiers Defined
1. **Public Subnet:** Associated with a route table directing default outbound traffic (`0.0.0.0/0`) directly to an **Internet Gateway (IGW)**. Hosts public load balancers and NAT gateways.
2. **Private Subnet:** Hosts application servers. Lacks direct routes to an IGW. To download external patches or call third-party APIs, traffic routes outbound through the **NAT Gateway** in the public subnet. Inbound connections from the internet are **physically impossible**.
3. **Isolated (Data) Subnet:** Contains sensitive databases (RDS). Possesses **zero internet routes** (no IGW, no NAT Gateway). Completely immune to external internet ingress and egress.

---

## 3. Defense-in-Depth Filtering: Security Groups vs Network ACLs

```
[ Incoming Network Packet ]
            |
            v
  +-------------------------------------------------------------------------+
  | LAYER 1: Network ACL (NACL) - Evaluated at Subnet Boundary              |
  | - Stateless: Must evaluate rules in numerical order                     |
  | - Explicit ALLOW and DENY rules                                         |
  +-------------------------------------------------------------------------+
            | (If Allowed by NACL)
            v
  +-------------------------------------------------------------------------+
  | LAYER 2: Security Group (SG) - Evaluated at Virtual ENI / Instance Level|
  | - Stateful: Return traffic automatically permitted                      |
  | - ALLOW rules only (Implicit deny by default)                           |
  +-------------------------------------------------------------------------+
            | (If Allowed by SG)
            v
[ Workload / EC2 Instance Process ]
```

---

## 4. Key Differences Matrix: Security Groups vs Network ACLs

| Dimension | Security Groups (SGs) | Network Access Control Lists (NACLs) |
| :--- | :--- | :--- |
| **Operates At** | **Instance / Elastic Network Interface (ENI)**| **Subnet Boundary** |
| **State Tracking** | **Stateful** (Inbound allowed $\to$ outbound reply auto-allowed)| **Stateless** (Must explicitly open ephemeral return ports)|
| **Rule Types** | **ALLOW rules only** (Implicit deny by default) | **ALLOW and DENY rules** |
| **Rule Processing**| Evaluates ALL rules before making a decision | Evaluates rules in **strict numerical order** (First match wins)|
| **Ephemeral Port Handling**| Automatic | Must explicitly permit ports $1024 - 65535$ for replies |
| **Best Used For** | Micro-segmentation between application tiers | Coarse-grained subnet boundaries & blocking malicious IPs |

---

## 5. Private Cloud Access: AWS PrivateLink & VPC Endpoints

### The Data Exfiltration Problem with Public Cloud APIs
By default, when an EC2 instance in a private subnet communicates with AWS services like Amazon S3 or DynamoDB, traffic travels across the internet via the NAT Gateway.
- **Risk:** High NAT Gateway data transfer costs, and compromised instances can route exfiltration traffic to arbitrary public S3 buckets.

### The Solution: VPC Endpoints
1. **Gateway Endpoints (S3 and DynamoDB):** Modifies the VPC route table to route traffic directly to S3 across the private AWS backbone without touching the internet or incurring NAT costs.
2. **Interface Endpoints (AWS PrivateLink):** Provisions an Elastic Network Interface (ENI) with a private IP directly inside your private subnet for any AWS or SaaS service. Traffic stays 100% inside the private AWS software-defined network.

---

## 6. Network Observability: VPC Flow Logs

VPC Flow Logs capture metadata for all IP network traffic entering, leaving, or flowing between network interfaces within the VPC:

```
<version> <account-id> <interface-id> <srcaddr> <dstaddr> <srcport> <dstport> <protocol> <packets> <bytes> <start> <end> <action> <log-status>
2 123456789012 eni-0a1b2c3d 203.0.113.195 10.0.2.15 44123 22 6 1 40 1775836800 1775836860 REJECT OK
```

### Critical SOC Threat Detection Patterns in Flow Logs:
- **`action == "REJECT"` on Port 22 / 3389:** Detects port scans and external brute-force attempts blocked by Security Groups.
- **High `bytes` outbound to abnormal external IP:** Detects data exfiltration events.
- **High packet count with single byte payloads:** Detects SYN floods or port enumeration.

---

## 7. Interview Takeaways & Rapid-Fire Q&A

### 60-Second Elevator Pitch
> *"Cloud network security enforces defense-in-depth through multi-tier subnet topologies and layered packet filtering. Workloads are strictly partitioned into Public Subnets for internet load balancers, Private Subnets for application logic routing outbound via NAT Gateways, and Isolated Subnets with zero internet connectivity for sensitive databases. Traffic is filtered statefully at the virtual network interface by Security Groups and statelessly at the subnet boundary by Network ACLs. To eliminate internet exposure entirely, architectures adopt AWS PrivateLink VPC Endpoints, keeping internal API traffic on the private cloud backbone. Finally, VPC Flow Logs provide essential observability, capturing source/destination IPs, ports, and ACCEPT/REJECT verdicts for threat hunting and exfiltration detection."*

### Candidate Traps & Common Mistakes
- **Trap 1:** Assuming Security Groups and NACLs are evaluated in the same way. *Correction:* SGs are stateful (track connections and auto-allow replies) and apply to instances; NACLs are stateless (require explicit ephemeral port rules for replies) and apply to subnets.
- **Trap 2:** Placing databases in a public subnet with a security group "blocking" traffic. *Correction:* Human error or misconfiguration in an SG could expose the DB globally; databases belong in Isolated Subnets with zero route to an Internet Gateway.
- **Trap 3:** Confusing VPC Peering with Transit Gateway. *Correction:* VPC Peering connects two VPCs point-to-point without transitive routing; AWS Transit Gateway acts as a centralized cloud router connecting thousands of VPCs in a scalable hub-and-spoke topology.

### Expected Follow-Up Questions
1. *Why does a Network ACL require opening outbound ephemeral ports ($1024 - 65535$)?*
   - Because NACLs are stateless. When an EC2 instance initiates an outbound HTTP request on port 80, the responding web server sends return packets to a randomly chosen client ephemeral port. Since NACLs do not remember the initial outbound request, return traffic is dropped unless ephemeral ports are explicitly permitted.
2. *What is the difference between a NAT Instance and a NAT Gateway?*
   - A NAT Instance is a customer-managed EC2 instance running Linux iptables (requires manual OS patching, scaling, and high-availability configuration); a NAT Gateway is a fully managed, redundant AWS cloud service that scales automatically up to 100 Gbps.
