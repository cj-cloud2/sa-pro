# Puzzle: The Ambiguous Path

## The Scenario

Your organization, **GlobalTech**, has a hybrid architecture. You have a primary data center in Virginia and a VPC in the `us-east-1` region. To ensure high availability and performance, the networking team has implemented the following:

1. **Primary Link:** A **10Gbps Direct Connect (DX)** via a Transit Gateway (TGW).
2. **Backup Link:** A **Site-to-Site VPN** also connected to the same Transit Gateway (TGW).

**The Incident:** During a routine audit, the Cloud Architect notices that traffic destined for the **Legacy Payroll Server** (IP: `10.0.1.50`) is experiencing high latency and jitter. Upon inspection, the traffic is bypassing the 10Gbps Direct Connect link and is instead traveling over the encrypted VPN tunnel, even though the Direct Connect link is reported as "Healthy" and has 95% available bandwidth.

---

## The Puzzle Challenge

**Why is the traffic choosing the "slow" path, and how do you fix it without deleting the VPN connection?**

### Guiding Hints

* **Hint 1:** Check the **Route Table** of the Transit Gateway. Are the destination CIDRs exactly the same?
* **Hint 2:** Remember the **"Golden Rule of Routing"** that applies to almost all routers, including AWS virtual routers. It is more powerful than link speed or cost.
* **Hint 3:** Look at how the routes were learned. The Direct Connect uses **BGP** to advertise the whole corporate range (`10.0.0.0/16`). The VPN was recently configured using **Static Routing** for the specific subnet where the Payroll Server lives (`10.0.1.0/24`).
* **Hint 4:** If you make the CIDRs identical, how does AWS decide between a DX path and a VPN path?

---

# Solution & Conceptual Justification

## 1. The Primary Culprit: Longest Prefix Match (LPM)

**The Concept:** In the world of networking, the **Longest Prefix Match** is the absolute highest priority. When a router receives a packet, it looks at the destination IP and searches its route table for the most specific match (the one with the largest subnet mask/prefix).

**Detailed Justification:**

* The Direct Connect was advertising `10.0.0.0/16`.
* The VPN had a static route for `10.0.1.0/24`.
* Because `/24` (256 IPs) is more specific than `/16` (65,536 IPs), the router **must** choose the `/24` path. It doesn't matter that the DX link is 100x faster; the routing logic dictates that the most specific path is the "correct" destination. This is why the Payroll Server traffic (`10.0.1.50`) was "sucked" into the VPN tunnel.

## 2. The Tie-Breaker: AWS Route Priority

**The Concept:** If the prefixes are **identical** (e.g., both links advertise `10.0.0.0/16`), AWS uses a predefined hierarchy to decide which path to prefer.

**Detailed Justification:**
If we update the VPN to also advertise/route `10.0.0.0/16`, the Transit Gateway follows this internal priority list for "External" routes:

1. **Direct Connect (DX) Dedicated/Transit VIF** (Highest Priority)
2. **Site-to-Site VPN (BGP/Dynamic)**
3. **Site-to-Site VPN (Static)** (Lowest Priority)

By making the CIDRs identical, the DX link naturally wins because AWS is programmed to prefer the dedicated hardware of Direct Connect over the public internet-based VPN.

## 3. The "Advanced" Fix: AS-Path Prepending

**The Concept:** If both links use **BGP (Dynamic Routing)** and have the same prefix, how do we tell the *On-Premises* router to prefer the DX for return traffic?

**Detailed Justification:**
BGP calculates the "shortest" path based on how many "hops" (Autonomous Systems) a packet must jump through.

* To ensure the VPN remains a backup, we can use **AS-Path Prepending**.
* We make the VPN path look "longer" by repeating our own AS number multiple times in the BGP advertisement (e.g., instead of Path `[65000]`, we send Path `[65000, 65000, 65000]`).
* The router sees the DX path as "shorter" and prefers it, only switching to the "longer" VPN path if the DX path disappears from the table entirely.

---

### **Summary Table for Students**

| Priority | Strategy | Why it works |
| --- | --- | --- |
| **1st (Override)** | **Longest Prefix Match** | `/24` always beats `/16`. This was the cause of our puzzle's failure. |
| **2nd (Standard)** | **AWS Path Preference** | DX is natively preferred over VPN if CIDRs are identical. |
| **3rd (Tuning)** | **BGP Attributes** | AS-Path Prepending makes a backup link look less attractive to BGP. |

**Would you like me to expand the "DNS Ghost" puzzle with similar guiding hints and detailed justifications?**