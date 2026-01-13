### The Solution: Diagram + Deep Explanation

```
On-Prem (172.16.0.0/16)
       |
       | Direct Connect Gateway Attachment
       v
   +-------------------+
   | Transit Gateway   |  <-- Central Hub (us-east-1)
   |                   |
   | Route Tables:     |
   | - Prod-RT         |  
   | - Shared-RT       |  
   +-------------------+  
      ^    ^       ^  
      |    |       |  
 VPC-C | VPC-A,B  | Internet Egress
(Prod) | (Dev/Test)| (via Shared VPC-D)
10.3   | 10.1/2   | 10.4  
       |  
   RAM Share <--- DevOps Account owns TGW
```

**Detailed Breakdown:**

1. **Why TGW Wins (vs. Peering):**
Peering: N×(N-1)/2 links (6 here, 1,225 for 50 VPCs). No hub routing. TGW: 5 attachments, transitive via central tables. Throughput: 100Gbps+ burst.
2. **Step-by-Step Build:**
    - Create TGW in DevOps account.
    - **Attach VPC-D (Shared):** Associate/propagate to Shared-RT (for egress).
    - **Attach VPC-A/B/C:** Prod VPC-C to Prod-RT (restricted propagation).
    - **On-Prem Attach:** Propagate to all RTs.
    - **VPC Route Tables:** `10.4.0.0/16 → tgw-attach-X` (egress), `172.16.0.0/16 → tgw-attach-Y`.
3. **Cross-Account Magic:** RAM share TGW → AppTeams accept → Attach VPC-A/B/C locally. One share rules all.
4. **Verification Flow (Prod EC2 → On-Prem):**
EC2 (10.3.x.x) → VPC RT (172.16 → TGW) → Prod-RT lookup → On-prem propagation → Success.

**Key Concepts Mastered:** TGW transforms VPCs into a scalable fabric—essential for enterprise multi-account designs. Total time: ~18 mins.

