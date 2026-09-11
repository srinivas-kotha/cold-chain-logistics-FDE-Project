# Instructions: Legacy MSSQL Setup (Azure Dev Environment)

This document details the complete end-to-end setup to provision the legacy MSSQL container in an Azure VM (equivalent to the EC2 setup in the tutorial), configure auto-shutdown, and manage daily development restarts.

---

## 1. Local Machine Prerequisites (Windows)

Install the Azure CLI and log in:

```powershell
# Install Azure CLI
Invoke-WebRequest -Uri "https://aka.ms/installazurecliwindows" -OutFile .\AzureCLI.msi; Start-Process msiexec.exe -Wait -ArgumentList '/I AzureCLI.msi /quiet'; Remove-Item .\AzureCLI.msi

# Refresh PATH (or restart PowerShell)
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")

# Log in to Azure
az login
```

---

## 2. Infrastructure Setup (Local PowerShell)

### Step 1: Create Resource Group & VM

```powershell
$RG = "rg-coldchain-central"
$LOCATION = "centralus"
$VM_NAME = "vm-fde-dev"

# Create Resource Group
az group create --name $RG --location $LOCATION

# Create VM (Standard_D2s_v3 avoids quota restrictions on dev accounts)
az vm create `
  --resource-group $RG `
  --name $VM_NAME `
  --location $LOCATION `
  --image Ubuntu2204 `
  --size Standard_D2s_v3 `
  --admin-username azureuser `
  --generate-ssh-keys
```

### Step 2: Open SQL Server Port (1433)

```powershell
az vm open-port --resource-group $RG --name $VM_NAME --port 1433 --priority 1001
```

### Step 3: Configure Daily Auto-Shutdown (Zero Idle Billing)

_Automatically stops the VM daily at 19:00 UTC._

```powershell
az vm auto-shutdown --resource-group $RG --name $VM_NAME --time 1900
```

---

## 3. Remote Setup: Docker & MSSQL (Inside Azure VM)

### Step 1: Connect via SSH

```bash
ssh -i ~/.ssh/id_rsa azureuser@<VM_PUBLIC_IP>
```

### Step 2: Install Docker Engine

```bash
sudo apt-get update && sudo apt-get install -y docker.io
sudo systemctl enable --now docker
```

### Step 3: Create Volume & Start SQL Server Container

_Creates a persistent volume and runs the container matching project tutorial defaults._

```bash
# Create persistent volume
sudo docker volume create mssql_data

# Run container
sudo docker run -v mssql_data:/var/opt/mssql \
 -e "ACCEPT_EULA=Y" \
 -e "MSSQL_SA_PASSWORD=FdeEnterprisePass123!" \
 -p 1433:1433 \
 --name legacy-mssql \
 --restart unless-stopped  -d mcr.microsoft.com/mssql/server:2022-latest
```

### Step 4: Verify Container

```bash
sudo docker ps
```

---

## 4. Connection Details

| Parameter             | Value                   |
| :-------------------- | :---------------------- |
| **Server / Host**     | `<VM_PUBLIC_IP>,1433`   |
| **Username**          | `sa`                    |
| **Password**          | `FdeEnterprisePass123!` |
| **Trust Certificate** | `True`                  |

---

## 5. Daily Dev Workflow (Start / Stop / Restart)

To keep Azure compute costs near zero, use these commands on your local PowerShell terminal:

### When Done for the Day (Stop VM)

```powershell
az vm deallocate --resource-group rg-coldchain-central --name vm-fde-dev
```

_(Auto-shutdown will also do this automatically at 19:00 UTC if forgotten)._

### When Resuming Work (Start VM)

```powershell
az vm start --resource-group rg-coldchain-central --name vm-fde-dev
```

### Get Updated IP After Restart

When the VM stops and restarts, the dynamic public IP can change. Fetch the current one:

```powershell
az vm list-ip-addresses --resource-group rg-coldchain-central --name vm-fde-dev --output table
```

_(Because the container was configured with `--restart unless-stopped`, SQL Server starts up automatically as soon as the VM powers on)._

---

## 6. Complete Teardown (When Project is Finished)

Delete all resources to permanently eliminate billing:

```powershell
az group delete --name rg-coldchain-central --yes --no-wait
```
