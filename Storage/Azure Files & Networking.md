# Lab 6 - Azure Files, storage firewalls, endpoints & object replication

> **Domain:** Implement and manage storage (15-20%) · **Time:** ~45-60 min · **Cost:** ~£0.10
>
> Part of my AZ-104 hands-on lab guide. Portal-first, with Azure CLI for speed and teardown.

## Scenario

Megatron wants to retire an on-prem file server for an Azure file share, and lock the storage account down so only its VNet can reach it, and set up to replicate blob data to a second region for resilience.

## Exam skills covered

| Skill | Weight |
| --- | --- |
| Create and configure a file share in Azure Files | High |
| Configure identity-based access for Azure Files | Medium |
| Configure snapshots and soft delete for Azure Files | Medium |
| Configure Azure Storage firewalls and virtual networks | Medium |
| Configure object replication | Medium |

---

## Task 1 - Create a file share and mount it

**Portal:** **Click on your Storage account → File shares → + File share** named `company-share`. Use **Connect** to get the ready-made mount script for Windows/Linux/macOS.

<img width="177" height="60" alt="image" src="https://github.com/user-attachments/assets/431fed56-d9fb-4e01-b2f4-f2758e593ebe" />

**CLI:**

```bash
RG=rg-files-lab
LOC=uksouth
SA=stfiles$RANDOM
az group create -n $RG -l $LOC
az storage account create -n $SA -g $RG -l $LOC --sku Standard_LRS --kind StorageV2 --min-tls-version TLS1_2
az storage share-rm create --storage-account $SA -g $RG --name company-share --quota 100
```

> [!NOTE]
> SMB uses **port 445**, which many home networks block outbound - so a mount can fail from your laptop even when everything's configured right. You can test this from Cloud Shell or a VM inside Azure, or reach it privately over VPN / a private endpoint.

---

## Task 2 - Snapshots and soft delete for Files

**Portal:** on the share, take a **snapshot** (point-in-time copy). On **Data protection**, confirm **soft delete for file shares** is on (it should be already if you used the above cli for 7 days), so a deleted share is recoverable.

<img width="432" height="196" alt="image" src="https://github.com/user-attachments/assets/287e9ab6-4982-47d2-919c-445d9ada6b89" />

<img width="839" height="150" alt="image" src="https://github.com/user-attachments/assets/679c7c57-bd14-421c-b6ee-82ad09bbfea8" />

**CLI:**

```bash
KEY=$(az storage account keys list -n $SA -g $RG --query "[0].value" -o tsv)
az storage share snapshot --account-name $SA --account-key $KEY --name company-share
az storage account file-service-properties update -n $SA -g $RG --enable-delete-retention true --delete-retention-days 7
```

---

## Task 3 - Identity-based access (concept)

**Portal:** **File share settings / Identity-based access.** Azure Files supports three identity sources for SMB auth: **on-prem AD DS**, **Microsoft Entra Domain Services**, and **Microsoft Entra Kerberos** (for hybrid/cloud identities). Enable the source that matches your environment, then set access with the two-layer model.

The two layers to remember:

- **Share level** - Azure RBAC roles (`Storage File Data SMB Share Reader / Contributor / Elevated Contributor`).
- **File/folder level** - standard Windows **NTFS** permissions (ACLs).

  <img width="773" height="479" alt="image" src="https://github.com/user-attachments/assets/10a65805-f952-4b25-81f6-51e62d718777" />


> [!NOTE]
> On a personal/lab tenant this may be more hassle than it's worth to configure AD DS or Entra DS to set this up fully - the important part for the exam is knowing the **two-layer model** (RBAC for the share, NTFS for files inside it) and the three identity sources.

---

## Task 4 - Lock the account behind a storage firewall + service endpoint

**Portal:** **Networking → Firewalls and virtual networks** → switch to **Enabled from selected virtual networks and IP addresses**, add your VNet/subnet (this turns on the `Microsoft.Storage` **service endpoint** on that subnet), and add your **client IP** so you don't lock yourself out. Leave **Allow trusted Microsoft services** ticked.

**CLI:**

```bash
az network vnet create -g $RG -n vnet-files --address-prefix 10.20.0.0/16 --subnet-name snet-app --subnet-prefix 10.20.1.0/24
az network vnet subnet update -g $RG --vnet-name vnet-files -n snet-app --service-endpoints Microsoft.Storage
SUBNET_ID=$(az network vnet subnet show -g $RG --vnet-name vnet-files -n snet-app --query id -o tsv)
az storage account network-rule add -g $RG --account-name $SA --subnet $SUBNET_ID
az storage account update -g $RG -n $SA --default-action Deny
```


<img width="506" height="430" alt="image" src="https://github.com/user-attachments/assets/18e2d755-817c-4e3b-916d-0f7d2cd10ce6" />

---

## Task 5 - Object replication (blobs) to a second account

Object replication copies **block blobs** from a *source* account to a *destination* account (often another region). Both accounts need **versioning**, and the source also needs the **change feed** enabled.

**Portal:** create a destination account (e.g. in `ukwest`), then **Object replication → Set up replication rules**, mapping a source container to a destination container.

> [!NOTE]
> Don't confuse **object replication** (async, container-to-container, your own rule) with **account redundancy** (GRS/GZRS - a platform-managed copy of the whole account). Redundancy is automatic, and object replication is a rule you define.

---

## Success criteria

- [ ] A 100 GiB `company-share` exists and you took a snapshot
- [ ] Soft delete for file shares is enabled
- [ ] You can explain the share-level (RBAC) + file-level (NTFS) permission model
- [ ] The account firewall denies by default and only allows your VNet subnet (service endpoint on)
- [ ] You can describe object replication's prerequisites (versioning + change feed)

---

## Break & fix - try before you peek

**Locked out of the portal data view after enabling the firewall.** "Selected networks" blocks your browser too - add your **client IP** to the allow list. "Allow trusted Microsoft services" does not cover your laptop.

**Mounting the share fails on a home network.** SMB port **445** is blocked outbound - mount from Cloud Shell or an Azure VM, or use VPN / a private endpoint.

**Object replication isn't copying.** Source and destination both need **versioning**, and the source needs **change feed**. New rules only replicate *new* writes unless you opt to copy existing data.

---

## Knowledge check

<details>
<summary><b>Service endpoint vs private endpoint for a storage account?</b></summary>

**Service endpoint** keeps subnet traffic on the Azure backbone and lets the storage firewall trust that subnet - but the account keeps its **public IP/DNS**. **Private endpoint** gives the account a **private IP inside your VNet** (via Private Link), reachable privately including from on-prem over VPN/ExpressRoute, and lets you switch public access off entirely.
</details>

<details>
<summary><b>Which identity sources can Azure Files use for SMB, and how do permissions work?</b></summary>

Three sources: **on-prem AD DS**, **Microsoft Entra Domain Services**, and **Microsoft Entra Kerberos** (hybrid identities). Access is two layers: **share-level** via Azure RBAC roles, then **file/folder-level** via standard NTFS ACLs.
</details>

---

## Cleanup

```bash
az group delete -n rg-files-lab --yes --no-wait
# Delete the object-replication destination account/RG too if you made one
```
