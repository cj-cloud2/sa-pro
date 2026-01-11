# Puzzle: The DNS Ghost

## The Scenario

**GlobalTech** is migrating its legacy applications to AWS. They have established a hybrid network using a Transit Gateway and a Direct Connect link.

* **On-Premises:** Uses a traditional DNS server (IP: `172.16.0.10`) managing the domain `corp.internal`.
* **AWS:** Uses a Private Hosted Zone (PHZ) associated with their VPC for the domain `aws.corp.internal`.

**The Setup:**
To allow name resolution between environments, the team has deployed **Route 53 Resolver Endpoints**:

1. **Outbound Endpoint:** With a rule that forwards all queries for `corp.internal` to the on-prem DNS IP (`172.16.0.10`).
2. **Inbound Endpoint:** Providing an IP (`10.0.0.50`) for the on-prem DNS server to forward queries for `aws.corp.internal`.

**The Incident:**
A developer on an EC2 instance tries to reach `legacy-app.corp.internal`. Instead of getting an IP, they receive a **SERVFAIL** or a "Connection Timeout." Strangely:

* The EC2 instance **can** ping the on-prem DNS server IP (`172.16.0.10`).
* The EC2 instance **can** resolve `google.com`.
* The on-prem server **can** resolve `web-app.aws.corp.internal` via the Inbound Endpoint.

---

## The Puzzle Challenge

**The connection is alive, and the rules are in place. Why is the "Ghost" preventing EC2 from resolving the on-prem domain?**

### Guiding Hints

* **Hint 1: The "Recursive Loop" Trap.** If the on-prem DNS server doesn't know an answer, where does it send the query? Check if it has a "Global Forwarder" pointing back to AWS.
* **Hint 2: The Security Group Sentry.** An Outbound Endpoint is like an EC2 instance. It lives in a subnet. Does its Security Group allow the "return" traffic from the on-prem DNS server?
* **Hint 3: The "Source IP" Mystery.** When the query leaves AWS via the Outbound Endpoint, what is its Source IP? Is the on-prem Firewall/ACL expecting that specific IP?
* **Hint 4: Rule Priority.** Route 53 Resolver evaluates rules from most specific to least specific. Is there a "dot" (`.`) rule (Autodefined or manual) interfering with the `corp.internal` rule?

---
