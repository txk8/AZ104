# Lab 9 - ARM templates & Bicep

> **Domain:** Deploy and manage Azure compute resources (20-25%) · **Time:** ~45-60 min · **Cost:** ~£0.02 (deploys empty storage accounts)
>
> Part of my AZ-104 hands-on lab guide. Portal-first, with Azure CLI for speed and teardown.

## Scenario

This is more to cover certain exam questions than any particular real-life scenario. Infrastructure-as-code shows up on the exam as "read this template - what does it do?" or "fix this parameter". You'll deploy the same resource with ARM JSON and with Bicep, preview a change with what-if, then export an existing resource to a template and convert between the two formats.

## Exam skills covered

| Skill | Weight |
| --- | --- |
| Interpret an ARM template or Bicep file | High |
| Modify an existing ARM template / Bicep file | High |
| Deploy resources via template or Bicep | High |
| Export a deployment / convert ARM to Bicep | Medium |

---

## Task 1 - Read and deploy an ARM template

**Portal:** **Deploy a custom template → Build your own template in the editor.** Paste the JSON, fill the parameter, deploy to a resource group. Read the four key sections: **parameters**, **variables**, **resources**, **outputs**.

Save this as `azuredeploy.json`:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": { "saName": { "type": "string" } },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2023-01-01",
      "name": "[parameters('saName')]",
      "location": "[resourceGroup().location]",
      "sku": { "name": "Standard_LRS" },
      "kind": "StorageV2"
    }
  ],
  "outputs": {
    "id": { "type": "string", "value": "[resourceId('Microsoft.Storage/storageAccounts', parameters('saName'))]" }
  }
}
```

**CLI:**

```bash
RG=rg-iac-lab
LOC=uksouth
az group create -n $RG -l $LOC
az deployment group create -g $RG --template-file azuredeploy.json --parameters saName=stiac$RANDOM
```

---

## Task 2 - Do the same in Bicep

Bicep is the cleaner DSL that compiles to ARM - note how much shorter it is. Save this as `main.bicep`:

```bicep
param saName string
param location string = resourceGroup().location

resource sa 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: saName
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}

output id string = sa.id
```

**CLI:**

```bash
az deployment group create -g $RG --template-file main.bicep --parameters saName=stbicep$RANDOM
```

---

## Task 3 - Preview with what-if before deploying

Before any real change, run **what-if** to see the adds/changes/deletes

```bash
az deployment group what-if -g $RG --template-file main.bicep --parameters saName=stbicep$RANDOM
```

---

## Task 4 - Export a template and convert ARM to Bicep

**Portal:** **Resource group → Automation → Export template** captures existing resources as ARM JSON. Then convert between formats with the CLI.

```bash
# Export the whole resource group to ARM JSON
az group export -n $RG > exported.json

# Convert ARM JSON -> Bicep, and Bicep -> ARM JSON
az bicep decompile --file exported.json
az bicep build --file main.bicep   # produces main.json
```

---

## Success criteria

- [ ] You deployed a storage account from ARM JSON and from Bicep
- [ ] You can name the four template sections and what each does
- [ ] You previewed a change with what-if before deploying
- [ ] You exported an RG to ARM and converted between ARM and Bicep

---

## Break & fix - try before you peek

**Deployment fails: storage name already taken.** Storage names are globally unique - parameterise the name and add a suffix (e.g. `uniqueString(resourceGroup().id)` in the template) rather than hard-coding.

**"InvalidTemplate" errors.** Usually malformed JSON (a trailing comma), a wrong `apiVersion`, or a typo in a function like `parameters()` / `resourceId()`. Validate with `az deployment group validate` before deploying.

---

## Knowledge check

<details>
<summary><b>Why Bicep over raw ARM JSON?</b></summary>

Bicep transpiles to ARM JSON (same engine, same idempotent deployments) but is far more readable - no JSON noise, simpler expressions, automatic dependency inference, modules, and better tooling/IntelliSense. Anything ARM can do, Bicep can.
</details>

<details>
<summary><b>Incremental vs complete deployment mode?</b></summary>

**Incremental** (default) - resources in the template are created/updated; existing resources *not* in the template are left alone. **Complete** - the resource group is made to match the template exactly, so resources not in the template are **deleted**. Use what-if first to see any deletions.
</details>

---

## Cleanup

```bash
az group delete -n rg-iac-lab --yes --no-wait
```
