## Puzzle 1: The "Version Ghost" (File Gateway)

### The Scenario

A global research firm uses **S3 File Gateway** to bridge their on-premises lab in Pune with an S3 bucket in `us-east-1`.

* **The Workflow:** A script on-premises writes a file `results.csv` to the NFS mount.
* **The Conflict:** Simultaneously, a Lambda function in AWS updates the same `results.csv` object directly in S3 with supplementary metadata.
* **The Issue:** When the Pune lab reads `results.csv` 30 seconds later via the NFS mount, they see their original data without the Lambda's updates. Even after waiting 10 minutes, the file remains "stale" on-premises, despite S3 being highly consistent.

### The Conceptual Challenge

1. Why does the Gateway's local view diverge from S3's "Source of Truth" even after the 30-second propagation window for S3?
2. If the team implements an automated `RefreshCache` every 60 seconds, what architectural "side effect" might they encounter regarding **S3 Object Versioning** and **Automated Cache Eviction**?

---

### Answer Key 

* **The Cause:** The File Gateway does not "watch" S3 for changes. It assumes it is the primary owner of the prefix. It only refreshes metadata based on its internal TTL or an explicit `RefreshCache` API call.
* **The Side Effect:** Frequent `RefreshCache` on a large bucket causes high metadata overhead and may lead to increased GET requests. Furthermore, if versioning is on, the Gateway always points to the "Latest" version, but manual S3 overrides can lead to "Split Brain" where the Gateway's write-back might overwrite the Lambda's changes if the file handle was kept open.

---


# Puzzle 2: The "Ghost in the Machine" (File Gateway)

### The Scenario
A media house uses an **AWS Storage Gateway (S3 File Gateway)** to allow editors in Pune to upload raw footage via an NFS mount. The files are backed by an S3 bucket. 

One afternoon, an editor saves `interview_final.mov` to the local mount. A cloud engineer in Seattle, looking directly at the **S3 Bucket via the AWS Console**, refreshers the page repeatedly for 5 minutes but cannot see the file. However, the Pune editor can see the file perfectly on their local mapped drive.

### The Conceptual Challenge
1. Why is there a discrepancy between the local NFS view and the S3 Console view?
2. What specific architectural mechanism/API call is required to "force" the cloud-side visibility if the file had been uploaded to S3 by a different process?

---

### Answer Key 
* **The Cause:** File Gateway uses a local cache. While writes are asynchronous to S3, the metadata is managed locally first. If the file was uploaded directly to S3 (bypassing the gateway), the Gateway wouldn't know until a **RefreshCache** is triggered.
* **Concept:** Understanding that File Gateway is a *cached* interface, not a real-time synchronous mirror of the S3 API.


