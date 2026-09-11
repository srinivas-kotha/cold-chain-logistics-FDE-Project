Data Ingestion & Azure VM Connection Setup

This guide walks through preparing the source dataset, connecting to the SQL Server container VM, updating local environment connection details, and optionally configuring a static IP to persist connections.

---

## 1. Prerequisites & Source Data

- **Source Dataset**: Locate and prepare the file at `data\source\data.txt`.
- **Database Infrastructure**: Ensure the Azure VM and SQL Server 2022 container have been initialized using the setup guide:
  - Reference: `setting_up_legacy_mssql_in_azure.md`
  - Image: `mcr.microsoft.com/mssql/server:2022-latest`

---

## 2. Dynamic IP Workflow (PowerShell)

Use this workflow to boot the virtual machine, obtain its assigned dynamic public IP, and inject the address into your local project environment.

### Step 1: Start the Azure Virtual Machine

```powershell
az vm start `
  --resource-group rg-coldchain-central `
  --name vm-fde-dev
```

### Step 2: Retrieve the Public IP Address

```powershell
$NEW_IP = az vm list-ip-addresses `
  --resource-group rg-coldchain-central `
  --name vm-fde-dev `
  --query "[0].virtualMachine.network.publicIpAddresses[0].ipAddress" `
  -o tsv
```

### Step 3: Verify the Resolved Address

```powershell
Write-Host "New Azure VM IP: $NEW_IP"
```

### Step 4: Synchronize the Local Environment File

Update the `DB_SERVER` entry inside your `.env` configuration file automatically:

```powershell
(Get-Content .env) -replace '^DB_SERVER=.*', "DB_SERVER=$NEW_IP" | Set-Content .env
```

---

## 3. Permanent Fix: Convert to a Static Public IP (Optional)

To avoid modifying your `.env` configuration after every machine restart, lock the VM's public IP address allocation method to `Static`.

### Run via PowerShell:

```powershell
# 1. Query the attached Public IP resource name
$PIP_NAME = (az vm show `
  --resource-group rg-coldchain-central `
  --name vm-fde-dev `
  --query "networkProfile.networkInterfaces[0].id" `
  -o tsv | ForEach-Object {
    az network nic show --ids $_ --query "ipConfigurations[0].publicIpAddress.id" -o tsv | ForEach-Object {
      az network public-ip show --ids $_ --query "name" -o tsv
    }
  })

# 2. Update the allocation method to Static
az network public-ip update `
  --resource-group rg-coldchain-central `
  --name $PIP_NAME `
  --allocation-method Static
```
