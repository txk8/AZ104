# Lab 7 - Virtual machines: disks, sizing, availability & moving

> **Domain:** Deploy and manage Azure compute resources (20-25%) · **Time:** ~45-60 min · **Cost:** ~£0.10 if deallocated/deleted promptly
>
> Part of my AZ-104 hands-on lab guide. Portal-first, with Azure CLI for speed and teardown.

## Scenario

Create a Linux VM for Megatron, attach and grow a data disk, resize the VM, place it for high availability, turn on encryption at host, then move it to another resource group - the everyday VM lifecycle an admin would do.

## Exam skills covered

| Skill | Weight |
| --- | --- |
| Create a virtual machine | High |
| Manage virtual machine disks | High |
| Manage virtual machine sizes | Medium |
| Deploy VMs to availability zones and availability sets | Medium |
| Configure encryption at host | Medium |
| Move a VM to another resource group / subscription / region | Medium |

> [!NOTE]
> VMs bill while running. **Deallocate** (not just "stop" from inside the OS) when you're not using one, and delete the resource group at the end. A stopped-but-allocated VM still costs you (due to the disks).
> Not all SKUs will be available in each region. I tried to create a VM with a SKU of B1s but got an error message as this is not available in the region I'd selected (UK South). This is a SKU I had encountered quite a few times on another lab platform but this was using US regions.

---

## Task 1 - Create a VM into an availability zone

**Portal:** **Virtual machines → Create.** Image = Ubuntu, Size = **B1s**, Availability options = **Availability zone → Zone 1**, SSH public key auth. Pinning to a zone gives you datacentre-level resilience.

**CLI:**

```bash
RG=rg-vm-lab
LOC=uksouth
az group create -n $RG -l $LOC
az vm create -g $RG -n vm-app1 --image Ubuntu2204 --size Standard_D2als_v6 --zone 1 --admin-username azureuser --generate-ssh-keys
```

---


## Task 2 - Attach and expand a data disk

**Portal:** **VM → Disks → Create and attach a new disk** (e.g. 32 GiB, Standard SSD). To resize a disk you must **deallocate** the VM first, then grow it - you can't shrink a managed disk.

**CLI:**

```bash
az vm disk attach -g $RG --vm-name vm-app1 --name data-disk1 --new --size-gb 32 --sku StandardSSD_LRS

# Expand: deallocate, grow, restart
az vm deallocate -g $RG -n vm-app1
az disk update -g $RG -n data-disk1 --size-gb 64
az vm start -g $RG -n vm-app1
```
<img width="278" height="86" alt="image" src="https://github.com/user-attachments/assets/196d55ee-05e6-4fdb-96fb-dd81fe1696bc" />

---

## Task 3 - Resize the VM

**Portal:** **VM → Size → pick a new size → Resize.** If the target size isn't offered on the current host, deallocate first. Available sizes depend on region and zone.

**CLI:**

```bash
az vm list-vm-resize-options -g $RG -n vm-app1 -o table
az vm deallocate -g $RG -n vm-app1
az vm resize -g $RG -n vm-app1 --size Standard_D4als_v6
az vm start -g $RG -n vm-app1
```
<img width="807" height="229" alt="image" src="https://github.com/user-attachments/assets/4608e179-4291-4c75-a096-4b8a9e458142" />

---

## Task 4 - Enable encryption at host

**Portal:** **VM → Disks → Additional settings → Encryption at host.** This encrypts the data on the VM host (temp disk + disk caches) on top of the default platform encryption. The feature must be registered on the subscription once.


<img width="249" height="95" alt="image" src="https://github.com/user-attachments/assets/7be101a8-71ac-4f26-b394-19e3270b7d4d" />

**CLI:**



```bash
# One-time per subscription (registration takes a few minutes to show Registered)
az feature register --namespace Microsoft.Compute --name EncryptionAtHost
az provider register -n Microsoft.Compute

az vm deallocate -g $RG -n vm-app1
az vm update -g $RG -n vm-app1 --set securityProfile.encryptionAtHost=true
az vm start -g $RG -n vm-app1
```

> [!NOTE]
> Three encryption types to keep straight: **SSE** (default - platform-managed encryption of managed disks at rest), **encryption at host** (host-side, covers temp disk + caches), and **Azure Disk Encryption / ADE** (BitLocker/dm-crypt *inside* the guest OS).

---

## Task 5 - Move the VM to another resource group

**Portal:** **VM → Move → Move to another resource group.** Select the VM and its dependent resources (disk, NIC, IP). Moving across *regions* uses Azure Resource Mover / Site Recovery instead.

**CLI:**

```bash
az group create -n rg-vm-moved -l $LOC
VM_ID=$(az vm show -g $RG -n vm-app1 --query id -o tsv)
az resource move --destination-group rg-vm-moved --ids $VM_ID
# You typically must move the dependent resources (disk / NIC / public IP) together
```

---

## Success criteria

- [ ] A VM is deployed pinned to an availability zone
- [ ] A data disk is attached and expanded (after deallocation)
- [ ] The VM was resized to a different SKU
- [ ] Encryption at host is enabled (feature registered)
- [ ] The VM (and dependencies) moved to another resource group

---

## Break & fix - try before you peek

**"Cannot resize" to the size you want.** That SKU isn't on the current host cluster - **deallocate** the VM and retry so Azure can place it where the size exists. If it's still missing, the size may not exist in that region/zone.

**Can't change the availability set or zone after creation.** Both are **fixed at creation and immutable** - to change them you recreate the VM (from the disk/snapshot) in the desired set/zone.

---

## Knowledge check

<details>
<summary><b>Availability set vs availability zone vs scale set?</b></summary>

**Availability set** spreads VMs across fault + update domains within one datacentre (survives rack/maintenance events, 99.95% SLA). **Availability zone** spreads VMs across physically separate datacentres in a region (survives a whole datacentre failing, 99.99% SLA). **Scale set** is a managed group of identical VMs that can autoscale and span zones.
</details>

<details>
<summary><b>Standard SSD vs Premium SSD vs Ultra disk?</b></summary>

**Standard SSD** - cost-effective, modest IOPS (dev/test, light workloads). **Premium SSD** - high, consistent IOPS with an SLA (production). **Ultra disk** - highest, independently tunable IOPS/throughput (demanding DB/transactional work). VM size can also cap disk throughput.
</details>

---

## Cleanup

```bash
az group delete -n rg-vm-lab --yes --no-wait
az group delete -n rg-vm-moved --yes --no-wait
```
