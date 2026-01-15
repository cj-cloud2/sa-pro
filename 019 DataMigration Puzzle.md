**Puzzle 2: Migration Strategy Puzzle**

**Scenario:** TechNova Inc. is migrating to AWS. They have diverse workloads: 200 on-prem VMs (Windows/Linux), 5PB file shares (NFS/SMB), Oracle/MySQL databases, and a discovery gap—no inventory. Constraints: Low bandwidth (10Mbps to cloud), compliance requires agentless options, and cutover must be <4hrs. Available tools: AWS Application Migration Service (MGN), Migration Hub, AWS DataSync.

**Workload Grid to Solve:**


| Workload | Primary Tool | Key Feature for Constraint | Why This Over Others? |
| :-- | :-- | :-- | :-- |
| VM lift-and-shift | ? | *Hint: Agentless replication with test cutovers for quick validation.* | ? |
| File shares (5PB) | ? | *Hint: Bandwidth-optimized transfer with scheduling/compression.* | ? |
| Database migration | ? | *Hint: Centralized tracking/strategy orchestration across tools.* | ? |
| **Overall:** Low-bw discovery + cutover plan | ? | *Hint: Hub for inventory, progress, and multi-tool coordination.* | ? |

**Your Task (10-15 mins):** Fill the grid. For each, pick **one primary tool**, note its top feature for the constraint, and justify exclusions. Bonus: Sequence steps for a hybrid cutover. Discuss: How does low bandwidth change choices?

**Submit:** Completed grid + 1-sentence rationale per row.

***

### Solution Key with Detailed Justifications

#### Workload 1: VM Lift-and-Shift → **AWS Application Migration Service (MGN)**

**Teaching from Scratch:** MGN (formerly Server Migration Service) enables *agentless* lift-and-shift of VMs from VMware/vSphere, Hyper-V, or physical servers to EC2/AMIs. It deploys a replication appliance on-prem (or AWS), continuously syncs block-level changes (optimized for low bandwidth via compression/deduplication), and supports non-disruptive test cutovers (launch replicas, validate, then final cutover in minutes). Post-migration: Convert to AMIs for resilience. No app changes needed—ideal for quick wins in 7 Rs (Rehost).

**Why Best Here:** For TechNova's 200 VMs, agentless replication handles low 10Mbps bandwidth; test cutovers ensure <4hr final switch.
**Action Plan:** Deploy MGN appliance → Discover VMs → Continuous replication → Test/launch AMI.
**Why Not Others?** DataSync is files-only (not block/VM); Migration Hub orchestrates but doesn't replicate.

#### Workload 2: File Shares (5PB) → **AWS DataSync**

**Teaching from Scratch:** DataSync is a managed data transfer service for *file/NFS/SMB/Object* from on-prem/cloud to S3/FSx/EFS/EFS. It uses agents (VMs) for discovery/transfer, with *bandwidth optimization* (compression, scheduling, multi-threaded), incremental syncs, and verification. Handles PB-scale via parallel tasks; supports VPC endpoints for security. Unlike manual rsync/SCP, it's serverless post-agent, with 99.999% durability.

**Why Best Here:** Compresses/transfers 5PB files over 10Mbps without downtime; schedules off-peak to fit constraints.
**Action Plan:** Deploy DataSync agent on-prem → Create task to S3/FSx → Run incremental syncs.
**Why Not Others?** MGN is VM/block-level (not files); Migration Hub tracks but doesn't transfer data.

#### Workload 3: Database Migration → **Migration Hub**

**Teaching from Scratch:** Migration Hub is a *central orchestration console* (free) for tracking migrations across tools/services (MGN, DMS, DataSync, etc.). It auto-discovers via agent/ agentless (CloudEndure/ADM connectors), provides wave/portfolios, progress dashboards, and custom KPIs. Doesn't migrate itself but unifies strategy—e.g., "80% VMs done, DBs pending." Integrates with Application Discovery Service for inventory.

**Why Best Here:** Ties DB schema/data tools (e.g., DMS) into low-bw plan; tracks compliance cutovers centrally. (Note: Pair with DMS for actual DB work.)
**Action Plan:** Connect sources → Create portfolio → Track DB waves with MGN/DataSync.
**Why Not Others?** MGN/DataSync handle VMs/files, not DB schema/logic; Hub provides the missing discovery/orchestration.

#### Overall: Low-BW Discovery + Cutover → **Migration Hub**

**Teaching from Scratch:** (Extends above) In low-connectivity scenarios, Hub's agentless discovery (via APIs/scans) builds inventory without heavy data pulls. Sequence: Discover → Replicate (MGN/DataSync) → Test in Hub → Cutover waves. Ensures <4hr switches by staging replicas.

**Why Best Here:** Coordinates all tools for 10Mbps limits; dashboard flags risks. Full Sequence: 1. Hub discovery, 2. MGN VM sync, 3. DataSync files, 4. Hub-track DB (DMS), 5. Test cutover.
**Why Not Others?** Others are workload-specific; Hub glues them for strategy.
