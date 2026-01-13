# Puzzle 1: The "Boot Disk" Dilemma (Volume Gateway)

### The Scenario
You are architecting a solution for a legacy SQL Server running on-premises. 
* **Requirement A:** The application requires sub-millisecond latency for all read/write operations because the database is extremely "chatty."
* **Requirement B:** You need a point-in-time recovery option in AWS in case the local data center loses power.
* **Constraint:** Your on-premises SAN (Storage Area Network) is almost at full capacity.

### The Conceptual Challenge
Which configuration of Volume Gateway do you choose: **Stored Volumes** or **Cached Volumes**? Explain why one of these options *fails* one of the requirements/constraints.

---

### Answer Key (For Trainer)
* **The Winner:** Neither is a perfect fit, but **Stored Volumes** is the only one meeting the "sub-millisecond" read/write requirement (as all data is local).
* **The Failure:** Stored Volumes will fail the **Constraint** (SAN at full capacity) because it requires the full data set to be stored locally. **Cached Volumes** would solve the capacity issue but would fail the "chatty" latency requirement if the "working set" exceeds the cache.


---

## Puzzle 2: The "Split-Path" Performance Trap (Volume Gateway)

### The Scenario

A high-performance computing (HPC) app uses **Volume Gateway (Cached Mode)**.

* The "Working Set" (active data) is 500GB.
* The Local Cache is 1TB (plenty of room).
* **The Anomaly:** During a specific 2-hour window, write latency spikes from 2ms to 200ms, even though the total throughput is well within the local SSD's hardware limits.
* **The Observation:** CloudWatch shows the `UploadBufferFree` metric is dropping to zero during this window.

### The Conceptual Challenge

1. Why is the *Read* performance unaffected while *Writes* are stalling, despite having a massive local cache?
2. Distinguish between the role of the **Cache Disk** and the **Upload Buffer** in this specific bottleneck.

---

### Answer Key 

* **The Cause:** The **Upload Buffer** is the bottleneck. In Volume Gateway, a write is only acknowledged as "successful" to the application once it is staged in the Upload Buffer. If the internet bandwidth to AWS cannot drain the buffer as fast as the app is writing, the buffer fills up.
* **The Fix:** The cache disk (for reads) is separate from the upload buffer (for staging writes to S3). Increasing cache size won't help; they need more upload buffer space or more outbound bandwidth.

---
