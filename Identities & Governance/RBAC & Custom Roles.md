# Lab 2 - Azure RBAC: roles, scope & a custom role
 
> **Domain:** Manage Azure identities & governance (20–25%) · **Time:** ~35–45 min · **Cost:** Free (control-plane only)
>
> Part of my AZ-104 hands-on lab guide. Portal-first, with Azure CLI for speed and teardown.
 
## Scenario
 
Megatron's project team (`project-enigma`) needs least-privilege access: read-only across everything, Contributor on just one resource group, and a custom role that can restart VMs but not delete them. You'll assign roles at three scopes - **management group → subscription → resource group** - to see how inheritance works, then read an assignment back and explain the effective permission.
 
## Exam skills covered
 
| Skill | Weight |
| --- | --- |
| Manage built-in Azure roles | High |
| Assign roles at different scopes | High |
| Interpret access assignments | High |
 
> [!IMPORTANT]
> RBAC assigns a **directory identity** to an **Azure scope**, so the assignee must live in the **same tenant as your subscription** - I mention as I used a different tenant for the free trial on Entra P2.
 
> [!NOTE]
> MG-scope actions need **Owner** or **User Access Administrator**. If the first attempt fails, elevate once: **Entra ID → Properties → Access management for Azure resources → Yes**, then sign out/in.
 
## The scope model
 
A role assigned at any level **inherits down** to everything beneath it. Permissions from different assignments **stack** - you get everything they grant put together - unless a **deny assignment** blocks something.
 
```
mg-megatron  (management group)      assign: Reader -> project-enigma
   |
   v  (inherits down)
subscription
   |
   v
rg-rbac-lab  (resource group)        assign: Contributor + custom role -> project-enigma
   |
   v
resources    (VMs, storage, any individual resource)
```
 
So `project-enigma` ends up **Reader everywhere** (from the MG) and **Contributor on `rg-rbac-lab`** (direct) - on that one RG the two stack, leaving it with Contributor.
 
---
 
## Task 1 - Prep: a management group
 
**Portal:** **Management groups → Create** → `mg-megatron`. Then move your subscription under it (**mg-megatron → Subscriptions → Add**).

<img width="1034" height="140" alt="image" src="https://github.com/user-attachments/assets/9cc4d5fe-7be3-4d81-82c2-6a6e7eb034cc" />

 
**CLI:**
 
```bash
az account management-group create --name mg-megatron --display-name "Megatron"
SUB=$(az account show --query id -o tsv)
az account management-group subscription add --name mg-megatron --subscription "$SUB"
```
 
---
 
## Task 2 - Assign a built-in role at management-group scope
 
**Portal:** **mg-megatron → Access control (IAM) → Add → Add role assignment** → role **Reader** → assign to **project-enigma**. Note the **Scope** shown on the assignment - it's the whole management group.

<img width="463" height="178" alt="image" src="https://github.com/user-attachments/assets/9d503861-6a28-4f4b-827d-c375183df8fd" />

 
**CLI:**
 
```bash
GID=$(az ad group show --group "project-enigma" --query id -o tsv)
az role assignment create --assignee-object-id "$GID" --assignee-principal-type Group --role "Reader" --scope "/providers/Microsoft.Management/managementGroups/mg-megatron"
```
 
`project-enigma` can now **read every subscription, resource group and resource beneath `mg-megatron`** - one assignment covers the whole hierarchy. That's scope inheritance in action.
 
> [!TIP]
> No curly braces around the scope. `{...}` in docs is a placeholder; a real scope is literal - `/providers/Microsoft.Management/managementGroups/mg-megatron`, never `.../managementGroups/{mg-megatron}`.
 
---
 
## Task 3 - Assign Contributor at resource-group scope
 
**Portal:** create `rg-rbac-lab`, open its **Access control (IAM)** blade, and grant **project-enigma** the **Contributor** role scoped to that RG only.
 
**CLI:**
 
```bash
az group create -n rg-rbac-lab -l uksouth
RG_ID=$(az group show -n rg-rbac-lab --query id -o tsv)
az role assignment create --assignee-object-id "$GID" --assignee-principal-type Group --role "Contributor" --scope "$RG_ID"
```
 
Now the group is **Reader everywhere** (inherited from the MG) *and* **Contributor on this one RG** (direct). On `rg-rbac-lab` the two stack, so it effectively has Contributor.

 <img width="584" height="267" alt="image" src="https://github.com/user-attachments/assets/9d20debe-692a-4fda-b61f-f4bed9fa9134" />

---
 
## Task 4 - Author and assign a custom role
 
**Portal:** **Subscription → IAM → Add → Add custom role → Start from JSON**, upload the file below, then assign the new role at `rg-rbac-lab`.
 
Save this as `vm-operator.json` (replace the subscription ID - get it from the portal or with `az account show --query id -o tsv`):
 
```json
{
  "Name": "VM Operator (restart only)",
  "Description": "Start/restart/stop VMs but not create or delete them.",
  "Actions": [
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/deallocate/action"
  ],
  "NotActions": [],
  "AssignableScopes": [ "/subscriptions/<YOUR_SUBSCRIPTION_ID>" ]
}
```
 
**CLI:**
 
```bash
az role definition create --role-definition vm-operator.json
az role assignment create --assignee-object-id "$GID" --assignee-principal-type Group --role "VM Operator (restart only)" --scope "$(az group show -n rg-rbac-lab --query id -o tsv)"
```

 
 <img width="537" height="168" alt="image" src="https://github.com/user-attachments/assets/4cade142-19fd-40fb-94c0-fd164776faf9" />

> [!NOTE]
> `AssignableScopes` sets **where the role can be assigned**, not who gets it - and can't be broader than your own authority.
 
---
 
## Task 5 - Interpret the access
 
**Portal:** **IAM → Role assignments** and **IAM → Check access**. Pick `project-enigma` and read off each row: which **role**, what **scope**, and whether it's **inherited** (shown against the MG) or **assigned here** (shown against the RG). **View my access** shows effective permissions for the signed-in user.


 
**CLI:**
 
```bash
# Everything assigned to the group, with scope and how it got there
az role assignment list --assignee "$GID" --all -o table
 
# What applies specifically on the lab resource group?
az role assignment list --assignee "$GID" --scope "$(az group show -n rg-rbac-lab --query id -o tsv)" -o table
```
 
Read the **scope** column: `Reader` shows the management-group scope (inherited), while `Contributor` and the custom role show the resource-group scope (direct).
 
---
 
## Success criteria
 
- [ ] `mg-megatron` exists with your subscription placed under it
- [ ] `project-enigma` is **Reader at management-group scope** (inherits to every subscription/RG/resource below)
- [ ] `project-enigma` is **Contributor on `rg-rbac-lab` only**
- [ ] A custom role **"VM Operator (restart only)"** exists and is assigned at `rg-rbac-lab`
- [ ] You can read any assignment and state its **role + scope + inherited vs direct**
---
 
## Break & fix - try before you peek
 
**"Client does not have authorization" at MG scope.** You need Owner or User Access Administrator there - elevate access, then retry.
 
**Assignee not found.** The group must be in the same tenant as the subscription; recreate `project-enigma` here if it's in the P2 tenant.
 
**`InvalidScope`.** Curly braces again - scopes are literal, never `{ }`.
 
**Can read but not act.** Permissions stack, so if something's still blocked look for a **deny assignment** - a deny beats any allow.
 
---
 
## Knowledge check
 
<details>
<summary><b>How are permissions worked out when several roles apply?</b></summary>
They stack - you get everything the assignments grant put together. The one exception: a **deny assignment** always overrides an allow.
</details>
<details>
<summary><b>Actions vs DataActions vs NotActions?</b></summary>
**Actions** = management operations. **DataActions** = data operations (e.g. read a blob's contents). **NotActions/NotDataActions** are subtracted from what the role grants - exclusions, not denies.
</details>
<details>
<summary><b>Why assign at management-group scope?</b></summary>
It applies to every subscription, resource group and resource beneath it - including ones you add later - so you grant access once instead of per subscription.
</details>
---
 
## Cleanup
 
```bash
# Remove the resource group and custom role
az group delete -n rg-rbac-lab --yes --no-wait
az role definition delete --name "VM Operator (restart only)"
 
# Remove the management-group Reader assignment
az role assignment delete --assignee "$GID" --role "Reader" --scope "/providers/Microsoft.Management/managementGroups/mg-megatron"
 
# Optional: move the subscription back to the tenant root and delete mg-megatron in the portal
```
 
---
