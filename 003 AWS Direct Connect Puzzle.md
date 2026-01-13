# AWS Direct Connect Puzzle: Hybrid Latency Mystery

## **Puzzle Overview**

Your organization has deployed AWS Direct Connect with a 1 Gbps Link Aggregation Group (LAG) and a public Virtual Interface (VIF). Customers report **high latency** when accessing public AWS services (S3, DynamoDB) through the Direct Connect path versus the public internet. Your task is to identify the **conceptual misconfigurations** in BGP peering and encryption options causing this performance degradation.

***

## **Puzzle Scenario**

**Current Hybrid Setup:**

```
On-Premises Data Center
    ↓ 1 Gbps Direct Connect LAG (2x 500 Mbps links)
AWS Direct Connect Location
    ↓ Public VIF (advertises public AWS prefixes)
AWS Public Services (S3, DynamoDB, CloudFront POPs)
```

**Configuration Details:**

- **Direct Connect Location:** Equinix NY4 (co-located with major carriers)
- **LAG:** 2 physical connections (500 Mbps each), bonded as 1 Gbps LAG
- **Public VIF:** VLAN 200, BGP peering established with AWS, MTU 1500
- **BGP Configuration:** Single-session BGP with AWS public ASN (16509)
- **Encryption:** MACsec enabled on both member links
- **On-premises routing:** Advertises customer public prefixes (203.0.113.0/24)

**Performance Metrics:**

```
Internet Path (Baseline):
S3 PUT: 850 Mbps, 15ms latency
DynamoDB Writes: 1200 ops/sec, 12ms latency

Direct Connect Path (Problem):
S3 PUT: 420 Mbps (50% slower), 85ms latency (5x slower)
DynamoDB Writes: 650 ops/sec (46% slower), 78ms latency (6x slower)
```

**Key Observations:**

- **Physical layer** tests show 1 Gbps line rate with <1% packet loss
- **Traceroute** reveals packets taking Direct Connect path as expected
- **AWS Health Dashboard** shows Direct Connect link healthy
- **BGP session** stable, all expected prefixes advertised/received
- On-premises network engineers insist "the pipe is clean"

**The Puzzle:** Why is Direct Connect performing **5-6x worse** than plain internet for public AWS services?

***

## **Guiding Hints for Students**

### **Hint 1: Public VIF Purpose vs Reality**

Public VIFs exist to exchange **public IP prefixes** between your network and AWS. Ask yourself:

- What prefixes does AWS advertise over Public VIF?
- What traffic is supposed to flow through this path?
- Are S3/DynamoDB endpoints using the prefixes advertised on Public VIF?


### **Hint 2: BGP Path Selection Drama**

BGP doesn't magically choose the "fastest" path. It follows **deterministic rules**. Consider:

- How does your router decide Direct Connect > Internet for S3 traffic?
- What BGP attributes influence this decision?
- Could suboptimal attributes cause traffic to prefer a congested path?


### **Hint 3: Encryption Tax**

MACsec provides Layer 2 encryption, but encryption = computation. Ask:

- What's the CPU overhead of encrypting/decrypting 1 Gbps of traffic?
- Does your router's forwarding ASIC support hardware MACsec?
- How does encryption impact latency vs throughput?


### **Hint 4: MTU Mismatch Mayhem**

Standard Ethernet MTU = 1500 bytes. But AWS services love **jumbo frames**. Consider:

- What's the effective throughput at 1500 MTU vs 9000 MTU?
- How does Direct Connect handle Path MTU Discovery?
- Could fragmentation explain the 50% throughput drop?


### **Hint 5: LAG Load Balancing Illusion**

LAG (EtherChannel/LACP) distributes traffic across member links using a **hash algorithm**. Think:

- What 5-tuple does LAG hash use? (SIP, DIP, SP, DP, protocol)
- Single TCP flows = single member link (500 Mbps max)
- Multiple connections = better distribution
- S3/DynamoDB use connection pooling. Is it enough?


### **Hint 6: The Real Question**

You're not just fixing latency. You're learning **when Direct Connect makes sense** for public services vs private resources.

***


