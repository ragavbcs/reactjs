# Azure SQL VM Provisioning — REST API Guide (ARM Template)

This guide documents the complete REST API flow for provisioning and deprovisioning an Azure SQL Server virtual machine using ARM Template Deployments. Each CREATE step is a single `PUT` to the ARM Deployments endpoint — Azure handles parallel resource creation internally.

---

## Variables Reference

| Variable | Value |
|---|---|
| `SUBSCRIPTION_ID` | `bf18f464-1469-4216-834f-9c6694dbfe26` |
| `RESOURCE_GROUP` | `az-vorwerk-rg` |
| `LOCATION` | `westeurope` |
| `VM_NAME` | `mysql-vorwerk-poc` |
| `VM_COMPUTER_NAME` | `mysql-vwk-poc` (**max 15 chars** — Windows limit) |
| `VM_SIZE` | `Standard_D2as_v4` |
| `OS_DISK` | `sql-vorwerk-poc-osdisk` |
| `VNET` | `az-vorwerk-vn` |
| `SUBNET` | `az-vorwerk-sb` |
| `NIC` | `mysql-vorwerk-poc-nic` |
| `DISK_DATA` | `sql-vorwerk-data` — Premium_LRS, 8 GB, LUN 0, caching: ReadOnly |
| `DISK_LOG` | `sql-vorwerk-log` — Premium_LRS, 8 GB, LUN 1, caching: None |
| `DISK_TEMPDB` | `sql-vorwerk-tempdb` — Premium_LRS, 8 GB, LUN 2, caching: ReadOnly |
| SQL Image | `microsoftsqlserver / sql2022-ws2022 / enterprise-gen2 / latest` |
| SQL Connectivity | PRIVATE, port 1433 |
| SQL License | PAYG |
| SQL Collation | `SQL_Latin1_General_CP1_CI_AS` |
| maxDop | `4` |
| maxServerMemoryMB | `24576` |
| minServerMemoryMB | `0` |
| Data path | `F:\data` |
| Log path | `G:\log` |
| TempDB path | `H:\tempDb` |

### API Versions

| Resource Type | API Version |
|---|---|
| ARM Deployments | `2021-04-01` |
| Compute — Disks | `2023-10-02` |
| Network | `2023-09-01` |
| Compute — VM | `2023-09-01` |
| SQL IaaS Extension | `2022-07-01-preview` |

### Deployment URL Pattern

```
PUT https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Resources/deployments/{DEPLOYMENT_NAME}?api-version=2021-04-01
```

Headers (all steps):
```
Authorization: Bearer <BEARER_TOKEN>
Content-Type: application/json
```

> **VM SKU Policy:** This subscription enforces `DenySpecificVMSKUs`. Allowed sizes include `Standard_D2as_v4`, `Standard_D2ads_v5`, `Standard_D4as_v4`, `Standard_B4as_v2`, `Standard_B4s_v2`. Using an unlisted size returns HTTP 403.

> **Windows computerName limit:** `osProfile.computerName` must be **15 characters or fewer**. The Azure resource name has no such restriction.

> **SQL Auth:** `sqlAuthUpdateUserName` and `sqlAuthUpdatePassword` in Step 4 are mandatory. Without them SQL Server starts in Windows-Auth-only mode and remote SSMS connections fail with Error 18456.

---

## Dry Run (What-If)

Before running any deployment (Steps 2, 3, or 4), you can preview exactly what Azure will create, modify, or delete — without making any actual changes. This is the ARM equivalent of `terraform plan`.

### What-If URL Pattern

Replace the deployment `PUT` with a `POST` to the same URL with `/whatIf` appended:

```
POST https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Resources/deployments/{DEPLOYMENT_NAME}/whatIf?api-version=2021-04-01
```

**Headers:**
```
Authorization: Bearer <BEARER_TOKEN>
Content-Type: application/json
```

**Body:** Identical to the corresponding deployment body (same JSON — no changes needed).

### What-If Response

The operation is async — HTTP `202 Accepted` with `Azure-AsyncOperation` or `Location` header. Poll until `"status": "Succeeded"`, then read the `properties.changes` array:

```json
{
  "status": "Succeeded",
  "properties": {
    "changes": [
      {
        "resourceId": "/subscriptions/.../providers/Microsoft.Compute/disks/sql-vorwerk-data",
        "changeType": "Create",
        "before": null,
        "after": { "location": "westeurope", "sku": { "name": "Premium_LRS" }, "..." : "..." }
      },
      {
        "resourceId": "/subscriptions/.../providers/Microsoft.Compute/disks/sql-vorwerk-log",
        "changeType": "NoChange",
        "before": { "..." : "..." },
        "after": { "..." : "..." }
      }
    ]
  }
}
```

### Change Types

| changeType | Meaning |
|---|---|
| `Create` | Resource does not exist — will be created |
| `Modify` | Resource exists — one or more properties will change |
| `NoChange` | Resource exists and is already in the desired state — no action |
| `Delete` | Resource will be deleted (only occurs in `Complete` mode) |
| `Ignore` | Resource is out of scope (e.g. pre-existing resources not in template) |
| `Unsupported` | Resource type does not support What-If |

### Dry Run Flow

Run What-If for each deployment before the real apply:

```
Step 2 What-If  →  POST .../deploy-sql-disks-nic/whatIf   (preview disks + NIC)
Step 2 Apply    →  PUT  .../deploy-sql-disks-nic

Step 3 What-If  →  POST .../deploy-sql-vm/whatIf           (preview VM)
Step 3 Apply    →  PUT  .../deploy-sql-vm

Step 4 What-If  →  POST .../deploy-sql-iaas/whatIf         (preview SQL IaaS)
Step 4 Apply    →  PUT  .../deploy-sql-iaas
```

> **Note:** What-If uses the same `Azure-AsyncOperation` polling as regular deployments. Poll until `Succeeded`, then inspect `properties.changes` — if all changes look correct, proceed with the real `PUT`.

---

## CREATE Flow

```
Step 1  →  Acquire Bearer Token
Step 2  →  Deploy: Disks + NIC             (4 resources in 1 deployment, all parallel)
Step 3  →  Deploy: Virtual Machine         (1 resource, depends on Step 2 completing)
Step 4  →  Deploy: SQL IaaS Extension      (1 resource, depends on Step 3 completing)
```

---

### Step 1 — Acquire Bearer Token

**Method:** `POST`

**URL:**
```
https://login.microsoftonline.com/{TENANT_ID}/oauth2/v2.0/token
```

**Headers:**
```
Content-Type: application/x-www-form-urlencoded
```

**Body (form-encoded):**
```
grant_type=client_credentials
&client_id={CLIENT_ID}
&client_secret={CLIENT_SECRET}
&scope=https://management.azure.com/.default
```

**Response:**
```json
{
  "token_type": "Bearer",
  "expires_in": 3599,
  "access_token": "<BEARER_TOKEN>"
}
```

> Store the `access_token`. All subsequent requests use `Authorization: Bearer <BEARER_TOKEN>`. Tokens expire after ~1 hour — re-authenticate if you receive HTTP 401.

---

### Step 2 — Deploy Disks + NIC

Creates all three data disks and the NIC in a single deployment. Azure provisions them in parallel internally.

**Method:** `PUT`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Resources/deployments/deploy-sql-disks-nic?api-version=2021-04-01
```

**Body:**
```json
{
  "properties": {
    "mode": "Incremental",
    "template": {
      "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
      "contentVersion": "1.0.0.0",
      "resources": [
        {
          "type": "Microsoft.Compute/disks",
          "apiVersion": "2023-10-02",
          "name": "sql-vorwerk-data",
          "location": "westeurope",
          "sku": { "name": "Premium_LRS" },
          "properties": {
            "creationData": { "createOption": "Empty" },
            "diskSizeGB": 8
          }
        },
        {
          "type": "Microsoft.Compute/disks",
          "apiVersion": "2023-10-02",
          "name": "sql-vorwerk-log",
          "location": "westeurope",
          "sku": { "name": "Premium_LRS" },
          "properties": {
            "creationData": { "createOption": "Empty" },
            "diskSizeGB": 8
          }
        },
        {
          "type": "Microsoft.Compute/disks",
          "apiVersion": "2023-10-02",
          "name": "sql-vorwerk-tempdb",
          "location": "westeurope",
          "sku": { "name": "Premium_LRS" },
          "properties": {
            "creationData": { "createOption": "Empty" },
            "diskSizeGB": 8
          }
        },
        {
          "type": "Microsoft.Network/networkInterfaces",
          "apiVersion": "2023-09-01",
          "name": "mysql-vorwerk-poc-nic",
          "location": "westeurope",
          "properties": {
            "ipConfigurations": [
              {
                "name": "ipconfig1",
                "properties": {
                  "privateIPAllocationMethod": "Dynamic",
                  "subnet": {
                    "id": "/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Network/virtualNetworks/az-vorwerk-vn/subnets/az-vorwerk-sb"
                  }
                }
              }
            ]
          }
        }
      ]
    }
  }
}
```

**Response / Notes:**
- HTTP `201 Created` — asynchronous deployment.
- Poll the `Azure-AsyncOperation` header URL until `"status": "Succeeded"`.
- Expected duration: 1–2 minutes.
- Do **not** proceed to Step 3 until this deployment is `Succeeded`.

---

### Step 3 — Deploy Virtual Machine

Attaches the three disks and NIC created in Step 2. All referenced resources must be in `Succeeded` state.

**Method:** `PUT`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Resources/deployments/deploy-sql-vm?api-version=2021-04-01
```

**Body:**
```json
{
  "properties": {
    "mode": "Incremental",
    "template": {
      "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
      "contentVersion": "1.0.0.0",
      "resources": [
        {
          "type": "Microsoft.Compute/virtualMachines",
          "apiVersion": "2023-09-01",
          "name": "mysql-vorwerk-poc",
          "location": "westeurope",
          "properties": {
            "hardwareProfile": {
              "vmSize": "Standard_D2as_v4"
            },
            "storageProfile": {
              "imageReference": {
                "publisher": "microsoftsqlserver",
                "offer": "sql2022-ws2022",
                "sku": "enterprise-gen2",
                "version": "latest"
              },
              "osDisk": {
                "name": "sql-vorwerk-poc-osdisk",
                "createOption": "FromImage",
                "managedDisk": { "storageAccountType": "Premium_LRS" }
              },
              "dataDisks": [
                {
                  "lun": 0,
                  "name": "sql-vorwerk-data",
                  "createOption": "Attach",
                  "caching": "ReadOnly",
                  "managedDisk": {
                    "id": "/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Compute/disks/sql-vorwerk-data"
                  }
                },
                {
                  "lun": 1,
                  "name": "sql-vorwerk-log",
                  "createOption": "Attach",
                  "caching": "None",
                  "managedDisk": {
                    "id": "/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Compute/disks/sql-vorwerk-log"
                  }
                },
                {
                  "lun": 2,
                  "name": "sql-vorwerk-tempdb",
                  "createOption": "Attach",
                  "caching": "ReadOnly",
                  "managedDisk": {
                    "id": "/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Compute/disks/sql-vorwerk-tempdb"
                  }
                }
              ]
            },
            "osProfile": {
              "computerName": "mysql-vwk-poc",
              "adminUsername": "<ADMIN_USERNAME>",
              "adminPassword": "<ADMIN_PASSWORD>",
              "windowsConfiguration": {
                "provisionVMAgent": true,
                "enableAutomaticUpdates": true
              }
            },
            "networkProfile": {
              "networkInterfaces": [
                {
                  "id": "/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Network/networkInterfaces/mysql-vorwerk-poc-nic",
                  "properties": { "primary": true }
                }
              ]
            }
          }
        }
      ]
    }
  }
}
```

**Response / Notes:**
- HTTP `201 Created` — asynchronous deployment.
- Poll the `Azure-AsyncOperation` header URL until `"status": "Succeeded"`.
- Expected duration: 5–15 minutes.
- Do **not** proceed to Step 4 until this deployment is `Succeeded`.

---

### Step 4 — Deploy SQL IaaS Extension

Registers the VM with the SQL IaaS Agent, enables mixed-mode SQL Authentication, creates the SQL login, and configures storage paths. The VM must be `Running` before this step.

**Method:** `PUT`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Resources/deployments/deploy-sql-iaas?api-version=2021-04-01
```

**Body:**
```json
{
  "properties": {
    "mode": "Incremental",
    "template": {
      "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
      "contentVersion": "1.0.0.0",
      "resources": [
        {
          "type": "Microsoft.SqlVirtualMachine/sqlVirtualMachines",
          "apiVersion": "2022-07-01-preview",
          "name": "mysql-vorwerk-poc",
          "location": "westeurope",
          "properties": {
            "virtualMachineResourceId": "/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Compute/virtualMachines/mysql-vorwerk-poc",
            "sqlManagement": "Full",
            "sqlServerLicenseType": "PAYG",
            "sqlImageSku": "Enterprise",
            "enableAutomaticUpgrade": true,
            "serverConfigurationsManagementSettings": {
              "sqlConnectivityUpdateSettings": {
                "connectivityType": "PRIVATE",
                "port": 1433,
                "sqlAuthUpdateUserName": "<ADMIN_USERNAME>",
                "sqlAuthUpdatePassword": "<ADMIN_PASSWORD>"
              },
              "sqlInstanceSettings": {
                "collation": "SQL_Latin1_General_CP1_CI_AS",
                "maxDop": 4,
                "isOptimizeForAdHocWorkloadsEnabled": true,
                "maxServerMemoryMB": 24576,
                "minServerMemoryMB": 0
              }
            },
            "storageConfigurationSettings": {
              "sqlDataSettings": {
                "luns": [0],
                "defaultFilePath": "F:\\data"
              },
              "sqlLogSettings": {
                "luns": [1],
                "defaultFilePath": "G:\\log"
              },
              "sqlTempDbSettings": {
                "luns": [2],
                "defaultFilePath": "H:\\tempDb",
                "dataFileCount": 8,
                "dataFileSize": 8,
                "dataGrowth": 64,
                "logFileSize": 8,
                "logGrowth": 64
              },
              "storageWorkloadType": "GENERAL"
            }
          }
        }
      ]
    }
  }
}
```

**Response / Notes:**
- HTTP `201 Created` — asynchronous deployment.
- Poll the `Azure-AsyncOperation` header URL until `"status": "Succeeded"`.
- Expected duration: 5–20 minutes (SQL IaaS Agent installs inside the VM).
- After `Succeeded`, connect via SSMS: server `<private IP>`, Authentication `SQL Server Authentication`, Login `<ADMIN_USERNAME>`, Password `<ADMIN_PASSWORD>`, Encryption `Optional`.

---

## DELETE Flow

Resources must be deleted individually in reverse dependency order. ARM template `Complete` mode is not used here as it would delete all resources in the group not listed in the template.

```
Step 1  →  Acquire Bearer Token
Step 2  →  DELETE SQL IaaS Extension
Step 3  →  DELETE Virtual Machine
Step 4  →  DELETE NIC
Step 5  →  DELETE Data Disk
Step 6  →  DELETE Log Disk
Step 7  →  DELETE TempDB Disk
```

Steps 5, 6, and 7 are independent and may be submitted in parallel after Step 4 completes.

---

### Step 1 — Acquire Bearer Token

Same as CREATE Step 1.

---

### Step 2 — DELETE SQL IaaS Extension

**Method:** `DELETE`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.SqlVirtualMachine/sqlVirtualMachines/mysql-vorwerk-poc?api-version=2022-07-01-preview
```

**Response / Notes:**
- HTTP `200 OK` or `202 Accepted`.
- If `202`, poll `Azure-AsyncOperation` until `Succeeded`.
- Expected duration: 1–3 minutes.

---

### Step 3 — DELETE Virtual Machine

**Method:** `DELETE`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Compute/virtualMachines/mysql-vorwerk-poc?api-version=2023-09-01
```

**Response / Notes:**
- HTTP `202 Accepted` — asynchronous.
- Poll `Azure-AsyncOperation` until `Succeeded`.
- Expected duration: 2–5 minutes.
- The OS disk (`sql-vorwerk-poc-osdisk`) is **not** auto-deleted. To delete it with the VM, append `?forceDeletion=true` to the URL.

---

### Step 4 — DELETE NIC

**Method:** `DELETE`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Network/networkInterfaces/mysql-vorwerk-poc-nic?api-version=2023-09-01
```

**Response / Notes:**
- HTTP `202 Accepted` — asynchronous.
- Poll until `Succeeded`.
- Expected duration: 15–30 seconds.

---

### Step 5 — DELETE Data Disk

**Method:** `DELETE`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Compute/disks/sql-vorwerk-data?api-version=2023-10-02
```

**Response / Notes:**
- HTTP `202 Accepted` — asynchronous.
- Steps 5, 6, 7 may run in parallel.

---

### Step 6 — DELETE Log Disk

**Method:** `DELETE`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Compute/disks/sql-vorwerk-log?api-version=2023-10-02
```

**Response / Notes:**
- HTTP `202 Accepted` — asynchronous.

---

### Step 7 — DELETE TempDB Disk

**Method:** `DELETE`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Compute/disks/sql-vorwerk-tempdb?api-version=2023-10-02
```

**Response / Notes:**
- HTTP `202 Accepted` — asynchronous.

---

## Symphony Node Flow Diagram

```
CREATE

  [Token]
     |
     +----------------------------------------+
     | Deployment: deploy-sql-disks-nic        |
     |  ├─ Disk: sql-vorwerk-data              |
     |  ├─ Disk: sql-vorwerk-log               |
     |  ├─ Disk: sql-vorwerk-tempdb            |
     |  └─ NIC:  mysql-vorwerk-poc-nic         |
     +----------------------------------------+
                          |
     +----------------------------------------+
     | Deployment: deploy-sql-vm               |
     |  └─ VM: mysql-vorwerk-poc               |
     +----------------------------------------+
                          |
     +----------------------------------------+
     | Deployment: deploy-sql-iaas             |
     |  └─ SQL IaaS Extension (SQL Auth)       |
     +----------------------------------------+
                          |
                       [DONE]


DELETE

               [SQL IaaS Extension]
                          |
                  [VM: mysql-vorwerk-poc]
                          |
                  [NIC: mysql-vorwerk-poc-nic]
                          |
          +---------------+---------------+
          |               |               |
    [Disk: data]    [Disk: log]    [Disk: tempdb]
          |               |               |
          +---------------+---------------+
                          |
                       [DONE]
```

---

## Async Operation Polling

All ARM Deployment PUTs and individual DELETEs return async operations.

### Response Headers to Capture

| Header | Purpose |
|---|---|
| `Azure-AsyncOperation` | URL to poll for operation status |
| `Location` | Alternative polling URL (used when `Azure-AsyncOperation` is absent) |
| `Retry-After` | Suggested polling interval in seconds (typically 15–30) |

### Poll Request

**Method:** `GET`

**URL:** Exact URL from the `Azure-AsyncOperation` header.

**Headers:**
```
Authorization: Bearer <BEARER_TOKEN>
```

### Poll Response

```json
{ "status": "InProgress | Succeeded | Failed | Canceled" }
```

### Polling Algorithm

```
1. Submit PUT or DELETE.
2. If HTTP 200/201 with no Azure-AsyncOperation header → already complete.
3. If HTTP 201/202:
   a. Extract Azure-AsyncOperation URL from response header.
   b. Wait Retry-After seconds (default: 15s).
   c. GET the Azure-AsyncOperation URL.
   d. If status == "InProgress" → go to b.
   e. If status == "Succeeded"  → proceed to next step.
   f. If status == "Failed"     → halt, read error details.
4. Re-authenticate if 401 is received (token expired after ~1 hour).
```

---

## Deployment Polling (Alternative)

Instead of polling the `Azure-AsyncOperation` header URL, you can poll the deployment resource directly:

**Method:** `GET`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Resources/deployments/{DEPLOYMENT_NAME}?api-version=2021-04-01
```

Check `properties.provisioningState` in the response:

```json
{
  "properties": {
    "provisioningState": "Running | Succeeded | Failed"
  }
}
```

---

## Timing Reference

| Step | Description | Typical Duration |
|---|---|---|
| Token | OAuth2 client credentials | < 5 seconds |
| Step 2 | Deploy Disks + NIC (4 resources, parallel) | 1–2 minutes |
| Step 3 | Deploy VM | 5–15 minutes |
| Step 4 | Deploy SQL IaaS Extension | 5–20 minutes |
| **Total CREATE** | | **~12–37 minutes** |
| DELETE Step 2 | Delete SQL IaaS Extension | 1–3 minutes |
| DELETE Step 3 | Delete VM | 2–5 minutes |
| DELETE Step 4 | Delete NIC | 15–30 seconds |
| DELETE Steps 5–7 | Delete Disks (parallel) | 30–60 seconds |
| **Total DELETE** | | **~5–10 minutes** |

---

## Notes

### OS Disk

The OS disk (`sql-vorwerk-poc-osdisk`) is created implicitly during VM creation. It is **not** deleted automatically when the VM is deleted. Options:
- Append `?forceDeletion=true` to the VM DELETE URL, or
- Issue a separate DELETE after the VM is removed.

### SSMS Connection Settings

After provisioning completes:

| Setting | Value |
|---|---|
| Server name | `<private IP of VM>` |
| Authentication | SQL Server Authentication |
| Login | `<ADMIN_USERNAME>` |
| Password | `<ADMIN_PASSWORD>` |
| Encryption | Optional |
| Trust server certificate | not required |

### Authentication

- Tokens issued via `client_credentials` are valid for ~3600 seconds.
- Re-authenticate before a long polling step if the token is close to expiry.
- Service principal requires **Contributor** role on resource group `az-vorwerk-rg`.
