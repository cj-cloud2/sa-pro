# Puzzle 1: The "Last Mile" Latency (AWS Wavelength)

### The Scenario
An Augmented Reality (AR) gaming company has deployed their backend on EC2 instances in the `ap-south-1` (Mumbai) region. Players on 5G devices in Delhi are experiencing a 120ms lag. 

The architect suggests moving the "Game Engine" component to an **AWS Wavelength Zone**. However, after the move, the players' phones still show 120ms latency because they are connecting to the application via a standard Public Load Balancer.

### The Conceptual Challenge
1. Why did moving to Wavelength not automatically fix the latency?
2. What specific networking component (Gateway type) must be utilized to route traffic from the 5G carrier network directly to the Wavelength Zone?

---

### Answer Key (For Trainer)
* **The Cause:** Traffic is still "tromboning" (going out to the internet and back). 
* **Concept:** To get the benefit of Wavelength, the app must use a **Carrier Gateway**. This allows the device to stay within the Telecommunication Carrier's network, avoiding the hops through the public internet.

---

## Puzzle 2: The "Non-Local" Local Breakout (AWS Wavelength)

### The Scenario

A developer deploys a sub-millisecond latency app in a **Wavelength Zone** (Carrier: Vodafone, Location: London).

* The EC2 instance has a **Carrier IP** assigned.
* **The Failure:** A user on a **different** mobile carrier (e.g., EE or O2) in the same city tries to connect but experiences 150ms latency, exactly the same as if they were hitting the main AWS Region.

### The Conceptual Challenge

1. Why does the "Wavelength Magic" only work for users on the *specific* carrier hosting the Wavelength Zone?
2. From a routing perspective, what happens to the packet when it originates from a "Competitor" Carrier network?

---

### Answer Key 

* **The Cause:** Wavelength Zones are physically embedded *inside* the service provider’s (Carrier’s) data center, behind their Gi-LAN.
* **The Concept:** Traffic from Carrier A cannot hop directly into Carrier B's Wavelength Zone. It must exit Carrier A's network to the Public Internet (Internet Exchange), then enter Carrier B's network. This "Internet Hop" destroys the low-latency benefit.

---