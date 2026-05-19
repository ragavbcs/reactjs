# Azure SQL VM Provisioning — REST API Guide

This guide documents the complete REST API flow for provisioning and deprovisioning an Azure SQL Server virtual machine (IaaS). All requests use the Azure Resource Manager (ARM) REST API with Bearer token authentication.

---

## Variables Reference

| Variable | Value |
|---|---|
| `SUBSCRIPTION_ID` | `bf18f464-1469-4216-834f-9c6694dbfe26` |
| `RESOURCE_GROUP` | `az-vorwerk-rg` |
| `LOCATION` | `westeurope` |
| `VM_NAME` | `mysql-vorwerk-poc` |
| `VM_COMPUTER_NAME` | `mysql-vwk-poc` (**max 15 chars** — Windows limit; different from VM_NAME which can be longer) |
| `VM_SIZE` | `Standard_D2as_v4` (see VM SKU policy note below) |
| `OS_DISK` | `sql-vorwerk-poc-osdisk` |
| `VNET` | `az-vorwerk-vn` |
| `SUBNET` | `az-vorwerk-sb` |
| `NIC` | `mysql-vorwerk-poc-nic` |
| `DISK_DATA` | `sql-vorwerk-data` — Premium_LRS, 8 GB, LUN 0, caching: ReadOnly |
| `DISK_LOG` | `sql-vorwerk-log` — Premium_LRS, 8 GB, LUN 1, caching: None |
| `DISK_TEMPDB` | `sql-vorwerk-tempdb` — Premium_LRS, 8 GB, LUN 2, caching: ReadOnly |
| SQL Image | `microsoftsqlserver / sql2022-ws2022 / enterprise-gen2 / latest` |
| SQL Connectivity | PRIVATE, port 1433 |
| SQL Auth Username | `azureuser` |
| SQL Auth Password | set via `sqlAuthUpdatePassword` in Step 7 |
| SQL License | PAYG |
| SQL Collation | `SQL_Latin1_General_CP1_CI_AS` |
| maxDop | `4` |
| maxServerMemoryMB | `24576` |
| minServerMemoryMB | `0` |
| Data path | `F:\data` |
| Log path | `G:\log` |
| TempDB path | `H:\tempDb` |

> **VM SKU Policy:** This subscription enforces a `DenySpecificVMSKUs` policy. Allowed sizes include `Standard_D2as_v4`, `Standard_D2ads_v5`, `Standard_D4as_v4`, `Standard_B4as_v2`, `Standard_B4s_v2`. `Standard_B2s` and other B-series (except B4) are blocked. Use one of the allowed sizes or the deployment will return HTTP 403.

> **Windows computerName limit:** The `osProfile.computerName` field must be **15 characters or fewer**. The Azure resource name (`VM_NAME`) has no such restriction. Always set a short `computerName` (e.g. `mysql-vwk-poc` = 13 chars) even if the resource name is longer.

### API Versions

| Resource Type | API Version |
|---|---|
| Compute — Disks | `2023-10-02` |
| Network | `2023-09-01` |
| Compute — VM | `2023-09-01` |
| SQL IaaS Extension | `2022-07-01-preview` |

### Base URL pattern

```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg
```

---

## CREATE Flow

The provisioning sequence must follow the dependency order below. Asynchronous operations must be polled to completion before the next step proceeds.

```
Step 1  →  Acquire Bearer Token
Step 2  →  Create Data Disk
Step 3  →  Create Log Disk
Step 4  →  Create TempDB Disk
Step 5  →  Create NIC
Step 6  →  Create Virtual Machine          (attaches disks + NIC; computerName ≤ 15 chars)
Step 7  →  Register SQL IaaS Extension     (include sqlAuthUpdateUserName + sqlAuthUpdatePassword)
Step 7b →  [Remediation] Enable SQL Auth   (only if Step 7 was run without SQL credentials)
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

> Store the `access_token` value. All subsequent requests use the header `Authorization: Bearer <BEARER_TOKEN>`.

---

### Step 2 — Create Data Disk

**Method:** `PUT`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Compute/disks/sql-vorwerk-data?api-version=2023-10-02
```

**Headers:**
```
Authorization: Bearer <BEARER_TOKEN>
Content-Type: application/json
```

**Body:**
```json
{
  "location": "westeurope",
  "sku": {
    "name": "Premium_LRS"
  },
  "properties": {
    "creationData": {
      "createOption": "Empty"
    },
    "diskSizeGB": 8
  }
}
```

**Response / Notes:**
- HTTP `202 Accepted` — asynchronous operation.
- Poll the URL in the `Azure-AsyncOperation` response header until `"status": "Succeeded"`.
- Expected duration: 30–90 seconds.

---

### Step 3 — Create Log Disk

**Method:** `PUT`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Compute/disks/sql-vorwerk-log?api-version=2023-10-02
```

**Headers:**
```
Authorization: Bearer <BEARER_TOKEN>
Content-Type: application/json
```

**Body:**
```json
{
  "location": "westeurope",
  "sku": {
    "name": "Premium_LRS"
  },
  "properties": {
    "creationData": {
      "createOption": "Empty"
    },
    "diskSizeGB": 8
  }
}
```

**Response / Notes:**
- HTTP `202 Accepted` — asynchronous operation.
- Poll until `"status": "Succeeded"`.
- Steps 2, 3, and 4 (disk creation) may be submitted in parallel. Wait for all three before proceeding to Step 5.

---

### Step 4 — Create TempDB Disk

**Method:** `PUT`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Compute/disks/sql-vorwerk-tempdb?api-version=2023-10-02
```

**Headers:**
```
Authorization: Bearer <BEARER_TOKEN>
Content-Type: application/json
```

**Body:**
```json
{
  "location": "westeurope",
  "sku": {
    "name": "Premium_LRS"
  },
  "properties": {
    "creationData": {
      "createOption": "Empty"
    },
    "diskSizeGB": 8
  }
}
```

**Response / Notes:**
- HTTP `202 Accepted` — asynchronous operation.
- Poll until `"status": "Succeeded"`.

---

### Step 5 — Create NIC

Requires the VNet and subnet to already exist. Retrieve the subnet resource ID before calling this step.

**Subnet resource ID format:**
```
/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Network/virtualNetworks/az-vorwerk-vn/subnets/az-vorwerk-sb
```

**Method:** `PUT`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Network/networkInterfaces/mysql-vorwerk-poc-nic?api-version=2023-09-01
```

**Headers:**
```
Authorization: Bearer <BEARER_TOKEN>
Content-Type: application/json
```

**Body:**
```json
{
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
```

**Response / Notes:**
- HTTP `201 Created` or `200 OK` — may be synchronous.
- If `202 Accepted` is returned, poll the `Azure-AsyncOperation` header until succeeded.
- Expected duration: 15–30 seconds.

---

### Step 6 — Create Virtual Machine

All three data disks and the NIC must be in `Succeeded` state before this call. The body references them by resource ID.

**Method:** `PUT`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Compute/virtualMachines/mysql-vorwerk-poc?api-version=2023-09-01
```

**Headers:**
```
Authorization: Bearer <BEARER_TOKEN>
Content-Type: application/json
```

**Body:**
```json
{
  "location": "westeurope",
  "properties": {
    "hardwareProfile": {
      "vmSize": "Standard_B4as_v2"
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
        "managedDisk": {
          "storageAccountType": "Premium_LRS"
        }
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
          "properties": {
            "primary": true
          }
        }
      ]
    }
  }
}
```

**Response / Notes:**
- HTTP `201 Created` — asynchronous operation.
- Poll the `Azure-AsyncOperation` header until `"status": "Succeeded"`.
- Expected duration: 5–15 minutes.
- The VM will be in `Running` state when provisioning is complete.

---

### Step 7 — Register SQL IaaS Extension

The VM must be in `Running` / `Succeeded` state before this step. This registers the VM with the SQL IaaS Agent extension, enables mixed-mode SQL Authentication, creates the SQL login, and applies storage configuration.

> **Critical:** The `sqlConnectivityUpdateSettings` block must include `sqlAuthUpdateUserName` and `sqlAuthUpdatePassword`. If omitted, the SQL Server will start in **Windows Authentication only** mode and you will not be able to connect remotely via SQL Server Authentication (Error 18456). Without these fields, a separate PUT call is needed later to enable SQL Auth.

**Method:** `PUT`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.SqlVirtualMachine/sqlVirtualMachines/mysql-vorwerk-poc?api-version=2022-07-01-preview
```

**Headers:**
```
Authorization: Bearer <BEARER_TOKEN>
Content-Type: application/json
```

**Body:**
```json
{
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
```

**Response / Notes:**
- HTTP `200 OK` or `201 Created` — asynchronous operation.
- Poll the `Azure-AsyncOperation` header until `"status": "Succeeded"`.
- Expected duration: 5–20 minutes (the SQL IaaS Agent provisions inside the VM).
- After this step completes, connect via SSMS using **SQL Server Authentication** with the username/password provided above, Encryption set to `Optional`.

---

### Step 7b — Enable SQL Auth on Existing Registration (remediation only)

If Step 7 was run without `sqlAuthUpdateUserName`/`sqlAuthUpdatePassword`, use this PUT to enable SQL Auth retroactively. This is a full replacement of the SQL VM registration — include all existing properties.

**Method:** `PUT` (same URL as Step 7)

**Body:** Same as Step 7 body above — the `sqlAuthUpdateUserName` and `sqlAuthUpdatePassword` fields inside `sqlConnectivityUpdateSettings` are the only additions needed.

> **Note:** PATCH is not supported for this operation. A full PUT with all properties is required. Omitting fields will reset them to defaults.

---

## DELETE Flow

Resources must be deleted in reverse dependency order. Attempting to delete a disk that is still attached to a running VM will return a `409 Conflict` error.

```
Step 1  →  Delete SQL IaaS Extension
Step 2  →  Delete Virtual Machine
Step 3  →  Delete NIC
Step 4  →  Delete Data Disk
Step 5  →  Delete Log Disk
Step 6  →  Delete TempDB Disk
```

---

### Step 1 — Delete SQL IaaS Extension

**Method:** `DELETE`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.SqlVirtualMachine/sqlVirtualMachines/mysql-vorwerk-poc?api-version=2022-07-01-preview
```

**Headers:**
```
Authorization: Bearer <BEARER_TOKEN>
```

**Response / Notes:**
- HTTP `200 OK` or `202 Accepted`.
- Poll until succeeded before deleting the VM.

---

### Step 2 — Delete Virtual Machine

**Method:** `DELETE`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Compute/virtualMachines/mysql-vorwerk-poc?api-version=2023-09-01
```

**Headers:**
```
Authorization: Bearer <BEARER_TOKEN>
```

**Response / Notes:**
- HTTP `202 Accepted` — asynchronous operation.
- Poll until succeeded.
- The OS disk (`sql-vorwerk-poc-osdisk`) is a managed disk created from the image. By default it is **not** automatically deleted. Add the query parameter `forceDeletion=true` to the URL if you want the OS disk removed with the VM.
- Expected duration: 2–5 minutes.

---

### Step 3 — Delete NIC

**Method:** `DELETE`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Network/networkInterfaces/mysql-vorwerk-poc-nic?api-version=2023-09-01
```

**Headers:**
```
Authorization: Bearer <BEARER_TOKEN>
```

**Response / Notes:**
- HTTP `202 Accepted` — asynchronous operation.
- Poll until succeeded.

---

### Step 4 — Delete Data Disk

**Method:** `DELETE`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Compute/disks/sql-vorwerk-data?api-version=2023-10-02
```

**Headers:**
```
Authorization: Bearer <BEARER_TOKEN>
```

**Response / Notes:**
- HTTP `202 Accepted` — asynchronous operation.
- Steps 4, 5, and 6 (disk deletion) may be submitted in parallel.

---

### Step 5 — Delete Log Disk

**Method:** `DELETE`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Compute/disks/sql-vorwerk-log?api-version=2023-10-02
```

**Headers:**
```
Authorization: Bearer <BEARER_TOKEN>
```

**Response / Notes:**
- HTTP `202 Accepted` — asynchronous operation.

---

### Step 6 — Delete TempDB Disk

**Method:** `DELETE`

**URL:**
```
https://management.azure.com/subscriptions/bf18f464-1469-4216-834f-9c6694dbfe26/resourceGroups/az-vorwerk-rg/providers/Microsoft.Compute/disks/sql-vorwerk-tempdb?api-version=2023-10-02
```

**Headers:**
```
Authorization: Bearer <BEARER_TOKEN>
```

**Response / Notes:**
- HTTP `202 Accepted` — asynchronous operation.

---

## Symphony Node Flow Diagram

The diagram below represents the directed acyclic graph of provisioning steps, showing which steps can run in parallel and which have hard dependencies.

```
CREATE

  [Token]
     |
     +--------------------+--------------------+
     |                    |                    |
[Disk: data]        [Disk: log]        [Disk: tempdb]
     |                    |                    |
     +--------------------+--------------------+
                          |
                     [NIC: nic]
                          |
                  [VM: mysql-vorwerk-poc]
                          |
               [SQL IaaS Extension: PAYG]
                          |
                       [DONE]


DELETE

               [SQL IaaS Extension]
                          |
                  [VM: mysql-vorwerk-poc]
                          |
                     [NIC: nic]
                          |
     +--------------------+--------------------+
     |                    |                    |
[Disk: data]        [Disk: log]        [Disk: tempdb]
     |                    |                    |
     +--------------------+--------------------+
                          |
                       [DONE]
```

---

## Async Operation Polling

Many ARM operations return `202 Accepted` with a long-running operation handle in the response headers. You must poll this handle and wait for a terminal state before proceeding to the next step.

### Response Headers to Capture

| Header | Purpose |
|---|---|
| `Azure-AsyncOperation` | URL to poll for operation status |
| `Location` | Alternative polling URL (used when `Azure-AsyncOperation` is absent) |
| `Retry-After` | Suggested polling interval in seconds (typically 15–30) |

### Poll Request

**Method:** `GET`

**URL:** The exact URL from the `Azure-AsyncOperation` header.

**Headers:**
```
Authorization: Bearer <BEARER_TOKEN>
```

### Poll Response Schema

```json
{
  "status": "InProgress | Succeeded | Failed | Canceled",
  "error": {
    "code": "...",
    "message": "..."
  }
}
```

### Terminal States

| Status | Meaning |
|---|---|
| `Succeeded` | Operation completed successfully. Proceed to next step. |
| `Failed` | Operation failed. Read `error.message` for details. |
| `Canceled` | Operation was cancelled. |

### Polling Algorithm

```
1. Submit the resource operation (PUT or DELETE).
2. If response is 200/201 with no Azure-AsyncOperation header → already complete.
3. If response is 202:
   a. Extract the Azure-AsyncOperation URL from the response header.
   b. Wait for the number of seconds indicated by Retry-After (default: 15s).
   c. GET the Azure-AsyncOperation URL.
   d. If status == "InProgress" → go to step b.
   e. If status == "Succeeded" → continue to next step.
   f. If status == "Failed" or "Canceled" → halt and handle error.
4. Token expiry: Bearer tokens expire after ~1 hour. Re-authenticate if you receive 401.
```

---

## Notes

### Parallel Execution

The following groups of steps are independent and may be submitted concurrently to reduce total provisioning time:

| Parallel Group | Steps |
|---|---|
| Disk creation | Steps 2, 3, 4 (all three data disks) |
| Disk deletion | Steps 4, 5, 6 (all three data disks) |

All other steps have strict sequential dependencies.

### Wait Conditions

| Transition | Condition Before Proceeding |
|---|---|
| After disk creation → NIC | All three disks must be in `Succeeded` state |
| After NIC → VM | NIC must be in `Succeeded` state |
| After VM → SQL IaaS | VM must be in `Running` / `Succeeded` state |
| After SQL IaaS delete → VM delete | SQL IaaS extension must be fully removed |
| After VM delete → NIC delete | VM must be fully deleted and disks detached |
| After NIC delete → disk deletes | NIC must be fully deleted |

### Expected Timing

| Step | Typical Duration |
|---|---|
| Token acquisition | < 5 seconds |
| Single disk creation | 30–90 seconds |
| NIC creation | 15–30 seconds |
| VM creation | 5–15 minutes |
| SQL IaaS Extension registration | 5–20 minutes |
| VM deletion | 2–5 minutes |
| NIC deletion | 15–30 seconds |
| Single disk deletion | 30–60 seconds |
| **Total CREATE (sequential)** | **~20–40 minutes** |
| **Total DELETE (sequential)** | **~10–15 minutes** |

### OS Disk Handling

The OS disk (`sql-vorwerk-poc-osdisk`) is created implicitly during VM creation from the marketplace image. It is **not** automatically deleted when the VM is deleted. To delete it explicitly, either:

- Use the `forceDeletion=true` query parameter on the VM DELETE call, or
- Issue a separate DELETE request against the disk resource after the VM is removed.

### Authentication

- All requests require a valid Bearer token in the `Authorization` header.
- Tokens issued via the `client_credentials` flow are valid for approximately 3600 seconds (1 hour).
- If a long provisioning operation causes token expiry, re-authenticate and retry the polling request with the new token.
- Use a service principal with at least the **Contributor** role scoped to the resource group `az-vorwerk-rg`.
