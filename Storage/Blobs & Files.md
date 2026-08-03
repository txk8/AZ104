# Lab 5 - Blob Storage: tiers, lifecycle, versioning & soft delete

> **Domain:** Implement and manage storage (15-20%) · **Time:** ~40-55 min · **Cost:** ~£0.05
>
> Part of my AZ-104 hands-on lab guide. Portal-first, with Azure CLI for speed and teardown. Builds on the storage account from Lab 4 (or make a fresh one below).

Meet Mr Blobby, our test blob for this lab 
<img width="1038" height="500" alt="image" src="https://github.com/user-attachments/assets/906c6050-daed-488d-8b0c-911707747c9c" />


*Meet Mr Blobby - the star blob for this lab. (The real Blobby belongs to the BBC.)*

## Scenario

Megatron's logs are piling up and a couple of accidental deletes have already bitten. You'll tier the data to cut cost, add a lifecycle rule so old logs age out automatically, turn on the data-protection options (soft delete + versioning), and move data with AzCopy. Our guinea-pig blob for the tiering and delete demos is a blobby character: **Mr Blobby**.

## Exam skills covered

| Skill | Weight |
| --- | --- |
| Configure storage tiers | High |
| Configure blob lifecycle management | High |
| Configure blob versioning | Medium |
| Configure soft delete for blobs and containers | Medium |
| Manage data with AzCopy and Storage Explorer | Medium |

---

## Task 1 - Set access tiers on a blob

**Portal:** upload a blob, then **blob → Change tier** and move it between **Hot**, **Cool**, **Cold** and **Archive**. Note that Archive is offline - reading it needs a **rehydrate** first (hours). If you can't see change tier, make sure you are on the blob itself, it doesn't appear on the storage account or container page.

<img width="321" height="243" alt="image" src="https://github.com/user-attachments/assets/4aad2e73-cd13-4e06-ac7f-677d36b59f7a" />

Sending Mr Blobby to **Archive** is basically putting him into cold storage - he can't come out (be read) until you rehydrate him, which is very on-brand.

**CLI:**

```bash
RG=rg-blob-lab
LOC=uksouth
SA=stblob$RANDOM
az group create -n $RG -l $LOC
az storage account create -n $SA -g $RG -l $LOC --sku Standard_LRS --kind StorageV2 --min-tls-version TLS1_2 --allow-blob-public-access false
KEY=$(az storage account keys list -n $SA -g $RG --query "[0].value" -o tsv)
az storage container create -n logs --account-name $SA --account-key $KEY
echo "Blobby! Blobby! Blobby!" > mr-blobby.txt
az storage blob upload --account-name $SA --account-key $KEY -c logs -f mr-blobby.txt -n mr-blobby.txt --tier Cool
az storage blob set-tier --account-name $SA --account-key $KEY -c logs -n mr-blobby.txt --tier Archive
```

---

## Task 2 - Enable soft delete and versioning

**Portal:** **Data protection** blade → tick **soft delete for blobs** (e.g. 7 days), **soft delete for containers**, and **versioning for blobs**. Delete Mr Blobby, then restore him from the **Show deleted blobs** view - he bursts straight back out of the recycle bin.

<img width="556" height="449" alt="image" src="https://github.com/user-attachments/assets/7d501fbb-9910-4930-8d8b-911a1fd76ca9" />

Make sure you change the toggle on the top right hand side to show active and deleted blobs too, or else the blob you just deleted won't appear.

<img width="203" height="83" alt="image" src="https://github.com/user-attachments/assets/4b5a11df-27ae-401c-a168-8ba4ed6f975d" />

Soft delete reminds me of the recycle bin on windows - although things get deleted when the retention period is hit rather than when the bin is full, but the idea is similar.

<img width="91" height="74" alt="image" src="https://github.com/user-attachments/assets/af2c19fa-f9f1-43ee-b14f-41634afc089c" />

**CLI:**

```bash
az storage account blob-service-properties update -n $SA -g $RG --enable-delete-retention true --delete-retention-days 7 --enable-container-delete-retention true --container-delete-retention-days 7 --enable-versioning true
```

<img width="278" height="70" alt="image" src="https://github.com/user-attachments/assets/08263b0d-9283-4192-be94-bf73460a19c6" />

---

## Task 3 - Author a lifecycle management rule

**Portal:** **Lifecycle management → Add rule.** Condition: base blobs *last modified > 30 days → Cool*, *> 90 days → Archive*, *> 365 days → delete*. Scope it to the `logs` container by prefix. (After 365 days, Mr Blobby's contract finally runs out and he's deleted automatically.)

<img width="249" height="447" alt="image" src="https://github.com/user-attachments/assets/164245a0-137d-4b25-88cd-6828321a7d05" />

**CLI:** save this as `lifecycle.json`:

```json
{
  "rules": [
    {
      "enabled": true,
      "name": "logs-tiering",
      "type": "Lifecycle",
      "definition": {
        "filters": { "blobTypes": ["blockBlob"], "prefixMatch": ["logs/"] },
        "actions": {
          "baseBlob": {
            "tierToCool":    { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
            "delete":        { "daysAfterModificationGreaterThan": 365 }
          }
        }
      }
    }
  ]
}
```

Then apply it:

```bash
az storage account management-policy create --account-name $SA -g $RG --policy @lifecycle.json
```

---

## Task 4 - Move data with AzCopy

**Portal:** Storage Explorer (in-portal or the desktop app) gives the same copy/move operations with a GUI - worth knowing both.

**CLI:**

```bash
# Entra login - no keys needed if you have the Storage Blob Data Contributor role
azcopy login
azcopy copy "./localfolder/*" "https://$SA.blob.core.windows.net/logs/" --recursive

# Or append a SAS to the URL instead of logging in:
# azcopy copy "./localfolder/*" "https://$SA.blob.core.windows.net/logs?<SAS>" --recursive
```

---

## Success criteria

- [ ] Mr Blobby has been moved through Hot / Cool / Archive tiers
- [ ] Soft delete (blobs + containers) and versioning are enabled, and you restored a deleted blob
- [ ] A lifecycle rule auto-tiers and expires blobs in the `logs` container
- [ ] You copied data in with AzCopy (login or SAS)

---

## Break & fix - try before you peek

**Can't read an archived blob.** Archive is offline - you must **rehydrate** it to Hot/Cool first (can take hours). Keep hot-path data out of Archive.

**Lifecycle rule "did nothing".** It runs on a daily schedule (not instantly) and acts on **last-modified age**, so a fresh blob won't match a ">30 days" rule yet. Also check the prefix filter matches the container path exactly.

---

## Knowledge check

<details>
<summary><b>Hot vs Cool vs Cold vs Archive - the trade-off?</b></summary>

Moving down the tiers lowers **storage** cost but raises **access/retrieval** cost and latency, plus minimum-retention penalties. **Hot** = frequent access; **Cool** = infrequent (≥30 days); **Cold** = rare (≥90 days); **Archive** = offline (≥180 days, must rehydrate to read). Match the tier to how often the data is actually read.
</details>

<details>
<summary><b>Soft delete vs versioning vs snapshots?</b></summary>

**Soft delete** = a recycle bin: deleted blobs/containers are recoverable for a retention window. **Versioning** = a new version is saved automatically on every write, so you can roll back. **Snapshots** = manual point-in-time copies you take yourself. Versioning is the hands-off protection; snapshots are deliberate checkpoints.
</details>

---

## Cleanup

```bash
az group delete -n rg-blob-lab --yes --no-wait
```
