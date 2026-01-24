# Cross-Cluster Pod Communication in Nephio: A Complete Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Basics-Part1](#Basics-Part1)
3. [The Scenario](#the-scenario)
4. [Cross-Cluster Communication with MacVLAN](#cross-cluster-communication-with-macvlan)
5. [The IPAM Challenge](#the-ipam-challenge)
6. [IPAM Solutions Comparison](#ipam-solutions-comparison)
7. [IP Persistence Challenge](#ip-persistence-challenge)
8. [Nephio's Architecture](#nephios-architecture)
9. [Complete Packet Flow Example](#complete-packet-flow-example)
10. [Key Takeaways](#key-takeaways)
11. [SD-Core Specific](#SD-Core's-Network-Interface-Configuration)
12. [Basics-ChatGPT](#Basics-ChatGPT)
13. [CNI Plugin Hierarcy](#CNI-Plugin-Hierarchy)
14. [NADs](#NADs)

---

## Introduction

In telco/5G deployments using Nephio, workloads are distributed across multiple Kubernetes clusters. Understanding how pods communicate across these clusters and maintain stable IP addresses is critical for reliable network function operation.

**Key Questions This Guide Answers:**
- How do pods in different clusters communicate?
- How are IP addresses managed across clusters?
- How do pods maintain the same IP after restarts/upgrades?
- How does Nephio manage all of this without direct access to workload clusters?

---
## Basics-Part1

### Two Separate IP Address Spaces

#### Primary Network (Kubernetes-managed)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: amf-0
  namespace: core-amf
spec:
  containers:
  - name: amf
    image: amf:v1
```

When this pod is created:
```bash
$ kubectl get pod amf-0 -o wide
NAME    READY   STATUS    IP           NODE
amf-0   1/1     Running   10.244.1.5   core-node-1
                          ↑
                          This is the Kubernetes pod IP
```

**This IP (10.244.1.5):**
- ✅ Managed by Kubernetes CNI (Calico, Flannel, etc.)
- ✅ Used for cluster-internal communication
- ✅ Used by Services (ClusterIP, NodePort)
- ✅ Part of Kubernetes pod CIDR (e.g., 10.244.0.0/16)
- ✅ Appears in `kubectl get pods`
- ✅ Used for readiness/liveness probes
- ✅ Used for Pod-to-Pod communication within cluster

#### Secondary Network (MacVLAN - NOT Kubernetes-managed)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: amf-0
  namespace: core-amf
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [{
        "name": "n2-network",
        "interface": "n2",
        "ips": ["192.168.100.100/24"]
      }]
spec:
  containers:
  - name: amf
    image: amf:v1
```

Inside the pod:
```bash
$ kubectl exec -it amf-0 -- ip addr show

1: lo: <LOOPBACK,UP,LOWER_UP>
    inet 127.0.0.1/8 scope host lo

2: eth0@if10: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 10.244.1.5/24 brd 10.244.1.255 scope global eth0
    ↑
    Kubernetes-managed (primary interface)

3: n2@if5: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.100.100/24 brd 192.168.100.255 scope global n2
    ↑
    MacVLAN (secondary interface) - NOT managed by Kubernetes!
```

**This IP (192.168.100.100):**
- ❌ NOT managed by Kubernetes
- ❌ NOT part of pod CIDR
- ❌ NOT visible in `kubectl get pods -o wide`
- ❌ NOT used by Kubernetes Services
- ❌ NOT used for health checks (unless explicitly configured)
- ✅ Managed by MacVLAN IPAM (host-local, whereabouts, static, etc.)
- ✅ Used for external/cross-cluster communication
- ✅ Directly routable on physical network

---

### Visual Comparison

```
┌─────────────────────────────────────────────────────────┐
│                     Pod: amf-0                          │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Application Container                           │  │
│  │                                                   │  │
│  │  Listens on multiple interfaces:                 │  │
│  │  - 10.244.1.5:8080  (Kubernetes network)        │  │
│  │  - 192.168.100.100:38412  (N2 interface)        │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  Network Interfaces:                                   │
│  ┌─────────────────┐        ┌──────────────────┐      │
│  │ eth0            │        │ n2               │      │
│  │ (Primary)       │        │ (Secondary)      │      │
│  │                 │        │                  │      │
│  │ 10.244.1.5/24   │        │ 192.168.100.100  │      │
│  │                 │        │                  │      │
│  │ K8s managed ✅  │        │ MacVLAN managed  │      │
│  └────────┬────────┘        └────────┬─────────┘      │
└───────────┼──────────────────────────┼─────────────────┘
            │                          │
            │                          │
    ┌───────▼────────┐         ┌───────▼────────┐
    │ Kubernetes CNI │         │ MacVLAN CNI    │
    │ (e.g., Calico) │         │ (via Multus)   │
    └───────┬────────┘         └───────┬────────┘
            │                          │
    ┌───────▼────────┐         ┌───────▼────────┐
    │ veth pair      │         │ Physical NIC   │
    │ (virtual)      │         │ eth1 (real)    │
    └───────┬────────┘         └───────┬────────┘
            │                          │
    ┌───────▼────────┐         ┌───────▼────────┐
    │ Node bridge    │         │ Physical       │
    │ cni0           │         │ Network        │
    └────────────────┘         └────────────────┘
```

---

### IP Address Space Breakdown

#### Kubernetes IP Ranges (Managed)

```yaml
# Kubernetes cluster configuration
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
networking:
  podSubnet: 10.244.0.0/16        # Pod IPs
  serviceSubnet: 10.96.0.0/12     # Service IPs
```

**These are managed by Kubernetes:**
- **Pod CIDR:** `10.244.0.0/16`
  - Example: `10.244.1.5`, `10.244.2.10`, etc.
  - Allocated by Kubernetes CNI
  
- **Service CIDR:** `10.96.0.0/12`
  - Example: `10.96.0.1` (kubernetes service)
  - Example: `10.96.5.10` (your custom service)
  - Created with `kubectl expose` or Service resources

#### MacVLAN IP Ranges (NOT Managed by Kubernetes)

```yaml
# NetworkAttachmentDefinition
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: n2-network
spec:
  config: |
    {
      "type": "macvlan",
      "ipam": {
        "type": "host-local",
        "ranges": [[{
          "subnet": "192.168.100.0/24",    # NOT in Kubernetes CIDR
          "rangeStart": "192.168.100.10",
          "rangeEnd": "192.168.100.200"
        }]]
      }
    }
```

**These are separate from Kubernetes:**
- **MacVLAN subnet:** `192.168.100.0/24`
  - Example: `192.168.100.100`, `192.168.100.50`, etc.
  - Allocated by MacVLAN IPAM (not Kubernetes)
  - Routable on physical network
  - Completely independent from pod/service CIDRs

---

### Practical Examples

#### Example 1: Pod with Both IPs

```bash
# Create pod
kubectl apply -f amf-pod.yaml

# Check Kubernetes IP
$ kubectl get pod amf-0 -o wide
NAME    READY   STATUS    IP           NODE
amf-0   1/1     Running   10.244.1.5   node-1
                          ↑
                          Only shows Kubernetes IP!

# Check all IPs (inside pod)
$ kubectl exec -it amf-0 -- ip addr
eth0: inet 10.244.1.5/24        ← Kubernetes
n2:   inet 192.168.100.100/24   ← MacVLAN (not visible to K8s)
```

#### Example 2: Service Creation

```yaml
apiVersion: v1
kind: Service
metadata:
  name: amf-service
spec:
  selector:
    app: amf
  ports:
  - port: 8080
    targetPort: 8080
  type: ClusterIP
```

```bash
$ kubectl get svc amf-service
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
amf-service   ClusterIP   10.96.10.100    <none>        8080/TCP
                          ↑
                          Kubernetes Service IP (from service CIDR)
```

**This Service:**
- ✅ Routes to pod's Kubernetes IP: `10.244.1.5:8080`
- ❌ Does NOT route to MacVLAN IP: `192.168.100.100`
- ❌ Cannot expose MacVLAN interface

**To access MacVLAN IP, you must connect directly:**
```bash
# From another pod with MacVLAN interface
curl http://192.168.100.100:38412

# NOT through Kubernetes Service
```

#### Example 3: DNS Resolution

```bash
# Inside any pod in the cluster
$ nslookup amf-service
Name:      amf-service.core-amf.svc.cluster.local
Address 1: 10.96.10.100
           ↑
           Resolves to Service IP (Kubernetes managed)

# MacVLAN IPs are NOT in Kubernetes DNS
$ nslookup 192.168.100.100
# No DNS record (unless you add to external DNS)
```

---

### Why This Separation Exists

#### Kubernetes Network (eth0) - For Cluster Operations

**Purpose:**
- Internal cluster communication
- Service discovery via DNS
- Health checks (readiness/liveness)
- Metrics collection
- Log aggregation
- Control plane communication

**Example traffic:**
```
Prometheus → Pod metrics endpoint (10.244.1.5:9090)
Ingress Controller → Pod HTTP endpoint (10.244.1.5:8080)
Kubernetes API → Pod health check (10.244.1.5:8080/health)
```

#### MacVLAN Network (n2, n3, etc.) - For Workload Data Plane

**Purpose:**
- 5G signaling interfaces (N2, N4, etc.)
- User plane traffic (N3, N6, etc.)
- Cross-cluster communication
- External network integration
- Direct physical network access

**Example traffic:**
```
gNB (external) → AMF N2 interface (192.168.100.100:38412)
UPF N3 interface (192.168.100.50) → External Data Network
SMF → UPF N4 interface (192.168.100.51:8805)
```

---

### How IP Assignment Works

#### Kubernetes IP Assignment (Automatic)

```
1. Pod created
    ↓
2. Kubernetes scheduler assigns to node
    ↓
3. Kubelet on node calls CNI plugin
    ↓
4. CNI plugin (e.g., Calico) allocates IP from pod CIDR
    ↓
5. Creates veth pair: pod eth0 ↔ host vethXXX
    ↓
6. Assigns IP to pod eth0: 10.244.1.5
    ↓
7. Updates Kubernetes API with pod IP
    ↓
8. IP visible in `kubectl get pods`
```

#### MacVLAN IP Assignment (Manual/Declarative)

```
1. Pod created with Multus annotation
    ↓
2. Kubelet calls primary CNI (Kubernetes network)
    ↓
3. Multus detects annotation
    ↓
4. Multus calls MacVLAN CNI plugin
    ↓
5. MacVLAN IPAM allocates IP: 192.168.100.100
    ↓
6. Creates MacVLAN interface: n2
    ↓
7. Assigns IP to pod n2: 192.168.100.100
    ↓
8. IP NOT reported to Kubernetes API
    ↓
9. IP NOT visible in `kubectl get pods`
```

---

### Configuration Comparison

#### Pod with Only Kubernetes Network

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

**Result:**
- One interface: `eth0` with Kubernetes IP
- Accessible via Services
- Visible in `kubectl get pods`
- Standard Kubernetes networking

#### Pod with Kubernetes + MacVLAN Networks

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: amf-0
  annotations:
    k8s.v1.cni.cncf.io/networks: n2-network  # Multus annotation
spec:
  containers:
  - name: amf
    image: amf:v1
```

**Result:**
- Two interfaces:
  - `eth0`: Kubernetes IP (`10.244.1.5`)
  - `n2`: MacVLAN IP (`192.168.100.100`)
- Kubernetes IP visible in `kubectl get pods`
- MacVLAN IP only visible inside pod
- Need to query pod directly to see MacVLAN IP

---

### Verification Commands

#### Check Kubernetes-Managed IPs

```bash
# Pod IPs
kubectl get pods -o wide -A

# Service IPs
kubectl get svc -A

# Endpoints (pod IPs behind services)
kubectl get endpoints -A
```

#### Check MacVLAN IPs (NOT in Kubernetes)

```bash
# Must exec into pod
kubectl exec -it amf-0 -- ip addr show

# Or describe pod and look for annotations
kubectl describe pod amf-0 | grep -A 10 Annotations

# Check NetworkAttachmentDefinition status (if available)
kubectl get network-attachment-definitions
```

#### Check IPAM Allocations

```bash
# For Whereabouts (MacVLAN IPAM)
kubectl get ippools.whereabouts.cni.cncf.io -A

# For Nephio (MacVLAN IPAM)
kubectl get ipallocations -A

# These show MacVLAN IPs, NOT Kubernetes IPs
```

---

### Common Confusion Points

#### ❌ Misconception 1: "MacVLAN IP is just another pod IP"

**Wrong:** MacVLAN IP is completely separate from Kubernetes pod IP.

**Correct:**
- Pod IP: Managed by Kubernetes CNI
- MacVLAN IP: Managed by separate IPAM (host-local, whereabouts, etc.)

#### ❌ Misconception 2: "I can create a Service for MacVLAN IP"

**Wrong:** Kubernetes Services only work with Kubernetes pod IPs (eth0).

**Correct:**
- Services route to `eth0` interface only
- MacVLAN interfaces (`n2`, `n3`) are invisible to Services
- Must connect directly to MacVLAN IP

#### ❌ Misconception 3: "kubectl get pods shows all pod IPs"

**Wrong:** Only shows primary (Kubernetes) IP.

**Correct:**
```bash
$ kubectl get pod amf-0 -o wide
IP: 10.244.1.5    ← Only Kubernetes IP shown

# MacVLAN IP (192.168.100.100) is NOT shown
# Must exec into pod to see it
```

#### ❌ Misconception 4: "Health checks use MacVLAN IP"

**Wrong:** Kubernetes probes use primary network (eth0).

**Correct:**
```yaml
livenessProbe:
  httpGet:
    port: 8080  # Connects to 10.244.1.5:8080 (Kubernetes IP)
                # NOT to 192.168.100.100:8080 (MacVLAN IP)
```

---

### Summary Table

| Aspect | Kubernetes Network (eth0) | MacVLAN Network (n2, n3, etc.) |
|--------|--------------------------|-------------------------------|
| **IP Example** | 10.244.1.5 | 192.168.100.100 |
| **Managed By** | Kubernetes CNI | MacVLAN IPAM |
| **Visible in kubectl** | ✅ Yes | ❌ No |
| **Used by Services** | ✅ Yes | ❌ No |
| **Used by DNS** | ✅ Yes | ❌ No |
| **Health Checks** | ✅ Yes | ❌ No |
| **Pod-to-Pod (cluster)** | ✅ Yes | ⚠️ Can but shouldn't |
| **Cross-Cluster** | ❌ No (requires gateway) | ✅ Yes (direct) |
| **External Network** | ❌ No (NAT required) | ✅ Yes (direct) |
| **IP Persistence** | ❌ No | ✅ Yes (with static config) |
| **IPAM Source** | Kubernetes pod CIDR | Physical network ranges |

---

### Key Takeaway

**MacVLAN IPs are completely separate from Kubernetes IP management.**

Think of it this way:
- **Kubernetes network (eth0):** For Kubernetes to manage the pod
- **MacVLAN network (n2, n3):** For the application workload to communicate externally

Kubernetes doesn't know about, manage, or route to MacVLAN IPs. They're as external to Kubernetes as any other physical network device.

---

## The Scenario

### Deployment Setup

```
┌─────────────────────┐         ┌─────────────────────┐
│   my-ran Cluster    │         │   my-core Cluster   │
│                     │         │                     │
│  ┌──────────────┐   │         │  ┌──────────────┐   │
│  │  RAN Pod     │   │         │  │  AMF Pod     │   │
│  │  (gNB/CU/DU) │   │         │  │  (Core NF)   │   │
│  │              │   │         │  │              │   │
│  │ IP: ???      │   │         │  │ IP: ???      │   │
│  └──────────────┘   │         │  └──────────────┘   │
│                     │         │                     │
└─────────────────────┘         └─────────────────────┘
```

**Requirements:**
- Pods must communicate across clusters (N2, N3 interfaces)
- IP addresses must not conflict
- IP addresses must persist across pod restarts/upgrades
- Low latency, high throughput (telco requirements)

---

## Cross-Cluster Communication with MacVLAN

### What is MacVLAN?

MacVLAN creates virtual network interfaces with unique MAC addresses, making pods appear as physical devices on the network.

**Key Characteristics:**
- Pods get direct Layer 2/3 connectivity
- No overlay networking (VXLAN/GENEVE)
- No encapsulation overhead
- Suitable for telco workloads requiring direct network access

### Network Topology

```
                   Physical Network
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
   ┌────▼─────┐     ┌─────▼────┐     ┌─────▼────┐
   │ Gateway  │     │ran-node-1│     │core-node-1│
   │192.168.  │     │  (eth1)  │     │  (eth1)  │
   │100.1     │     └────┬─────┘     └────┬─────┘
   └──────────┘          │                │
                    MacVLAN           MacVLAN
                         │                │
                    ┌────▼───┐       ┌────▼───┐
                    │RAN Pod │       │AMF Pod │
                    │        │       │        │
                    └────────┘       └────────┘
```

### Detailed Packet Flow: RAN Pod → Core Pod

**Setup:**
- RAN Pod: IP `192.168.100.50`, MAC `02:42:c0:a8:64:32` (ran-node-1)
- Core Pod: IP `192.168.100.100`, MAC `02:42:c0:a8:64:64` (core-node-1)
- Both on same subnet: `192.168.100.0/24`

**Step-by-Step Flow:**

```
1. Application Layer
   RAN Pod process → Connect to 192.168.100.100:38412

2. IP Packet Creation
   Source IP: 192.168.100.50
   Dest IP: 192.168.100.100
   Protocol: SCTP/TCP

3. ARP Resolution (if needed)
   RAN Pod → "Who has 192.168.100.100?"
   Core Pod → "I have it, my MAC is 02:42:c0:a8:64:64"

4. Packet Egress (RAN Node)
   RAN Pod (net1 interface)
      ↓ MacVLAN
   ran-node-1 eth1
      ↓ Physical wire
   Network Switch

5. Network Traversal
   Switch forwards based on destination MAC
   No NAT (pods have real IPs)
   No encapsulation (direct L2/L3)

6. Packet Ingress (Core Node)
   Network Switch
      ↓ Physical wire
   core-node-1 eth1
      ↓ MacVLAN demux (by destination MAC)
   Core Pod (net1 interface)
      ↓ IP stack
   Core Pod application

7. Return Path
   Same process in reverse
   Connection established
```

**Key Point:** The network doesn't know or care about Kubernetes cluster boundaries. Pods communicate as if they were physical devices on the same network.

### When is the Gateway Used?

**Gateway (`192.168.100.1`) is used for:**
- Cross-subnet communication (e.g., `192.168.100.50` → `192.168.101.100`)
- Internet/external access
- Routing to different networks

**Gateway is NOT used for:**
- Same-subnet communication (e.g., `192.168.100.50` → `192.168.100.100`)
- Pods communicate directly via L2 (ARP + Ethernet frames)

**Pod routing table:**
```
default via 192.168.100.1 dev net1          # For external traffic
192.168.100.0/24 dev net1 scope link        # Direct L2, no gateway
```

---

## The IPAM Challenge

### The Problem

By default, each Kubernetes cluster has **independent IPAM** (IP Address Management):

```
my-ran cluster:
  IPAM allocates: 192.168.100.0/24
  Pod A → 192.168.100.50

my-core cluster:
  IPAM allocates: 192.168.100.0/24
  Pod B → 192.168.100.50  ❌ CONFLICT!
```

**Result:** IP address conflicts! Both clusters could assign the same IP to different pods.

### Question: How Do Pods Get IPs in the Same Subnet?

**Answer:** Through coordinated IPAM strategies that prevent conflicts while allowing pods to share the same Layer 2 network.

---

## IPAM Solutions Comparison

### Solution 1: Non-Overlapping IP Ranges (Manual Partitioning)

**Approach:** Manually partition the IP address space between clusters.

```yaml
# my-ran cluster - NetworkAttachmentDefinition
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-ran-net
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "ranges": [[{
          "subnet": "192.168.100.0/24",
          "rangeStart": "192.168.100.10",
          "rangeEnd": "192.168.100.99",
          "gateway": "192.168.100.1"
        }]]
      }
    }
```

```yaml
# my-core cluster - NetworkAttachmentDefinition
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-core-net
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "ranges": [[{
          "subnet": "192.168.100.0/24",
          "rangeStart": "192.168.100.100",
          "rangeEnd": "192.168.100.199",
          "gateway": "192.168.100.1"
        }]]
      }
    }
```

**IP Allocation:**
- RAN cluster: `192.168.100.10` - `192.168.100.99`
- Core cluster: `192.168.100.100` - `192.168.100.199`

**Pros:**
- ✅ Simple to understand
- ✅ No external dependencies
- ✅ Prevents cross-cluster conflicts

**Cons:**
- ❌ Manual management (error-prone)
- ❌ Doesn't scale well
- ❌ No IP persistence by default

---

### Solution 2: Completely Separate Subnets

**Approach:** Give each cluster its own subnet.

```
my-ran cluster:  192.168.100.0/24
my-core cluster: 192.168.101.0/24
```

**Requires:** Layer 3 routing between subnets (physical router or BGP).

**Network Setup:**
```
     ┌─────────────┐
     │   Router    │
     │ Routes:     │
     │ .100.0/24   │
     │ .101.0/24   │
     └──────┬──────┘
            │
    ┌───────┴───────┐
    │               │
┌───▼────┐      ┌───▼────┐
│  RAN   │      │  Core  │
│.100.x  │      │.101.x  │
└────────┘      └────────┘
```

**Pros:**
- ✅ Clear separation
- ✅ Easier to manage firewall rules
- ✅ Prevents conflicts

**Cons:**
- ❌ Pods not on same L2 segment
- ❌ Requires L3 routing infrastructure
- ❌ No IP persistence by default

---

### Solution 3: Centralized IPAM (Whereabouts with Shared Storage)

**Approach:** Use Whereabouts CNI plugin with shared etcd/Kubernetes API.

```yaml
# Both clusters point to shared IPAM backend
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-shared-net
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.100.0/24",
        "gateway": "192.168.100.1",
        "etcd_host": "shared-etcd.example.com:2379"
      }
    }
```

**How It Works:**
```
RAN Cluster                     Core Cluster
     │                               │
     ├─ Pod requests IP              ├─ Pod requests IP
     │                               │
     └──────────┐         ┌──────────┘
                │         │
          ┌─────▼─────────▼─────┐
          │   Shared etcd       │
          │   IPAM Database     │
          │                     │
          │  192.168.100.10: ✅ │
          │  192.168.100.11: ✅ │
          │  192.168.100.12: ❌ │
          └─────────────────────┘
```

**Pros:**
- ✅ Dynamic allocation
- ✅ Automated conflict prevention
- ✅ Single source of truth

**Cons:**
- ❌ Requires shared infrastructure (etcd/API server)
- ❌ Added complexity
- ❌ No IP persistence by default

---

### Solution 4: External IPAM (Infoblox, NetBox, phpIPAM)

**Approach:** Integrate with enterprise IPAM systems.

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-infoblox
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "ipam": {
        "type": "infoblox",
        "network": "192.168.100.0/24"
      }
    }
```

**How It Works:**
- CNI plugin requests IPs from external IPAM via API
- IPAM system manages allocations across all clusters
- Integrates with existing network infrastructure

**Pros:**
- ✅ Enterprise-grade
- ✅ Centralized management
- ✅ Audit trails and compliance
- ✅ May provide IP persistence (depends on configuration)

**Cons:**
- ❌ Requires external system
- ❌ Potential API latency
- ❌ Additional licensing costs

---

### Solution 5: Nephio's Declarative IPAM

**Approach:** Package-based declarative IP management with GitOps.

**Step 1: Create IPClaim in package**
```yaml
apiVersion: ipam.nephio.org/v1alpha1
kind: IPClaim
metadata:
  name: amf-n2-interface
  namespace: core-amf
spec:
  kind: network
  networkInstance:
    name: vpc-core
  selector:
    nephio.org/network-name: n2
```

**Step 2: Nephio IPAM controller allocates IP**
```yaml
# Generated by IPAM controller
apiVersion: ipam.nephio.org/v1alpha1
kind: IPAllocation
metadata:
  name: amf-n2-interface
  namespace: core-amf
spec:
  kind: network
  networkInstance:
    name: vpc-core
  prefix: 192.168.100.100/32  # Specific IP allocated
  gateway: 192.168.100.1
```

**Step 3: Nephio generates NetworkAttachmentDefinition**
```yaml
# Generated by Nephio
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: n2-network
  namespace: core-amf
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "ipam": {
        "type": "static"
      }
    }
```

**Step 4: Nephio generates pod with static IP annotation**
```yaml
# Generated deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: amf
  namespace: core-amf
spec:
  template:
    metadata:
      annotations:
        k8s.v1.cni.cncf.io/networks: |
          [{
            "name": "n2-network",
            "interface": "n2",
            "ips": ["192.168.100.100/24"]
          }]
    spec:
      containers:
      - name: amf
        image: amf:v1
```

**Pros:**
- ✅ Declarative and GitOps-native
- ✅ Automatic IP allocation
- ✅ **IP persistence by default**
- ✅ Prevents conflicts across clusters
- ✅ Full lifecycle management

**Cons:**
- ❌ Requires Nephio infrastructure
- ❌ Learning curve for package model

---

## IP Persistence Challenge

### Why IP Persistence Matters in Telco

In 5G/telco deployments, **IP addresses must remain stable** across pod lifecycle events:

```
❌ What breaks if IPs change:

Pod restart: IP changes from .50 to .51
  → N2 signaling connections break
  → AMF can't reach gNB
  → User registrations fail

Pod upgrade: IP changes from .100 to .101
  → Routing tables become stale
  → Neighboring NFs can't communicate
  → Service disruption
```

### Default Behavior (Without Persistence)

Most IPAM solutions allocate IPs dynamically:

```
Pod created  → Gets 192.168.100.50
Pod deleted  → IP released to pool
Pod recreated → Gets 192.168.100.51  ❌ Different IP!
```

### IP Persistence by Solution

| Solution | Default IP Persistence? | How to Achieve It |
|----------|------------------------|-------------------|
| Solution 1: Non-overlapping ranges | ❌ NO | Add static annotations or IP reservations |
| Solution 2: Separate subnets | ❌ NO | Add static annotations or IP reservations |
| Solution 3: Shared Whereabouts | ❌ NO | Add static annotations or IP reservations |
| Solution 4: External IPAM | ⚠️ MAYBE | Depends on external system capabilities |
| Solution 5: Nephio | ✅ **YES** | Built-in via IPAllocation + static annotations |

### Adding Persistence to Solutions 1-3

**Option A: Manual Static IP Annotations**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: amf-0
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [{
        "name": "n2-network",
        "ips": ["192.168.100.100/24"]  # Explicit IP
      }]
spec:
  containers:
  - name: amf
    image: amf:v1
```

Change IPAM to `static` type:
```yaml
spec:
  config: |
    {
      "type": "macvlan",
      "ipam": {
        "type": "static"  # Honors the annotation
      }
    }
```

**Problem:** Manual management for every pod, error-prone, doesn't scale.

**Option B: Whereabouts IP Reservations**

```yaml
apiVersion: whereabouts.cni.cncf.io/v1alpha1
kind: OverlappingRangeIPReservation
metadata:
  name: amf-0-reservation
spec:
  containerid: "amf-0"
  ifname: "net1"
  ip: "192.168.100.100"
  podref: core-namespace/amf-0
```

**Problem:** Still requires manual reservation creation and management.

### Why Only Nephio Has Default Persistence

**Solutions 1-3 solve:** Cross-cluster IP conflict prevention (cluster-level)

**Nephio solves:** IP conflict prevention + Pod-to-IP binding (workload-level)

```
Nephio's approach:

IPClaim (declarative intent)
    ↓
IPAM Controller allocates specific IP
    ↓
Creates IPAllocation (binding record)
    ↓
Generates static IP annotation in pod spec
    ↓
Pod always gets same IP ✅
```

**Key difference:** Nephio **automatically generates static IP annotations** and stores the binding in the package, making persistence the default behavior.

---

## Nephio's Architecture

### How Nephio Manages Workload Clusters Without Direct Access

**Key Principle:** Nephio uses **GitOps pull model**, not push-based deployment.

```
┌──────────────────────────────────────────────────────────┐
│              Nephio Management Cluster                    │
│                                                           │
│  ┌──────────┐  ┌───────────┐  ┌──────────────────┐     │
│  │  Porch   │  │   IPAM    │  │ Network Config   │     │
│  │(Package  │  │ Controller│  │   Controller     │     │
│  │  API)    │  └─────┬─────┘  └────────┬─────────┘     │
│  └────┬─────┘        │                 │               │
│       └──────────────┼─────────────────┘               │
│                      │                                 │
│              ┌───────▼────────┐                        │
│              │  Git Repos     │                        │
│              │ (Deployments)  │                        │
│              └───────┬────────┘                        │
└──────────────────────┼─────────────────────────────────┘
                       │
                       │ Git commits (PUSH)
                       │
      ┌────────────────┼────────────────┐
      │                │                │
┌─────▼─────┐    ┌─────▼─────┐    ┌────▼──────┐
│Git Repo:  │    │Git Repo:  │    │Git Repo:  │
│ran-cluster│    │core-cluster│   │edge-cluster│
└─────┬─────┘    └─────┬─────┘    └────┬──────┘
      │                │                │
      │ ConfigSync     │ ConfigSync     │ ConfigSync
      │ (PULL)         │ (PULL)         │ (PULL)
      │                │                │
┌─────▼─────┐    ┌─────▼─────┐    ┌────▼──────┐
│  my-ran   │    │  my-core  │    │  my-edge  │
│ Cluster   │    │  Cluster  │    │  Cluster  │
│┌─────────┐│    │┌─────────┐│    │┌─────────┐│
││ConfigSync││    ││ConfigSync││    ││ConfigSync││
│└─────────┘│    │└─────────┘│    │└─────────┘│
└───────────┘    └───────────┘    └───────────┘
```

### ConfigSync Configuration

On each workload cluster:

```yaml
apiVersion: configsync.gke.io/v1beta1
kind: RootSync
metadata:
  name: root-sync
  namespace: config-management-system
spec:
  sourceFormat: unstructured
  git:
    repo: https://github.com/myorg/deployment-repo
    branch: main
    dir: my-ran-deployment  # Cluster-specific directory
    auth: token
    secretRef:
      name: git-creds
  pollInterval: 15s  # Check for changes every 15 seconds
```

### Complete IPClaim Processing Flow

**Step 1: User creates PackageRevision with IPClaim**
```bash
# On management cluster
kpt alpha rpkg init my-ran-deployment \
  --repository=deployment \
  --workspace=v1
```

**Step 2: Nephio controllers process the package**
```
IPAM Controller watches PackageRevisions
    ↓
Finds IPClaim in package
    ↓
Allocates IP from IPAM database
    ↓
Injects IPAllocation resource
    ↓
Injects NetworkAttachmentDefinition
    ↓
Injects static IP annotation in pod spec
    ↓
Updates package via Porch API
```

**Step 3: Porch commits to Git**
```
deployment-repo/my-ran-deployment/
├── ipclaim.yaml          # Original
├── ipallocation.yaml     # Injected
├── nad.yaml              # Injected
├── deployment.yaml       # Updated with IP annotation
└── Kptfile
```

**Step 4: ConfigSync pulls and applies**
```
ConfigSync on my-ran cluster:
  Polls Git every 15s
      ↓
  Detects changes
      ↓
  Pulls package contents
      ↓
  Applies all resources to cluster
      ↓
  Reports sync status
```

**Step 5: Resources created on workload cluster**
```bash
kubectl get ipclaim -n my-ran-deployment
kubectl get ipallocation -n my-ran-deployment
kubectl get network-attachment-definitions -n my-ran-deployment
kubectl get pods -n my-ran-deployment
```

### How Management Cluster "Reads" from Workload Clusters

**Answer: It doesn't need to!** But feedback mechanisms exist:

**Method 1: Status via Git commits**
```yaml
# ConfigSync writes back to Git
deployment-repo/my-ran-deployment/.status/sync-status.yaml

apiVersion: configsync.gke.io/v1beta1
kind: SyncStatus
status:
  sync:
    commit: abc123
    lastSyncTime: "2026-01-24T10:30:00Z"
  conditions:
  - type: Synced
    status: "True"
```

**Method 2: Metrics (out-of-band)**
```
Workload Cluster → Prometheus → Central Monitoring → Management Cluster
```

**Method 3: Porch PackageRevision status**
```yaml
apiVersion: porch.kpt.dev/v1alpha1
kind: PackageRevision
status:
  deployment:
    deployed: true
    conditions:
    - type: Ready
      status: "True"
```

### IPAM Database Location

The IPAM backend runs on the **management cluster**:

```yaml
# NetworkInstance tracks available IP space
apiVersion: ipam.nephio.org/v1alpha1
kind: NetworkInstance
metadata:
  name: vpc-ran
spec:
  topology: layer3
  prefixes:
  - prefix: 192.168.100.0/24

# IPPrefix tracks allocations
apiVersion: ipam.nephio.org/v1alpha1
kind: IPPrefix
metadata:
  name: ran-n2-pool
spec:
  prefix: 192.168.100.0/24
  allocations:
  - prefix: 192.168.100.10/28
    claimRef:
      name: my-ran-deployment-n2
      cluster: my-ran
  - prefix: 192.168.100.100/28
    claimRef:
      name: my-core-deployment-n2
      cluster: my-core
```

---

## Complete Packet Flow Example

### Real-World 5G Scenario

**Setup:**
- RAN cluster: gNB pod needs to communicate with AMF in core cluster
- Interface: N2 (control plane signaling)
- Both using MacVLAN on same subnet: `192.168.100.0/24`

### Nephio Configuration

**RAN Package:**
```yaml
# IPClaim for gNB N2 interface
apiVersion: ipam.nephio.org/v1alpha1
kind: IPClaim
metadata:
  name: gnb-n2
  namespace: ran-gnb
spec:
  kind: network
  networkInstance:
    name: n2-network
  selector:
    nephio.org/network-name: n2
```

**Nephio IPAM allocates:**
- gNB: `192.168.100.50`
- AMF: `192.168.100.100`

### Deployed Configuration

**RAN Cluster (my-ran):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gnb-0
  namespace: ran-gnb
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [{
        "name": "n2-network",
        "interface": "n2",
        "ips": ["192.168.100.50/24"]
      }]
spec:
  containers:
  - name: gnb
    image: gnb:v1
```

**Core Cluster (my-core):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: amf-0
  namespace: core-amf
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [{
        "name": "n2-network",
        "interface": "n2",
        "ips": ["192.168.100.100/24"]
      }]
spec:
  containers:
  - name: amf
    image: amf:v1
```

### Packet Flow: UE Registration (gNB → AMF)

```
1. UE attaches to gNB
   gNB needs to send N2 message to AMF

2. gNB pod creates SCTP packet
   Source: 192.168.100.50:38412
   Dest: 192.168.100.100:38412
   Protocol: SCTP
   Message: N2 Initial UE Message

3. Kernel routing decision (inside gNB pod)
   ip route show:
     192.168.100.0/24 dev n2 scope link
   → Same subnet, send directly (no gateway)

4. ARP resolution
   gNB pod: "Who has 192.168.100.100?"
   
   ARP request travels:
   gNB pod (n2 interface)
     → MacVLAN (ran-node-1)
     → Physical NIC eth1
     → Network switch
     → Physical NIC eth1 (core-node-1)
     → MacVLAN demux
     → AMF pod (n2 interface)
   
   AMF pod: "192.168.100.100 is at MAC 02:42:c0:a8:64:64"

5. SCTP packet transmission
   gNB pod constructs Ethernet frame:
     - Source MAC: 02:42:c0:a8:64:32 (gNB's MAC)
     - Dest MAC: 02:42:c0:a8:64:64 (AMF's MAC)
     - Payload: IP packet with SCTP data

6. Frame egress from RAN cluster
   gNB pod (n2)
     → MacVLAN interface
     → ran-node-1 eth1 (physical)
     → Network cable
     → Switch

7. Switch forwarding
   Switch MAC table lookup:
     MAC 02:42:c0:a8:64:64 → Port connected to core-node-1
   
   Frame forwarded to core-node-1

8. Frame ingress to Core cluster
   Switch
     → Network cable
     → core-node-1 eth1 (physical)
     → MacVLAN demultiplexes by dest MAC
     → AMF pod (n2 interface)

9. Packet processing in AMF
   AMF kernel receives Ethernet frame
     → Strips Ethernet header
     → Processes IP packet
     → Delivers SCTP data to AMF application
     → AMF processes N2 Initial UE Message

10. Response (AMF → gNB)
    Same process in reverse:
    AMF constructs response
      Source: 192.168.100.100
      Dest: 192.168.100.50
    
    Packet travels back through network
    gNB receives response
    N2 signaling complete
```

### Timeline of Complete Transaction

```
T+0ms:   UE sends attach request to gNB (radio interface)
T+1ms:   gNB processes, needs to contact AMF
T+2ms:   gNB pod creates N2 Initial UE Message
T+3ms:   ARP resolution (if not cached)
T+4ms:   SCTP packet created with N2 message
T+5ms:   Packet egresses gNB pod → MacVLAN
T+6ms:   Physical transmission through network
T+7ms:   Packet arrives at core-node-1
T+8ms:   MacVLAN delivers to AMF pod
T+9ms:   AMF processes N2 message
T+10ms:  AMF sends response back to gNB
T+15ms:  gNB receives response
T+16ms:  N2 signaling complete, UE registration proceeds
```

**Total latency: ~16ms (sub-20ms typical for direct L2/L3)**

### What Happens During Pod Restart?

**Scenario: AMF pod crashes and restarts**

```
1. AMF pod crashes
   Kubernetes detects failure

2. Kubernetes recreates AMF pod
   Same name: amf-0
   Same namespace: core-amf
   Same static IP annotation: 192.168.100.100

3. CNI plugin invoked (on new pod creation)
   Reads annotation: "ips": ["192.168.100.100/24"]
   IPAM type: static
   
4. MacVLAN interface configured
   IP: 192.168.100.100 (SAME IP ✅)
   New MAC: 02:42:c0:a8:64:65 (different MAC)

5. Pod sends gratuitous ARP
   "192.168.100.100 is now at MAC 02:42:c0:a8:64:65"
   
   Network updates ARP caches:
   - Switch updates MAC table
   - gNB pod updates ARP cache
   - Gateway updates ARP cache

6. Communication resumes
   gNB sends next N2 message to 192.168.100.100
   ARP resolves to new MAC
   Packet delivered successfully ✅
```

**Key Point:** IP stays the same (192.168.100.100), so gNB doesn't need reconfiguration. Only MAC address changes, which is handled transparently by ARP.

---

## Key Takeaways

### 1. Cross-Cluster Communication with MacVLAN

**Essential Concepts:**
- MacVLAN makes pods appear as physical devices on the network
- No overlay networking or encapsulation overhead
- Pods communicate directly via Layer 2/3, regardless of cluster boundaries
- Gateway is only used for cross-subnet or external traffic
- Suitable for telco workloads requiring low latency and high throughput

**Remember:**
```
Same subnet → Direct L2 communication (ARP + Ethernet)
Different subnet → L3 routing via gateway
```

### 2. IPAM: Preventing Cross-Cluster IP Conflicts

**The Challenge:**
- Each cluster has independent IPAM by default
- Can lead to IP address conflicts across clusters
- Pods on same network must have unique IPs

**Solutions Summary:**

| Solution | What It Solves | IP Persistence |
|----------|---------------|----------------|
| Manual partitioning | Conflicts via range separation | ❌ No (requires manual static IPs) |
| Separate subnets | Conflicts via network isolation | ❌ No (requires manual static IPs) |
| Shared IPAM (Whereabouts) | Conflicts via centralized allocation | ❌ No (requires reservations) |
| External IPAM | Conflicts via enterprise system | ⚠️ Maybe (system-dependent) |
| **Nephio** | **Conflicts + Persistence via declarative binding** | **✅ Yes (automatic)** |

### 3. IP Persistence: Critical for Telco

**Why It Matters:**
- 5G network functions have static interface requirements
- N2, N3, N4 interfaces must maintain stable IPs
- IP changes break signaling, routing, and user sessions

**How Nephio Achieves It:**
```
IPClaim (intent)
    ↓
IPAM Controller (allocation)
    ↓
IPAllocation (binding record)
    ↓
Static IP annotation (enforcement)
    ↓
Pod lifecycle (persistence)
```

**Result:** Pods always get the same IP across restarts, upgrades, and rescheduling.

### 4. Nephio Architecture: GitOps Pull Model

**Key Principles:**
- Management cluster **never** directly accesses workload clusters
- Management cluster writes enriched packages to Git
- Workload clusters pull and apply via ConfigSync
- Git is the single source of truth

**Benefits:**
- ✅ Security: No exposed API servers
- ✅ Scalability: No connection overhead
- ✅ Auditability: All changes in Git history
- ✅ Resilience: Clusters work independently

**Communication Flow:**
```
Management Cluster → Git (push)
Git ← Workload Cluster (pull)
Git → Status feedback (optional)
```

### 5. Complete Solution Comparison

**For Simple Testing/Development:**
- Use Solution 1 (manual partitioning)
- Add static IP annotations manually
- Good for learning and small-scale deployments

**For Enterprise Without Nephio:**
- Use Solution 4 (external IPAM like Infoblox)
- Integrate with existing network infrastructure
- Leverage enterprise features (audit, compliance)

**For Production Telco/5G:**
- Use Solution 5 (Nephio)
- Declarative, GitOps-native
- Built-in IP persistence
- Full lifecycle management
- Industry best practice for cloud-native 5G

### 6. Practical Verification Commands

**Check pod network configuration:**
```bash
# View pod IP addresses
kubectl get pods -n <namespace> -o wide

# Check MacVLAN interface inside pod
kubectl exec -it <pod-name> -- ip addr show net1

# Verify routing table
kubectl exec -it <pod-name> -- ip route show
```

**Verify IPAM allocations:**
```bash
# For Nephio
kubectl get ipclaims -A
kubectl get ipallocations -A

# For Whereabouts
kubectl get ippools.whereabouts.cni.cncf.io -A
```

**Test connectivity:**
```bash
# Ping across clusters
kubectl exec -it <ran-pod> -- ping 192.168.100.100

# Check ARP cache
kubectl exec -it <ran-pod> -- ip neigh show

# Verify no IP conflicts
kubectl exec -it <test-pod> -- arping -c 5 192.168.100.100
# Should see only ONE MAC responding
```

**Monitor ConfigSync status:**
```bash
# Check sync status
kubectl get rootsync -n config-management-system

# View sync errors
kubectl describe rootsync root-sync -n config-management-system
```

### 7. Common Pitfalls and Solutions

**Pitfall 1: Forgetting Static IP Annotations**
- Symptom: IPs change on pod restart
- Solution: Always use static annotations or Nephio's declarative model

**Pitfall 2: IP Conflicts Across Clusters**
- Symptom: Intermittent connectivity, duplicate IP warnings
- Solution: Implement coordinated IPAM (Whereabouts shared or Nephio)

**Pitfall 3: Expecting Gateway for Same-Subnet Traffic**
- Symptom: Misunderstanding packet flow
- Solution: Remember same-subnet = direct L2, different-subnet = L3 via gateway

**Pitfall 4: Not Handling MAC Address Changes**
- Symptom: Brief connectivity loss on pod restart
- Solution: Use gratuitous ARP, implement proper health checks

**Pitfall 5: Manual IPAM Management at Scale**
- Symptom: IP conflicts, management overhead, errors
- Solution: Adopt Nephio or external IPAM for automation

### 8. Design Recommendations

**For New Deployments:**
1. Start with Nephio if building cloud-native 5G
2. Use declarative IPClaims for all network interfaces
3. Let Nephio handle IPAM and persistence automatically
4. Leverage GitOps for full lifecycle management

**For Existing Deployments:**
1. Assess current IPAM solution
2. Check if IP persistence is properly implemented
3. Consider migration path to Nephio
4. Implement gradual transition (cluster by cluster)

**Network Design:**
1. Plan IP address space carefully
2. Allocate sufficient ranges for growth
3. Document IP allocations
4. Implement monitoring and alerting
5. Use network policies for security

---

## Conclusion

Cross-cluster pod communication in Nephio combines several technologies:

**MacVLAN** provides direct Layer 2/3 connectivity between pods across clusters, enabling low-latency communication essential for telco workloads.

**Coordinated IPAM** prevents IP conflicts across clusters while allowing pods to share the same network segment.

**IP Persistence** ensures network functions maintain stable addresses across their lifecycle, critical for 5G signaling and routing.

**GitOps Architecture** enables declarative, auditable, and scalable management without requiring direct access to workload clusters.

**Nephio** integrates all of these capabilities into a unified platform, providing the only solution with default IP persistence through its declarative package model.

Understanding these concepts is essential for deploying and operating cloud-native 5G infrastructure at scale.

---

## Additional Resources

**Nephio Documentation:**
- https://nephio.org/
- Package Specialization: https://github.com/nephio-project/docs

**CNI Plugins:**
- MacVLAN: https://www.cni.dev/plugins/current/main/macvlan/
- Multus: https://github.com/k8snetworkplumbingwg/multus-cni
- Whereabouts: https://github.com/k8snetworkplumbingwg/whereabouts

**ConfigSync:**
- Google Cloud Config Sync: https://cloud.google.com/anthos-config-management/docs/config-sync-overview

**5G Network Interfaces:**
- 3GPP TS 23.501: 5G System Architecture
- N2 Interface: NG-RAN ↔ AMF
- N3 Interface: NG-RAN ↔ UPF

---
## SD-Core's Network Interface Configuration

### N2 Interface (Control Plane) - NO MacVLAN ❌

From the Aether documentation, the **N2 interface uses the node's primary IP address** (the Kubernetes-managed network):

```yaml
amf:
  ip: "10.76.28.113"  # Physical server IP address
```

**How N2 works:**
```
gNB (external, e.g., 10.76.28.187)
    ↓
    SCTP packets on port 38412
    ↓
Physical server IP: 10.76.28.113
    ↓
Kubernetes NodePort / port forwarding
    ↓
AMF pod (inside Kubernetes network)
```

**From the documentation:**
> "RAN elements (whether they are physical gNBs or gNBsim) reach the AMF using the server's actual IP address (10.76.28.113 in our running example). Kubernetes is configured to forward SCTP packets arriving on port 38412 to the AMF container."

**Key point:** N2 uses **standard Kubernetes networking** (eth0, the primary interface), NOT MacVLAN!

---

### N3 Interface (User Plane) - YES MacVLAN ✅

The **N3 interface uses MacVLAN** for direct connectivity:

```yaml
router:
  data_iface: ens18
  macvlan:
    iface: gnbaccess
    subnet_prefix: "172.20"

upf:
  ip_prefix: "192.168.252.0/24"
  iface: "access"  # This is the MacVLAN interface
  default_upf:
    ip:
      access: "192.168.252.3/24"  # N3 interface
      core: "192.168.250.3/24"    # N6 interface
```

**How N3 works:**
```
gNB (external, e.g., 10.76.28.187 or 172.20.x.x)
    ↓
    GTP packets (port 2152) with user plane data
    ↓
Physical network (ens18)
    ↓
MacVLAN bridge: "access" (192.168.252.0/24)
    ↓
UPF pod access interface: 192.168.252.3
```

**From the documentation:**
> "The access bridge connects the UPF downstream to the RAN (this corresponds to 3GPP's N3 interface) and is assigned IP subnet 192.168.252.0/24."

And:
> "$ ip link add access link ens18 type macvlan mode bridge 192.168.252.3"

**Key point:** N3 uses **MacVLAN** for direct Layer 2 connectivity to physical network!

---

### Why This Design?

#### N2 (Control Plane) - No MacVLAN Needed

**Reasons:**
1. **Low traffic volume** - Signaling messages are relatively small and infrequent
2. **No strict latency requirements** - Control plane can tolerate slight overhead
3. **Simplicity** - Using the node's IP address simplifies configuration
4. **Standard Kubernetes features** - Can use NodePort, LoadBalancer, or direct port forwarding
5. **No need for physical network integration** - gNB can reach via standard IP routing

**Architecture:**
```
┌─────────────────────────────────────────────────┐
│              Physical Server                     │
│                                                  │
│  ┌──────────────────────────────────────────┐  │
│  │     Kubernetes Cluster                   │  │
│  │                                          │  │
│  │  ┌────────────┐                         │  │
│  │  │  AMF Pod   │                         │  │
│  │  │            │                         │  │
│  │  │ eth0:      │ ← Kubernetes network    │  │
│  │  │ 10.244.x.x │                         │  │
│  │  └─────┬──────┘                         │  │
│  │        │                                 │  │
│  │        │ (NodePort / port forward)      │  │
│  │        │                                 │  │
│  └────────┼─────────────────────────────────┘  │
│           │                                     │
│     ┌─────▼──────┐                             │
│     │ Node IP:   │                             │
│     │10.76.28.113│                             │
│     │Port: 38412 │                             │
│     └─────┬──────┘                             │
└───────────┼─────────────────────────────────────┘
            │
            │ SCTP (N2)
            │
      ┌─────▼──────┐
      │    gNB     │
      │10.76.28.187│
      └────────────┘
```

#### N3 (User Plane) - MacVLAN Required

**Reasons:**
1. **High throughput** - User data traffic requires maximum performance
2. **Low latency** - Direct L2/L3 access eliminates overhead
3. **No encapsulation** - Avoids GTP-in-GTP tunneling complications
4. **Physical network integration** - UPF must be directly reachable by gNB
5. **Standard 3GPP requirements** - N3 expects direct IP connectivity

**Architecture:**
```
┌─────────────────────────────────────────────────┐
│              Physical Server                     │
│                                                  │
│  ┌──────────────────────────────────────────┐  │
│  │     Kubernetes Cluster                   │  │
│  │                                          │  │
│  │  ┌────────────┐                         │  │
│  │  │  UPF Pod   │                         │  │
│  │  │            │                         │  │
│  │  │ eth0:      │ ← Kubernetes network    │  │
│  │  │ 10.244.x.x │                         │  │
│  │  │            │                         │  │
│  │  │ access:    │ ← MacVLAN interface     │  │
│  │  │192.168.    │   (N3)                  │  │
│  │  │  252.3     │                         │  │
│  │  └─────┬──────┘                         │  │
│  │        │                                 │  │
│  │        │ (MacVLAN)                       │  │
│  │        │                                 │  │
│  └────────┼─────────────────────────────────┘  │
│           │                                     │
│     ┌─────▼──────┐                             │
│     │   ens18    │ Physical NIC                │
│     │192.168.    │                             │
│     │  252.1     │                             │
│     └─────┬──────┘                             │
└───────────┼─────────────────────────────────────┘
            │
            │ GTP (N3) - Direct L2
            │
      ┌─────▼──────┐
      │    gNB     │
      │192.168.    │
      │  252.2     │
      └────────────┘
```

---

### Verification from SD-Core Configuration

Looking at the configuration structure:

```yaml
# vars/main.yaml (from aether-5gc repo)
core:
  data_iface: ens18              # Physical interface for data plane
  
  upf:
    ip_prefix: "192.168.252.0/24"  # N3 interface subnet
    iface: "access"                 # MacVLAN interface name
    
  amf:
    ip: "10.76.28.113"              # Uses node's primary IP (NOT MacVLAN)

router:
  data_iface: ens18
  macvlan:
    iface: gnbaccess                # MacVLAN for N3
    subnet_prefix: "172.20"         # Optional RAN subnet
```

**The playbook creates MacVLAN interfaces only for UPF:**
```bash
# From the router.yml playbook
$ ip link add access link ens18 type macvlan mode bridge
$ ip link add core link ens18 type macvlan mode bridge
```

**These correspond to:**
- `access` = N3 interface (to gNB)
- `core` = N6 interface (to Internet/DN)

**AMF has no MacVLAN interface created!**

---

### Summary Table

| Interface | Protocol | SD-Core Implementation | Network Type | Why? |
|-----------|----------|----------------------|--------------|------|
| **N2** (gNB ↔ AMF) | SCTP | Node IP + Port 38412 | **Kubernetes network (eth0)** | Control plane, low volume, NodePort works fine |
| **N3** (gNB ↔ UPF) | GTP-U | MacVLAN (access interface) | **MacVLAN** (192.168.252.0/24) | User plane, high throughput, needs direct L2 access |
| **N6** (UPF ↔ Internet) | IP | MacVLAN (core interface) | **MacVLAN** (192.168.250.0/24) | Data network connectivity |
| N4 (SMF ↔ UPF) | PFCP | Kubernetes network | **Kubernetes network (eth0)** | Control plane, internal to cluster |

---

### Packet Capture Commands from Documentation

The documentation even shows this distinction:

```bash
# N2 (SCTP) - captured on ANY interface (not specific MacVLAN)
$ sudo tcpdump -i any sctp -w sctp-test.pcap

# N3 (GTP) - captured on SPECIFIC interfaces (ens18 and access MacVLAN)
$ sudo tcpdump -i ens18 port 2152 -w gtp-outside.pcap
$ sudo tcpdump -i access port 2152 -w gtp-inside.pcap
                        ↑
                        MacVLAN interface!
```

---

### Why This Makes Sense

**Control Plane (N2):**
- Signaling traffic is lightweight
- Can go through Kubernetes networking layers
- Benefits from Kubernetes service discovery
- Easier to manage and monitor

**User Plane (N3):**
- Data traffic is heavy (Gbps)
- Needs minimal latency
- Requires direct physical network access
- Benefits from kernel bypass (af_packet or DPDK)
- No Kubernetes overhead

---

## Basics-ChatGPT

Yes — **this split is absolutely standard in telco**, and you’ve just articulated it in the exact language operators use:

> **Primary IP = Kubernetes / cloud-native**
> **Secondary IP = telecom / underlay / non-Kubernetes**

This pattern shows up everywhere: 4G EPC, 5GC, IMS, UPF, BNG, firewalls, SBCs, etc.

Let’s anchor this properly and then survey **all realistic “secondary IP” options** that are *not* managed by Kubernetes.

---

### 1️⃣ Why this split is common in telco (not an accident)

Telco workloads have **hard constraints** that cloud-native networking violates:

| Telco requirement    | Kubernetes default |
| -------------------- | ------------------ |
| Stable IP identity   | Ephemeral          |
| L2/L3 adjacency      | Overlay            |
| Line-rate throughput | Encapsulation      |
| Deterministic path   | Load-balanced      |
| No NAT               | kube-proxy NAT     |

So telco adopted Kubernetes **without surrendering the data plane**.

That’s why:

* Primary interface → Kubernetes
* Secondary interface → Underlay

This is not a hack — it’s a *design pattern*.

---

### 2️⃣ Taxonomy of “Secondary IP” options (non-Kubernetes)

Below is the **full menu**, from simplest to most powerful.

---

#### Option A — **MACVLAN (bridge / passthru)**

**(Most common, SD-Core default)**

##### What it is

* Kernel MACVLAN device
* Pod gets:

  * Its own MAC
  * Its own IP
* Shares the host NIC

##### Why telco likes it

* Simple
* Deterministic
* Near line-rate
* No overlay
* No NAT

##### Typical use

* UPF N3/N6
* gNB N3
* BNG
* DPI

##### Limitations

* Host ↔ pod communication requires routing tricks
* No live migration
* Node-affine

---

#### Option B — **IPVLAN (L2 or L3 mode)**

##### What it is

* Similar to MACVLAN
* Shares the **same MAC**
* IP-based separation

##### Why it’s used

* Some switches limit MACs per port
* Cleaner ARP behavior in L3 mode

##### Telco usage

* High-density VNFs
* When MAC scaling matters

##### Tradeoff

* Slightly more complex routing
* Less intuitive debugging

---

#### Option C — **SR-IOV (VF passthrough)**

##### What it is

* Virtual Function exposed directly to pod
* Hardware-enforced isolation
* DPDK-friendly

##### Why telco uses it

* Ultra-high throughput
* Ultra-low latency
* Predictable performance

##### Typical use

* High-scale UPF
* Edge gateways
* 5G core in production

##### Tradeoffs

* Complex ops
* Node pinning
* Finite VFs
* No oversubscription

---

#### Option D — **DPDK + host networking**

##### What it is

* Pod uses:

  * `hostNetwork: true`
  * DPDK binds NIC directly
* Kubernetes almost disappears

##### Why it exists

* Absolute performance
* Legacy VNF compatibility

##### Tradeoffs

* No network isolation
* Hard to schedule
* Essentially “VM in a pod”

Still very common in telco cores.

---

#### Option E — **OVS / bridge-based attachment (underlay)**

*(Less common now, but important historically)*

##### What it is

* Pod connects to:

  * Open vSwitch bridge
* Bridge connects to:

  * Physical NIC

##### Why it was used

* SDN control
* VLAN/VXLAN mapping
* Flow steering

##### Why it’s fading

* Complexity
* Kernel overhead
* MACVLAN/IPVLAN simpler now

Still used in NFV-heavy environments.

---

#### Option F — **Bare-metal / VM outside Kubernetes**

*(Still counts as “secondary”)*

##### What it is

* UPF not in Kubernetes at all
* Kubernetes manages control plane only

##### Why it’s valid

* Maximum determinism
* Easier certification
* Easier HA with classic IP failover

Very common in tier-1 networks.

---

### 3️⃣ What all these have in common (this is the real point)

Regardless of mechanism:

| Property                              | True |
| ------------------------------------- | ---- |
| IP is **not allocated by Kubernetes** | ✅    |
| IP is **stable across restarts**      | ✅    |
| IP is **routable outside cluster**    | ✅    |
| Traffic bypasses kube-proxy           | ✅    |
| No overlay encapsulation              | ✅    |

That’s what “secondary IP” really means.

---

### 4️⃣ What is *not* a valid “secondary IP” (important)

These are **anti-patterns** in telco:

❌ Service IPs
❌ Pod IPs
❌ Ingress IPs
❌ Overlay CNIs (VXLAN, Geneve)
❌ NAT gateways

If you see these on N3/N6, something is wrong.

---

### 5️⃣ How telcos talk about this (real terminology)

In operator language, you’ll hear:

* **“Primary interface”** → Mgmt / control
* **“Fabric interface”** → Data plane
* **“Underlay attachment”**
* **“Bare-metal equivalence”**
* **“Non-cloud-native interface”**

Kubernetes is often described as:

> “An application lifecycle manager, not a network.”

That’s exactly what you’ve discovered.

---

### 6️⃣ How SD-Core fits perfectly into this pattern

SD-Core:

* Primary IP:

  * Pod network
  * SBI
  * AMF/SMF/NRF
* Secondary IP:

  * MACVLAN
  * UPF only
* Static IPAM
* No illusion of elasticity

That’s why it *works*.

---

### Final takeaway (lock this in)

> **Yes — separating Kubernetes “primary” networking from telecom “secondary” networking is not just common, it’s foundational in telco cloud.**
> Secondary interfaces are underlay-attached, statically addressed, and deliberately *outside* Kubernetes control.

If you want, next we can:

* Compare **MACVLAN vs IPVLAN vs SR-IOV for UPF**
* Talk about **IP failover patterns outside Kubernetes**
* Map this to **3GPP deployment models**

You’re thinking like a network architect now — this is the right layer to be reasoning at.

---

## Perfect. Let’s do this **slow, concrete, and architectural**, because this is one of those topics where one wrong abstraction poisons everything downstream.

I’ll explicitly connect this back to **“why SD-Core steps outside the CNI”**, since that’s the anchor.

---

## Static IPAM vs Whereabouts IPAM

*(and why SD-Core deliberately avoids dynamic IPAM)*

---

### First principle (the non-negotiable rule)

> **The N3 network is not a Kubernetes network.**
> It is a **telecom underlay network** that Kubernetes workloads *attach to*.

Once you accept that, most IPAM decisions become obvious.

---

### Option A — **Static IPAM (SD-Core default)**

#### What it really means

Static IPAM does **not** mean “hardcoded forever” in a naive sense.

It means:

* IP ownership lives **outside Kubernetes**
* Kubernetes is only told:

  > “Attach *this* interface with *this* IP”

Kubernetes never allocates, reclaims, or reasons about the address.

---

#### How it’s implemented (mechanically)

**NetworkAttachmentDefinition (MACVLAN):**

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: n3-net
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "mode": "bridge",
      "master": "eno1",
      "ipam": {
        "type": "static"
      }
    }
```

**UPF Pod annotation:**

```yaml
k8s.v1.cni.cncf.io/networks: |
  [
    {
      "name": "n3-net",
      "ips": ["192.168.251.10/24"]
    }
  ]
```

That’s it.

---

#### Why this gives **perfect IP persistence**

* Pod restarts → interface reattached → **same IP**
* Pod upgrade → interface reattached → **same IP**
* SMF, gNB, RAN configs remain valid

There is **no allocator** that could decide otherwise.

---

#### Why SD-Core *likes* this

From SD-Core’s perspective:

* UPF is:

  * Stateful
  * IP-anchored
  * Node-affine
* Kubernetes is:

  * A lifecycle manager
  * *Not* a networking authority

This is why SD-Core **steps outside the CNI**.

> CNIs optimize for elasticity
> UPFs optimize for determinism

These goals are incompatible.

---

#### Pros / Cons (honest)

**Pros**

* Absolute determinism
* Cross-cluster by definition
* No shared state
* No split-brain risk
* Easy to reason about failures

**Cons**

* Manual bookkeeping
* No “scale to 100 UPFs with magic”

But telecom networks have *never* been magically elastic — SD-Core is honest about that.

---

### Option B — **Whereabouts IPAM (Multus)**

Now let’s talk about the tempting option.

---

#### What Whereabouts actually does

Whereabouts is:

* A **distributed IP allocator**
* For secondary interfaces (Multus)
* Backed by:

  * Kubernetes API objects **or**
  * etcd

It hands out IPs dynamically from a pool.

---

#### Why people like it (on paper)

* No manual IP assignment
* Looks Kubernetes-native
* Pods can come and go

This feels “cloud-correct”.

---

#### Why it is **dangerous for N3 / UPF**

Let’s walk through the failure modes.

---

##### 🚨 Failure mode 1: UPF restart timing

* Pod deleted
* IP lease released
* New pod scheduled
* **Allocator may give a different IP**

Result:

* SMF config invalid
* gNB still sending to old IP
* Silent blackhole

This alone disqualifies it for UPF.

---

##### 🚨 Failure mode 2: Multi-cluster coordination

For cross-cluster:

* You now need:

  * Shared etcd **or**
  * Cross-cluster Kubernetes API access

That introduces:

* Split-brain risk
* Latency
* Operational coupling between clusters

Exactly what you were trying to avoid.

---

##### 🚨 Failure mode 3: Debuggability

When GTP-U breaks, you want:

* “UPF is at X”
* “gNB sends to X”

Not:

* “Whereabouts thinks the lease is X but node Y disagrees”

Telecom debugging hates indirection.

---

#### When Whereabouts *is* acceptable

Whereabouts can be okay for:

* **gNB pods**
* **test labs**
* **non-critical RAN simulators**

Because:

* gNBs can re-register
* N3 TEIDs can be re-negotiated

UPF cannot.

---

#### SD-Core’s implicit stance

SD-Core does not ban Whereabouts — but it **architecturally assumes static IPs** for:

* UPF N3
* UPF N6

That assumption leaks everywhere:

* Helm values
* SMF config
* Documentation
* Operational playbooks

This is intentional.

---

### Verdict (Part 1)

| Component     | IPAM choice               |
| ------------- | ------------------------- |
| UPF           | **Static IPAM only** ✅    |
| gNB           | Static or Whereabouts     |
| AMF / SMF     | Pod IPs (CNI)             |
| Cross-cluster | **Never Kubernetes IPAM** |

---

## Concrete multi-cluster IP plan (Core + RAN)

Let’s make this **real**, not abstract.

---

### Goal

* Core in **Cluster A**
* RAN in **Cluster B**
* No cluster networking federation
* Clean failure domains

---

### Step 1: Define networks (globally)

| Network             | Purpose       | CIDR                 |
| ------------------- | ------------- | -------------------- |
| Core Pod Network    | Control plane | 10.244.0.0/16        |
| RAN Pod Network     | Control plane | 10.245.0.0/16        |
| **N3 Data Network** | gNB ↔ UPF     | **192.168.251.0/24** |
| N6 Data Network     | UPF ↔ DN      | 192.168.252.0/24     |
| UE Pool             | PDU sessions  | 172.16.0.0/16        |

Only **N3 / N6** cross clusters.

---

### Step 2: Physical / underlay reality

```
           ┌───────────── Data Plane Network ─────────────┐
           │              (L2 or Routed L3)                │
           │                                                │
   Cluster A NIC                                  Cluster B NIC
   eno1                                           eno1
```

This is a **real switch / fabric**, not Kubernetes.

---

### Step 3: Cluster A (Core)

**UPF**

* Pod network: `10.244.x.x` (irrelevant for N3)
* MACVLAN interface:

  * IP: `192.168.251.10`
  * Parent: `eno1`

**AMF / SMF**

* Pod network only
* Exposed via Service (SCTP/HTTP)

---

### Step 4: Cluster B (RAN)

**OAI gNB**

* Pod network:

  * `10.245.x.x` → N2
* MACVLAN interface:

  * IP: `192.168.251.101`
  * Parent: `eno1`

---

### Step 5: What talks to what

| Path           | Network          | Mechanism |
| -------------- | ---------------- | --------- |
| gNB → AMF (N2) | Pod network      | Service   |
| gNB → UPF (N3) | **Data network** | MACVLAN   |
| UPF → DN (N6)  | Data network     | MACVLAN   |

Clusters never “see” each other.

Pods do.

---

### Step 6: Why this survives restarts

* UPF pod restarts
* MACVLAN reattaches
* `192.168.251.10` returns
* gNB still sends to same IP
* SMF config still valid

This is **IP persistence by design**, not by orchestration tricks.

---

### Bringing it back to your quoted line

> *“This is why SD-Core steps outside the CNI.”*

Now the meaning should be very clear:

* CNI = elastic, ephemeral, cluster-scoped
* N3 = deterministic, stable, multi-cluster

So SD-Core **refuses to lie** and pretend those are compatible.

---

### Where you are architecturally (important)

At this point, you are no longer “deploying SD-Core”.

You are **designing a telecom underlay with Kubernetes as a control plane**.

That’s the correct mental model.

---
## CNI Plugin Hierarchy

### The Actual Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Pod Creation                          │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                  Kubelet                                 │
│  "I need to create network for this pod"                │
└────────────────────┬────────────────────────────────────┘
                     │
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
┌───────────────┐         ┌──────────────────┐
│ PRIMARY CNI   │         │ Does pod have    │
│ (Calico)      │         │ Multus annotation│
│               │         │ ?                │
│ ALWAYS called │         └────────┬─────────┘
│ first         │                  │
└───────┬───────┘                  │
        │                          │
        │ Creates eth0             │ YES → Multus invoked
        │ (10.244.x.x)             │ NO  → Skip
        │                          │
        ▼                          ▼
┌───────────────┐         ┌──────────────────┐
│ eth0 ready    │         │ Multus CNI       │
│               │         │ (meta-plugin)    │
│ Pod has       │         │                  │
│ Kubernetes IP │         │ Calls additional │
│               │         │ CNI plugins      │
└───────────────┘         └────────┬─────────┘
                                   │
                          ┌────────┴─────────┐
                          │                  │
                          ▼                  ▼
                   ┌──────────────┐   ┌──────────────┐
                   │ MacVLAN CNI  │   │ SRIOV CNI    │
                   │              │   │              │
                   │ Creates net1 │   │ Creates net2 │
                   └──────────────┘   └──────────────┘
```

### Key Points

**1. Primary CNI (Calico) is called DIRECTLY by Kubelet**
```
Kubelet → Calico CNI → eth0 created
```

**2. Multus is called SEPARATELY by Kubelet (after primary CNI)**
```
Kubelet → Multus CNI → MacVLAN CNI → net1 created
                     → SRIOV CNI   → net2 created
```

**3. They run in SEQUENCE, not nested**
```
Step 1: Primary CNI runs → Creates eth0
Step 2: Multus runs (if annotation present) → Creates additional interfaces
```

---

### CNI Configuration Files

Let me show you the actual CNI configuration structure on a Kubernetes node:

#### Location: `/etc/cni/net.d/`

```bash
$ ls -la /etc/cni/net.d/
total 24
-rw-r--r-- 1 root root  617 Jan 10 10:00 00-multus.conf
-rw-r--r-- 1 root root  292 Jan 10 09:55 10-calico.conflist
-rw-r--r-- 1 root root  156 Jan 10 10:05 macvlan-conf.conf
```

#### Primary CNI: `10-calico.conflist`

```json
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "ipam": {
        "type": "calico-ipam"
      },
      "policy": {
        "type": "k8s"
      },
      "kubernetes": {
        "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "capabilities": {"portMappings": true}
    }
  ]
}
```

**Kubelet reads this and calls Calico directly.**

#### Multus Configuration: `00-multus.conf`

```json
{
  "name": "multus-cni-network",
  "type": "multus",
  "cniVersion": "0.3.1",
  "kubeconfig": "/etc/cni/net.d/multus.d/multus.kubeconfig",
  "delegates": [
    {
      "cniVersion": "0.3.1",
      "name": "k8s-pod-network",
      "plugins": [
        {
          "type": "calico",
          "log_level": "info",
          "datastore_type": "kubernetes",
          "ipam": {
            "type": "calico-ipam"
          },
          "policy": {
            "type": "k8s"
          },
          "kubernetes": {
            "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
          }
        },
        {
          "type": "portmap",
          "capabilities": {"portMappings": true}
        }
      ]
    }
  ]
}
```

**Notice:** The `delegates` section contains the **SAME** Calico configuration!

#### NetworkAttachmentDefinition: MacVLAN

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.100.0/24"
      }
    }
```

**This is NOT a CNI config file - it's a Kubernetes CRD that Multus reads.**

---

### How It Actually Works

#### Without Multus (Standard Kubernetes)

```
┌─────────────────────────────────────────┐
│  Kubelet CNI Plugin Invocation          │
└────────────────┬────────────────────────┘
                 │
                 ▼
      ┌──────────────────────┐
      │  /etc/cni/net.d/     │
      │                      │
      │  Reads configs in    │
      │  lexicographic order │
      └──────────┬───────────┘
                 │
                 ▼
      ┌──────────────────────┐
      │  10-calico.conflist  │ ← First file (alphabetically)
      │                      │
      │  Kubelet calls this  │
      └──────────┬───────────┘
                 │
                 ▼
      ┌──────────────────────┐
      │   Calico CNI binary  │
      │   /opt/cni/bin/      │
      │   calico             │
      └──────────┬───────────┘
                 │
                 ▼
      ┌──────────────────────┐
      │   eth0 created       │
      │   10.244.1.5/24      │
      └──────────────────────┘
```

#### With Multus (Multi-Interface)

```
┌─────────────────────────────────────────┐
│  Kubelet CNI Plugin Invocation          │
└────────────────┬────────────────────────┘
                 │
                 ▼
      ┌──────────────────────┐
      │  /etc/cni/net.d/     │
      │                      │
      │  Reads configs in    │
      │  lexicographic order │
      └──────────┬───────────┘
                 │
                 ▼
      ┌──────────────────────┐
      │  00-multus.conf      │ ← First file (00- prefix)
      │                      │
      │  Kubelet calls this  │
      └──────────┬───────────┘
                 │
                 ▼
      ┌──────────────────────────────────────┐
      │   Multus CNI binary                  │
      │   /opt/cni/bin/multus                │
      │                                      │
      │   Multus does TWO things:            │
      │                                      │
      │   1. Calls "delegate" (primary CNI)  │
      │      → Calico                        │
      │      → Creates eth0                  │
      │                                      │
      │   2. Checks pod annotations          │
      │      k8s.v1.cni.cncf.io/networks     │
      │                                      │
      │      If annotation exists:           │
      │      → Reads NetworkAttachment       │
      │         Definitions                  │
      │      → Calls additional CNI plugins  │
      │         (MacVLAN, SRIOV, etc.)       │
      └──────────┬───────────────────────────┘
                 │
        ┌────────┴────────┐
        │                 │
        ▼                 ▼
┌───────────────┐  ┌──────────────────┐
│ Calico CNI    │  │ MacVLAN CNI      │
│               │  │                  │
│ Creates eth0  │  │ Creates net1     │
│ 10.244.1.5    │  │ 192.168.100.100  │
└───────────────┘  └──────────────────┘
```

---

### Detailed Call Sequence

#### Step 1: Kubelet Determines Which CNI to Call

```bash
# Kubelet looks in /etc/cni/net.d/
# Picks the FIRST configuration file (alphabetically)

$ ls /etc/cni/net.d/
00-multus.conf        ← This one! (starts with 00)
10-calico.conflist    ← Would be used if no multus
```

#### Step 2: Multus is Invoked

```bash
# Kubelet executes
/opt/cni/bin/multus < 00-multus.conf

# Multus receives CNI ADD command with:
# - Container ID
# - Network namespace
# - Pod name/namespace
```

#### Step 3: Multus Calls Delegate (Primary CNI)

```go
// Inside Multus code (simplified)

// Read the delegate configuration
delegateConf := config.Delegates[0]  // Points to Calico

// Call the delegate CNI plugin
result, err := invoke.DelegateAdd(
    context.TODO(),
    delegateConf.Name,  // "calico"
    delegateConf.Config,
    containerID,
    netns,
)

// Calico creates eth0 with IP 10.244.1.5
```

#### Step 4: Multus Checks for Annotations

```go
// Get pod information
pod := k8sClient.GetPod(namespace, podName)

// Check for network annotation
networksAnnotation := pod.Annotations["k8s.v1.cni.cncf.io/networks"]

if networksAnnotation != "" {
    // Parse annotation: "macvlan-conf, sriov-conf"
    networks := parseNetworks(networksAnnotation)
    
    for _, network := range networks {
        // Get NetworkAttachmentDefinition
        nad := k8sClient.GetNetworkAttachmentDefinition(network)
        
        // Call the specified CNI plugin
        result, err := invoke.DelegateAdd(
            context.TODO(),
            nad.Spec.Config.Type,  // "macvlan"
            nad.Spec.Config,
            containerID,
            netns,
        )
        
        // MacVLAN creates net1 with IP 192.168.100.100
    }
}
```

#### Step 5: All Interfaces Created

```bash
# Final result inside pod
$ ip addr

1: lo: <LOOPBACK,UP>
    inet 127.0.0.1/8

2: eth0@if10: <BROADCAST,MULTICAST,UP>
    inet 10.244.1.5/24
    ↑ Created by Calico (via Multus delegate)

3: net1@if5: <BROADCAST,MULTICAST,UP>
    inet 192.168.100.100/24
    ↑ Created by MacVLAN (via Multus additional networks)
```

---

### Configuration File Naming Convention

The `00-` prefix is crucial:

```bash
/etc/cni/net.d/
├── 00-multus.conf         ← Loaded FIRST (Multus as wrapper)
├── 10-calico.conflist     ← Referenced by Multus as delegate
└── macvlan-conf.conf      ← Not used by Kubelet directly!
                              Only used by Multus via NAD
```

**Without the `00-` prefix:**
```bash
/etc/cni/net.d/
├── 10-calico.conflist     ← Loaded FIRST (no Multus)
├── 20-multus.conf         ← IGNORED! (Calico already handled it)
```

In this case, Multus would never be called!

---

### Why This Design?

#### Multus is a "Meta-Plugin" or "Wrapper"

Multus doesn't manage networks itself - it:
1. **Delegates** primary network to another CNI (Calico, Flannel, etc.)
2. **Orchestrates** additional network attachments

#### Benefits of This Architecture

**1. Backward Compatibility**
```
Existing CNI plugins (Calico, Flannel) work unchanged
No modification needed to primary CNI
```

**2. Single Entry Point**
```
Kubelet only calls ONE CNI plugin (Multus)
Multus handles all the complexity
```

**3. Flexibility**
```
Primary CNI: Can be ANY CNI (Calico, Flannel, Cilium, etc.)
Secondary CNIs: Can be ANY number of additional networks
```

**4. Clean Separation**
```
Primary network: Kubernetes management, Services, DNS
Secondary networks: Workload-specific (N2, N3, storage, etc.)
```

---

### Common Misconception vs Reality

#### ❌ Misconception

```
Kubelet
  ↓
Multus (manages everything)
  ↓
├─ Calico (creates eth0)
├─ MacVLAN (creates net1)
└─ SRIOV (creates net2)

"Multus controls all CNI plugins"
```

#### ✅ Reality

```
Kubelet
  ↓
Multus (orchestrator/wrapper)
  ↓
  ├─ Delegates to Calico (for eth0)
  │   └─ Calico creates eth0
  │
  └─ Calls additional plugins (for net1, net2, etc.)
      ├─ MacVLAN creates net1
      └─ SRIOV creates net2

"Multus delegates primary, adds secondary"
```

---

## Verification Commands

#### Check CNI Configuration Order

```bash
# On Kubernetes node
$ ls -la /etc/cni/net.d/
-rw-r--r-- 1 root root  617 Jan 10 10:00 00-multus.conf
-rw-r--r-- 1 root root  292 Jan 10 09:55 10-calico.conflist

# Kubelet uses 00-multus.conf (first alphabetically)
```

#### Check Multus Delegate Configuration

```bash
# Inside 00-multus.conf
$ cat /etc/cni/net.d/00-multus.conf | jq .delegates

[
  {
    "name": "k8s-pod-network",
    "plugins": [
      {
        "type": "calico",    ← Delegates to Calico
        ...
      }
    ]
  }
]
```

#### Check Pod Network Setup

```bash
# Create pod with annotation
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-conf
spec:
  containers:
  - name: test
    image: alpine
    command: ["sleep", "3600"]
EOF

# Check interfaces
$ kubectl exec test-pod -- ip addr
2: eth0@if10: ...      ← Created by Calico (via Multus delegate)
3: net1@if5: ...       ← Created by MacVLAN (via Multus additional)
```

#### Check Multus Logs

```bash
# On Kubernetes node
$ journalctl -u kubelet | grep multus

Multus: Adding pod to network k8s-pod-network (delegate)
Multus: Adding pod to network macvlan-conf (additional)
```

---

### Summary

#### The Key Distinction

| Aspect | Primary CNI (Calico) | Multus |
|--------|---------------------|--------|
| **Called by** | Kubelet (via Multus delegate) | Kubelet (directly) |
| **Purpose** | Create primary interface (eth0) | Orchestrate all networking |
| **Configuration** | Referenced in Multus delegate | CNI config file (00-multus.conf) |
| **Relationship** | Peer (via delegation) | **NOT parent/child** |
| **Always runs?** | Yes (for every pod) | Only if Multus is configured |

#### The Correct Mental Model

```
Multus is NOT a container for other CNIs.
Multus is a COORDINATOR that:
  1. Delegates primary networking to another CNI
  2. Adds additional network interfaces based on annotations
```

**Calico does NOT run "under" Multus.**
**Calico runs "via delegation from" Multus.**

This is an important distinction! Multus doesn't "wrap" or "contain" other CNI plugins - it **orchestrates** them through delegation and additional network attachment.

Does this clarify the relationship between Multus and the primary CNI?
---

## NADs
**NetworkAttachmentDefinition (NAD) is ONLY for secondary interfaces, NOT for the primary interface.**

---

### NetworkAttachmentDefinition Scope

#### Primary Interface (eth0) - NO NAD ❌

The primary interface is configured through:
1. **CNI configuration file** in `/etc/cni/net.d/`
2. **Cluster-wide CNI installation** (Calico, Flannel, etc.)
3. **NOT** through NetworkAttachmentDefinition CRD

```bash
# Primary CNI configuration
/etc/cni/net.d/10-calico.conflist

# This is a FILE, not a Kubernetes resource
# Applied at cluster setup time
# Affects ALL pods automatically
```

#### Secondary Interfaces (net1, net2, etc.) - YES NAD ✅

Secondary interfaces are configured through:
1. **NetworkAttachmentDefinition** CRD (Kubernetes resource)
2. **Per-pod annotation** referencing the NAD
3. **Managed by Multus**

```yaml
# This IS a Kubernetes resource
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf
spec:
  config: |
    {
      "type": "macvlan",
      ...
    }
```

---

### Complete Comparison

#### Primary Interface Configuration

**Method: CNI Config File**

```bash
# Location: /etc/cni/net.d/10-calico.conflist
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "ipam": {
        "type": "calico-ipam"
      },
      "policy": {
        "type": "k8s"
      }
    }
  ]
}
```

**Characteristics:**
- ❌ NOT a Kubernetes resource
- ❌ NOT visible with `kubectl get`
- ❌ NOT namespaced
- ✅ Cluster-wide configuration
- ✅ Applied to ALL pods automatically
- ✅ No pod annotation needed

**Pod Creation:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  # NO annotation needed for primary interface!
spec:
  containers:
  - name: test
    image: alpine
```

**Result:**
```bash
$ kubectl exec test-pod -- ip addr
2: eth0@if10: <BROADCAST,MULTICAST,UP>
    inet 10.244.1.5/24
    ↑
    Automatically created by Calico (primary CNI)
    No NAD required!
```

---

#### Secondary Interface Configuration

**Method: NetworkAttachmentDefinition**

```yaml
# This IS a Kubernetes resource (CRD)
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf
  namespace: default  # NADs are namespaced
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.100.0/24"
      }
    }
```

**Characteristics:**
- ✅ IS a Kubernetes resource (CRD)
- ✅ Visible with `kubectl get net-attach-def`
- ✅ Namespaced (belongs to a namespace)
- ✅ Applied per-pod via annotation
- ✅ Optional (pods can ignore it)
- ✅ Managed by Multus

**Pod Creation:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-conf  # EXPLICIT reference needed
spec:
  containers:
  - name: test
    image: alpine
```

**Result:**
```bash
$ kubectl exec test-pod -- ip addr
2: eth0@if10: <BROADCAST,MULTICAST,UP>
    inet 10.244.1.5/24
    ↑ Primary (from CNI config file)

3: net1@if5: <BROADCAST,MULTICAST,UP>
    inet 192.168.100.100/24
    ↑ Secondary (from NetworkAttachmentDefinition)
```

---

### Why This Separation?

#### Design Rationale

**Primary Interface:**
- Required for Kubernetes to function
- Must exist on EVERY pod
- Provides basic connectivity (Services, DNS, etc.)
- Cluster administrator decides (cluster-wide setting)
- Can't be different per pod

**Secondary Interfaces:**
- Optional, workload-specific
- Only attached when explicitly requested
- Provide specialized connectivity (telco, storage, etc.)
- Application developer decides (per-pod annotation)
- Can be different for each pod

---

### Detailed Architecture

#### Without NetworkAttachmentDefinition

```
┌────────────────────────────────────────────┐
│           Kubernetes Cluster               │
│                                            │
│  CNI Config: /etc/cni/net.d/              │
│  ┌────────────────────────────┐           │
│  │  10-calico.conflist        │           │
│  │  (cluster-wide)            │           │
│  └────────────┬───────────────┘           │
│               │                            │
│               │ Applied to ALL pods        │
│               │                            │
│         ┌─────┴──────┬──────────┐         │
│         │            │          │         │
│    ┌────▼───┐   ┌───▼────┐ ┌───▼────┐   │
│    │Pod A   │   │Pod B   │ │Pod C   │   │
│    │        │   │        │ │        │   │
│    │eth0    │   │eth0    │ │eth0    │   │
│    │10.244  │   │10.244  │ │10.244  │   │
│    │.1.5    │   │.1.6    │ │.1.7    │   │
│    └────────┘   └────────┘ └────────┘   │
│                                            │
│  All pods get eth0 automatically          │
│  No NAD, no annotation needed             │
└────────────────────────────────────────────┘
```

#### With NetworkAttachmentDefinition

```
┌─────────────────────────────────────────────────────────┐
│              Kubernetes Cluster                         │
│                                                         │
│  CNI Config: /etc/cni/net.d/                           │
│  ┌────────────────────────────┐                        │
│  │  00-multus.conf            │                        │
│  │  (delegates to calico)     │                        │
│  └────────────┬───────────────┘                        │
│               │                                         │
│               │ ALL pods get eth0                       │
│               │                                         │
│         ┌─────┴──────┬──────────┐                      │
│         │            │          │                      │
│    ┌────▼───┐   ┌───▼────┐ ┌───▼────┐                │
│    │Pod A   │   │Pod B   │ │Pod C   │                │
│    │(no ann)│   │(with   │ │(with   │                │
│    │        │   │ ann)   │ │ ann)   │                │
│    │eth0    │   │eth0    │ │eth0    │                │
│    │10.244  │   │10.244  │ │10.244  │                │
│    │.1.5    │   │.1.6    │ │.1.7    │                │
│    └────────┘   │        │ │        │                │
│                 │net1    │ │net1    │                │
│                 │192.168 │ │192.168 │                │
│                 │.100.50 │ │.100.51 │                │
│                 └───▲────┘ └───▲────┘                │
│                     │          │                      │
│  ┌──────────────────┴──────────┴─────────────┐       │
│  │  NetworkAttachmentDefinition               │       │
│  │  name: macvlan-conf                        │       │
│  │  (Kubernetes CRD)                          │       │
│  │                                             │       │
│  │  Only pods WITH annotation get net1        │       │
│  └─────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────┘
```

---

### Can You Use NAD for Primary Interface?

#### Technically: Yes, but WRONG ❌

There's an advanced (and rarely used) Multus feature that allows defining the primary interface via NAD:

```yaml
# DON'T DO THIS (unless you have very specific reasons)
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: calico-primary
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/resourceName: "primary"
spec:
  config: |
    {
      "type": "calico",
      "ipam": {
        "type": "calico-ipam"
      }
    }
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  annotations:
    k8s.v1.cni.cncf.io/networks: '[
      {"name": "calico-primary", "interface": "eth0", "default-route": ["true"]}
    ]'
spec:
  containers:
  - name: test
    image: alpine
```

**Why this is BAD:**

1. ❌ **Breaks Kubernetes assumptions** - Every pod should have eth0 automatically
2. ❌ **Violates cluster-wide policy** - Primary CNI should be cluster-wide decision
3. ❌ **Complicates management** - Each pod needs explicit annotation
4. ❌ **No isolation** - NADs are namespaced, CNI config is cluster-wide
5. ❌ **Debugging nightmare** - Why doesn't my pod have eth0?
6. ❌ **Breaks default behavior** - Pods without annotation won't have network

#### Practically: No, DON'T DO IT ❌

**Standard Practice:**
- Primary interface: CNI config file (cluster-wide)
- Secondary interfaces: NetworkAttachmentDefinition (per-pod)

---

### Managing NetworkAttachmentDefinitions

#### Create NAD

```bash
# Create MacVLAN network definition
kubectl apply -f - <<EOF
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-n3
  namespace: ran
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "static"
      }
    }
EOF
```

#### View NADs

```bash
# List all NetworkAttachmentDefinitions
$ kubectl get network-attachment-definitions -A

NAMESPACE   NAME          AGE
ran         macvlan-n3    5m
core        macvlan-n2    10m
default     sriov-net     15m

# Short form
$ kubectl get net-attach-def -A
```

#### Describe NAD

```bash
$ kubectl describe net-attach-def macvlan-n3 -n ran

Name:         macvlan-n3
Namespace:    ran
Labels:       <none>
Annotations:  <none>
API Version:  k8s.cni.cncf.io/v1
Kind:         NetworkAttachmentDefinition
Metadata:
  ...
Spec:
  Config:  {
    "cniVersion": "0.3.1",
    "type": "macvlan",
    "master": "eth1",
    "mode": "bridge",
    "ipam": {
      "type": "static"
    }
  }
```

#### Use NAD in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: upf-0
  namespace: ran
  annotations:
    # Reference NAD by name
    k8s.v1.cni.cncf.io/networks: macvlan-n3
spec:
  containers:
  - name: upf
    image: upf:v1
```

---

### Multiple NADs Per Pod

You can attach multiple secondary networks to a pod:

#### Multiple NADs - Simple Syntax

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: upf-0
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-n3, macvlan-n6, sriov-net
    #                            net1      net2        net3
spec:
  containers:
  - name: upf
    image: upf:v1
```

**Result:**
```bash
$ kubectl exec upf-0 -- ip addr
2: eth0@if10: ...     # Primary (Calico)
3: net1@if5: ...      # macvlan-n3
4: net2@if6: ...      # macvlan-n6
5: net3@if7: ...      # sriov-net
```

#### Multiple NADs - Advanced Syntax with IP Assignment

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: upf-0
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [
        {
          "name": "macvlan-n3",
          "interface": "n3",
          "ips": ["192.168.100.10/24"]
        },
        {
          "name": "macvlan-n6",
          "interface": "n6",
          "ips": ["192.168.200.10/24"]
        }
      ]
spec:
  containers:
  - name: upf
    image: upf:v1
```

**Result:**
```bash
$ kubectl exec upf-0 -- ip addr
2: eth0@if10: inet 10.244.1.5/24       # Primary
3: n3@if5: inet 192.168.100.10/24      # macvlan-n3
4: n6@if6: inet 192.168.200.10/24      # macvlan-n6
```

---

### Cross-Namespace NAD References

NADs are **namespaced resources**, but you can reference NADs from other namespaces:

#### NAD in Different Namespace

```yaml
# NAD in 'network-config' namespace
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: shared-macvlan
  namespace: network-config
spec:
  config: |
    {
      "type": "macvlan",
      "master": "eth1",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.100.0/24"
      }
    }
```

```yaml
# Pod in 'ran' namespace referencing NAD from 'network-config' namespace
apiVersion: v1
kind: Pod
metadata:
  name: gnb-0
  namespace: ran
  annotations:
    k8s.v1.cni.cncf.io/networks: network-config/shared-macvlan
    #                            namespace/nad-name
spec:
  containers:
  - name: gnb
    image: gnb:v1
```

---

### Summary Table

| Aspect | Primary Interface (eth0) | Secondary Interfaces (net1, net2, ...) |
|--------|-------------------------|---------------------------------------|
| **Configuration Method** | CNI config file | NetworkAttachmentDefinition CRD |
| **Location** | `/etc/cni/net.d/` | Kubernetes API (stored in etcd) |
| **Kubernetes Resource?** | ❌ No (plain file) | ✅ Yes (CRD) |
| **Visible with kubectl?** | ❌ No | ✅ Yes (`kubectl get net-attach-def`) |
| **Namespaced?** | ❌ No (cluster-wide) | ✅ Yes |
| **Applied to Pods** | Automatically (all pods) | Explicitly (via annotation) |
| **Managed By** | Cluster Admin | Application Developer |
| **Annotation Required?** | ❌ No | ✅ Yes |
| **Can be Different Per Pod?** | ❌ No (same for all) | ✅ Yes (per-pod choice) |
| **Created At** | Cluster setup time | Runtime (like any K8s resource) |
| **Modified By** | Editing file + restart | `kubectl apply` |
| **Example** | Calico, Flannel, Cilium | MacVLAN, SRIOV, IPVLAN |

---

### Practical Example: SD-Core

In SD-Core deployment:

#### Primary Interface - NO NAD

```bash
# On each node: /etc/cni/net.d/10-calico.conflist
# (or whatever primary CNI you use)

# ALL pods get eth0 automatically
# Used for:
# - Kubernetes Services
# - DNS resolution
# - Inter-pod communication (within cluster)
# - Health checks
```

#### Secondary Interfaces - YES NAD

```yaml
# NetworkAttachmentDefinition for N3
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: n3-network
  namespace: core
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "ens18",
      "mode": "bridge",
      "ipam": {
        "type": "static"
      }
    }
```

```yaml
# UPF pod using the NAD
apiVersion: v1
kind: Pod
metadata:
  name: upf-0
  namespace: core
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [{
        "name": "n3-network",
        "interface": "access",
        "ips": ["192.168.252.3/24"]
      }]
spec:
  containers:
  - name: upf
    image: upf:latest
```

**Result:**
- `eth0`: Primary Kubernetes network (Calico) - automatic
- `access`: N3 interface (MacVLAN) - via NAD

---

### Key Takeaway

**NetworkAttachmentDefinition is ONLY for secondary interfaces.**

- ✅ **Primary interface (eth0)**: CNI config file (cluster-wide)
- ✅ **Secondary interfaces (net1, net2, ...)**: NetworkAttachmentDefinition (per-pod)

This separation allows:
- Cluster admins to control primary networking (stability, security)
- Application developers to add specialized networks as needed (flexibility)

**Never use NAD for primary interface unless you have an extremely specific use case and fully understand the implications!**

Does this clarify when and how NetworkAttachmentDefinition is used?

---
## IPAM Overview

IPAM plugins are responsible for **allocating and managing IP addresses** for network interfaces. They work with CNI plugins (like MacVLAN, bridge, etc.) to assign IPs to pod interfaces.

---

### 1. host-local IPAM

#### What is host-local?

`host-local` is the **simplest and most basic IPAM plugin**. It manages IP addresses **locally on each node** using files stored on the node's filesystem.

#### Key Characteristics

- ✅ **Node-local**: Each node manages its own IP allocations independently
- ✅ **File-based storage**: Uses files in `/var/lib/cni/networks/` to track allocations
- ✅ **Simple**: No coordination between nodes
- ✅ **Fast**: No network calls, just local file operations
- ❌ **No cross-node awareness**: Different nodes don't know about each other's allocations
- ❌ **Conflicts possible**: Multiple nodes can allocate the same IP if misconfigured

#### How It Works

```
┌─────────────────────────────────────────────────────────┐
│                    Node 1                               │
│                                                         │
│  Pod A created with MacVLAN                            │
│      ↓                                                  │
│  host-local IPAM called                                │
│      ↓                                                  │
│  Reads: /var/lib/cni/networks/macvlan-net/             │
│      ├── 192.168.100.10  (in use)                      │
│      ├── 192.168.100.11  (in use)                      │
│      └── 192.168.100.12  (free)                        │
│      ↓                                                  │
│  Allocates: 192.168.100.12                             │
│      ↓                                                  │
│  Creates file: /var/lib/cni/networks/macvlan-net/      │
│                192.168.100.12                          │
│      ↓                                                  │
│  Returns IP to MacVLAN plugin                          │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    Node 2                               │
│                                                         │
│  Pod B created with MacVLAN                            │
│      ↓                                                  │
│  host-local IPAM called                                │
│      ↓                                                  │
│  Reads: /var/lib/cni/networks/macvlan-net/             │
│      ├── 192.168.100.13  (in use)                      │
│      └── 192.168.100.14  (free)                        │
│      ↓                                                  │
│  Allocates: 192.168.100.14                             │
│      ↓                                                  │
│  Node 2 has NO IDEA what Node 1 allocated!             │
│  Could allocate 192.168.100.12 if range overlaps ❌    │
└─────────────────────────────────────────────────────────┘
```

#### Configuration Example

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-hostlocal
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "ranges": [
          [{
            "subnet": "192.168.100.0/24",
            "rangeStart": "192.168.100.10",
            "rangeEnd": "192.168.100.100",
            "gateway": "192.168.100.1"
          }]
        ],
        "routes": [
          {
            "dst": "0.0.0.0/0",
            "gw": "192.168.100.1"
          }
        ]
      }
    }
```

#### File-Based Storage

**On each node:**
```bash
# IP allocation storage location
$ ls -la /var/lib/cni/networks/macvlan-hostlocal/

total 16
drwxr-xr-x 2 root root 4096 Jan 25 10:00 .
drwxr-xr-x 3 root root 4096 Jan 25 09:50 ..
-rw-r--r-- 1 root root   73 Jan 25 10:00 192.168.100.10
-rw-r--r-- 1 root root   73 Jan 25 10:01 192.168.100.11
-rw-r--r-- 1 root root   73 Jan 25 10:02 192.168.100.12
-rw-r--r-- 1 root root   48 Jan 25 09:55 last_reserved_ip.0
```

**File content:**
```bash
$ cat /var/lib/cni/networks/macvlan-hostlocal/192.168.100.10

7f8a3b2c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b
```
This is the **container ID** that owns this IP.

**Last reserved IP:**
```bash
$ cat /var/lib/cni/networks/macvlan-hostlocal/last_reserved_ip.0

192.168.100.12
```
Tracks the last allocated IP to continue from.

### Allocation Algorithm (host-local)

```go
// Simplified host-local logic

func AllocateIP(subnet, rangeStart, rangeEnd) (IP, error) {
    // 1. Read last allocated IP from file
    lastIP := readLastReservedIP()
    
    // 2. Try next IP in sequence
    nextIP := incrementIP(lastIP)
    
    // 3. Check if IP is already allocated
    for nextIP <= rangeEnd {
        ipFile := filepath.Join(dataDir, nextIP.String())
        
        // If file doesn't exist, IP is free
        if !fileExists(ipFile) {
            // Allocate it
            writeFile(ipFile, containerID)
            writeLastReservedIP(nextIP)
            return nextIP, nil
        }
        
        // Try next IP
        nextIP = incrementIP(nextIP)
    }
    
    // No IPs available
    return nil, errors.New("no IPs available")
}
```

#### Deallocation (host-local)

```bash
# When pod is deleted
$ rm /var/lib/cni/networks/macvlan-hostlocal/192.168.100.10

# IP is now available for reuse
```

#### Use Cases for host-local

✅ **Good for:**
- Single-node clusters
- Non-overlapping IP ranges across nodes (manual partitioning)
- Simple test/dev environments
- Scenarios where you control IP ranges per node

❌ **Bad for:**
- Multi-node clusters with shared IP pool
- Dynamic node scaling
- Cross-cluster scenarios (like Nephio)
- Any case requiring coordinated IP management

---

### 2. whereabouts IPAM

#### What is whereabouts?

`whereabouts` is a **cluster-aware IPAM plugin** that coordinates IP allocations across all nodes using a shared backend (Kubernetes API or etcd).

#### Key Characteristics

- ✅ **Cluster-wide**: All nodes share the same IP pool
- ✅ **Coordination**: Prevents duplicate allocations across nodes
- ✅ **Dynamic**: Automatically handles node additions/removals
- ✅ **Multiple backends**: Kubernetes API (default) or etcd
- ✅ **Reconciliation**: Can detect and fix stale allocations
- ✅ **Overlapping ranges**: Supports same subnet across multiple networks
- ⚠️ **Slightly slower**: Network API calls vs local file I/O
- ⚠️ **Requires permissions**: Needs RBAC to access Kubernetes API

#### How It Works

```
┌─────────────────────────────────────────────────────────┐
│                    Node 1                               │
│                                                         │
│  Pod A created with MacVLAN                            │
│      ↓                                                  │
│  whereabouts IPAM called                               │
│      ↓                                                  │
│  Queries Kubernetes API:                               │
│  "What IPs are allocated in this range?"               │
│      ↓                                                  │
│      ┌──────────────────────────────────┐             │
│      │   Kubernetes API Server          │             │
│      │   (Shared State)                 │             │
│      │                                   │             │
│      │   IPPools:                        │             │
│      │   - 192.168.100.10 (Node 1, Pod A)│             │
│      │   - 192.168.100.11 (Node 2, Pod B)│             │
│      │   - 192.168.100.12 (free)         │             │
│      └──────────────┬───────────────────┘             │
│                     │                                  │
│      Allocates: 192.168.100.12                        │
│      ↓                                                  │
│  Creates IPPool CRD in Kubernetes                      │
│      ↓                                                  │
│  Returns IP to MacVLAN plugin                          │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    Node 2                               │
│                                                         │
│  Pod B created with MacVLAN                            │
│      ↓                                                  │
│  whereabouts IPAM called                               │
│      ↓                                                  │
│  Queries Kubernetes API:                               │
│  "What IPs are allocated?"                             │
│      ↓                                                  │
│      ┌──────────────────────────────────┐             │
│      │   Kubernetes API Server          │             │
│      │                                   │             │
│      │   IPPools:                        │             │
│      │   - 192.168.100.10 (Node 1)      │             │
│      │   - 192.168.100.11 (Node 2)      │             │
│      │   - 192.168.100.12 (Node 1)  ← Sees this!      │
│      │   - 192.168.100.13 (free)         │             │
│      └──────────────┬───────────────────┘             │
│                     │                                  │
│      Allocates: 192.168.100.13                        │
│      (Skips .12 because Node 1 already has it!)       │
│                                                         │
│  ✅ No conflicts!                                      │
└─────────────────────────────────────────────────────────┘
```

#### Configuration Example

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-whereabouts
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.100.0/24",
        "range_start": "192.168.100.10",
        "range_end": "192.168.100.200",
        "gateway": "192.168.100.1",
        "routes": [
          {
            "dst": "0.0.0.0/0",
            "gw": "192.168.100.1"
          }
        ],
        "kubernetes": {
          "kubeconfig": "/etc/cni/net.d/whereabouts.d/whereabouts.kubeconfig"
        }
      }
    }
```

#### Kubernetes API Storage (Default)

Whereabouts stores allocations as **Kubernetes Custom Resources**:

```bash
# Check IP allocations
$ kubectl get ippools.whereabouts.cni.cncf.io -A

NAMESPACE   NAME                                          AGE
kube-system macvlan-whereabouts-192-168-100-0-24         5m
```

```bash
# Describe the IPPool
$ kubectl describe ippool macvlan-whereabouts-192-168-100-0-24 -n kube-system

Name:         macvlan-whereabouts-192-168-100-0-24
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>
API Version:  whereabouts.cni.cncf.io/v1alpha1
Kind:         IPPool
Metadata:
  ...
Spec:
  Allocations:
    192.168.100.10:
      Container ID: 7f8a3b2c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a
      Pod Name:     upf-0
      Pod Namespace: ran
      Interface:    net1
    192.168.100.11:
      Container ID: 9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c
      Pod Name:     amf-0
      Pod Namespace: core
      Interface:    net1
    192.168.100.12:
      Container ID: 1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c
      Pod Name:     smf-0
      Pod Namespace: core
      Interface:    net1
  Range: 192.168.100.0/24
  Range Start: 192.168.100.10
  Range End: 192.168.100.200
```

### Allocation Algorithm (whereabouts)

```go
// Simplified whereabouts logic

func AllocateIP(config IPAMConfig) (IP, error) {
    // 1. Get or create IPPool from Kubernetes API
    pool := getOrCreateIPPool(config.Range)
    
    // 2. Lock the pool (using leader election)
    lock := acquireLock(pool)
    defer lock.Release()
    
    // 3. Find first available IP
    for ip := config.RangeStart; ip <= config.RangeEnd; ip++ {
        if !pool.IsAllocated(ip) {
            // 4. Mark as allocated
            pool.Allocations[ip] = AllocationInfo{
                ContainerID: containerID,
                PodName:     podName,
                PodNamespace: podNamespace,
                Interface:   interfaceName,
            }
            
            // 5. Update IPPool in Kubernetes API
            updateIPPool(pool)
            
            return ip, nil
        }
    }
    
    return nil, errors.New("no IPs available")
}
```

#### Deallocation (whereabouts)

```bash
# When pod is deleted
# whereabouts removes entry from IPPool

$ kubectl get ippool macvlan-whereabouts-192-168-100-0-24 -o yaml

spec:
  allocations:
    192.168.100.10:  # Still there
    192.168.100.11:  # Still there
    # 192.168.100.12 removed when pod deleted
```

#### Reconciliation Feature

Whereabouts has a **reconciler** that periodically checks for stale allocations:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: whereabouts-reconciler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: whereabouts
        image: ghcr.io/k8snetworkplumbingwg/whereabouts:latest
        command:
        - /bin/sh
        - -c
        - |
          /ip-reconciler \
            --log-level=debug \
            --interval=30  # Check every 30 seconds
```

**What reconciler does:**
1. Lists all IPPools
2. For each allocated IP, checks if pod still exists
3. If pod doesn't exist, removes the allocation
4. Frees up stale IPs from crashed pods, failed deletions, etc.

---

### Comparison: host-local vs whereabouts

#### Feature Comparison

| Feature | host-local | whereabouts |
|---------|-----------|-------------|
| **Storage Backend** | Local filesystem | Kubernetes API / etcd |
| **Scope** | Per-node | Cluster-wide |
| **Coordination** | ❌ None | ✅ Kubernetes API |
| **Conflict Prevention** | ❌ Manual (non-overlapping ranges) | ✅ Automatic |
| **Stale IP Cleanup** | ❌ Manual | ✅ Automatic (reconciler) |
| **Performance** | ⚡ Very fast (local files) | 🐌 Slightly slower (API calls) |
| **Complexity** | ✅ Simple | ⚠️ Moderate |
| **RBAC Required** | ❌ No | ✅ Yes |
| **Multi-cluster** | ❌ No | ✅ Yes (with shared etcd) |
| **Scalability** | Limited (per-node) | High (cluster-wide) |
| **Debugging** | Easy (just check files) | Moderate (check CRDs) |

#### IP Range Management

**host-local: Manual Partitioning Required**
```
Node 1: 192.168.100.10 - 192.168.100.50   (manually configured)
Node 2: 192.168.100.51 - 192.168.100.100  (manually configured)
Node 3: 192.168.100.101 - 192.168.100.150 (manually configured)

If ranges overlap → IP conflicts! ❌
```

**whereabouts: Shared Pool**
```
All nodes: 192.168.100.10 - 192.168.100.200 (shared pool)

Node 1 pod gets: 192.168.100.10
Node 2 pod gets: 192.168.100.11
Node 3 pod gets: 192.168.100.12
Node 1 pod gets: 192.168.100.13

No conflicts, automatic! ✅
```

### Performance Characteristics

**host-local:**
```
IP Allocation Time: ~1ms
  - Read local file
  - Write local file
  - Done
```

**whereabouts:**
```
IP Allocation Time: ~10-50ms
  - Query Kubernetes API (network call)
  - Acquire lock (leader election)
  - Update IPPool CRD
  - Wait for API write to complete
  - Done
```

For most use cases, this difference is negligible!

---

### Advanced whereabouts Features

#### 1. Multiple Ranges per Network

```yaml
spec:
  config: |
    {
      "type": "macvlan",
      "ipam": {
        "type": "whereabouts",
        "ranges": [
          {
            "range": "192.168.100.0/24",
            "range_start": "192.168.100.10",
            "range_end": "192.168.100.100"
          },
          {
            "range": "192.168.101.0/24",
            "range_start": "192.168.101.10",
            "range_end": "192.168.101.100"
          }
        ]
      }
    }
```

Whereabouts will allocate from first available pool.

#### 2. Exclude IPs

```yaml
spec:
  config: |
    {
      "type": "macvlan",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.100.0/24",
        "exclude": [
          "192.168.100.1/32",    # Gateway
          "192.168.100.2/32",    # DNS server
          "192.168.100.50-192.168.100.60"  # Reserved range
        ]
      }
    }
```

#### 3. IP Reservations

```yaml
apiVersion: whereabouts.cni.cncf.io/v1alpha1
kind: OverlappingRangeIPReservation
metadata:
  name: amf-n2-reservation
  namespace: core
spec:
  containerid: "amf-0"
  ifname: "net1"
  ip: "192.168.100.100"
  podref: "core/amf-0"
```

This ensures `192.168.100.100` is always reserved for `amf-0`.

#### 4. Static IP with whereabouts

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: upf-0
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [{
        "name": "macvlan-whereabouts",
        "ips": ["192.168.100.50/24"]  # Request specific IP
      }]
spec:
  containers:
  - name: upf
    image: upf:v1
```

Whereabouts will:
1. Check if `192.168.100.50` is available
2. If yes, allocate it
3. If no, fail (won't allocate different IP)

#### 5. Shared etcd Backend (Multi-Cluster)

```yaml
spec:
  config: |
    {
      "type": "macvlan",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.100.0/24",
        "etcd_endpoints": "https://etcd1:2379,https://etcd2:2379",
        "etcd_key_file": "/etc/whereabouts/etcd-key.pem",
        "etcd_cert_file": "/etc/whereabouts/etcd-cert.pem",
        "etcd_ca_cert_file": "/etc/whereabouts/etcd-ca.pem"
      }
    }
```

**Use case:** Multiple Kubernetes clusters sharing the same IP pool (like Nephio multi-cluster).

---

### Which Should You Use?

#### Use host-local When:

✅ **Single-node cluster**
```
Simple dev/test environment with one node
```

✅ **Manual IP range partitioning**
```
You carefully partition ranges per node:
Node 1: .10-.50
Node 2: .51-.100
```

✅ **Maximum performance critical**
```
Extremely high pod churn, microseconds matter
```

✅ **Simplicity is priority**
```
Don't want to deal with CRDs, RBAC, etc.
```

#### Use whereabouts When:

✅ **Multi-node cluster**
```
Standard production multi-node setup
```

✅ **Shared IP pool needed**
```
All nodes should allocate from same pool
```

✅ **Dynamic scaling**
```
Nodes added/removed dynamically
```

✅ **Multi-cluster scenarios**
```
Nephio, federated clusters sharing IP space
```

✅ **Automatic stale IP cleanup**
```
Reconciler handles orphaned allocations
```

✅ **Production telco deployments**
```
Reliability and conflict prevention critical
```

---

### Real-World Example: SD-Core

In SD-Core (as you discovered), they likely use:

#### For N3 Interface (UPF)

**Small deployments:**
```yaml
# host-local with manual range per node
ipam:
  type: host-local
  ranges:
    - subnet: 192.168.252.0/24
      rangeStart: 192.168.252.10
      rangeEnd: 192.168.252.20
```

**Production deployments:**
```yaml
# whereabouts for cluster-wide coordination
ipam:
  type: whereabouts
  range: 192.168.252.0/24
  range_start: 192.168.252.10
  range_end: 192.168.252.200
```

---

### Debugging Commands

#### host-local Debugging

```bash
# Check allocations on current node
ls -la /var/lib/cni/networks/macvlan-hostlocal/

# Check which container has an IP
cat /var/lib/cni/networks/macvlan-hostlocal/192.168.100.10

# Manually free an IP (use with caution!)
rm /var/lib/cni/networks/macvlan-hostlocal/192.168.100.10
```

#### whereabouts Debugging

```bash
# Check IP pools
kubectl get ippools.whereabouts.cni.cncf.io -A

# Detailed allocation info
kubectl describe ippool <pool-name> -n kube-system

# Check for specific IP
kubectl get ippool <pool-name> -n kube-system -o yaml | grep "192.168.100.10"

# View reconciler logs
kubectl logs -n kube-system -l app=whereabouts-reconciler

# Force reconciliation (delete and recreate reconciler pod)
kubectl delete pod -n kube-system -l app=whereabouts-reconciler
```

---

### Summary

**host-local:**
- Simple, fast, file-based
- Per-node scope
- Requires manual range management
- Good for simple/single-node setups

**whereabouts:**
- Cluster-aware, API-based
- Cluster-wide scope
- Automatic coordination and cleanup
- Good for production/multi-node/multi-cluster setups

For **Nephio and telco deployments**, `whereabouts` is generally the better choice due to its cluster-wide coordination and multi-cluster support via shared etcd.

## The Problem: host-local is Node-Scoped

### What Happens When Pod Reschedules to Different Node

```
Initial State:
┌─────────────────────────────────────┐
│           Node 1                    │
│                                     │
│  Pod: upf-0                         │
│  IP: 192.168.100.50                 │
│                                     │
│  /var/lib/cni/networks/macvlan/     │
│  └── 192.168.100.50                 │
│      (contains upf-0 container ID)  │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│           Node 2                    │
│                                     │
│  /var/lib/cni/networks/macvlan/     │
│  └── (empty or different IPs)       │
└─────────────────────────────────────┘
```

**Node 1 crashes or pod is rescheduled:**

```
After Rescheduling:
┌─────────────────────────────────────┐
│           Node 1 (crashed)          │
│                                     │
│  /var/lib/cni/networks/macvlan/     │
│  └── 192.168.100.50                 │
│      ⚠️ STALE! Pod no longer here   │
│      ⚠️ IP locked but unused        │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│           Node 2                    │
│                                     │
│  Pod: upf-0 (rescheduled here)      │
│  IP: ???                            │
│                                     │
│  /var/lib/cni/networks/macvlan/     │
│  └── (Node 2 has NO knowledge       │
│       of 192.168.100.50!)           │
│                                     │
│  host-local allocates NEW IP:       │
│  IP: 192.168.100.51  ❌ Different!  │
│                                     │
│  /var/lib/cni/networks/macvlan/     │
│  └── 192.168.100.51                 │
└─────────────────────────────────────┘
```

---

### Detailed Sequence of Events

#### Scenario 1: Node Failure

**Step 1: Pod running on Node 1**
```bash
# On Node 1
$ ls /var/lib/cni/networks/macvlan-net/
192.168.100.50  # upf-0 container ID inside

# Pod has interface
$ kubectl exec upf-0 -- ip addr show net1
3: net1: inet 192.168.100.50/24
```

**Step 2: Node 1 becomes unavailable**
```
Node 1 status: NotReady
Kubernetes waits for grace period (default: 5 minutes)
```

**Step 3: Kubernetes reschedules pod to Node 2**
```bash
# Kubernetes scheduler picks Node 2
# Node 2 has NO knowledge of Node 1's allocations

# On Node 2
$ ls /var/lib/cni/networks/macvlan-net/
192.168.100.10  # Some other pod
192.168.100.11  # Another pod
# NO record of 192.168.100.50!
```

**Step 4: host-local IPAM on Node 2 allocates IP**
```bash
# host-local reads its LOCAL files only
# Finds next available IP from ITS perspective

# On Node 2
$ ls /var/lib/cni/networks/macvlan-net/
192.168.100.10
192.168.100.11
192.168.100.12  # NEW - allocated to upf-0
```

**Step 5: Pod gets DIFFERENT IP**
```bash
$ kubectl exec upf-0 -- ip addr show net1
3: net1: inet 192.168.100.12/24  ❌ Changed from .50 to .12!
```

**Step 6: Original IP is LOST**
```bash
# On Node 1 (when it recovers)
$ ls /var/lib/cni/networks/macvlan-net/
192.168.100.50  # STALE! Container no longer exists
                # But host-local thinks it's still allocated
```

---

### Scenario 2: Pod Deletion and Recreation

**Step 1: Delete pod on Node 1**
```bash
$ kubectl delete pod upf-0

# host-local receives CNI DEL command
# Removes allocation file on Node 1
$ ls /var/lib/cni/networks/macvlan-net/
# 192.168.100.50 file deleted ✅
```

**Step 2: Pod recreated, scheduled to Node 2**
```bash
# Kubernetes scheduler decides: Node 2

# host-local on Node 2 has NO IDEA about previous allocation
# It allocates from ITS local pool

$ ls /var/lib/cni/networks/macvlan-net/
192.168.100.15  # NEW allocation for upf-0
```

**Result: Different IP again!**

---

### Scenario 3: StatefulSet with Node Affinity

Even with StatefulSet, if the pod moves nodes:

**Initial deployment:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: upf
spec:
  replicas: 1
  template:
    metadata:
      annotations:
        k8s.v1.cni.cncf.io/networks: macvlan-net
    spec:
      # No node affinity - can schedule anywhere
      containers:
      - name: upf
        image: upf:v1
```

```bash
# upf-0 scheduled to Node 1
$ kubectl get pod upf-0 -o wide
NAME    NODE    IP
upf-0   node-1  10.244.1.5  # Kubernetes IP (visible)

# MacVLAN IP (not visible in kubectl)
$ kubectl exec upf-0 -- ip addr show net1
net1: inet 192.168.100.50/24
```

**Node 1 goes down:**
```bash
# upf-0 rescheduled to Node 2 (no node affinity preventing it)

$ kubectl get pod upf-0 -o wide
NAME    NODE    IP
upf-0   node-2  10.244.2.5  # Different K8s IP

$ kubectl exec upf-0 -- ip addr show net1
net1: inet 192.168.100.22/24  # Different MacVLAN IP! ❌
```

---

### Why host-local Doesn't Sync

#### Architecture Limitation

```
┌────────────────────────────────────────────────────────┐
│              Kubernetes Cluster                        │
│                                                        │
│  ┌──────────────────┐      ┌──────────────────┐      │
│  │     Node 1       │      │     Node 2       │      │
│  │                  │      │                  │      │
│  │  host-local      │  ❌  │  host-local      │      │
│  │  IPAM plugin     │  NO  │  IPAM plugin     │      │
│  │                  │ COMM │                  │      │
│  │  Local storage:  │      │  Local storage:  │      │
│  │  /var/lib/cni/   │      │  /var/lib/cni/   │      │
│  │  └─ 192.168.     │      │  └─ 192.168.     │      │
│  │     100.50       │      │     100.15       │      │
│  └──────────────────┘      └──────────────────┘      │
│                                                        │
│  No shared state, no communication!                   │
└────────────────────────────────────────────────────────┘
```

**host-local design principles:**
1. **Stateless**: No daemon, just a binary called by CNI
2. **Local only**: Only reads/writes local filesystem
3. **No networking**: Never makes network calls
4. **No coordination**: Each node is independent

This is by design for **simplicity and speed**, but it means **no cross-node awareness**.

---

### Workarounds for host-local

#### Option 1: Pin Pods to Specific Nodes (Node Affinity)

**Force pod to always run on the same node:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: upf-0
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-net
spec:
  nodeSelector:
    kubernetes.io/hostname: node-1  # Always on node-1
  containers:
  - name: upf
    image: upf:v1
```

**With StatefulSet:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: upf
spec:
  replicas: 1
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - node-1  # upf-0 always on node-1
```

**Problem:** Defeats the purpose of Kubernetes scheduling flexibility! Node fails = pod can't run.

#### Option 2: Use Static IP Annotations

**Combine host-local with static IP requests:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: upf-0
  annotations:
    k8s.v1.cni.cncf.io/networks: |
      [{
        "name": "macvlan-net",
        "ips": ["192.168.100.50/24"]  # Request specific IP
      }]
spec:
  containers:
  - name: upf
    image: upf:v1
```

**Change IPAM to `static` type:**
```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-net
spec:
  config: |
    {
      "type": "macvlan",
      "master": "eth1",
      "ipam": {
        "type": "static"  # Not host-local!
      }
    }
```

**How it works:**
- Pod carries its own IP in the annotation
- Doesn't matter which node it runs on
- `static` IPAM just uses the IP from annotation

**Problem:** 
- You must manually manage IP assignments
- Still need to partition ranges to avoid conflicts across nodes
- Loses dynamic allocation benefits

#### Option 3: Manual IP Range Partitioning

**Assign different ranges to each node:**

```yaml
# Node 1: NetworkAttachmentDefinition
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-node1
spec:
  config: |
    {
      "type": "macvlan",
      "ipam": {
        "type": "host-local",
        "ranges": [[{
          "subnet": "192.168.100.0/24",
          "rangeStart": "192.168.100.10",
          "rangeEnd": "192.168.100.50"  # Node 1 range
        }]]
      }
    }
```

```yaml
# Node 2: NetworkAttachmentDefinition
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-node2
spec:
  config: |
    {
      "type": "macvlan",
      "ipam": {
        "type": "host-local",
        "ranges": [[{
          "subnet": "192.168.100.0/24",
          "rangeStart": "192.168.100.51",
          "rangeEnd": "192.168.100.100"  # Node 2 range
        }]]
      }
    }
```

**Use node affinity to match:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: upf-0
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-node1  # Use node1's NAD
spec:
  nodeSelector:
    kubernetes.io/hostname: node-1  # Schedule to node-1
  containers:
  - name: upf
    image: upf:v1
```

**Problem:** 
- Complex management
- Tight coupling between pods and nodes
- Defeats Kubernetes scheduling flexibility
- Still loses IP if pod moves nodes

---

### The Real Solution: Use whereabouts!

This is exactly why `whereabouts` was created:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-whereabouts
spec:
  config: |
    {
      "type": "macvlan",
      "master": "eth1",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.100.0/24",
        "range_start": "192.168.100.10",
        "range_end": "192.168.100.200"
      }
    }
```

**With whereabouts:**

```
Pod upf-0 on Node 1:
  IP: 192.168.100.50
  Stored in Kubernetes API (cluster-wide)

Node 1 fails:
  Pod rescheduled to Node 2
  
whereabouts on Node 2:
  - Queries Kubernetes API
  - Sees 192.168.100.50 was allocated to upf-0
  - Checks if that allocation is still valid
  - Container ID matches? Reuse same IP ✅
  - Or, use IP reservation for guaranteed persistence
  
Pod upf-0 on Node 2:
  IP: 192.168.100.50  ✅ Same IP!
```

---

### Comparison Table: IP Persistence Across Node Migration

| Scenario | host-local | whereabouts |
|----------|-----------|-------------|
| **Pod deleted & recreated on SAME node** | ✅ Same IP (from local file) | ✅ Same IP (from API) |
| **Pod rescheduled to DIFFERENT node** | ❌ Different IP | ✅ Same IP (with reservation) |
| **Node failure, pod migrates** | ❌ Different IP | ✅ Same IP (with reservation) |
| **StatefulSet pod recreation** | ⚠️ Same IP only if on same node | ✅ Same IP regardless of node |
| **Manual pod migration** | ❌ Different IP | ✅ Same IP (with reservation) |
| **Cluster autoscaling (node drain)** | ❌ Different IP | ✅ Same IP (with reservation) |

---

### Example: Complete Flow Comparison

#### With host-local (IP LOST)

```bash
# Day 1: Deploy upf-0 on node-1
$ kubectl apply -f upf-pod.yaml
$ kubectl get pod upf-0 -o wide
NAME    NODE     IP
upf-0   node-1   10.244.1.5

$ kubectl exec upf-0 -- ip addr show net1
net1: inet 192.168.100.50/24  # MacVLAN IP

# Day 2: node-1 fails
$ kubectl get nodes
NAME     STATUS
node-1   NotReady  ❌
node-2   Ready

# Kubernetes reschedules upf-0 to node-2
$ kubectl get pod upf-0 -o wide
NAME    NODE     IP
upf-0   node-2   10.244.2.5

$ kubectl exec upf-0 -- ip addr show net1
net1: inet 192.168.100.73/24  # DIFFERENT IP! ❌

# External devices (gNB) trying to reach 192.168.100.50
# Connection fails! ❌
```

#### With whereabouts + IP Reservation (IP PRESERVED)

```bash
# Day 0: Create IP reservation
$ kubectl apply -f - <<EOF
apiVersion: whereabouts.cni.cncf.io/v1alpha1
kind: OverlappingRangeIPReservation
metadata:
  name: upf-0-n3-ip
spec:
  containerid: "upf-0"
  ip: "192.168.100.50"
  podref: "ran/upf-0"
EOF

# Day 1: Deploy upf-0 on node-1
$ kubectl apply -f upf-pod.yaml
$ kubectl get pod upf-0 -o wide
NAME    NODE     IP
upf-0   node-1   10.244.1.5

$ kubectl exec upf-0 -- ip addr show net1
net1: inet 192.168.100.50/24  # From reservation ✅

# Day 2: node-1 fails
$ kubectl get nodes
NAME     STATUS
node-1   NotReady  ❌
node-2   Ready

# Kubernetes reschedules upf-0 to node-2
$ kubectl get pod upf-0 -o wide
NAME    NODE     IP
upf-0   node-2   10.244.2.5  # K8s IP changes

$ kubectl exec upf-0 -- ip addr show net1
net1: inet 192.168.100.50/24  # SAME IP! ✅

# External devices (gNB) can still reach 192.168.100.50
# Connection works! ✅
```

---

### Key Takeaway

**With host-local, when a pod reschedules to a different node:**

❌ **The new node has NO KNOWLEDGE of the previous IP allocation**
❌ **A new IP will be allocated from the new node's local pool**
❌ **The old IP remains locked on the old node (until cleaned up)**
❌ **IP persistence is LOST across node migration**

This is a **fundamental limitation** of `host-local`'s design - it's purely node-local with no cross-node coordination.

**For telco/5G workloads where IP stability is critical**, you must use:
1. `whereabouts` with IP reservations, OR
2. `static` IPAM with explicit IP annotations, OR
3. External IPAM (Infoblox, NetBox), OR
4. Nephio's declarative IPAM (which generates static annotations automatically)

**Never rely on host-local for IP persistence in multi-node production environments!**
--
## Glossary

**AMF (Access and Mobility Management Function):** 5G core network function handling connection and mobility management

**ARP (Address Resolution Protocol):** Protocol for mapping IP addresses to MAC addresses

**CNI (Container Network Interface):** Plugin interface for configuring container networking

**ConfigSync:** Tool for syncing Git repositories to Kubernetes clusters

**gNB (Next Generation NodeB):** 5G radio base station

**IPAM (IP Address Management):** System for planning, tracking, and managing IP address space

**IPClaim:** Nephio CRD for declaring IP address requirements

**IPAllocation:** Nephio CRD representing allocated IP addresses

**MacVLAN:** Network driver that assigns unique MAC addresses to containers

**Multus:** CNI plugin enabling multiple network interfaces per pod

**N2 Interface:** 5G interface between RAN and AMF (control plane)

**N3 Interface:** 5G interface between RAN and UPF (user plane)

**NetworkAttachmentDefinition:** Multus CRD defining network attachment configurations

**Porch:** Nephio package orchestration component

**StatefulSet:** Kubernetes workload for stateful applications with stable identities

**UPF (User Plane Function):** 5G core network function handling user data traffic

**Whereabouts:** IPAM CNI plugin with support for shared storage backends

