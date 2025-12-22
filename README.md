# Reichman University Cloud Workspace

Welcome to the Reichman University Cloud Environment. This guide outlines the critical rules and workflows to ensure high performance and cost efficiency while working with our infrastructure.

---

## ⚠️ Critical Rules: Read Before Starting

### 1. DO NOT Run Models on Network Drives
All network-attached drives are **Azure Files** storage.
* **Risk:** Running high-IOPS operations (heavy reading/writing) directly on these drives will cause significant high costs.
* **Policy:** You must treat the network drive as **cold storage** only.

### 2. The Correct Workflow
To ensure performance and avoid cost spikes, follow this strictly:
1.  **Transfer:** Copy your model, scripts, and datasets from the Network Drive (FileShare) to the **Local Drive (C:)**.
2.  **Execute:** Run your models and perform all data processing locally on the **C:** drive.
3.  **Backup:** Once finished, copy your results back to the Network Drive for safe storage.

---

## 💰 Optimizing Cost & Performance (Money Saving Tips)

### Rule #1: ZIP Your Files Before Uploading
**Why?** In Azure, you pay for every single "transaction" (interaction with a file).
* **Moving 1,000 small files** = 4,000+ transactions (Requires opening, writing, and closing every single file).
* **Moving 1 Zip file** = ~4 transactions.

**✅ DO:** Compress your folders into a `.zip` or `.tar` archive before moving them to the cloud.
**❌ DON'T:** Upload folders containing thousands of small images or code files directly.

#### 📊 Cost Comparison
| Step | Operation | 1,000 Small Files ❌ | 1 Zip File ✅ |
| :--- | :--- | :--- | :--- |
| **1** | Open/Create | 1,000 IOPS | 1 IOPS |
| **2** | Transfer Data | 1,000+ IOPS | 1 IOPS (Streamed) |
| **3** | Fix Date/Time | 1,000 IOPS | 1 IOPS |
| **4** | Close Handle | 1,000 IOPS | 1 IOPS |
| **TOTAL** | **Billable Events** | **~4,000 Transactions** | **~4 Transactions** |

<details>
<summary><b>🔍 Deep Dive: Why does this happen? (Click to Expand)</b></summary>

### The Technical Explanation
In Azure Files, every interaction is considered a **"Transaction."** A single file copy isn't just one action, it is a conversation between the client and the server.

**Scenario A: Copying 1,000 files (Unzipped)**
1.  **Create Handle:** "Hi Azure, I need to create `file1.txt`" (1 IOPS)
2.  **Write Data:** "Here is the content for `file1.txt`" (1 IOPS)
3.  **Update Attributes:** "Set the timestamp for `file1.txt`" (1 IOPS)
4.  **Close Handle:** "I am done with `file1.txt`" (1 IOPS)
    * *Result:* **~4,000+ IOPS used.** You hit your throttle limit immediately and pay for every interaction.

**Scenario B: Copying 1 Zip file**
1.  **Create Handle:** "Hi Azure, I need to create `archive.zip`" (1 IOPS)
2.  **Write Data:** "Here is the content..." (Streamed continuously, counts as fewer IOPS because it's sustained throughput).
3.  **Close Handle:** "I am done." (1 IOPS)
    * *Result:* **~10-50 IOPS used.** cost effective.

</details>

---

## 📂 Moving & Managing Files (Admins Only)

> [!NOTE]
> **Target Audience:** This section is intended for **System Administrators** and **Data Managers**.
> The following guidelines are critical for managing **large datasets** or high-volume file transfers to prevent unnecessary billing costs.

### How to Move Files *Inside* the File Share (No Cost)
If you need to move files that are already on the Network Drive (e.g., moving from `/data/raw` to `/data/processed`):

✅ **Use Azure Storage Explorer**:
* When you "Move" files within the same share using Azure Storage Explorer, it utilizes a **server-side copy**.
* **Cost:** Free (or negligible). It updates the metadata without actually moving the data bytes back and forth.

❌ **Avoid AzCopy or Windows Explorer for Internal Moves**:
* Tools like standard Windows Explorer (Drag & Drop) or generic copy commands often perform a "Read" (download) followed by a "Write" (upload).
* **Cost:** This incurs double transaction costs and is significantly slower.

### Tools to use for Transfers (Local ↔ Cloud)
**Azure Storage Explorer** (which utilizes AzCopy in the background) or the **AzCopy** CLI are the recommended tools.

* **🚀 Maximum Performance:** AzCopy uses the **Azure Storage REST APIs** directly with multi-threading to saturate your available bandwidth.
* **💰 Cost Effective:** Optimized API usage and concurrency reduce overhead compared to standard file copy protocols.
* **🔄 Resumable:** If your transfer is interrupted (network glitch, sleep mode), AzCopy can resume exactly where it left off.
* **🛡️ Reliability:** It has built-in retry logic and robust error handling, ensuring data integrity for large transfers.
