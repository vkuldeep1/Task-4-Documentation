# Amazon EKS Hands‑On Deployment & Debugging (Free Tier)

> **Context**
> This document records a complete, real‑world Amazon EKS setup on an AWS Free Tier account.
> It is written so that *future‑me or another engineer* can recognize the situation instantly, understand why things failed or worked, and reproduce or debug the same class of issues without trial‑and‑error.

---

## High‑Level Goal

* Create an EKS cluster using `eksctl`
* Attach a managed node group
* Verify cluster health
* Deploy a sample workload (nginx)
* Debug real failures encountered during setup
* Understand **why** each issue occurred

This is **not** a happy‑path tutorial. It documents **failures and fixes**, which is what actually matters in production.

---

# Phase 0 — Prerequisites & Assumptions

### Local Environment

* OS: Linux (WSL / Ubuntu)
* Tools installed:

  * `aws` CLI
  * `kubectl`
  * `eksctl`

Verification commands:

```bash
aws --version
kubectl version --client
eksctl version
```

### AWS Account Constraints

* AWS **Free Tier** account
* EC2 restrictions apply:

  * Only Free Tier–eligible instance types allowed
  * Typical options: `t2.micro`, `t3.micro`

This constraint **directly affects EKS node groups**.

---

# Phase 1 — Initial Cluster Creation Attempt

## Command Used

```bash
eksctl create cluster \
  --name eks-lab \
  --region ap-south-1 \
  --nodegroup-name ng-1 \
  --node-type t3.medium \
  --nodes 2
```

## Result

* Control plane creation **started**
* CloudFormation stack for node group **failed**

Key failure:

```text
CREATE_FAILED
AsgInstanceLaunchFailures
The specified instance type is not eligible for Free Tier
```

## Root Cause

* `t3.medium` is **not Free Tier eligible**
* AWS Auto Scaling Group could not launch EC2 instances
* Managed Node Group entered `CREATE_FAILED`

### Important Insight

> EKS failures often look like Kubernetes issues but are **pure EC2 / billing constraints**.

---

# Phase 2 — Second Attempt (Instance Type Adjustment)

### Change Made

Switched to `t3.large` to avoid capacity issues.

### Result

* Same failure
* Free Tier restriction still applies

### Lesson

Instance size must satisfy **both**:

* Capacity availability
* Account billing eligibility

---

# Phase 3 — Correct Cluster Creation (Free Tier Safe)

## Final Working Command

```bash
eksctl create cluster \
  --name eks-lab \
  --region ap-south-1 \
  --zones ap-south-1a,ap-south-1b \
  --nodegroup-name ng-1 \
  --node-type t3.micro \
  --nodes 2 \
  --with-oidc
```

### Why These Flags Matter

* `t3.micro` → Free Tier eligible
* `--with-oidc` → Enables IAM OIDC provider (required for modern EKS add‑ons)
* Limited AZs → Reduces networking and capacity complexity

## Result

```text
EKS cluster "eks-lab" is ready
nodegroup "ng-1" has 2 node(s)
node is ready
```

Verification:

```bash
kubectl get nodes
```

Output:

```text
NAME                                            STATUS   VERSION
ip-192-168-14-188.ap-south-1.compute.internal   Ready    v1.32.x
ip-192-168-63-246.ap-south-1.compute.internal   Ready    v1.32.x
```

---

# Phase 4 — Cluster Health Validation

## System Pods Check

```bash
kubectl get pods -n kube-system
```

Observed running components:

* `aws-node` (VPC CNI)
* `kube-proxy`
* `coredns`
* `metrics-server`

All pods in `Running` state.

### Meaning

* Networking works
* DNS works
* IAM permissions for CNI are sufficient
* Cluster is operational

---

# Phase 5 — Application Deployment (nginx)

## Deployment

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=ClusterIP --port=80
```

## Observed Behavior

```bash
kubectl get pods
```

```text
nginx-5869d7778c-nzrx2   0/1   Pending
```

The pod **never scheduled**.

---

# Phase 6 — Debugging the Pending Pod

## Describe Pod

```bash
kubectl describe pod nginx-5869d7778c-nzrx2
```

### Events Section

```text
Warning  FailedScheduling  default-scheduler  
0/2 nodes are available: 2 Too many pods.
preemption: 0/2 nodes are available: No preemption victims found
```

---

# Phase 7 — Root Cause Analysis

### What "Too many pods" Means

This is **not** about replicas or limits.

It means:

* Each EC2 instance has a **maximum pod count**
* This limit is enforced by the AWS VPC CNI
* `t3.micro` supports **very few pods per node**

### Current Resource Usage

Each node already runs:

* aws-node
* kube-proxy
* coredns
* metrics-server

System pods consume:

* CPU
* Memory
* Pod slots

No room left for nginx.

---

# Phase 8 — Correct Interpretation (Key Learning)

This is **expected behavior** on Free Tier EKS.

Important distinctions:

* ❌ Not a Kubernetes bug
* ❌ Not an EKS bug
* ❌ Not a networking bug
* ✅ Correct scheduler behavior

> **Nodes being Ready does NOT mean workloads are schedulable.**

---

# Core Concepts Explained

## Amazon EKS

* Managed Kubernetes control plane
* AWS manages:

  * API server
  * etcd
  * scheduler

You manage:

* Worker nodes
* Networking
* IAM integration

## Node Group

* Collection of EC2 instances
* Backed by Auto Scaling Group
* Resource limits are **hard limits**

## Pending Pod

A pod in `Pending` means:

* Scheduler cannot find a node that satisfies requirements
* It is a signal, not an error

## VPC CNI

* Assigns real VPC IPs to pods
* Limits pod density per node
* Tightly coupled with instance type

---

# Final State Summary

✔ EKS cluster created successfully
✔ Managed node group attached
✔ System components healthy
✔ Scheduling failure correctly diagnosed

This setup is **sufficient to understand EKS fundamentals**.

---

# Cleanup Command (Important)

```bash
eksctl delete cluster --name eks-lab --region ap-south-1
```

Always clean up to avoid orphaned AWS resources.

---

# Closing Note

If you encounter a similar situation in the future:

1. Check **instance type**
2. Check **Free Tier restrictions**
3. Check **pod density limits**
4. Read pod events first, not logs


