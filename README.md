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
<summary><b>� Deep Dive: Technical Explanation & Pricing (Click to Expand)</b></summary>

### Azure Files Pricing Breakdown

#### 1. Storage (Data at Rest)
This is the cost for simply keeping the files on the disk.
* **Transaction Optimized:** `$0.06 / GB`.
    * *Note:* Expensive for storage. Cost-effective only for workloads with extremely high read/write activity.
* **Hot Tier (Recommended):** `$0.0255 / GB`.
    * *Note:* Cheaper for storage, balanced operational costs. This is the standard tier.

#### 2. Operations (Pay-per-Action)
In the **Hot Tier**, you pay for every 10,000 operations.

| Operation Type | Cost (per 10,000 ops) | Activity |
| :--- | :--- | :--- |
| **Write & List** | **$0.065** | Uploading files, saving data, browsing folders (`ls`). |
| **Read** | **$0.0052** | Downloading or reading file content. |
| **Delete** | **$0.00 (Free)** | Removing files incurs no transaction cost. |

### 🧮 Why does this matter? (The Hidden Cost of SMB)

It is not just about the "Write" operation. The **SMB Protocol** (used by network drives) is "chatty."
For every single file you copy, the computer and the server have a conversation requiring multiple **billable transactions**:

1.  **Open Handle:** "I want to create this file" (1 Transaction)
2.  **Write Data:** "Here is the data" (1 Transaction)
3.  **Set Attributes:** "Set the creation date/time" (1 Transaction)
4.  **Close Handle:** "I am finished" (1 Transaction)

**The Math:**
* **Copying 100,000 files:**
    * 100,000 Writes + 300,000 Overhead operations = **400,000 Transactions**.
    * Cost: ~$2.60 (vs expected $0.65).
* **Copying 1 Zip file:**
    * 1 Write + 3 Overhead operations = **4 Transactions**.
    * Cost: ~$0.00.

**Conclusion:** The protocol overhead triples your cost when handling many small files.

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
