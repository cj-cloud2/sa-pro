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
