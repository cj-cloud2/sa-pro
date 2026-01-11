## **Solution \& Teaching Justification** (10 min to review)

### **Issue \#1: CIDR Overlap - Critical Ambiguity**

**The Problem:**
The on-premises network uses 10.0.1.0/24, and the AWS VPC Private Subnet A also uses 10.0.1.0/24. This creates a **routing ambiguity** that violates a fundamental networking principle: **never use overlapping address spaces between sites connected via VPN**.

**What Went Wrong:**
When the VPC route table needs to send traffic destined for 10.0.1.0/24, it has a decision problem:

1. Route A: Send via local subnet (eth0)
2. Route B: Send via Virtual Private Gateway (VPN)

Most routing algorithms prefer **more specific routes** or **local routes**. The route table likely interprets "10.0.1.0/24 destined traffic" as "local traffic on the private subnet" rather than "traffic to the remote on-premises network."

**Teaching Justification - Why CIDR Overlap Breaks Routing:**

Imagine two cities both named "Springfield" in different states. If you send a letter addressed only to "Springfield, Main Street, House \#5," the postal service doesn't know which Springfield you mean. Similarly, routing relies on **unique address spaces** to make unambiguous decisions.

**How Routing Works (Conceptually):**

```
Route Table Decision Logic:
┌─────────────────────────────────────┐
│ Packet arrives: Destination IP = 10.0.1.50 │
├─────────────────────────────────────┤
│ Consult Route Table for best match  │
│                                     │
│ Rule 1: 10.0.1.0/24 → Local (eth0) │ ← Matches!
│ Rule 2: 10.0.0.0/16 → VGW (VPN)    │ ← More specific match: Rule 1
│ Rule 3: 0.0.0.0/0 → IGW            │
├─────────────────────────────────────┤
│ Decision: Send via eth0 (Local)     │
│ Problem: This is on-premises traffic!
│ It should go via VGW (VPN)          │
└─────────────────────────────────────┘
```

**The Real-World Impact:**

- On-premises can reach 10.0.1.0/24 in AWS because the on-premises route table explicitly says "10.0.1.0/24 is local" and "10.0.0.0/16 is via VPN." On-premises routing is unambiguous.
- But AWS route table is confused: it sees 10.0.1.0/24 and thinks "that's my local subnet," not realizing some of that space is also on-premises.

**The Solution:**
**Change the on-premises CIDR to something that doesn't overlap with the VPC.**

For example, redesign the network:

- **On-premises CIDR:** 10.0.100.0/24 (or 192.168.1.0/24, or any non-overlapping range)
- **AWS VPC CIDR:** 10.0.0.0/16 (unchanged)
- **AWS Private Subnet A:** 10.0.1.0/24
- **AWS Private Subnet B:** 10.0.2.0/24

Now routing is unambiguous:

- EC2 in 10.0.1.0/24 looking for 10.0.100.0/24 → Definitely on-premises → Route via VGW ✓

**Why This Is Non-Negotiable:**
This isn't a "best practice"—it's a **fundamental requirement** of any network-to-network connection (VPN, Direct Connect, peering, etc.). Overlapping address spaces break routing logic at its core.

***

### **Issue \#2: Route Table Propagation Disabled**

**The Problem:**
Even if you fix the CIDR overlap, there's still a configuration issue: **VPN routes are not being propagated to the VPC route tables**.

**What Went Wrong:**
The Virtual Private Gateway is attached to the VPC, but route propagation may be disabled. This means:

- The VGW knows about the on-premises network (10.0.1.0/24, after CIDR redesign)
- But this information is **not automatically added** to the VPC's subnet route tables
- Without explicit routes pointing to the VGW, traffic destined for on-premises is not sent through the VPN tunnel

**Teaching Justification - Understanding Route Propagation:**

AWS has a feature called **route table propagation** that automatically adds routes for VPN connections and Direct Connect connections to route tables. However, this feature must be **explicitly enabled** for each route table.

**Two Ways to Get Routes into a Route Table:**

1. **Manual Route Entry:**
    - Go to Route Table → Edit Routes
    - Add: Destination = 10.0.100.0/24, Target = Virtual Private Gateway
    - This is explicit and clear but requires manual maintenance if networks change
2. **Route Propagation (Automatic):**
    - Enable "propagate routes" on the route table
    - The VGW automatically learns about the on-premises network (via BGP or static configuration)
    - Routes automatically appear in the route table without manual intervention
    - If the on-premises network changes, the route updates automatically (if using dynamic routing)

**How Route Propagation Works:**

```
Virtual Private Gateway (VGW)
    ↓
Learns on-premises routes (via BGP or static config)
    ↓
Checks: Is route propagation enabled on the VPC route tables?
    ├─ YES: Automatically injects routes into route tables
    └─ NO:  Routes remain unknown to route tables
               (only explicit routes in route tables matter)
```

**The Problem Scenario:**

- VGW knows: "On-premises is 10.0.100.0/24"
- But route tables don't know about this route (propagation disabled)
- Traffic destined for 10.0.100.0/24 has no matching route
- Default route (0.0.0.0/0 → IGW) is used instead
- Packets exit through Internet Gateway instead of VGW
- On-premises sees traffic coming from AWS public IP, not VPN
- On-premises firewall blocks it (unexpected source)

**The Solution:**

**Option A - Enable Route Propagation (Recommended):**

1. Go to VPC → Route Tables
2. Select each private subnet's route table
3. Click "Route Propagation" or "Propagate Routes"
4. Enable propagation for the Virtual Private Gateway
5. Within minutes, routes for 10.0.100.0/24 automatically appear

**Advantages:**

- Automatic and dynamic
- Scales well when on-premises networks change
- Less manual configuration
- Works with dynamic routing protocols (BGP)

**Option B - Manual Static Routes:**

1. Go to each route table
2. Add route: Destination = 10.0.100.0/24, Target = Virtual Private Gateway
3. Repeat for every subnet route table

**Disadvantages:**

- Manual and static
- Requires updates when on-premises networks change
- Doesn't scale well for multiple on-premises networks
- Error-prone

***

### **Issue \#3: Security Group Configuration (Return Traffic)**

**The Problem:**
After fixing CIDR overlap and route propagation, there's one more obstacle: **the EC2 instances' security groups may not allow return traffic** from the on-premises network.

**What Went Wrong:**
When an on-premises client initiates a connection to an EC2 instance:

- **Outbound (On-Premises → EC2):** On-premises firewall allows this
- **Inbound (EC2 ← On-Premises):** EC2 security group must allow return traffic

If the EC2 instance's security group has **no inbound rules allowing the on-premises CIDR**, the return traffic is blocked.

**Teaching Justification - Why Return Traffic Matters:**

In TCP connections, both directions must be explicitly allowed:

```
Stateful Firewalls: Return traffic is "remembered"
────────────────────────────────────────────────

Initial Connection (Outbound - usually allowed):
┌─────────────────────────────────┐
│ Source:      10.0.100.5 (on-prem) │
│ Destination: 10.0.1.10 (EC2)      │
│ Port:        80 (web request)    │
├─────────────────────────────────┤
│ On-Premises Firewall: OK, allowed │
│ Security Group: Sees inbound      │
│                traffic from on-prem
│                → Check rules      │
└─────────────────────────────────┘

Return Traffic (Inbound - must be explicitly allowed OR established):
┌──────────────────────────────────┐
│ Source:      10.0.1.10 (EC2)       │
│ Destination: 10.0.100.5 (on-prem) │
│ Port:        80 (web response)    │
├──────────────────────────────────┤
│ Security Group: Is this allowed?  │
│ ├─ If SG has rule for on-prem:    │
│ │  ✓ Allowed (explicit rule)      │
│ └─ If SG has no on-prem rule:     │
│    ✗ Blocked                      │
└──────────────────────────────────┘
```

**Important Note About Stateful Inspection:**
AWS security groups are **stateful**, meaning:

- If you allow **outbound** traffic from the EC2 to on-premises, return traffic is **automatically allowed** (the SG remembers the outbound connection)
- However, if on-premises initiates the connection, the EC2 must have an **inbound rule** explicitly allowing the on-premises source

**The Solution:**

Add inbound rules to EC2 instances' security groups:


| **Rule** | **Type** | **Protocol** | **Port Range** | **Source** | **Description** |
| :-- | :-- | :-- | :-- | :-- | :-- |
| 1 | Inbound | TCP | 22 | 10.0.100.0/24 | SSH from on-premises |
| 2 | Inbound | TCP | 443 | 10.0.100.0/24 | HTTPS from on-premises |
| 3 | Inbound | TCP | 3306 | 10.0.100.0/24 | MySQL from on-premises |
| 4 | Inbound | ICMP | N/A | 10.0.100.0/24 | Ping from on-premises |

**Or (Less Secure - for testing):**

- Allow all TCP/UDP from 10.0.100.0/24 (testing only, restrict later)

**Best Practice:**

- Create a dedicated security group for resources that accept on-premises traffic
- Name it descriptively: `sg-on-premises-access`
- Document which ports and protocols are needed
- Review and audit these rules regularly

***

### **Issue \#4: Verify Virtual Private Gateway Route Table (AWS-Side)**

**The Problem:**
The VGW itself has a route table that defines which routes it knows about and propagates. If the VGW route table is misconfigured, it won't propagate the correct routes even if propagation is enabled.

**What Went Wrong:**
The Virtual Private Gateway needs to know about the on-premises network. This happens via:

1. **BGP (Dynamic Routing):** If you're using dynamic BGP, on-premises must advertise its routes
2. **Static Routes:** You manually specify on-premises networks in the VGW route table

If neither is configured, the VGW doesn't know about 10.0.100.0/24 and can't propagate it.

**Teaching Justification - VGW Route Table:**

Think of the VGW as a gateway with its own intelligence:

```
VGW: "I'm connected to on-premises via tunnel."
     "What networks are reachable through that tunnel?"

Option 1 - BGP (automatic):
  On-Premises sends: "Here's my network: 10.0.100.0/24"
  VGW learns this dynamically
  VGW propagates: 10.0.100.0/24 → VGW (to all route tables)

Option 2 - Static (manual):
  You tell VGW: "Know that 10.0.100.0/24 is reachable via tunnel"
  VGW stores this statically
  VGW propagates: 10.0.100.0/24 → VGW (to all route tables)
```

**The Solution:**

**For BGP (Recommended if on-premises supports it):**

1. Check VPN Connection details → Route Options
2. Ensure "Dynamic routing" is selected (instead of Static)
3. Configure BGP on on-premises side to advertise 10.0.100.0/24
4. VGW will automatically learn and propagate routes

**For Static Routes (If BGP not available):**

1. Go to VPC → Virtual Private Gateways
2. Select your VGW
3. View Route Propagation or manage static routes
4. Add static route: 10.0.100.0/24 (on-premises network)
5. Save

**Which is Better?**

- **BGP:** Better for production, scales to multiple networks, automatic failover handling
- **Static:** Simple for small deployments, but requires manual updates when on-premises networks change

***

## **Summary of Misconfigurations**

| **Configuration** | **What It Controls** | **Why It Broke Return Traffic** | **Key Learning Point** |
| :-- | :-- | :-- | :-- |
| **CIDR Overlap** | Address space uniqueness | VPC route table confused 10.0.1.0/24 as local, not remote | Overlapping CIDRs break routing logic entirely |
| **Route Propagation** | Route table population from VGW | VPC route tables didn't know about on-premises network, so packets went to IGW instead | Routes must be in the route table for packets to be sent via VPN |
| **Security Group Rules** | Instance-level firewall | EC2 blocked inbound traffic from on-premises CIDR | Even if packets reach the VPC, instances have their own firewall |
| **VGW Route Configuration** | Routes known by the gateway | VGW didn't advertise on-premises networks to route tables | VGW must know about on-premises networks to propagate them |


***

## **Troubleshooting Decision Tree**

Students can use this to diagnose similar issues:

```
VPN Tunnel Status = UP?
├─ NO → Tunnel configuration issue (encryption, authentication)
│       (Not covered in this puzzle)
│
└─ YES → Tunnels are UP, but traffic fails?
   │
   ├─ Check 1: Do address spaces overlap?
   │  ├─ YES → REDESIGN: Change on-premises CIDR
   │  └─ NO  → Continue
   │
   ├─ Check 2: Are there routes to on-premises in VPC route tables?
   │  ├─ NO  → Enable route propagation OR add static routes
   │  └─ YES → Continue
   │
   ├─ Check 3: Do EC2 security groups allow on-premises CIDR?
   │  ├─ NO  → Add inbound rules for on-premises CIDR
   │  └─ YES → Continue
   │
   ├─ Check 4: Is VGW configured with on-premises networks (BGP or static)?
   │  ├─ NO  → Configure BGP or add static routes to VGW
   │  └─ YES → All done!
   │
   └─ If still failing → Advanced troubleshooting (tcpdump, VPC Flow Logs)
```


***

## **Real-World Analogies**

**CIDR Overlap:**
Imagine a postal system where two different towns are both named "Springfield." When you send a letter to "Springfield, Main St, House 5," the post office doesn't know which Springfield. Similarly, when traffic is destined for 10.0.1.0/24, the router doesn't know if it's local or remote.

**Route Propagation:**
The VGW is a guard at a gate. The guard knows what's on the other side (on-premises networks), but if the guard doesn't have a bulletin board to announce this information, the townspeople (route tables) won't know. Route propagation is the bulletin board.

**Security Group Rules:**
Even after packages reach the right building, they must pass through the building's reception desk (security group). If the reception desk has no rule allowing visitors from Springfield, packages are rejected at the door.

***

## **Common Misconceptions to Avoid**

1. **"VPN tunnel UP means traffic flows"** ❌
    - UP = tunnel established. Traffic flow requires routes, proper addressing, and firewall rules.
2. **"If on-premises can reach AWS, AWS can reach on-premises"** ❌
    - Asymmetric routing is common. Verify both directions explicitly.
3. **"The entire VPC is reachable via VPN once you connect"** ❌
    - Only subnets with proper route table entries and security group rules can communicate.
4. **"CIDR overlap is just a naming inconvenience"** ❌
    - It fundamentally breaks routing logic and violates the principle of unique address spaces.
5. **"BGP is always better than static routes"** ❌
    - BGP is better for dynamic/complex networks. Static routes are fine for simple, stable setups.

***

## **Verification Steps (For Instructors)**

Have students verify the corrected configuration:

1. **Check for CIDR Overlap:**
    - On-premises network CIDR should NOT overlap with VPC CIDR
    - All subnets should have unique, non-overlapping address spaces
2. **Verify Route Propagation:**
    - Go to each private subnet's route table
    - Check if "Route Propagation" is enabled for the VGW
    - Verify that routes for on-premises CIDR appear in the route table
3. **Check Security Groups:**
    - Select EC2 instances
    - Inbound rules should allow on-premises CIDR (10.0.100.0/24) for required ports
    - Test: `telnet 10.0.1.10 22` (from on-premises) should connect if SSH is allowed
4. **Verify VGW Configuration:**
    - Check if VGW is configured with on-premises routes (BGP or static)
    - If BGP: Check BGP status and advertised routes
    - If Static: Verify static routes are defined
5. **Test Connectivity (Both Directions):**
    - From on-premises: `ping 10.0.1.10` (should work)
    - From EC2: `ping 10.0.100.5` (should work)
    - Both should be consistent and reliable

***

## **Advanced Considerations (For SA-PRO)**

1. **MTU Size:** VPN tunnels may have reduced MTU (1436 bytes). Packets larger than this are fragmented. Discuss `Path MTU Discovery` implications.
2. **BGP ASN Conflict:** If on-premises and AWS both use same BGP ASN, routing fails. Each must have unique ASN.
3. **Redundancy:** Two VPN tunnels provide redundancy, but both must be configured identically for proper failover.
4. **Asymmetric Routing with Multiple Tunnels:** One tunnel might work in one direction, another in the reverse direction, causing intermittent failures.
5. **VPC Flow Logs:** Show students how to use VPC Flow Logs to diagnose where packets are being rejected (at instance level, security group level, or elsewhere).

***

This puzzle teaches students that **Site-to-Site VPN connectivity requires multiple layers of configuration**—addressing, routing, gateway configuration, and security rules. A failure in any layer breaks bidirectional communication.

