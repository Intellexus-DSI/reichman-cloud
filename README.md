# Reichman University Cloud Workspace

Welcome to the Reichman University Cloud Environment. This guide outlines the critical rules and workflows to ensure high performance and cost efficiency while working with our infrastructure.

---

## ⚠️ Critical Rules: Read Before Starting

### 1. DO NOT Run Models on Network Drives
All network-attached drives are **Azure Files** storage.
* **Risk:** Running high-IOPS operations (heavy reading/writing) directly on these drives will cause significant high costs.
* **Policy:** Treat the network drive as **Shared Storage** (for backups and sharing data between VMs) rather than active workspace.

### 2. The Correct Workflow
To ensure performance and avoid cost spikes, follow this strictly:
1.  **Setup:** Clone your code/scripts from **GitHub** (do not copy code manually).
2.  **Transfer Data:** Copy only necessary datasets/models from the Network Drive to the **Local Machine Storage**.
3.  **Execute:** Run your models and perform all data processing locally on the machine's local storage.
4.  **Backup:** Once finished, copy only essential results back to the Network Drive for retention or sharing.

---
## 💰 Optimizing Cost & Performance (Money Saving Tips)

### Rule #1: ZIP Your Files Before Uploading
**Why?** In Azure, you pay for every single "transaction" (interaction with a file).
* **The "Data Cost" is fixed:** Moving 10GB of data always generates ~10,000 "Write" transactions (because Azure splits data into 1MB chunks), whether it's one file or many.
* **The "Overhead Cost" is variable:** This depends on the *count* of files. Moving 10,000 separate files generates **30,000+ extra** billable operations just to open, attribute, and close them.

**✅ DO:** Compress your folders into a `.zip` or `.tar.gz` archive before moving them to the cloud.  
**❌ DON'T:** Upload folders containing thousands of small images or code files directly.

#### 📊 Cost Comparison (Example: 10GB Data)
| Step | Operation | 10,000 Small Files (1MB each) ❌ | 1 Large Zip File (10GB) ✅ |
| :--- | :--- | :--- | :--- |
| **1** | **Overhead** (Open/Create) | 10,000 Transactions | **1 Transaction** |
| **2** | **Overhead** (Set Metadata) | 10,000 Transactions | **1 Transaction** |
| **3** | **Overhead** (Close File) | 10,000 Transactions | **1 Transaction** |
| **4** | **Data Transfer** (Writes) | ~10,000 Writes (1 per file) | ~10,000 Writes (1 per 1MB chunk) |
| **TOTAL** | **Billable Events** | **~40,000 Transactions** | **~10,003 Transactions** |
| **RESULT** | **Efficiency** | 💸 **4x More Expensive** | 💰 **75% Cost Reduction** |

<details>
<summary>🛠️ How to Compress and Moving Files(Quick Tutorial)</summary>

### 🛠️ How to Compress Files
Use these commands to prepare your data before uploading to the network drive.

#### Compress using Tar (recursive)
This uses `pigz` to compress using all CPU cores (High Speed).
```bash
tar -I pigz -cf name.tar.gz folder_name
```
Extract
```bash
tar -I pigz -xf name.tar.gz
```

### 🚚 Moving Files: `mv` vs `rsync`

When moving data to the network drive, you have two options.

#### ❌ Option 1: The "Move" Command (Not Recommended for Large Files)
The `mv` command moves the file and **automatically deletes** the source folder from your local machine once finished.

* **Risk:** If the transfer fails (internet disconnect/VPN drop), you might lose data or end up with a corrupted file. There is no progress bar.

```bash
# Moves folder and deletes the original from local disk
mv folder_name path_location
```

#### ✅ Option 2: The Rsync Command (Recommended)

**rsync** copies the files safely. Unlike `mv`, it shows a progress bar and ensures the file is fully transferred before you manually delete the original.

**Benefit:** If the connection drops, you can simply run the command again, and it will resume where it left off (if using the `-P` flag). You never risk losing data.

```bash
# 1. Copy safely
rsync -ahP folder_name path_copy

# 2. Verify the file is there, then delete locally
rm -rf folder_Name
```

</details>

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
