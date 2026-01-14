### EKS Puzzle: "Cluster Chaos – Diagnose the Outage"

**Puzzle Scenario**
You're the lead architect for ShopFast, an e-commerce platform running on EKS. During Black Friday, the cluster experiences cascading failures: pods evict unexpectedly, scaling lags, and 40% of traffic fails despite 80% CPU utilization across nodes.

**Provided Clues:**

```
EKS Cluster: 3 AZs (us-west-2a/b/c), 2 node groups (Spot + On-Demand).
Workloads: frontend (Deployment, 10 replicas), backend (Deployment, 5 replicas), db-cache (StatefulSet).
Metrics snippet:
- HPA on frontend: target CPU 60%, current avg 75% → no scaling.
- Node utilization: 85% CPU, but pods pending.
- Logs: "Pod disrupted due to PDB violation during node drain."
Cluster addon: Cluster Autoscaler enabled, but Spot nodes terminate frequently.
```

**Your Task (10-15 mins):** Answer these 3 questions conceptually. Use the clues above as guiding hints.

1. **Why isn't HPA scaling frontend pods despite high CPU?**
2. **What's causing the single point of failure in this multi-AZ setup?**
3. **How do Pod Disruption Budgets (PDBs) contribute to the cascading failures, and what's the conceptual fix?**

**Guiding Hints (Embedded):**

- Think: HPA relies on metrics server—custom vs. built-in?
- AZ clue: Control plane is managed, but data plane spreads how?
- PDB hint: Drains happen on Spot termination—what's the minAvailable math?

***

### Detailed Solutions with Teaching Justifications

#### 1. HPA Not Scaling Despite High CPU

**Answer:** HPA fails because it's likely using built-in CPU metrics aggregated at the pod level, but Cluster Autoscaler isn't provisioning nodes fast enough for pending pods. The "high CPU but pods pending" indicates a capacity bottleneck, not a metrics issue—HPA scales pods only if nodes have room.

**Teaching Justification from Basics:**
Horizontal Pod Autoscaler (HPA) is Kubernetes' auto-scaling mechanism for pods based on observed metrics versus targets. Start with the fundamentals: HPA queries the Metrics Server (deployed in EKS by default) every 15 seconds. For CPU, it uses **resource metrics** (kubectl top pods), calculated as pod CPU requests/limits usage ratio.

- **Built-in vs. Custom Metrics:** Built-in metrics (CPU/memory) are simple but pod-granular. If your Deployment has no `resources.requests.cpu` defined, HPA can't compute utilization—defaults to 0. Here, avg 75% implies requests are set, but scaling stalls due to no available nodes (clue: pods pending).
- **HPA Flow:**

1. Desired replicas = current × (current metric / target metric). E.g., 10 pods at 75% CPU (target 60%) → scale to ~13.
2. But Kubernetes scheduler rejects new pods without node capacity.
- **EKS-Specific:** Cluster Autoscaler (CA) watches unschedulable pods and scales node groups. Spot nodes terminate (unpredictable), so CA expels pods via drain, but if On-Demand group scales slowly (expansion limits), backlog builds.
- **Conceptual Fix:** Enable **custom metrics** via Prometheus Adapter for workload-specific signals (e.g., queue length). Pair with Karpenter (EKS-optimized CA) for <1min node provisioning. Justification: Reduces scale-out time from 5-10 mins (ASG) to seconds.


#### 2. Single Point of Failure in Multi-AZ Setup

**Answer:** The EKS control plane is highly available across AZs by default, but worker nodes likely have uneven topology spread—e.g., more Spot nodes in one AZ, causing total failure if that AZ's nodes drain simultaneously.

**Teaching Justification from Basics:**
EKS splits responsibilities: **Control plane** (API server, etcd, etc.) is AWS-managed, spanning 3 AZs with 99.95% SLA—no single AZ failure. **Data plane** (worker nodes/pods) is your responsibility.

- **Multi-AZ Fundamentals:** Use `topologySpreadConstraints` in pod spec for even distribution:

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: ScheduleAnyway
```

This ensures no AZ has >1 more pods than others.
- **Node Group Pitfalls:** 2 groups (Spot + On-Demand) might have AZ selectors skewed (clue: 3 AZs but Spot terminations cluster). If 60% nodes in us-west-2a fail (AZ outage + Spot drain), schedulers can't place pods elsewhere fast.
- **EKS HA Concepts:** IRSA for pod identity, but true HA needs Node Affinity/Anti-Affinity. Single PoF here: PDBs + CA not honoring topology during drains.
- **Conceptual Fix:** Deploy with `podAntiAffinity` preferring different nodes/AZs, and use EKS Managed Node Groups with multi-AZ diversity. Justification: Ensures 33% capacity per AZ minimum, surviving one AZ loss (Kubernetes HA principle: N+1 redundancy).


#### 3. Pod Disruption Budgets (PDBs) and Cascading Failures

**Answer:** PDBs are too restrictive (e.g., minAvailable=100%), blocking node drains during Spot terminations, leading to forced evictions and unavailable pods. Cascading happens as frontend pods die, starving backend.

**Teaching Justification from Basics:**
PDBs protect workloads from voluntary disruptions (drains, upgrades). They define "safe to disrupt" quanta.

- **PDB Mechanics:** Two types:


| Type | Formula | Example (5 replicas) |
| :-- | :-- | :-- |
| minAvailable | N × percent or absolute | 80% → 4 healthy |
| maxUnavailable | N × percent or absolute | 20% → 1 disruptible |

During drain (CA/Spot termination), K8s checks PDB before evicting. If violated, pods stay → node stuck → CA timeouts → more terminations.
- **Clue Tie-In:** "PDB violation during node drain" + high CPU = frontend PDB blocks drain, nodes overutilize, pods OOM/evict forcefully (no graceful shutdown).
- **Cascading Effect:** Frontend unavailable → no traffic to backend → idle resources but perceived outage.
- **Conceptual Fix:** Set PDBs to allow churn: `minAvailable: 70%` for stateless, `maxUnavailable: 1` for stateful. Use `kubectl drain --ignore-pdb` sparingly; prefer `Cluster Autoscaler Expander: Priority` for stable nodes. Justification: Balances availability (SLA) with efficiency (Spot cost savings ~70%), embodying "disruptible by design" in cloud-native arch.

This puzzle reinforces EKS conceptual pillars: metrics-driven scaling, topological resilience, and disruption tolerance. Students learn by tracing symptoms to root causes.

