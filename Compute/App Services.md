# Lab 11 - App Service: plans, scaling, slots, custom domain & backup

> **Domain:** Deploy and manage Azure compute resources (20-25%) · **Time:** ~50-65 min · **Cost:** ~£0.10-0.30 (needs Standard tier for slots/backup - delete promptly)
>
> Part of my AZ-104 hands-on lab guide. Portal-first, with Azure CLI for speed and teardown.

## Scenario

Megatron wants a web app on App Service: pick a plan, scale it, add a staging deployment slot for zero-downtime releases, bind a custom domain with a managed certificate, configure backup, and integrate it with a VNet.

## Exam skills covered

| Skill | Weight |
| --- | --- |
| Provision an App Service plan | High |
| Configure scaling for an App Service plan | Medium |
| Create an App Service | High |
| Configure certificates and TLS | Medium |
| Map a custom DNS name to an App Service | Medium |
| Configure backup for an App Service | Medium |
| Configure deployment slots | Medium |

> [!NOTE]
> Deployment slots and backup need **Standard (S1) tier or higher** - they're greyed out on Free/Basic. This lab uses S1, which does bill while it exists, so delete the resource group when you're done.

---

## Task 1 - Create a plan and a web app

**Portal:** **App Services → Create.** Runtime = Node, OS = Linux, Plan = create new at **S1** (Standard - needed for slots and backup later).

**CLI:**

```bash
RG=rg-appsvc-lab
LOC=uksouth
APP=webapp$RANDOM
az group create -n $RG -l $LOC
az appservice plan create -g $RG -n plan-lab --sku S1 --is-linux
az webapp create -g $RG -p plan-lab -n $APP --runtime "NODE:20-lts"
```

---

## Task 2 - Scale up (tier) and scale out (instances)

**Portal:** **App Service plan → Scale up** changes the SKU/hardware. **Scale out** adds instances - add an autoscale rule on CPU like the scale-set lab. Remember: **up = bigger box, out = more boxes**.

**CLI:**

```bash
az appservice plan update -g $RG -n plan-lab --sku P1V3        # scale UP (bigger SKU)

az monitor autoscale create -g $RG --resource "$(az appservice plan show -g $RG -n plan-lab --query id -o tsv)" --name plan-autoscale --min-count 1 --max-count 3 --count 1   # scale OUT baseline
```

---

## Task 3 - Add a staging deployment slot and swap

**Portal:** **Web app → Deployment slots → Add slot** named `staging`. Deploy a change to staging, validate, then **Swap** with production for a near-zero-downtime release. Note which settings are **slot-sticky**.

**CLI:**

```bash
az webapp deployment slot create -g $RG -n $APP --slot staging
az webapp deployment slot swap -g $RG -n $APP --slot staging --target-slot production
```

---

## Task 4 - Map a custom domain + TLS

**Portal:** **Custom domains → Add.** Add a CNAME/TXT at your DNS provider to validate ownership, then create a free **App Service Managed Certificate** and bind it (SNI SSL). Set **HTTPS Only** and a minimum TLS version.

**CLI:**

```bash
# After you've added the DNS records at your registrar:
az webapp config hostname add -g $RG --webapp-name $APP --hostname www.yourdomain.com
az webapp update -g $RG -n $APP --https-only true
# Managed cert: az webapp config ssl create / bind (needs the custom domain validated first)
```

> [!NOTE]
> No domain handy? You can still learn the flow - the exam tests that you (1) add + verify the hostname, (2) create/upload a cert, (3) bind it (SNI vs IP-based SSL), and (4) enforce HTTPS-only.

---

## Task 5 - Configure backup and VNet integration

**Portal:** **Backups** (Standard+): point to a storage account, set schedule + retention, run a manual backup. **Networking → VNet integration** lets the app make **outbound** calls into a VNet (e.g. to a private database); a **private endpoint** makes the app reachable **inbound** privately.

**CLI:**

```bash
# Regional VNet integration for outbound access into a delegated subnet
az webapp vnet-integration add -g $RG -n $APP --vnet vnet-app --subnet snet-appsvc
```

---

## Success criteria

- [ ] A web app runs on an S1+ plan
- [ ] You scaled up (SKU) and configured scale-out / autoscale
- [ ] A staging slot exists and you performed a swap
- [ ] HTTPS-only is enforced and you can describe the custom-domain + managed-cert binding flow
- [ ] A backup is configured, and you understand VNet integration (outbound) vs private endpoint (inbound)

---

## Break & fix - try before you peek

**"Add slot" or Backup is greyed out.** Both need **Standard tier or higher** - scale the plan up to S1+ and the options appear.

**App settings didn't move on swap.** Some settings are marked **deployment slot settings (slot-sticky)** and stay with the slot rather than swapping - decide per-setting whether it should be sticky (e.g. a per-environment connection string).

---

## Knowledge check

<details>
<summary><b>Scale up vs scale out for App Service?</b></summary>

**Scale up** = move the *plan* to a bigger/more capable SKU (more CPU/RAM, and features like slots, custom domains, autoscale, VNet integration unlock at higher tiers). **Scale out** = run *more instances* of the same SKU, manually or via autoscale, to handle more load. Up = bigger box; out = more boxes.
</details>

<details>
<summary><b>VNet integration vs private endpoint for a web app?</b></summary>

**VNet integration** handles **outbound** traffic - the app reaches resources *inside* a VNet (private DB, internal API). **Private endpoint** handles **inbound** traffic - it gives the app a private IP so clients reach it privately and you can turn off public access. Opposite directions, often used together.
</details>

---

## Cleanup

```bash
az group delete -n rg-appsvc-lab --yes --no-wait
```
