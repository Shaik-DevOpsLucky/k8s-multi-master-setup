# k8s-multi-master-setup

---

# 🚀 What is Multi-Master Kubernetes? (End-to-End Theory)

## 🔹 Simple Definition (One-liner)

**Multi-master Kubernetes** means running **more than one control plane node** so the cluster **never goes down if one master fails**.

---

## 🔹 First: What does a “Master” actually do?

A **Kubernetes master (control plane)** is NOT running your app containers.

It runs **cluster brain components**:

| Component               | Responsibility                              |
| ----------------------- | ------------------------------------------- |
| kube-apiserver          | Entry point for kubectl, nodes, controllers |
| etcd                    | Cluster state database                      |
| kube-scheduler          | Decides *where* pods run                    |
| kube-controller-manager | Keeps desired vs actual state               |

If **master goes down → cluster control is gone**.

---

## ❌ Problem with Single Master

### What happens in single master?

```
Apps running?         ✅ YES
kubectl works?        ❌ NO
New pods scheduling?  ❌ NO
Node joins?           ❌ NO
Autoscaling?          ❌ NO
```

### Business Impact

* Deployments stuck
* Auto-healing stops
* Production changes blocked
* SLA violation

👉 **Unacceptable for production**

---

# ✅ What is Multi-Master (High Availability)?

Multi-master means:

* **2 or 3 control plane nodes**
* **Load balancer in front**
* **Shared etcd cluster**

```
User / kubectl
      |
   Load Balancer (6443)
      |
 ┌────┴────┬────┴────┐
 │ Master1 │ Master2 │ Master3
 │  etcd   │  etcd   │  etcd
 └─────────┴─────────┴─────────┘
```

---

## 🔹 Key Behavior (Very Important)

### kube-apiserver

* All masters run it
* LB sends traffic to any healthy master

### scheduler & controller

* All masters run them
* **Leader election**
* Only one active at a time
* Others standby

### etcd

* All masters run etcd
* Uses **RAFT consensus**
* Needs **majority (quorum)**

---

# 🧠 Why Quorum Matters (etcd)

| Nodes | Quorum | Failure Allowed |
| ----- | ------ | --------------- |
| 1     | ❌      | 0               |
| 3     | 2      | 1               |
| 5     | 3      | 2               |

👉 That’s why orgs use **3 or 5 masters**

---

# 🏆 Benefits of Multi-Master (Clear Cut)

## 1️⃣ High Availability (Biggest Reason)

If one master dies:

* Cluster continues
* API still reachable
* Pods still schedule

✔️ No downtime
✔️ Zero impact on users

---

## 2️⃣ Production-Grade Reliability

* Rolling upgrades
* Maintenance without outage
* Master reboot safe

---

## 3️⃣ No Single Point of Failure

Single master = SPOF
Multi-master = **fault-tolerant**

---

## 4️⃣ Scalability of Control Plane

* Large clusters (1000+ nodes)
* High API traffic
* CI/CD systems constantly deploying

---

## 5️⃣ Required for Compliance & SLA

Most orgs have:

* 99.9% or 99.99% uptime
* Change windows
* Disaster recovery requirements

Single master ❌
Multi-master ✅

---

# 🏢 Are Organizations REALLY Using Multi-Master?

### Short answer:

**YES — EVERY serious Kubernetes user**

### Real Examples

| Organization   | Usage                  |
| -------------- | ---------------------- |
| AWS (EKS)      | Multi-AZ, multi-master |
| Google (GKE)   | Regional control plane |
| Azure (AKS)    | Managed HA masters     |
| Netflix        | Custom HA clusters     |
| Banks          | Mandatory HA           |
| SaaS Companies | Default                |

👉 **Single master is for labs & learning only**

---

# 🧩 Managed Kubernetes vs Self-Managed

| Feature | Managed (EKS/GKE/AKS) | Self-Managed  |
| ------- | --------------------- | ------------- |
| Masters | Hidden                | You manage    |
| HA      | Automatic             | You design    |
| etcd    | Managed               | You manage    |
| LB      | Built-in              | You configure |

Organizations prefer **managed** to avoid master headaches
But **concept is the SAME**

---

# 🎯 Use Cases (Real-World)

## 1️⃣ Production Microservices

* 50–500 services
* Continuous deployments
* No downtime allowed

---

## 2️⃣ Financial / Banking Apps

* Compliance (PCI, SOC2)
* Zero control plane downtime

---

## 3️⃣ E-commerce

* Sales events
* High traffic spikes
* Autoscaling must always work

---

## 4️⃣ Platform Engineering Teams

* Internal developer platforms
* Shared clusters
* Self-service deployments

---

## 5️⃣ Hybrid / On-Prem Kubernetes

* Data centers
* Edge clusters
* Bare metal (no cloud LB)

---

# ⚖️ Single vs Multi Master (Interview Gold)

| Question                      | Answer            |
| ----------------------------- | ----------------- |
| Is single master OK for prod? | ❌ No              |
| Why multi-master?             | HA, no SPOF       |
| How many masters?             | 3 or 5            |
| How is traffic routed?        | Load balancer     |
| What happens if master fails?  | Cluster continues working |
| Who uses it?                  | All real orgs     |

---

# 🧠 When NOT to Use Multi-Master?

* Learning Kubernetes
* POCs
* Personal labs
* Cost-sensitive environments

---

# 🎤 One-Line Interview Answer

> “Multi-master Kubernetes provides high availability for the control plane by running multiple API servers and etcd members behind a load balancer, eliminating single points of failure and making the cluster production-ready.”

🔥 That answer alone puts you ahead of 80% candidates.

---


Going from **single control plane → multi-master + multi-worker** is the *real* Kubernetes architecture step. Let’s do this **cleanly**, **theoretically first**, then **practically step-by-step**, AWS-ready.

---

# 🧠 PART 1: THEORY — Multi-Master + Multi-Worker Kubernetes

## What changes from single master?

| Component      | Single Master | Multi-Master (HA)             |
| -------------- | ------------- | ----------------------------- |
| kube-apiserver | 1             | **3+ (behind Load Balancer)** |
| etcd           | 1 (local)     | **3 or 5 members (quorum)**   |
| scheduler      | 1             | 3 (active-leader)             |
| controller     | 1             | 3 (active-leader)             |
| workers        | any           | any                           |

### Key Concepts

### 1️⃣ Load Balancer (VERY IMPORTANT)

* Workers **never talk directly to a master IP**
* Workers talk to **one stable endpoint**
* Example:

```
k8s-api.mycompany.local → AWS NLB → masters
```

This endpoint is used in:

* `kubeadm init`
* `kubeadm join`

---

### 2️⃣ etcd Quorum Rule

etcd requires **majority to function**:

| Members | Quorum           |
| ------- | ---------------- |
| 1       | ❌ Not HA         |
| 3       | ✅ Recommended    |
| 5       | ✅ Large clusters |

👉 **Always use odd numbers**

---

### 3️⃣ Control Plane Behavior

* kube-scheduler & controller-manager use **leader election**
* Only one is active at a time
* Others are hot standby

---

# 🧱 PART 2: ARCHITECTURE (AWS)

```
                 ┌─────────────────────────┐
                 │   AWS NLB (TCP 6443)     │
                 │ k8s-api.internal.local  │
                 └───────────┬─────────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
┌─────────▼────────┐ ┌─────────▼────────┐ ┌─────────▼────────┐
│ controlplane-1   │ │ controlplane-2   │ │ controlplane-3   │
│ etcd member      │ │ etcd member      │ │ etcd member      │
└──────────────────┘ └──────────────────┘ └──────────────────┘

        ┌──────────────┐   ┌──────────────┐
        │ worker-1     │   │ worker-2     │
        └──────────────┘   └──────────────┘
```

---

# 🛠️ PART 3: PRACTICAL SETUP (STEP-BY-STEP)

## 🔹 Prerequisites (ALL NODES)

✅ Do this on:

* controlplane-1
* controlplane-2
* controlplane-3
* worker-1, worker-2…

✔️ What next is:

* swap disabled
* Install containerd
* Install kubeadm / kubelet / kubectl
* Install sysctl, br_netfilter
* Install Calico networking

## Here is the reference url to perform the above steps:
*https://github.com/Shaik-DevOpsLucky/k8s-kubeadm-multi-node-cluster-setup*

💯 You’re good here.

---

## 🔹 Step 1: Create AWS Network Load Balancer

### AWS Console → EC2 → Load Balancers

* Type: **Network Load Balancer**
* Listener:

  * TCP **6443**
* Target group:

  * Targets = **all control plane private IPs**
  * Port: **6443**

Example DNS:

```
k8s-api.internal.elb.amazonaws.com
```

👉 **This becomes your API endpoint**

---

## 🔹 Step 2: DNS (Optional but Best Practice)

Create Route53 record:

```
k8s-api.mycompany.local → NLB DNS
```

---

## 🔹 Step 3: Initialize FIRST Control Plane

### On controlplane-1 ONLY

```bash
sudo kubeadm init \
  --control-plane-endpoint "k8s-api.internal.elb.amazonaws.com:6443" \
  --upload-certs \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket unix:///var/run/containerd/containerd.sock \
  --node-name controlplane-1
```

✔️ Save **TWO things** from output:

1. `kubeadm join ... --control-plane`
2. `--certificate-key`

---

## 🔹 Step 4: Configure kubectl

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 🔹 Step 5: Install CNI (ONCE ONLY)

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Wait:

```bash
kubectl get pods -n kube-system
```

---

## 🔹 Step 6: Join OTHER Control Planes

### On controlplane-2 & controlplane-3

```bash
sudo kubeadm join k8s-api.internal.elb.amazonaws.com:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key> \
  --cri-socket unix:///var/run/containerd/containerd.sock
```

---

## 🔹 Step 7: Join Worker Nodes

### On worker nodes ONLY

```bash
sudo kubeadm join k8s-api.internal.elb.amazonaws.com:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --cri-socket unix:///var/run/containerd/containerd.sock
```

---

## 🔹 Step 8: Verify Cluster

```bash
kubectl get nodes
```

Expected:

```
controlplane-1   Ready   control-plane
controlplane-2   Ready   control-plane
controlplane-3   Ready   control-plane
worker-1         Ready
worker-2         Ready
```

---

# 🧪 PART 4: VALIDATION (IMPORTANT)

### Kill a control plane node

```bash
sudo systemctl stop kubelet
```

✔️ Cluster should continue working
✔️ `kubectl get nodes` still works
✔️ Pods still schedule

---

# 🔐 SECURITY GROUPS (IMPORTANT)

### Control Plane SG

* 6443 TCP (from workers + masters)
* 2379–2380 TCP (masters only)
* 10250 TCP
* 10257, 10259 TCP

### Worker SG

* 10250 TCP
* 30000–32767 TCP/UDP
* Calico: TCP 179

---

# 🎯 SUMMARY

✔️ Multi-master = **High Availability**
✔️ Load balancer = **single stable endpoint**
✔️ etcd quorum = **odd number**
✔️ kubeadm supports HA **natively**
✔️ Your current setup is already **90% compatible**

---

# Prepared by:
*Shaik Moulali*
