# AWS Client VPN Puzzle: Remote Workforce Secure Access
**15 Mins to Solve **
**10 Mins to Review Solution **

## **Puzzle Overview**

Your organization needs to enable remote developers to securely access private EC2 instances in their VPC. A Client VPN endpoint has been partially configured, but the remote team reports they cannot access the resources even though their VPN connection is successful. Your task is to troubleshoot and identify what's missing.

***

## **Puzzle Scenario**

**Current Setup:**

- **VPC CIDR:** 10.0.0.0/16
- **Private Subnet 1:** 10.0.1.0/24 (contains development EC2 instances)
- **Private Subnet 2:** 10.0.2.0/24 (contains database servers)
- **Client VPN Endpoint:** Created with Client IPv4 CIDR 172.16.0.0/16
- **Authentication:** Mutual certificate-based authentication is already configured
- **Tunneling Mode:** Split-Tunnel is enabled

**Problem Statement:**
Remote developers connect successfully to the Client VPN endpoint (they receive VPN IP addresses like 172.16.0.x), but when they attempt to ping EC2 instances on 10.0.1.5 or 10.0.2.10, the requests timeout. Interestingly, they can access their home internet and external websites without issues.

***

## **Guiding Hints for Students**

### Hint 1: Layer-by-Layer Troubleshooting Approach

Think about the OSI model. The VPN connection succeeds (Layer 3 connectivity exists), but resource access fails. What are the potential layers where rules or configurations could block traffic? Consider:

- Network-level access controls
- Firewall rules within the VPC
- Routing decisions


### Hint 2: Focus on Authorization

The puzzle involves two types of authorization mechanisms in AWS Client VPN:

1. **Network-based authorization** - What networks can VPN clients access?
2. **Instance-level authorization** - What security groups allow VPN traffic to resources?

Ask yourself: "Even if the Client VPN endpoint knows about the VPC network, how does the EC2 instance know it should accept traffic from the VPN clients?"

### Hint 3: Split-Tunnel Implications

With split-tunnel enabled, the VPN client receives specific routes from the VPN endpoint. What routes does the client need to know about? Hint: Without these routes, the client might attempt to send VPC-destined traffic through the internet gateway instead of the VPN tunnel.

### Hint 4: Security Group Perspective

When a remote user connects via Client VPN and tries to access an EC2 instance, the EC2 instance's security group acts as a stateful firewall. The EC2 sees traffic coming from... what source? (Hint: The VPN endpoint applies a security group to its network interfaces in the target subnet.)

### Hint 5: Multi-Part Problem

This isn't a single-issue puzzle. There are likely **two or three missing configurations** that need to be addressed for complete connectivity.

***


