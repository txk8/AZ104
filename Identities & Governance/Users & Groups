# Lab 1 - Entra ID users, groups, guests & SSPR
 
> **Domain:** Manage Azure identities & governance (20–25%) · **Time:** ~40 min · **Cost:** Free (Entra operations); licences optional
>
> Part of my AZ-104 hands-on lab guide. Portal-first, with Azure CLI for speed and teardown.
 
## Scenario
 
Megatron is onboarding a new project team plus an external auditor. You need staff accounts, a dynamic group that auto-populates by department, a security group for the project, a guest invite for the auditor, and self-service password reset so the help desk stops resetting passwords.
 
## Exam skills covered
 
| Skill | Weight |
| --- | --- |
| Create users and groups | High |
| Manage user and group properties | Medium |
| Manage licences in Microsoft Entra ID | Medium |
| Manage external users | Medium |
| Configure self-service password reset (SSPR) | Medium |
 
> [!IMPORTANT]
> Dynamic groups, group-based licensing, and SSPR write-back need **Entra ID P1**; Conditional Access / PIM / Identity Protection need **P2**. A pay-as-you-go tenant can start a free P1/P2 trial — activate it **from inside the tenant your users live in** (Microsoft 365 admin center → Billing → Purchase services), not via a fresh signup (that spins up a *separate* empty tenant).
 
---
 
## Task 1 — Create the users (Ada Lovelace & Grace Hopper)
 
**Portal:** Microsoft Entra ID → **Users → New user → Create new user**. Set a UPN, name, and an initial password. On the **Properties** tab set **Department = Engineering** for *both* users (you'll target this with a dynamic group next). Give one a Job title too, e.g. *Network Engineer*.
 
**CLI:**
 
> [!WARNING]
> `az ad user create` does **not** accept `--department` or `--job-title` — those aren't parameters of the command (you'll get `unrecognized arguments: --department`). Create the user first, then set those properties through Microsoft Graph with `az rest`. Commands below are single-line on purpose so they paste cleanly in **any** shell (Bash or PowerShell).
 
```bash
# Get your verified *.onmicrosoft.com (initial/default) domain
DOMAIN=$(az rest --method get --url "https://graph.microsoft.com/v1.0/domains" --query "value[?isDefault].id" -o tsv)
 
# Create the two users (no --department here — it isn't a valid flag)
az ad user create --display-name "Ada Lovelace" --user-principal-name "ada@$DOMAIN" --password "Lab-Passw0rd!2026"
az ad user create --display-name "Grace Hopper" --user-principal-name "grace@$DOMAIN" --password "Lab-Passw0rd!2026"
 
# Set department + job title via Microsoft Graph (the az wrapper doesn't expose these)
az rest --method PATCH --url "https://graph.microsoft.com/v1.0/users/ada@$DOMAIN" --headers "Content-Type=application/json" --body '{"department":"Engineering","jobTitle":"Network Engineer"}'
az rest --method PATCH --url "https://graph.microsoft.com/v1.0/users/grace@$DOMAIN" --headers "Content-Type=application/json" --body '{"department":"Engineering","jobTitle":"Compiler Engineer"}'
 
# Verify
az ad user show --id "ada@$DOMAIN" --query "{name:displayName, dept:department, title:jobTitle}" -o table
```
 
> [!TIP]
> `{something}` or `<something>` in Azure docs/commands is a fill-in-the-blank placeholder — replace the whole token **including the brackets** with your value. A subscription ID is `/subscriptions/xxxx-...`, never `/subscriptions/{xxxx-...}`.
 
---
 
## Task 2 — Create an assigned group and a dynamic group
 
**Portal:** **Groups → New group.**
 
1. **Assigned:** type *Security*, membership *Assigned*, name `project-enigma`, add Ada and Grace manually.
2. **Dynamic:** New group again, **Membership type = Dynamic User**, add a rule: `user.department -eq "Engineering"`. Save and watch it auto-populate (can take a minute).
> [!NOTE]
> The dynamic-rule builder is **only** in the Entra admin center (or Azure portal), *not* the M365 admin center. You can also convert an existing assigned group: open it → **Properties → Membership type → Dynamic User → Add dynamic query**. Converting hands membership control to the rule — manual members who don't match get removed.
 
**CLI:**
 
```bash
# Assigned security group
az ad group create --display-name "project-enigma" --mail-nickname "project-enigma"
GID=$(az ad group show --group "project-enigma" --query id -o tsv)
 
# Add both members by object id
AID=$(az ad user show --id "ada@$DOMAIN" --query id -o tsv)
HID=$(az ad user show --id "grace@$DOMAIN" --query id -o tsv)
az ad group member add --group $GID --member-id $AID
az ad group member add --group $GID --member-id $HID
 
# NOTE: the dynamic membershipRule is set in the portal / Graph, not via a first-class
# 'az ad group' flag. Use the Entra admin center for the dynamic rule.
```
 
---
 
## Task 3 — Invite an external (guest) user
 
**Portal:** **Users → New user → Invite external user.** Enter an email you control, add a personal message, and optionally add them straight into `project-enigma`. Accept the invite from that mailbox to see the redemption flow and the `#EXT#` UPN that gets created.
 
**CLI:**
 
```bash
# B2B guest invite via Microsoft Graph
az rest --method post --url "https://graph.microsoft.com/v1.0/invitations" --headers "Content-Type=application/json" --body '{ "invitedUserEmailAddress": "auditor@example.com", "inviteRedirectUrl": "https://myapps.microsoft.com", "sendInvitationMessage": true }'
```
 
---
 
## Task 4 — Manage licences
 
**Portal:** **Entra ID → Billing → Licenses → All products.** If you have any (e.g. from a trial), select a product → **Assign** → pick a user or group. Practise **group-based licensing** by assigning to `project-enigma` and confirming members inherit it.
 
---
 
## Task 5 — Turn on Self-Service Password Reset (SSPR)
 
**Portal:** **Entra ID → Password reset → Properties.**
 
- Set **Self service password reset enabled** to *Selected* and choose `project-enigma`.
- Under **Authentication methods**, require **2** methods.
- Under **Registration**, require users to register at sign-in.
- Test as a member at <https://aka.ms/sspr>.
> [!NOTE]
> SSPR for **cloud** users is included; **password write-back** to on-prem AD needs **P1 + Entra Connect**. Don't confuse SSPR (users reset their own password) with an admin reset.
 
---
 
## Success criteria
 
- [ ] Two cloud users exist — **Ada Lovelace** and **Grace Hopper** — both with Department = Engineering
- [ ] An **assigned** security group and a **dynamic** group both exist; the dynamic group auto-contains Ada and Grace
- [ ] A guest (`#EXT#`) user appears in the directory after invite redemption
- [ ] SSPR is enabled for a selected group with 2 required methods
---
 
## Break & fix — try before you peek
 
**Dynamic group is empty.** You set the rule to `user.department -eq "engineering"` (lowercase) but the users have `Engineering`. Dynamic rule *values* are case-sensitive. Fix the rule or the property, then wait for re-evaluation.
 
**`unrecognized arguments: --department`.** `az ad user create` has no `--department` flag. Create the user, then set department via `az rest` PATCH to `.../users/<upn>` (see Task 1). Same applies to `jobTitle`.
 
**Can't add the guest to a Microsoft 365 group.** Guest access can be blocked by an external collaboration setting — check **Entra ID → External Identities → External collaboration settings**.
 
---
 
## Knowledge check
 
<details>
<summary><b>Assigned vs dynamic group membership — when does each make sense?</b></summary>
**Assigned:** you add/remove members manually — predictable, no licence requirement, good for small/static teams.
 
**Dynamic:** membership is computed from a rule over user (or device) attributes — self-maintaining as people join/move/leave, but requires **Entra ID P1** and the attributes must be populated accurately. You cannot manually add a member to a dynamic group.
</details>
<details>
<summary><b>What's the difference between SSPR and password write-back?</b></summary>
**SSPR** lets users reset/unlock their own *cloud* account after registering authentication methods.
 
**Password write-back** (via Entra Connect, needs P1) pushes those cloud resets back to on-premises Active Directory so the two stay in sync in a hybrid setup.
</details>
<details>
<summary><b>Why did the department not set on <code>az ad user create</code>?</b></summary>
Because `az ad user` only exposes a handful of properties (name, UPN, password, mail-nickname, account-enabled). Department and job title aren't among them, so you set them through the underlying Microsoft Graph API with `az rest --method PATCH`. Running `az ad user create --help` shows the exact accepted arguments — the fastest way to catch this yourself.
</details>
---
 
## Cleanup
 
```bash
# Remove the lab users/groups (adjust UPNs as needed)
az ad group delete --group "project-enigma"
az ad user delete --id "ada@$DOMAIN"
az ad user delete --id "grace@$DOMAIN"
# Guests: Entra ID → Users → select the #EXT# user → Delete
```
 
---
