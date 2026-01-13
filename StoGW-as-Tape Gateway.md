# Puzzle 1: The "Missing Vault" Paradox (Tape Gateway)

### The Scenario
A law firm is migrating 20 years of records to the cloud using **AWS Tape Gateway**. They have integrated it with their existing Backup Exec software. The architect creates 10 virtual tapes. 

The backup software completes the job and "ejects" the tapes. The firm’s compliance officer logs into the **S3 Console** to verify that the data is stored in the **S3 Glacier Deep Archive** bucket to save costs, but they cannot find any objects or buckets related to these tapes.

### The Conceptual Challenge
1. Where does the data "live" once the tape is ejected by the backup software?
2. Why can't the officer see the tapes in the S3 Console?

---

### Answer Key 
* **The Cause:** Tape Gateway data is stored in AWS-managed S3 storage, but it is **not visible as objects** in the customer’s S3 buckets. 
* **Concept:** Tapes must be managed via the **Storage Gateway Console (VTL)** or the backup software. Once archived, they move to the "Virtual Shelf" (Glacier), but they are never directly accessible as flat files in a user-owned S3 bucket.


---
---

## Puzzle 2: The "Air-Gapped" Restoration (Tape Gateway)

### The Scenario

A financial institution uses **Tape Gateway** for long-term compliance.

* They have 500 Virtual Tapes archived in **S3 Glacier Deep Archive**.
* **The Crisis:** The local Tape Gateway VM appliance is accidentally deleted from the on-premises hypervisor, and all local cache data is lost.
* **The Requirement:** They need to restore a specific tape (Barcode: `FIN_2022_Q4`) immediately for an audit.

### The Conceptual Challenge

1. Since the Gateway VM is gone, is the data lost?
2. Describe the architectural "Relocation" process. Can they simply spin up a new Gateway and see the archived tapes immediately? What is the "missing link" in the recovery workflow?

---

### Answer Key 

* **The Solution:** No, data is not lost. Tapes in S3 Glacier are independent of the Gateway instance.
* **The Workflow:** They must create a **new** Gateway, but the archived tapes will not appear in the "VTL" (Virtual Tape Library). They must first "Retrieve" the tape from the Archive (Glacier) to the S3 layer, and then **assign** the retrieved tape to the new Gateway's VTL.
* **Key Concept:** The decoupling of the *Gateway* (the access point) from the *Virtual Tape* (the data entity).

---
