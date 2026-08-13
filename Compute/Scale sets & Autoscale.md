# Lab 8 - Virtual Machine Scale Sets & autoscale

> **Domain:** Deploy and manage Azure compute resources (20-25%) · **Time:** ~20 min · **Cost:** ~£0.15 (2+ small VMs - delete once done)
>
> Part of my AZ-104 hands-on lab guide. Portal-first, with Azure CLI for speed and teardown.

## Scenario

Megatron's web tier needs to grow when demand increases and shrink back down when it's quiet. You'll deploy a scale set behind a load balancer, then add autoscale rules driven by CPU so the instance count follows demand automatically.

## Exam skills covered

| Skill | Weight |
| --- | --- |
| Deploy and configure a Virtual Machine Scale Set | High |
| Manage sizing and scaling for a scale set | Medium |

> [!NOTE]
> Scale sets run multiple VMs, so this one costs a little more - **delete the resource group** when you're done. If a size isn't available in your region (SkuNotAvailable), swap `--vm-sku` for one that is.

---

## Task 1 - Deploy a scale set

**Portal:** **Virtual machine scale sets → Create.** Ubuntu, size **D2als_v6**, instance count **2**, orchestration = Uniform (simplest for autoscale), load balancing = **Azure load balancer**, spread across zones if offered.

**CLI:**

```bash
RG=rg-vmss-lab
LOC=uksouth
az group create -n $RG -l $LOC
az vmss create -g $RG -n vmss-web --image Ubuntu2204 --vm-sku Standard_D2als_v6 --instance-count 2 --admin-username azureuser --generate-ssh-keys --upgrade-policy-mode automatic --lb vmss-lb
```
<img width="622" height="190" alt="image" src="https://github.com/user-attachments/assets/a191dfa2-5b52-42ae-b69d-c55ba9a37acc" />

---

## Task 2 - Add CPU-based autoscale rules

**Portal:** **Scale set → Scaling → Custom autoscale.** Min **2** / Max **5** / Default **2**. Rule: *scale out +1 when average CPU > 70% for 5 min*; *scale in -1 when average CPU < 30% for 5 min*.

**CLI:**

```bash
az monitor autoscale create -g $RG --resource vmss-web --resource-type Microsoft.Compute/virtualMachineScaleSets --name autoscale-web --min-count 2 --max-count 5 --count 2

az monitor autoscale rule create -g $RG --autoscale-name autoscale-web --condition "Percentage CPU > 70 avg 5m" --scale out 1
az monitor autoscale rule create -g $RG --autoscale-name autoscale-web --condition "Percentage CPU < 30 avg 5m" --scale in 1
```

<img width="699" height="361" alt="image" src="https://github.com/user-attachments/assets/8e099fe3-1dbc-43c2-829c-6b8e1123cee4" />

---

## Task 3 - Manually scale and update the model

**Portal:** bump the instance count manually and watch new instances appear. Change the VM size in the model and apply an upgrade - the **upgrade policy** (Automatic / Rolling / Manual) governs how existing instances pick up model changes.

**CLI:**

```bash
az vmss scale -g $RG -n vmss-web --new-capacity 3
az vmss list-instances -g $RG -n vmss-web -o table
```
<img width="699" height="262" alt="image" src="https://github.com/user-attachments/assets/13f7a60a-0dd9-4cc2-9094-c36b486fdd7e" />

---

## Success criteria

- [ ] A scale set with 2 instances is running behind a load balancer
- [ ] Autoscale rules scale out above 70% CPU and in below 30% CPU, within min/max bounds
- [ ] You manually scaled the set and can explain the upgrade policy (Automatic / Rolling / Manual)

---

## Break & fix - try before you peek

**Autoscale never triggers.** The metric has to breach the threshold for the *full* window (e.g. 5 min), and the min/max must leave room to move (already at max can't scale out). Check the metric source and time grain.

**SkuNotAvailable on create.** The chosen `--vm-sku` has no capacity in your region/zone - pick an available size (`az vm list-skus -l uksouth --size Standard_D --output table`) and retry.

---

## Knowledge check

<details>
<summary><b>When pick a scale set over VMs in an availability set?</b></summary>

Use a **scale set** when instances are **identical and stateless** and you want **elastic, automatic** scaling with simplified management (one model, rolling upgrades, built-in load balancer integration, can span zones). Use discrete VMs in an **availability set** for a small number of **individually-managed** servers that don't need to scale dynamically.
</details>

<details>
<summary><b>Upgrade policy: Automatic vs Rolling vs Manual?</b></summary>

**Automatic** - instances are updated to the new model immediately, no ordering guarantees (brief disruption possible). **Rolling** - updated in batches with health checks between them (safer, gradual). **Manual** - existing instances stay on the old model until *you* trigger their upgrade.
</details>

---

## Cleanup

```bash
az group delete -n rg-vmss-lab --yes --no-wait
```
