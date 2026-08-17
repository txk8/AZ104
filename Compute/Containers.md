# Lab 10 - Containers: ACR, Container Instances & Container Apps

> **Domain:** Deploy and manage Azure compute resources (20-25%) · **Time:** ~40 min · **Cost:** ~£0.10 
>
> Part of my AZ-104 hands-on lab guide. Portal-first, with Azure CLI for speed and teardown.

## Scenario

Megatron wants to ship a small container image: build and store it in a private registry, run it as a one-off with Container Instances, then run it as a scalable service with Container Apps.

## Exam skills covered

| Skill | Weight |
| --- | --- |
| Create and manage an Azure Container Registry | High |
| Provision a container with Azure Container Instances | High |
| Provision a container with Azure Container Apps | Medium |
| Manage sizing and scaling for containers | Medium |

---

## Task 1 - Create a registry and build an image in the cloud

**Portal:** **Container registries → Create** (SKU **Basic**). `az acr build` builds your image *on Azure* (no local Docker needed) and pushes it to the registry.

<img width="555" height="40" alt="image" src="https://github.com/user-attachments/assets/c63c59d2-8d69-4fe2-b21f-bb46c33be5e6" />


**CLI:**

```bash
RG=rg-container-lab
LOC=uksouth
ACR=acrmegatron$RANDOM
az group create -n $RG -l $LOC
az acr create -g $RG -n $ACR --sku Basic

# Build the sample quickstart image in the registry (no Docker on your machine required)
az acr build -r $ACR -t web/hello:v1 https://github.com/Azure-Samples/acr-build-helloworld-node.git
az acr repository list -n $ACR -o table
```

> [!NOTE]
> On a new subscription or where you haven't used containers before you may hit `MissingSubscriptionRegistration`. Resource providers are enabled **per subscription** (only needed one-time), so register the ones this lab needs, wait for "Registered", then retry the create.

```bash
az provider register -n Microsoft.ContainerRegistry
az provider register -n Microsoft.ContainerInstance      # ACI (Task 2)
az provider register -n Microsoft.App                    # Container Apps (Task 3)
az provider register -n Microsoft.OperationalInsights    # Container Apps environment
az provider show -n Microsoft.ContainerRegistry --query registrationState -o tsv   # wait for "Registered"
```

---

## Task 2 - Run it with Azure Container Instances (ACI)

**Portal:** **Container instances → Create.** Source = your ACR image, give it CPU/ and a DNS label.

**CLI:**

```bash
az acr update -n $ACR --admin-enabled true
ACR_USER=$(az acr credential show -n $ACR --query username -o tsv)
ACR_PASS=$(az acr credential show -n $ACR --query "passwords[0].value" -o tsv)
DNS=hello$RANDOM

az container create -g $RG -n aci-hello --image $ACR.azurecr.io/web/hello:v1 --registry-username $ACR_USER --registry-password $ACR_PASS --dns-name-label $DNS --ports 80 --cpu 1 --memory 1
az container show -g $RG -n aci-hello --query ipAddress.fqdn -o tsv
```

<img width="140" height="43" alt="image" src="https://github.com/user-attachments/assets/f7fecc86-c90b-4506-b02b-6e6cab13a27d" />

Browse to the FQDN it prints and you should see the sample app.

---

## Task 3 - Run it as a scalable service with Container Apps

**Portal:** **Container Apps → Create.** Container Apps adds ingress, revisions, and scale-to-zero on top of a managed (Log Analytics-backed) environment.

**CLI:**

```bash
az extension add -n containerapp --upgrade
az provider register -n Microsoft.App
az provider register -n Microsoft.OperationalInsights

az containerapp env create -g $RG -n cae-lab -l $LOC
az containerapp create -g $RG -n ca-hello --environment cae-lab --image $ACR.azurecr.io/web/hello:v1 --registry-server $ACR.azurecr.io --registry-username $ACR_USER --registry-password $ACR_PASS --target-port 80 --ingress external --min-replicas 0 --max-replicas 5
```
<img width="586" height="114" alt="image" src="https://github.com/user-attachments/assets/6a7855b3-5f52-4e44-a9cb-9a2b5da7e5e1" />

---

## Success criteria

- [ ] A Basic ACR holds an image you built with `az acr build`
- [ ] An ACI container is reachable on its public FQDN
- [ ] A Container App serves the image with external ingress and 0-5 replica scaling

---

## Break & fix - try before you peek

**ACI can't pull the image (Unauthorized).** The registry is private. Either enable the admin user and pass credentials (as above), or - better - give the container a **managed identity** with the **AcrPull** role on the registry.

**Container App create fails on the provider.** The `Microsoft.App` / `Microsoft.OperationalInsights` providers must be registered on the subscription first (the `az provider register` lines above).

---

## Knowledge check

<details>
<summary><b>ACI vs Container Apps vs AKS - when to use which?</b></summary>

**ACI** - simplest: run a single or few containers, batch jobs, no orchestration, per-second billing. **Container Apps** - serverless containers with ingress, revisions, event-driven autoscaling (KEDA) and scale-to-zero - great for microservices/APIs without managing Kubernetes. **AKS** - full managed Kubernetes for complex orchestration and maximum control. For AZ-104: ACI = simplest, Container Apps = serverless scale.
</details>

<details>
<summary><b>Best way for a container to authenticate to ACR?</b></summary>

A **managed identity** with the **AcrPull** role - no stored credentials. The admin user + password works for quick tests but is a shared secret you have to manage and rotate, so don't use in production.
</details>

---

## Cleanup

```bash
az group delete -n rg-container-lab --yes --no-wait
```
