# Puzzle 1: The "Sovereignty" Toss-up (Outposts vs. Local Zones)

### The Scenario
Two startups have different needs for "Edge" computing:
1.  **Startup A (FinTech):** Must comply with a law stating that user data *physically cannot leave their owned office building*.
2.  **Startup B (Video Streaming):** Needs to deliver content to users in Kolkata with <10ms latency but does not want to manage any hardware or rent data center space.

### The Conceptual Challenge
Assign **AWS Outposts** or **AWS Local Zones** to each startup. Justify why the other service is architecturally "illegal" for Startup A's specific constraint.

---

### Answer Key (For Trainer)
* **Startup A:** Must use **AWS Outposts**. (Local Zones are physically located in AWS-managed facilities, which violates the "physical office" constraint).
* **Startup B:** Must use **AWS Local Zones**. (Provides low latency without the overhead of managing the physical Outpost rack/power/cooling).
* **Concept:** Outposts = Customer-owned site. Local Zones = AWS-owned site near a city.

---

## Puzzle 2: The "Disconnected" Outpost (Outposts vs. Local Zones)

### The Scenario

A manufacturing plant uses **AWS Outposts** to run an AI-based quality control system on a conveyor belt.

* The Outpost rack is connected to the Mumbai Region via a Direct Connect (DX).
* **The Incident:** A construction crew accidentally cuts the fiber optic cable, and the Outpost loses connectivity to the AWS Region for 4 hours.
* **The Behavior:** The EC2 instances on the Outpost keep running, but the local administrators find they cannot restart an instance or attach a new EBS volume during the outage.

### The Conceptual Challenge

1. Why is "Control Plane" functionality lost while "Data Plane" functionality remains during a disconnect?
2. If this were an **AWS Local Zone** instead of an **Outpost**, would the conveyor belt AI have continued to function during the fiber cut? Why or why not?

---

### Answer Key 

* **The Outpost issue:** The Outpost Control Plane (the "brain") resides in the Parent Region. Without a connection, you can't issue new API commands (Start/Stop/Attach), but the local hypervisor keeps existing workloads running.
* **Local Zone comparison:** If it were a Local Zone, the AI would likely **fail entirely**. Since Local Zones are AWS-managed facilities "in the cloud," the plant's connection to the Local Zone is over the same WAN/DX. If the line is cut, the plant loses access to the Local Zone entirely.
* **Key Concept:** Outposts provide **local survival** for the data plane; Local Zones do not.

---
