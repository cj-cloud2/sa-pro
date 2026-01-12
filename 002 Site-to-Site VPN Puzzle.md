# AWS Site-to-Site VPN Puzzle: On-Premises to VPC Connectivity (15 Min)

## **Puzzle Overview**

Your organization has established an AWS Site-to-Site VPN connection between your on-premises data center and a VPC in AWS. The VPN tunnel is up and operational, and ping tests show that traffic is reaching the remote network. However, return traffic from the VPC back to on-premises is failing intermittently. Your task is to identify the tunnel misconfiguration blocking bidirectional traffic flow.

***

## **Puzzle Scenario**

**Current Setup:**

- **On-Premises Network:** 10.0.1.0/24 (corporate LAN with servers and workstations)
- **AWS VPC CIDR:** 10.0.0.0/16
- **VPC Subnets:**
    - Private Subnet A: 10.0.1.0/24 (application servers)
    - Private Subnet B: 10.0.2.0/24 (database servers)
- **Customer Gateway (On-Premises):** 203.0.113.10 (public IP of on-premises VPN device)
- **Virtual Private Gateway (AWS):** Attached to VPC
- **VPN Connection:** Status shows "available" with tunnel 1 and tunnel 2 both "UP"
- **Tunnel Configuration:** Both tunnels configured with Phase 1 and Phase 2 encryption/authentication parameters

**Problem Statement:**
A junior network engineer has just completed the Site-to-Site VPN setup. When testing:

- ✓ On-premises user can ping 10.0.1.5 (EC2 in private subnet A) — **Works intermittently**
- ✗ EC2 instance in private subnet A cannot ping 10.0.1.100 (on-premises server) — **Always fails**
- ✗ EC2 instance in private subnet B cannot ping any on-premises server — **Always fails**

The on-premises network admin confirms that the Customer Gateway device is correctly configured and can establish the VPN tunnel. AWS Console shows both VPN tunnels are healthy with encrypted traffic flowing through them.

**Key Clue:** The on-premises admin says, "When I ping from our side, AWS EC2 instances sometimes respond, but when EC2 instances try to reach us, we see nothing."

***

## **Guiding Hints for Students**

### Hint 1: VPN Tunnels ≠ End-to-End Connectivity

The VPN tunnels being "UP" means the encrypted tunnel is established between the Customer Gateway and Virtual Private Gateway. But "UP" does **not** mean:

- Traffic is being routed through the tunnel
- Both directions of traffic are configured
- Return paths are established

Ask yourself: "A tunnel existing doesn't mean packets know to use it. What tells packets **where** to go?"

### Hint 2: Think About Routing on Both Sides

VPN connections require **bidirectional routing**:

- **On-premises side:** On-premises router must have a route for 10.0.0.0/16 (AWS VPC) pointing to the VPN tunnel
- **AWS side:** VPC route tables must have routes for 10.0.1.0/24 (on-premises) pointing to the Virtual Private Gateway

If either side is missing its route, traffic flows in one direction only or fails in both directions. Which direction is failing in this scenario, and what does that tell you?

### Hint 3: Asymmetric Routing Problem

You notice that on-premises can sometimes reach 10.0.1.0/24 in AWS, but AWS cannot reach 10.0.1.0/24 on-premises. This suggests:

- Forward path (on-premises → AWS) is working
- Return path (AWS → on-premises) is broken

What configuration would cause the return path to fail? Hint: What if the route table on the AWS side is incomplete or misconfigured?

### Hint 4: Multiple Subnets, Single Route?

The VPC has two private subnets: 10.0.1.0/24 and 10.0.2.0/24. Both are part of the broader 10.0.0.0/16 CIDR block. But is advertising the entire 10.0.0.0/16 to on-premises the same as explicitly routing traffic from both subnets through the VPN?

Think about: Route table propagation. Does every subnet automatically know about VPN routes, or must you configure them explicitly?

### Hint 5: CIDR Overlap and Ambiguity

Look carefully at the CIDRs:

- On-premises: 10.0.1.0/24
- AWS VPC: 10.0.0.0/16 (which includes 10.0.1.0/24, 10.0.2.0/24, etc.)
- Private Subnet A in VPC: 10.0.1.0/24

Wait—both on-premises and a subnet in AWS use the same CIDR (10.0.1.0/24)! Could this be causing routing ambiguity? What happens when the on-premises network is 10.0.1.0/24 and the VPC also has a subnet with 10.0.1.0/24?

### Hint 6: The Return Traffic Problem

When an EC2 instance in the VPC responds to a ping from on-premises:

- The EC2 instance receives a packet from 10.0.1.100 (on-premises)
- It constructs a response packet with Source IP = 10.0.1.x (its own IP)
- The EC2's operating system looks at the destination IP (10.0.1.100) and consults the route table: "Where should I send packets destined for 10.0.1.100?"

What does the route table say? Is there a route for 10.0.1.0/24 pointing to the VPN, or does the route point somewhere else?

***


