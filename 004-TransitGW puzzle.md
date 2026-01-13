### Transit Gateway Challenge: Taming the Multi-VPC Jungle

**The Scenario**
You are the lead architect at MegaCorp, a fast-growing fintech with:

- **4 VPCs** across **2 accounts** (DevOps owns central infra, AppTeams own workloads).
- VPC-A (Dev): 10.1.0.0/16 – App servers.
- VPC-B (Test): 10.2.0.0/16 – Test DBs.
- VPC-C (Prod): 10.3.0.0/16 – Live services.
- VPC-D (Shared): 10.4.0.0/16 – Centrally managed logging/S3 gateway (in DevOps account).
- **On-premises DC**: 172.16.0.0/16 via Direct Connect.

**Requirements:**

- All VPCs ↔ on-premises (transitive).
- Prod/Test → Shared VPC (no full mesh).
- Internet egress via Shared VPC only.
- Cross-account secure.

**Current State:** VPC peering = 6+ manual links, no transitivity, route table explosion. **Challenge:** Architect a scalable fix using Transit Gateway. Sketch it, explain routing logic, justify vs. peering (15-20 mins thinking time).

***

### Thought Process: How to Architect This Step-by-Step

1. **Reject Peering Fundamentals:** Calculate mesh (4 VPCs = 6 peerings). No transitivity (VPC-A ↔ B doesn't reach on-prem). Cross-account requires per-pair accepts. Route tables duplicate across VPCs. **Ask:** What if 50 VPCs? Quadratic disaster.
2. **Embrace Hub-and-Spoke:** Central Transit Gateway (TGW) as "network router." Attach all VPCs + on-prem + egress. Scales linearly (attachments ~\$0.05/hr each).
3. **Map Components:**
    - **TGW Creation:** Regional (e.g., us-east-1), auto-generates route tables.
    - **Attachments:** VPC (domain), Direct Connect Gateway.
    - **Route Tables:** Segment by policy (e.g., "Prod-RT" isolates Prod).
4. **Routing Logic:**
    - **Association:** Bind attachment to RT (e.g., VPC-C → Prod-RT).
    - **Propagation:** Auto-advertise CIDRs (e.g., on-prem propagates 172.16.0.0/16 to Prod-RT).
    - VPC route tables: Add TGW ID as target (e.g., 172.16.0.0/16 → tgw-123).
5. **Cross-Account:** Share TGW via RAM → AppTeams attach locally.
6. **Security/Scale:** Policies control propagation. Monitor via VPC Flow Logs.

**Time Check:** Spend 5 mins diagramming, 5 mins routing, 5 mins trade-offs, 5 mins cross-account.

***

