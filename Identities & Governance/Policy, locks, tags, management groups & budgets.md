# Lab 3 -  Policy, locks, tags & budgets

> **Domain:** Manage Azure identities & governance (20–25%) · **Time:** ~50–60 min · **Cost:** Free (governance); a £20 budget alert costs nothing
>
> Part of my AZ-104 hands-on lab guide. Portal-first, with Azure CLI for speed and teardown.

## Scenario

Megatron's finance team wants guardrails in place before any workloads are introduced. Every resource must be tagged with a cost centre, with nothing deployed outside approved regions, and production resource groups protected from accidental deletion, and an email sent when spend nears £20 (in real life likely more than £20 but it's to show as an example).

## Exam skills covered

| Skill | Weight |
| --- | --- |
| Implement and manage Azure Policy | High |
| Configure resource locks | Medium |
| Apply and manage tags on resources | Medium |
| Manage resource groups and subscriptions | Medium |
| Manage costs with alerts, budgets and Advisor | Medium |
| Configure management groups | Medium |

> [!NOTE]
> This lab runs in the tenant that holds your subscription from Lab 2 and  reuses `mg-megatron`.

---

## Task 1 - Assign policy at management-group scope

You already have `mg-megatron` (Lab 2). Assigning policy there means it cascades to every subscription beneath it - and from there any resource groups in those subscriptions, and then again for all the resources contained within those resource groups.

**Portal:** **Management groups → mg-megatron → Governance / Policy → Assignments.** You'll assign the policies below at this scope (or at subscription scope if you'd rather keep it contained).

**CLI:** This is assigning variables for the next tasks so don't worry about getting no output:

```bash
SUB=$(az account show --query id -o tsv)
MG="/providers/Microsoft.Management/managementGroups/mg-megatron"
```

---

## Task 2 - Assign a built-in policy: allowed locations

**Portal:** **Policy → Assignments → Assign policy.** Scope = your subscription (or `mg-megatron`). Definition = **Allowed locations**. Parameter = **UK South**. Effect = **Deny**. Then try to create a resource group in another region and watch it get blocked.

<img width="267" height="194" alt="image" src="https://github.com/user-attachments/assets/08df86c3-0678-4464-9dc5-7e0e0ad0901f" />

When creating a resource to test this, you get to the Review + Create stage, you should get validation failed. I have setup a VM with Japan West as the region and all of the components (NSG/IP/VNET etc) will generate an error message.

<img width="417" height="451" alt="image" src="https://github.com/user-attachments/assets/6dcc7f77-0950-4ca8-855f-7244eed77c9d" />



**CLI:**

```bash
az policy assignment create --name "allowed-locations" --display-name "Allowed locations: uksouth" --scope "/subscriptions/$SUB" --policy "e56962a6-4747-49cd-b67b-bf8b01975c4c" --params '{ "listOfAllowedLocations": { "value": ["uksouth"] } }'

# Prove it - this should be DENIED
az group create -n rg-should-fail -l northeurope
```

---

## Task 3 - Enforce and inherit tags with Policy

Two built-in tag policies: one **requires** a tag on new resource groups, the other **inherits** that tag down to the resources inside them.

**Portal:** assign **Require a tag on resource groups** (tag name = `CostCenter`) and **Inherit a tag from the resource group** (tag name = `CostCenter`). The inherit one uses a **Modify** effect, so its assignment needs a managed identity and a **remediation task** to fix resources that already exist.

**CLI:** apply a tag directly to a resource group and merge another on:

```bash
az group create -n rg-prod-app -l uksouth --tags CostCenter=FIN-1001 Env=Prod
az tag update --resource-id "$(az group show -n rg-prod-app --query id -o tsv)" --operation merge --tags Owner=kate
```

<img width="511" height="36" alt="image" src="https://github.com/user-attachments/assets/cd8ac8f7-31d9-4b70-86af-28a4e6cc26e8" />

> [!NOTE]
> **Modify** and **DeployIfNotExists** policies only fix things going forward. To fix resources that already exist, run a **remediation task** (Policy → Remediation), and make sure the assignment's managed identity has the role it needs.

---

## Task 4 - Protect a resource group with a lock

**Portal:** **rg-prod-app → Locks → Add** → a **CanNotDelete** lock named `no-delete`. Try to delete the RG and watch it fail. Switch it to **ReadOnly** to see it also block changes.

**CLI:**

```bash
az lock create --name no-delete --lock-type CanNotDelete --resource-group rg-prod-app

# Prove it - this should FAIL while the lock exists
az group delete -n rg-prod-app --yes
```

<img width="345" height="119" alt="image" src="https://github.com/user-attachments/assets/a0928a21-d43f-44a5-8a07-718d5c7a9dda" />

---

## Task 5 - Create a budget with an alert

**Portal:** **Subscription → Cost Management → Budgets → Add.** Amount = **£20**, monthly. Add alert conditions at **50%** and **90%**, email yourself. Then skim **Advisor → Cost** for idle-resource / right-sizing recommendations.

Worth noting that the currency is set when your billing account was created, depending on the country you choose signing up. So you may see $ instead of £.

```bash
# Budgets are done in the portal; this just lists any that exist
az consumption budget list -o table
```

---

## Success criteria

- [ ] A policy is assigned at `mg-megatron` (or subscription) scope
- [ ] Allowed-locations **denies** an RG created outside UK South
- [ ] A require-tag policy is assigned and `rg-prod-app` carries `CostCenter`
- [ ] A **CanNotDelete** lock blocks deletion of `rg-prod-app`
- [ ] A **£20** monthly budget with email alerts exists

---

## Break & fix - try before you peek

**Inherit-tag policy isn't tagging existing resources.** Modify only acts going forward - run a **remediation task** and check its managed identity has the right role.

**Can't delete an RG even after removing the lock.** Look for a lock inherited from the **subscription** or a parent scope, or a Policy **Deny** on delete.

**Allowed-locations didn't block anything.** Check the effect is **Deny** (not Audit) and the assignment's scope actually covers where you're deploying.

---

## Knowledge check

<details>
<summary><b>Policy vs RBAC - what's the difference?</b></summary>

**RBAC** controls *who can do what*. **Azure Policy** controls *what the resources are allowed to be* (regions, SKUs, required tags, enforced settings) - no matter who creates them. You usually need both.
</details>

<details>
<summary><b>Which policy effects need a managed identity and remediation?</b></summary>

**Modify** and **DeployIfNotExists** - because they change or deploy things on your behalf. **Audit** and **Deny** don't; they only report or block.
</details>

---

## Cleanup

```bash
az lock delete --name no-delete --resource-group rg-prod-app
az group delete -n rg-prod-app --yes --no-wait
az policy assignment delete --name allowed-locations --scope "/subscriptions/$SUB"
# Remove the tag policy assignments and the budget from the portal if you're done
```
