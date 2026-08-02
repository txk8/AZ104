# Lab 4 - Storage accounts: redundancy, keys, SAS & stored access policies
 
> **Domain:** Implement and manage storage (15–20%) · **Time:** ~45–60 min · **Cost:** ~£0.05 (a few MB of blobs)
>
> Part of my AZ-104 hands-on lab guide. Portal-first, with Azure CLI for speed and teardown.
 
## Scenario
 
A 3rd party needs time-limited, read-only access to one container of Megatron's data - without ever being handed the account keys. We'll create a storage account, understand its redundancy and encryption, then hand out scoped access with a SAS token governed by a stored access policy you can revoke instantly.
 
## Exam skills covered
 
| Skill | Weight |
| --- | --- |
| Create and configure storage accounts | High |
| Configure Azure Storage redundancy | Medium |
| Configure storage account encryption | Medium |
| Manage access keys | Medium |
| Create and use SAS tokens | High |
| Configure stored access policies | Medium |
 
---
 
## Task 1 — Create a storage account and choose redundancy
 
**Portal:** **Storage accounts → Create.** Performance = Standard, Redundancy = **LRS** (cheapest for a lab, in real life pick the most suitable option). On the **Encryption** tab, note it's encrypted at rest by default with Microsoft-managed keys. Names are **globally unique, lowercase, no dashes** Globally unique means it doesn't matter if you have used the name for a storage account in a different tenant, it must be unique everywhere (all tenants, so "global"). Worth knowing this for tenant to tenant migrations as the names cannot remain the same.

**Redundancy:
-Refers to how many copies of your data Azure stores (and where). The trade-off is between cost versus resilience. LRS is the cheapest but offers the least resilience. You set the redundancy when you create a storage account - sometimes it can be changed depending on the option.
Changing between LRS and GRS is simple and easy to do in the portal. However, changing from ZRS means conversion or migration of the storage account.**
 
**CLI:**
 
```bash
RG=rg-storage-lab
LOC=uksouth
SA=stmegatron$RANDOM   # must be globally unique, 3-24 lowercase letters/numbers
az group create -n $RG -l $LOC
az storage account create -n $SA -g $RG -l $LOC --sku Standard_LRS --kind StorageV2 --min-tls-version TLS1_2 --allow-blob-public-access false
```
 
---
 
## Task 2 — Inspect and rotate access keys
 
**Portal:** **Storage account → Security + networking → Access keys.** See key1/key2 and their connection strings. Click **Rotate key1** — the two-key design lets you rotate one while apps keep using the other, so there's no downtime.

<img width="490" height="268" alt="image" src="https://github.com/user-attachments/assets/3336c853-c829-454d-a62e-4d1334cd52dd" />

 
**CLI:**
 
```bash
az storage account keys list -n $SA -g $RG -o table
az storage account keys renew -n $SA -g $RG --key key1
```
 
---
 
## Task 3 — Create a container and upload a blob
 
**Portal:** **Containers → + Container** named `partner-data` (private). Upload any small file.
 
**CLI:**
 
```bash
KEY=$(az storage account keys list -n $SA -g $RG --query "[0].value" -o tsv)
az storage container create -n partner-data --account-name $SA --account-key $KEY
echo "hello az104" > sample.txt
az storage blob upload --account-name $SA --account-key $KEY -c partner-data -f sample.txt -n sample.txt
```
 
---
 
## Task 4 — Create a stored access policy, then a SAS bound to it
 
**Portal:** **Container → Access policy → + Add policy** → name `partner-ro`, permissions **Read, List**, set an expiry. Then **Shared access tokens** tab → pick the stored policy → **Generate SAS**. Because the SAS points at the policy, you can revoke it instantly by editing or deleting the policy.
 
**CLI:**
 
```bash
EXPIRY="2026-12-31T23:59Z"   # edit as needed
az storage container policy create --container-name partner-data --name partner-ro --permissions rl --expiry $EXPIRY --account-name $SA --account-key $KEY
 
# Service SAS that inherits the policy (no permissions/expiry on the SAS itself)
SAS=$(az storage container generate-sas -n partner-data --policy-name partner-ro --account-name $SA --account-key $KEY -o tsv)
echo "https://$SA.blob.core.windows.net/partner-data/sample.txt?$SAS"
```

<img width="474" height="116" alt="image" src="https://github.com/user-attachments/assets/c128839a-2582-4c57-ad4d-12733534dd31" />

 
> [!NOTE]
> Three SAS types: **user delegation** (signed by Entra ID — most secure, no account key), **service** (one service, can be tied to a stored access policy), **account** (broadest). Prefer user-delegation or a policy-backed service SAS so you can revoke.
 
---
 
## Task 5 — Revoke access by deleting the policy
 
**Portal:** delete the `partner-ro` access policy. The SAS you issued stops working immediately - that's the whole point of stored access policies over ad-hoc SAS.
 
**CLI:**
 
```bash
az storage container policy delete --container-name partner-data --name partner-ro --account-name $SA --account-key $KEY
# Re-open the SAS URL from Task 4 -> now 403
```

<img width="555" height="197" alt="image" src="https://github.com/user-attachments/assets/65e4b7bb-e455-49f0-a5d4-fd377c12d893" />

---
 
## Success criteria
 
- [ ] A StorageV2 LRS account exists with TLS 1.2 minimum and public blob access disabled
- [ ] You rotated an access key and can explain the two-key pattern
- [ ] A stored access policy (Read/List) governs a service SAS for `partner-data`
- [ ] Deleting the policy revokes the SAS (verified 403)
---
 
## Break & fix - try before you peek
 
**SAS returns AuthenticationFailed straight away.** Usually clock skew (omit the start time or set it a few minutes in the past), wrong permissions, or the account firewall blocking your IP.
 
**An anonymous URL works when it shouldn't.** The container access level is public — set it to Private and keep **allow blob public access** disabled on the account.
 
---
 
## Knowledge check
 
<details>
<summary><b>LRS vs ZRS vs GRS vs GZRS?</b></summary>
- **LRS** — 3 copies in one datacentre (cheapest; protects against disk/rack failure).
- **ZRS** — 3 copies across availability zones in the region (survives a datacentre/zone loss).
- **GRS** — LRS locally + async copies in a paired region (survives a regional outage).
- **GZRS** — ZRS locally + a copy in the paired region (zone *and* region protection).
Add **RA-** (RA-GRS/RA-GZRS) for read access to the secondary at any time.
</details>
<details>
<summary><b>Why use a stored access policy instead of an ad-hoc SAS?</b></summary>
An ad-hoc SAS bakes its permissions and expiry into the token, so revoking it early means rotating the account key (which breaks everything). A stored access policy holds those rules on the server, so the SAS just references it — you can change or revoke access by editing/deleting the policy, no key rotation needed.
</details>
---
 
## Cleanup
 
```bash
az group delete -n rg-storage-lab --yes --no-wait
```
 
