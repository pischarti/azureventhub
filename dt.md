# Azure Event Hub Audit Log with Dynatrace

Architecture design for ingesting Azure AD logs, Activity logs, and Resource logs into Dynatrace SaaS via Azure Event Hub.

---

## Overview

This document describes the end-to-end architecture for streaming Azure platform logs (Entra ID/Azure AD, Activity, and Resource logs) into Dynatrace SaaS using Azure Event Hub as the intermediary streaming platform. The architecture employs a dual-VNet design with an Azure Function Log Forwarder for event processing, Blob Storage for temporary log staging, and a dedicated Dynatrace ActiveGate for secure forwarding to Dynatrace SaaS.

### Log Types Covered

| Log Type | Description | Common Use Cases |
|----------|-------------|------------------|
| **Azure AD (Entra ID) Logs** | Sign-in logs, audit logs, provisioning logs, risky user/sign-in events | Security monitoring, identity analytics, compliance auditing |
| **Activity Logs** | Subscription-level control plane operations (ARM) | Change tracking, compliance, resource lifecycle monitoring |
| **Resource Logs** | Diagnostic logs from individual Azure resources | Performance troubleshooting, application debugging, security analysis |

---

## Architecture Design

### High-Level Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                 Azure Platform                                       │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────────────────────────┐ │
│  │   Azure AD   │   │   Activity   │   │           Resource Logs                  │ │
│  │ (Entra ID)   │   │    Logs      │   │  (App Service, AKS, SQL, Key Vault)      │ │
│  │   Logs       │   │              │   │                                          │ │
│  └──────┬───────┘   └──────┬───────┘   └──────────────────┬───────────────────────┘ │
│         │                  │                              │                          │
│         │      Diagnostic Settings / Export Configuration │                          │
│         └──────────────────┴──────────────────────────────┘                          │
│                                    │                                                 │
└────────────────────────────────────┼─────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           VNET-A: Log Ingestion VNet                                │
│                          (Event Hub & Function VNet)                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐│
│  │                        Azure Event Hub Namespace                                ││
│  │                      (Private Endpoint in VNET-A)                               ││
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                  ││
│  │  │  eventhub-ad    │  │ eventhub-       │  │ eventhub-       │                  ││
│  │  │  -logs          │  │ activity-logs   │  │ resource-logs   │                  ││
│  │  │  (partitions)   │  │ (partitions)    │  │ (partitions)    │                  ││
│  │  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘                  ││
│  │           └────────────────────┼────────────────────┘                            ││
│  └────────────────────────────────┼─────────────────────────────────────────────────┘│
│                                   │ Event Hub Trigger                                │
│                                   ▼                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐│
│  │                    Azure Function App (Log Forwarder)                           ││
│  │                     (VNet Integrated - VNET-A Subnet)                           ││
│  │  ┌─────────────────────────────────────────────────────────────────────────────┐││
│  │  │  • Consumes events from Event Hub                                           │││
│  │  │  • Transforms/enriches log data                                             │││
│  │  │  • Batches logs and stages to Blob Storage                                  │││
│  │  │  • Forwards processed logs to ActiveGate                                    │││
│  │  └─────────────────────────────────────────────────────────────────────────────┘││
│  └──────────────────────────┬──────────────────────────┬────────────────────────────┘│
│                             │                          │                             │
│                             │ Stage Logs               │ Forward Logs                │
│                             ▼                          │                             │
│  ┌──────────────────────────────────────────┐          │                             │
│  │      Azure Blob Storage Account          │          │                             │
│  │    (Private Endpoint in VNET-A)          │          │                             │
│  │  ┌────────────────────────────────────┐  │          │                             │
│  │  │  Container: log-staging            │  │          │                             │
│  │  │  • Temporary log batch storage     │  │          │                             │
│  │  │  • Retry buffer for failed sends   │  │          │                             │
│  │  │  • Consumer checkpoint storage     │  │          │                             │
│  │  │  • Lifecycle: auto-delete 24hrs    │  │          │                             │
│  │  └────────────────────────────────────┘  │          │                             │
│  └──────────────────────────────────────────┘          │                             │
│                                                        │                             │
└────────────────────────────────────────────────────────┼─────────────────────────────┘
                                                         │
                                                         │ HTTPS (Port 9999)
                                                         │ VNet Peering / Private Link
                                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          VNET-B: ActiveGate VNet                                    │
│                        (Dynatrace Integration VNet)                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐│
│  │                     Dynatrace ActiveGate Cluster                                ││
│  │                   (Environment ActiveGate - HA Pair)                            ││
│  │  ┌───────────────────────────┐  ┌───────────────────────────┐                   ││
│  │  │   ActiveGate Node 1      │  │   ActiveGate Node 2       │                   ││
│  │  │   (Primary)              │  │   (Secondary)             │                   ││
│  │  │   • Log Ingest Module    │  │   • Log Ingest Module     │                   ││
│  │  │   • Metrics Ingest       │  │   • Metrics Ingest        │                   ││
│  │  └───────────┬──────────────┘  └───────────┬───────────────┘                   ││
│  │              │                             │                                     ││
│  │              └──────────────┬──────────────┘                                     ││
│  │                             │                                                    ││
│  │              Internal Load Balancer (Port 9999)                                  ││
│  └─────────────────────────────┼────────────────────────────────────────────────────┘│
│                                │                                                     │
└────────────────────────────────┼─────────────────────────────────────────────────────┘
                                 │
                                 │ HTTPS (443)
                                 │ Outbound to Internet / ExpressRoute
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          Dynatrace SaaS Environment                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐│
│  │                          Dynatrace Cluster                                      ││
│  │  ┌─────────────────────────────────────────────────────────────────────────────┐││
│  │  │                    Log Ingest API Endpoint                                  │││
│  │  │              https://{env-id}.live.dynatrace.com/api/v2/logs/ingest        │││
│  │  └─────────────────────────────────────────────────────────────────────────────┘││
│  │                                    │                                            ││
│  │                                    ▼                                            ││
│  │  ┌─────────────────────────────────────────────────────────────────────────────┐││
│  │  │                        Dynatrace Log Storage                                │││
│  │  │       Grail / Log Viewer / Dashboards / Alerting / Davis AI Correlation    │││
│  │  └─────────────────────────────────────────────────────────────────────────────┘││
│  └─────────────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Network Architecture Summary

| Component | VNet | Subnet | Purpose |
|-----------|------|--------|---------|
| Event Hub Namespace | VNET-A | `snet-eventhub` | Private endpoint for Event Hub |
| Azure Function App | VNET-A | `snet-function` | VNet-integrated Function App |
| Blob Storage Account | VNET-A | `snet-storage` | Private endpoint for staging storage |
| ActiveGate Cluster | VNET-B | `snet-activegate` | Dynatrace ActiveGate VMs |
| Internal Load Balancer | VNET-B | `snet-activegate` | HA endpoint for ActiveGate |

### VNet Connectivity

| Connection | Type | Purpose |
|------------|------|---------|
| VNET-A ↔ VNET-B | VNet Peering | Function App to ActiveGate communication |
| VNET-B → Internet | NAT Gateway / Azure Firewall | ActiveGate to Dynatrace SaaS |

---

## Component Details

### 1. Azure Event Hub Namespace

The Event Hub Namespace serves as the central streaming platform for all Azure logs before forwarding to the Azure Function.

#### Namespace Configuration

| Setting | Recommended Value | Rationale |
|---------|-------------------|-----------|
| **Pricing Tier** | Premium | Required for private endpoints and zone redundancy |
| **Processing Units** | Start with 1-2 PUs | Scale based on log volume |
| **Zone Redundancy** | Enabled | Production resilience for critical audit logs |
| **Retention** | 1-7 days | Balance between replay capability and cost |
| **Public Network Access** | Disabled | All access via private endpoint in VNET-A |

#### Event Hub Instances

Create separate Event Hubs within the namespace for log isolation and independent scaling:

| Event Hub Name | Purpose | Partition Count | Consumer Groups |
|----------------|---------|-----------------|-----------------|
| `eventhub-aad-logs` | Azure AD sign-in and audit logs | 4-8 | `function-consumer`, `$Default` |
| `eventhub-activity-logs` | Subscription activity logs | 2-4 | `function-consumer`, `$Default` |
| `eventhub-resource-logs` | Resource diagnostic logs | 8-16 | `function-consumer`, `$Default` |

#### Shared Access Policies

| Policy Name | Rights | Used By |
|-------------|--------|---------|
| `DiagnosticSettingsSend` | Send | Azure Diagnostic Settings (log sources) |
| `FunctionConsumer` | Listen | Azure Function Log Forwarder |

#### Private Endpoint Configuration

```hcl
resource "azurerm_private_endpoint" "eventhub" {
  name                = "pe-eventhub-logs"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  subnet_id           = azurerm_subnet.eventhub.id

  private_service_connection {
    name                           = "psc-eventhub"
    private_connection_resource_id = azurerm_eventhub_namespace.main.id
    subresource_names              = ["namespace"]
    is_manual_connection           = false
  }

  private_dns_zone_group {
    name                 = "eventhub-dns-zone-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.eventhub.id]
  }
}
```

### 2. Azure AD (Entra ID) Log Export

Azure AD logs require tenant-level configuration via the Entra ID admin center or Azure Portal.

#### Supported Log Categories

| Category | Description | Event Hub Destination |
|----------|-------------|----------------------|
| `SignInLogs` | Interactive user sign-ins | `eventhub-aad-logs` |
| `NonInteractiveUserSignInLogs` | Non-interactive sign-ins (background, service) | `eventhub-aad-logs` |
| `ServicePrincipalSignInLogs` | Service principal/application sign-ins | `eventhub-aad-logs` |
| `ManagedIdentitySignInLogs` | Managed identity sign-ins | `eventhub-aad-logs` |
| `AuditLogs` | Directory changes, user/group management | `eventhub-aad-logs` |
| `ProvisioningLogs` | Azure AD provisioning service logs | `eventhub-aad-logs` |
| `RiskyUsers` | Users flagged for risk | `eventhub-aad-logs` |
| `RiskyServicePrincipals` | Service principals flagged for risk | `eventhub-aad-logs` |
| `UserRiskEvents` | Risk detection events for users | `eventhub-aad-logs` |
| `ServicePrincipalRiskEvents` | Risk detection events for service principals | `eventhub-aad-logs` |

#### Configuration Steps

1. **Navigate to Entra ID Diagnostic Settings**:
   - Azure Portal → Microsoft Entra ID → Monitoring → Diagnostic settings
   - Or: `https://portal.azure.com/#view/Microsoft_AAD_IAM/DiagnosticSettingsV2Blade`

2. **Create Diagnostic Setting**:
   - Name: `export-to-eventhub-dynatrace`
   - Select all required log categories
   - Destination: Stream to an event hub
   - Event Hub namespace: Select the target namespace
   - Event Hub name: `eventhub-aad-logs`
   - Event Hub policy: `DiagnosticSettingsSend`

3. **Required Permissions**:
   - Azure AD role: `Security Administrator` or `Global Administrator`
   - Event Hub: `Azure Event Hubs Data Sender` on the namespace or specific hub

#### Terraform Example

```hcl
resource "azurerm_monitor_aad_diagnostic_setting" "aad_to_eventhub" {
  name                           = "export-to-eventhub-dynatrace"
  eventhub_authorization_rule_id = azurerm_eventhub_namespace_authorization_rule.diagnostic_send.id
  eventhub_name                  = azurerm_eventhub.aad_logs.name

  enabled_log {
    category = "SignInLogs"
    retention_policy {
      enabled = false
    }
  }

  enabled_log {
    category = "AuditLogs"
    retention_policy {
      enabled = false
    }
  }

  enabled_log {
    category = "NonInteractiveUserSignInLogs"
    retention_policy {
      enabled = false
    }
  }

  enabled_log {
    category = "ServicePrincipalSignInLogs"
    retention_policy {
      enabled = false
    }
  }

  enabled_log {
    category = "ManagedIdentitySignInLogs"
    retention_policy {
      enabled = false
    }
  }

  enabled_log {
    category = "RiskyUsers"
    retention_policy {
      enabled = false
    }
  }

  enabled_log {
    category = "UserRiskEvents"
    retention_policy {
      enabled = false
    }
  }
}
```

### 3. Azure Activity Log Export

Activity logs capture subscription-level control plane operations and must be configured per subscription.

#### Log Categories

| Category | Description |
|----------|-------------|
| `Administrative` | Resource create, update, delete operations |
| `Security` | Security Center alerts and recommendations |
| `ServiceHealth` | Azure service health events |
| `Alert` | Azure Monitor alerts fired |
| `Recommendation` | Azure Advisor recommendations |
| `Policy` | Azure Policy evaluation events |
| `Autoscale` | Autoscale engine operations |
| `ResourceHealth` | Resource health status changes |

#### Configuration Steps

1. **Navigate to Activity Log Export**:
   - Azure Portal → Monitor → Activity log → Export Activity Logs
   - Or: Subscription → Activity log → Diagnostic settings

2. **Create Diagnostic Setting**:
   - Name: `activity-log-to-eventhub`
   - Select all applicable categories
   - Destination: Stream to an event hub
   - Event Hub namespace: Select target namespace
   - Event Hub name: `eventhub-activity-logs`
   - Event Hub policy: `DiagnosticSettingsSend`

#### Terraform Example

```hcl
resource "azurerm_monitor_diagnostic_setting" "activity_log_to_eventhub" {
  name                           = "activity-log-to-eventhub"
  target_resource_id             = "/subscriptions/${data.azurerm_subscription.current.subscription_id}"
  eventhub_authorization_rule_id = azurerm_eventhub_namespace_authorization_rule.diagnostic_send.id
  eventhub_name                  = azurerm_eventhub.activity_logs.name

  enabled_log {
    category = "Administrative"
  }

  enabled_log {
    category = "Security"
  }

  enabled_log {
    category = "ServiceHealth"
  }

  enabled_log {
    category = "Alert"
  }

  enabled_log {
    category = "Policy"
  }

  enabled_log {
    category = "ResourceHealth"
  }
}
```

### 4. Azure Resource Log Export

Resource logs (formerly Diagnostic Logs) are configured per-resource and provide granular insights into resource behavior.

#### Common Resource Log Categories

| Resource Type | Key Categories | Typical Use Cases |
|---------------|----------------|-------------------|
| **App Service** | `AppServiceHTTPLogs`, `AppServiceConsoleLogs`, `AppServiceAppLogs` | Request tracing, application errors |
| **Azure Kubernetes Service** | `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `cluster-autoscaler`, `guard` | Cluster operations, security auditing |
| **Azure SQL** | `SQLInsights`, `AutomaticTuning`, `QueryStoreRuntimeStatistics`, `Errors` | Query performance, database errors |
| **Key Vault** | `AuditEvent` | Secret access auditing, compliance |
| **Azure Firewall** | `AzureFirewallApplicationRule`, `AzureFirewallNetworkRule`, `AzureFirewallDnsProxy` | Network security monitoring |
| **Application Gateway** | `ApplicationGatewayAccessLog`, `ApplicationGatewayPerformanceLog`, `ApplicationGatewayFirewallLog` | WAF events, performance analysis |
| **Storage Account** | `StorageRead`, `StorageWrite`, `StorageDelete` | Data access auditing |
| **Cosmos DB** | `DataPlaneRequests`, `QueryRuntimeStatistics`, `ControlPlaneRequests` | Query performance, operations |

#### Configuration Pattern

```hcl
resource "azurerm_monitor_diagnostic_setting" "resource_logs" {
  for_each = var.monitored_resources

  name                           = "diag-to-eventhub-${each.key}"
  target_resource_id             = each.value.resource_id
  eventhub_authorization_rule_id = azurerm_eventhub_namespace_authorization_rule.diagnostic_send.id
  eventhub_name                  = azurerm_eventhub.resource_logs.name

  dynamic "enabled_log" {
    for_each = each.value.log_categories
    content {
      category = enabled_log.value
    }
  }

  dynamic "metric" {
    for_each = each.value.metric_categories
    content {
      category = metric.value
      enabled  = true
    }
  }
}
```

### 5. Azure Blob Storage (Log Staging)

The Blob Storage account provides temporary staging for log batches, enabling reliable delivery with retry capabilities.

#### Storage Account Configuration

| Setting | Recommended Value | Rationale |
|---------|-------------------|-----------|
| **Account Kind** | StorageV2 | General-purpose v2 for all features |
| **Performance** | Standard | Cost-effective for log staging |
| **Replication** | LRS or ZRS | Local redundancy sufficient for temporary staging |
| **Access Tier** | Hot | Frequent read/write for active batches |
| **Public Network Access** | Disabled | All access via private endpoint |
| **Hierarchical Namespace** | Disabled | Not required for blob staging |

#### Container Structure

| Container | Purpose | Lifecycle Policy |
|-----------|---------|-----------------|
| `log-staging` | Active log batches awaiting forward | Delete after 24 hours |
| `log-failed` | Failed deliveries for retry | Delete after 7 days |
| `log-checkpoint` | Event Hub consumer checkpoints | No deletion (managed by Function) |

#### Lifecycle Management Policy

```json
{
  "rules": [
    {
      "name": "delete-staging-logs",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["log-staging/"]
        },
        "actions": {
          "baseBlob": {
            "delete": {
              "daysAfterCreationGreaterThan": 1
            }
          }
        }
      }
    },
    {
      "name": "delete-failed-logs",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["log-failed/"]
        },
        "actions": {
          "baseBlob": {
            "delete": {
              "daysAfterCreationGreaterThan": 7
            }
          }
        }
      }
    }
  ]
}
```

#### Terraform Example

```hcl
resource "azurerm_storage_account" "log_staging" {
  name                            = "stlogstaging${var.environment}"
  resource_group_name             = azurerm_resource_group.main.name
  location                        = azurerm_resource_group.main.location
  account_tier                    = "Standard"
  account_replication_type        = "ZRS"
  account_kind                    = "StorageV2"
  min_tls_version                 = "TLS1_2"
  public_network_access_enabled   = false
  allow_nested_items_to_be_public = false

  blob_properties {
    delete_retention_policy {
      days = 7
    }
  }
}

resource "azurerm_storage_container" "log_staging" {
  name                  = "log-staging"
  storage_account_name  = azurerm_storage_account.log_staging.name
  container_access_type = "private"
}

resource "azurerm_storage_container" "log_failed" {
  name                  = "log-failed"
  storage_account_name  = azurerm_storage_account.log_staging.name
  container_access_type = "private"
}

resource "azurerm_storage_container" "log_checkpoint" {
  name                  = "log-checkpoint"
  storage_account_name  = azurerm_storage_account.log_staging.name
  container_access_type = "private"
}

resource "azurerm_private_endpoint" "storage_blob" {
  name                = "pe-storage-blob"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  subnet_id           = azurerm_subnet.storage.id

  private_service_connection {
    name                           = "psc-storage-blob"
    private_connection_resource_id = azurerm_storage_account.log_staging.id
    subresource_names              = ["blob"]
    is_manual_connection           = false
  }

  private_dns_zone_group {
    name                 = "storage-dns-zone-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.blob.id]
  }
}
```

### 6. Azure Function App (Log Forwarder)

The Azure Function App consumes events from Event Hub, stages logs to Blob Storage, and forwards to ActiveGate.

#### Function App Configuration

| Setting | Recommended Value | Rationale |
|---------|-------------------|-----------|
| **Runtime** | Python 3.11 or .NET 8 | Mature Event Hub SDK support |
| **Plan** | Premium EP1-EP3 | VNet integration, no cold starts, scale control |
| **VNet Integration** | VNET-A / snet-function | Access Event Hub and Storage private endpoints |
| **Outbound IPs** | Via VNet (NAT to VNET-B) | Route to ActiveGate via peering |
| **Instance Count** | Min 2, Max 10 | HA and burst handling |

#### Application Settings

| Setting | Value | Description |
|---------|-------|-------------|
| `EVENTHUB_CONNECTION` | `@Microsoft.KeyVault(SecretUri=...)` | Event Hub connection string |
| `STORAGE_CONNECTION` | `@Microsoft.KeyVault(SecretUri=...)` | Blob Storage connection string |
| `ACTIVEGATE_ENDPOINT` | `https://10.1.0.10:9999/e/{env-id}/api/v2/logs/ingest` | ActiveGate internal LB endpoint |
| `ACTIVEGATE_TOKEN` | `@Microsoft.KeyVault(SecretUri=...)` | Dynatrace API token |
| `BATCH_SIZE` | `1000` | Logs per batch |
| `BATCH_INTERVAL_SECONDS` | `30` | Max time before flush |
| `STAGING_CONTAINER` | `log-staging` | Blob container for staging |
| `FAILED_CONTAINER` | `log-failed` | Blob container for failed logs |
| `CHECKPOINT_CONTAINER` | `log-checkpoint` | Blob container for checkpoints |

#### Function Logic Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Azure Function Execution Flow                        │
│                                                                             │
│  ┌─────────────┐    ┌─────────────────┐    ┌──────────────────────────────┐│
│  │  Event Hub  │───▶│  Event Trigger  │───▶│  Parse & Transform Logs      ││
│  │  Trigger    │    │  (Batch Mode)   │    │  • Extract common fields     ││
│  └─────────────┘    └─────────────────┘    │  • Enrich with metadata      ││
│                                            │  • Validate schema           ││
│                                            └──────────────┬───────────────┘│
│                                                           │                 │
│                                                           ▼                 │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    Stage to Blob Storage                             │  │
│  │  • Write batch to log-staging/{timestamp}/{batch-id}.json           │  │
│  │  • Enables retry on downstream failure                               │  │
│  │  • Checkpoint Event Hub position after successful stage              │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                            │                                │
│                                            ▼                                │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    Forward to ActiveGate                             │  │
│  │  • POST to ActiveGate log ingest endpoint (HTTPS/9999)              │  │
│  │  • Include Dynatrace API token header                                │  │
│  │  • Retry with exponential backoff (3 attempts)                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                          │                              │                   │
│                    Success                           Failure                │
│                          │                              │                   │
│                          ▼                              ▼                   │
│  ┌─────────────────────────────────┐  ┌────────────────────────────────┐   │
│  │  Delete from log-staging       │  │  Move to log-failed            │   │
│  │  Update checkpoint             │  │  Alert on persistent failures  │   │
│  └─────────────────────────────────┘  └────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Sample Function Code (Python)

```python
import azure.functions as func
import logging
import json
import os
from datetime import datetime
from azure.storage.blob import BlobServiceClient
import requests

app = func.FunctionApp()

ACTIVEGATE_ENDPOINT = os.environ["ACTIVEGATE_ENDPOINT"]
ACTIVEGATE_TOKEN = os.environ["ACTIVEGATE_TOKEN"]
STORAGE_CONNECTION = os.environ["STORAGE_CONNECTION"]
STAGING_CONTAINER = os.environ.get("STAGING_CONTAINER", "log-staging")
FAILED_CONTAINER = os.environ.get("FAILED_CONTAINER", "log-failed")

blob_service = BlobServiceClient.from_connection_string(STORAGE_CONNECTION)

@app.function_name(name="EventHubLogForwarder")
@app.event_hub_message_trigger(
    arg_name="events",
    event_hub_name="eventhub-aad-logs",
    connection="EVENTHUB_CONNECTION",
    consumer_group="function-consumer",
    cardinality="many"
)
def eventhub_log_forwarder(events: list[func.EventHubEvent]):
    batch_id = datetime.utcnow().strftime("%Y%m%d%H%M%S%f")
    logs = []
    
    for event in events:
        try:
            log_data = json.loads(event.get_body().decode('utf-8'))
            # Handle Azure diagnostic log format (records array)
            if "records" in log_data:
                for record in log_data["records"]:
                    logs.append(transform_log(record))
            else:
                logs.append(transform_log(log_data))
        except Exception as e:
            logging.error(f"Failed to parse event: {e}")
    
    if not logs:
        return
    
    # Stage to blob storage
    staging_blob_name = f"{datetime.utcnow().strftime('%Y/%m/%d/%H')}/{batch_id}.json"
    staging_container = blob_service.get_container_client(STAGING_CONTAINER)
    staging_blob = staging_container.get_blob_client(staging_blob_name)
    staging_blob.upload_blob(json.dumps(logs), overwrite=True)
    
    logging.info(f"Staged {len(logs)} logs to {staging_blob_name}")
    
    # Forward to ActiveGate
    try:
        response = requests.post(
            ACTIVEGATE_ENDPOINT,
            headers={
                "Authorization": f"Api-Token {ACTIVEGATE_TOKEN}",
                "Content-Type": "application/json; charset=utf-8"
            },
            json=logs,
            timeout=30,
            verify=True
        )
        response.raise_for_status()
        
        # Success - delete staging blob
        staging_blob.delete_blob()
        logging.info(f"Successfully forwarded {len(logs)} logs to ActiveGate")
        
    except requests.exceptions.RequestException as e:
        logging.error(f"Failed to forward logs to ActiveGate: {e}")
        # Move to failed container for retry
        failed_container = blob_service.get_container_client(FAILED_CONTAINER)
        failed_blob = failed_container.get_blob_client(staging_blob_name)
        failed_blob.start_copy_from_url(staging_blob.url)
        staging_blob.delete_blob()
        raise

def transform_log(record: dict) -> dict:
    """Transform Azure log record to Dynatrace format."""
    return {
        "content": json.dumps(record),
        "log.source": "azure",
        "timestamp": record.get("time", datetime.utcnow().isoformat()),
        "cloud.provider": "azure",
        "azure.resource.id": record.get("resourceId", ""),
        "azure.category": record.get("category", ""),
        "azure.operation.name": record.get("operationName", ""),
        "azure.correlation.id": record.get("correlationId", ""),
        "severity": map_severity(record.get("level", record.get("resultType", "INFO")))
    }

def map_severity(level: str) -> str:
    """Map Azure log level to Dynatrace severity."""
    mapping = {
        "Critical": "ERROR",
        "Error": "ERROR", 
        "Warning": "WARN",
        "Informational": "INFO",
        "Information": "INFO",
        "Verbose": "DEBUG",
        "Success": "INFO",
        "Failure": "ERROR"
    }
    return mapping.get(level, "INFO")
```

#### Terraform Example

```hcl
resource "azurerm_linux_function_app" "log_forwarder" {
  name                = "func-log-forwarder-${var.environment}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  storage_account_name       = azurerm_storage_account.function.name
  storage_account_access_key = azurerm_storage_account.function.primary_access_key
  service_plan_id            = azurerm_service_plan.premium.id

  virtual_network_subnet_id = azurerm_subnet.function.id

  site_config {
    always_on                = true
    vnet_route_all_enabled   = true
    elastic_instance_minimum = 2

    application_stack {
      python_version = "3.11"
    }
  }

  app_settings = {
    "EVENTHUB_CONNECTION"      = "@Microsoft.KeyVault(SecretUri=${azurerm_key_vault_secret.eventhub_connection.id})"
    "STORAGE_CONNECTION"       = "@Microsoft.KeyVault(SecretUri=${azurerm_key_vault_secret.storage_connection.id})"
    "ACTIVEGATE_ENDPOINT"      = "https://${azurerm_lb.activegate.private_ip_address}:9999/e/${var.dynatrace_env_id}/api/v2/logs/ingest"
    "ACTIVEGATE_TOKEN"         = "@Microsoft.KeyVault(SecretUri=${azurerm_key_vault_secret.dt_token.id})"
    "STAGING_CONTAINER"        = "log-staging"
    "FAILED_CONTAINER"         = "log-failed"
    "CHECKPOINT_CONTAINER"     = "log-checkpoint"
    "BATCH_SIZE"               = "1000"
    "BATCH_INTERVAL_SECONDS"   = "30"
    "FUNCTIONS_WORKER_RUNTIME" = "python"
  }

  identity {
    type = "SystemAssigned"
  }
}
```

### 7. Dynatrace ActiveGate (VNET-B)

The ActiveGate cluster receives logs from the Azure Function and forwards to Dynatrace SaaS.

#### Deployment Architecture

| Component | Configuration | Purpose |
|-----------|---------------|---------|
| **VM Size** | Standard_D4s_v5 (4 vCPU, 16GB) | Log ingest processing capacity |
| **Instance Count** | 2 (Active-Active) | High availability |
| **Load Balancer** | Internal LB, port 9999 | HA endpoint for Function App |
| **Availability** | Availability Set or Zones | Fault tolerance |

#### ActiveGate Configuration

| Setting | Value |
|---------|-------|
| **Deployment Mode** | Environment ActiveGate |
| **Network Zone** | `azure-log-ingest` |
| **Modules Enabled** | Log Ingest, Metrics Ingest |
| **Listening Port** | 9999 |
| **TLS Certificate** | Internal CA or Let's Encrypt |

#### Installation Steps

1. **Download ActiveGate installer** from Dynatrace environment:
   - Settings → Deployment → ActiveGate → Linux
   - Copy installer URL with environment token

2. **Install ActiveGate on each VM**:

```bash
# Download installer
wget -O /tmp/Dynatrace-ActiveGate-Linux-x86.sh \
  "https://{env-id}.live.dynatrace.com/api/v1/deployment/installer/gateway/unix/latest?arch=x86&flavor=default" \
  --header="Authorization: Api-Token {PAAS_TOKEN}"

# Install with custom configuration
sudo /bin/sh /tmp/Dynatrace-ActiveGate-Linux-x86.sh \
  --set-network-zone=azure-log-ingest \
  --set-server=https://{env-id}.live.dynatrace.com/communication
```

3. **Enable Log Ingest Module**:

Edit `/var/lib/dynatrace/gateway/config/custom.properties`:

```properties
[collector]
MSGrouter = true

[http.client.external]
hostname-verification = yes

[connectivity]
networkZone = azure-log-ingest
```

4. **Configure TLS Certificate**:

```bash
# Import certificate to ActiveGate
sudo /opt/dynatrace/gateway/jre/bin/keytool -importkeystore \
  -srckeystore /path/to/certificate.pfx \
  -srcstoretype pkcs12 \
  -destkeystore /var/lib/dynatrace/gateway/ssl/keystore.jks \
  -deststoretype JKS
```

#### Internal Load Balancer Configuration

```hcl
resource "azurerm_lb" "activegate" {
  name                = "lb-activegate"
  location            = azurerm_resource_group.activegate.location
  resource_group_name = azurerm_resource_group.activegate.name
  sku                 = "Standard"

  frontend_ip_configuration {
    name                          = "activegate-frontend"
    subnet_id                     = azurerm_subnet.activegate.id
    private_ip_address_allocation = "Static"
    private_ip_address            = "10.1.0.10"
  }
}

resource "azurerm_lb_backend_address_pool" "activegate" {
  loadbalancer_id = azurerm_lb.activegate.id
  name            = "activegate-backend-pool"
}

resource "azurerm_lb_probe" "activegate" {
  loadbalancer_id = azurerm_lb.activegate.id
  name            = "activegate-health-probe"
  protocol        = "Https"
  port            = 9999
  request_path    = "/rest/health"
}

resource "azurerm_lb_rule" "activegate" {
  loadbalancer_id                = azurerm_lb.activegate.id
  name                           = "activegate-log-ingest"
  protocol                       = "Tcp"
  frontend_port                  = 9999
  backend_port                   = 9999
  frontend_ip_configuration_name = "activegate-frontend"
  backend_address_pool_ids       = [azurerm_lb_backend_address_pool.activegate.id]
  probe_id                       = azurerm_lb_probe.activegate.id
}
```

#### Dynatrace API Token Requirements

| Scope | Permission | Purpose |
|-------|------------|---------|
| `logs.ingest` | Write | Ingest logs via API |
| `metrics.ingest` | Write | Ingest custom metrics (optional) |

### 8. VNet Peering Configuration

Enable connectivity between VNET-A (Function) and VNET-B (ActiveGate).

```hcl
resource "azurerm_virtual_network_peering" "vnet_a_to_b" {
  name                      = "peer-vneta-to-vnetb"
  resource_group_name       = azurerm_resource_group.vnet_a.name
  virtual_network_name      = azurerm_virtual_network.vnet_a.name
  remote_virtual_network_id = azurerm_virtual_network.vnet_b.id

  allow_virtual_network_access = true
  allow_forwarded_traffic      = false
  allow_gateway_transit        = false
  use_remote_gateways          = false
}

resource "azurerm_virtual_network_peering" "vnet_b_to_a" {
  name                      = "peer-vnetb-to-vneta"
  resource_group_name       = azurerm_resource_group.vnet_b.name
  virtual_network_name      = azurerm_virtual_network.vnet_b.name
  remote_virtual_network_id = azurerm_virtual_network.vnet_a.id

  allow_virtual_network_access = true
  allow_forwarded_traffic      = false
  allow_gateway_transit        = false
  use_remote_gateways          = false
}
```

#### Network Security Group Rules

**VNET-A Function Subnet NSG**:

| Rule | Direction | Source | Destination | Port | Action |
|------|-----------|--------|-------------|------|--------|
| AllowVnetBOutbound | Outbound | snet-function | VNET-B CIDR | 9999 | Allow |
| AllowStorageOutbound | Outbound | snet-function | Storage PE | 443 | Allow |
| AllowEventHubOutbound | Outbound | snet-function | EventHub PE | 443,5671 | Allow |

**VNET-B ActiveGate Subnet NSG**:

| Rule | Direction | Source | Destination | Port | Action |
|------|-----------|--------|-------------|------|--------|
| AllowFunctionInbound | Inbound | VNET-A CIDR | snet-activegate | 9999 | Allow |
| AllowDynatraceSaaSOutbound | Outbound | snet-activegate | Internet | 443 | Allow |

---

## Security Considerations

### Network Security

| Control | Implementation | Rationale |
|---------|----------------|-----------|
| **Private Endpoints** | Event Hub, Blob Storage with private endpoints in VNET-A | No public internet exposure for data plane |
| **VNet Peering** | Controlled peering between VNET-A and VNET-B | Isolated network segments with explicit connectivity |
| **NSG Rules** | Restrictive inbound/outbound rules per subnet | Least-privilege network access |
| **TLS 1.2+** | Enforce minimum TLS on all endpoints | Encryption in transit |
| **NAT Gateway** | VNET-B outbound via NAT Gateway | Controlled egress for ActiveGate |

### Identity & Access

| Control | Implementation |
|---------|----------------|
| **Managed Identity** | Function App uses system-assigned identity for Blob/EventHub access |
| **RBAC** | `Event Hubs Data Receiver` for Function, `Storage Blob Data Contributor` for checkpoints |
| **Key Vault** | All secrets stored in Key Vault, referenced via `@Microsoft.KeyVault()` |
| **Key Rotation** | 90-day rotation for API tokens and connection strings |

### Data Protection

| Control | Implementation |
|---------|----------------|
| **Encryption at Rest** | Customer-managed keys for Storage and Event Hub |
| **Encryption in Transit** | TLS 1.2+ for all communications |
| **Data Masking** | Log processing rules to mask PII before Dynatrace storage |
| **Blob Lifecycle** | Auto-delete staging blobs after 24 hours, failed after 7 days |

---

## Operational Considerations

### Monitoring the Pipeline

Deploy monitoring for the log ingestion pipeline itself:

| Metric | Source | Alert Threshold |
|--------|--------|-----------------|
| `IncomingMessages` | Event Hub | < expected baseline (log source failure) |
| `OutgoingMessages` | Event Hub | Consumer lag > 10,000 messages |
| `ThrottledRequests` | Event Hub | > 0 sustained (scale PUs) |
| `FunctionExecutionCount` | Function App | < expected (function failure) |
| `FunctionExecutionDuration` | Function App | p95 > 60s (performance degradation) |
| `FunctionErrors` | Function App | > 0 sustained (code/connectivity issue) |
| `Blob Count (log-failed)` | Blob Storage | > 0 (delivery failures) |
| `ActiveGate Health` | Dynatrace | Unhealthy status |
| `Log Ingest Volume` | Dynatrace | < baseline (pipeline break) |

### Alerting Rules

```hcl
resource "azurerm_monitor_metric_alert" "function_errors" {
  name                = "alert-function-errors"
  resource_group_name = azurerm_resource_group.main.name
  scopes              = [azurerm_linux_function_app.log_forwarder.id]
  description         = "Alert when function execution errors exceed threshold"

  criteria {
    metric_namespace = "Microsoft.Web/sites"
    metric_name      = "FunctionExecutionCount"
    aggregation      = "Total"
    operator         = "LessThan"
    threshold        = 100  # Expected minimum per 5 minutes
  }

  window_size        = "PT5M"
  frequency          = "PT1M"
  severity           = 2

  action {
    action_group_id = azurerm_monitor_action_group.ops.id
  }
}

resource "azurerm_monitor_metric_alert" "failed_blobs" {
  name                = "alert-failed-log-blobs"
  resource_group_name = azurerm_resource_group.main.name
  scopes              = [azurerm_storage_account.log_staging.id]
  description         = "Alert when failed log container has blobs"

  criteria {
    metric_namespace = "Microsoft.Storage/storageAccounts/blobServices"
    metric_name      = "BlobCount"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 0

    dimension {
      name     = "BlobType"
      operator = "Include"
      values   = ["BlockBlob"]
    }
  }

  window_size = "PT15M"
  frequency   = "PT5M"
  severity    = 2

  action {
    action_group_id = azurerm_monitor_action_group.ops.id
  }
}
```

### Scaling Guidelines

| Log Volume (Events/day) | Event Hub | Function Plan | ActiveGate |
|-------------------------|-----------|---------------|------------|
| < 1 million | 1 PU, 4 partitions | EP1 (2 instances) | 2x D2s_v5 |
| 1-10 million | 1-2 PU, 8 partitions | EP2 (3 instances) | 2x D4s_v5 |
| 10-100 million | 2-4 PU, 16 partitions | EP3 (5 instances) | 2x D8s_v5 |
| > 100 million | 4+ PU, 32 partitions | EP3 (10 instances) | 4x D8s_v5 |

### Cost Optimization

| Strategy | Implementation |
|----------|----------------|
| **Filter at Source** | Export only required log categories in diagnostic settings |
| **Batch Processing** | Aggregate logs before forwarding (reduce API calls) |
| **Blob Lifecycle** | Aggressive cleanup of staging blobs (24hr) |
| **Reserved Capacity** | Reserved instances for ActiveGate VMs |
| **Function Plan Right-sizing** | Monitor utilization, scale plan tier as needed |

---

## Disaster Recovery

### Recovery Objectives

| Objective | Target | Strategy |
|-----------|--------|----------|
| **RTO** | 4 hours | Redeploy from IaC, restore from backup |
| **RPO** | 1 hour | Event Hub retention (7 days), Blob staging |

### Failure Scenarios

| Scenario | Impact | Recovery |
|----------|--------|----------|
| **Function App Failure** | Logs queue in Event Hub | Auto-restart, Event Hub retention provides buffer |
| **ActiveGate Failure** | Logs stage in Blob Storage | HA pair provides failover, retry from failed container |
| **Event Hub Failure** | New logs lost | Zonal redundancy (Premium), geo-DR for critical |
| **Blob Storage Failure** | Checkpoint loss, duplicate logs | ZRS replication, idempotent processing |
| **VNet Peering Failure** | Function cannot reach ActiveGate | Logs stage in Blob, alert and remediate |

### Retry Logic

The pipeline implements multi-level retry:

1. **Event Hub** → Function: Built-in retry with checkpoint (at-least-once delivery)
2. **Function** → Blob Staging: Synchronous write before forward attempt
3. **Function** → ActiveGate: 3 retries with exponential backoff (1s, 5s, 30s)
4. **Failed Logs**: Timer-triggered function processes `log-failed` container hourly

---

## Troubleshooting

### Common Issues

| Symptom | Possible Cause | Resolution |
|---------|----------------|------------|
| No logs in Dynatrace | Diagnostic setting misconfigured | Verify Event Hub name and auth rule |
| Logs delayed | Function cold start or throttling | Check function metrics, scale Premium plan |
| High failed blob count | ActiveGate unreachable | Verify VNet peering, NSG rules, ActiveGate health |
| Duplicate logs | Checkpoint failure, function retry | Implement idempotency key in log processing |
| Missing Azure AD logs | Insufficient Entra ID license | Verify P1/P2 license for sign-in logs |
| ActiveGate unhealthy | Memory/disk exhaustion | Scale VM size, check log volume |

### Diagnostic Commands

**Check Event Hub consumer lag**:

```bash
az eventhubs eventhub consumer-group list \
  --resource-group rg-logging \
  --namespace-name evhns-logs \
  --eventhub-name eventhub-aad-logs
```

**Check Function App logs**:

```bash
az webapp log tail \
  --name func-log-forwarder \
  --resource-group rg-logging
```

**Check failed blob count**:

```bash
az storage blob list \
  --account-name stlogstaging \
  --container-name log-failed \
  --query "length(@)"
```

**Test ActiveGate connectivity from Function VNet**:

```bash
# Deploy a test container in the Function subnet
az container create \
  --resource-group rg-logging \
  --name test-connectivity \
  --image curlimages/curl \
  --vnet vnet-logging-a \
  --subnet snet-function \
  --command-line "curl -v https://10.1.0.10:9999/rest/health"
```

### Validation Queries

**Verify logs are arriving (Dynatrace DQL)**:

```dql
fetch logs
| filter cloud.provider == "azure"
| summarize count(), by:{azure.category, bin(timestamp, 1h)}
| sort timestamp desc
```

**Check for ingestion latency**:

```dql
fetch logs
| filter cloud.provider == "azure"
| filter timestamp > now() - 1h
| fields ingestionDelay = now() - timestamp
| summarize avg(ingestionDelay), max(ingestionDelay), by:{azure.category}
```

**Identify missing log categories**:

```dql
fetch logs
| filter cloud.provider == "azure"
| filter timestamp > now() - 24h
| summarize lastSeen = max(timestamp), by:{azure.category}
| filter lastSeen < now() - 1h
```

---

## References

- [Dynatrace Azure Log Forwarding Documentation](https://docs.dynatrace.com/docs/setup-and-configuration/setup-on-cloud-platforms/microsoft-azure-services/azure-log-forwarder)
- [Dynatrace ActiveGate Documentation](https://docs.dynatrace.com/docs/setup-and-configuration/dynatrace-activegate)
- [Azure Event Hubs Documentation](https://learn.microsoft.com/en-us/azure/event-hubs/)
- [Azure Functions Event Hub Trigger](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-hubs-trigger)
- [Azure AD Diagnostic Settings](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-integrate-activity-logs-with-azure-monitor-logs)
- [Azure Activity Log Schema](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/activity-log-schema)
- [Azure Resource Log Categories](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/resource-logs-categories)
- [Azure Private Endpoints](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview)
- [Dynatrace Log Processing](https://docs.dynatrace.com/docs/observe-and-explore/logs/log-management-and-analytics/lma-log-processing)

## Questions on Implementation - Performance and Error Handling (Network and Processing)

The following questions should be addressed during implementation planning to ensure the architecture meets operational requirements.

---

### 1. Azure Event Hub

#### Capacity & Performance

- [ ] What is the expected daily log volume (events/day, MB/day) for each log type (Azure AD, Activity, Resource)?
- [ ] What is the peak ingestion rate (events/second) during high-activity periods (e.g., business hours, security incidents)?
- [ ] How many Processing Units (PUs) are required to handle peak load with 20% headroom?
- [ ] What partition count provides optimal parallelism for consumer processing without over-provisioning?
- [ ] Should auto-inflate be enabled, and what is the maximum PU limit to control costs?

#### Reliability & Error Handling

- [ ] What is the acceptable message retention period if downstream processing fails (1 day vs 7 days)?
- [ ] How will consumer lag be monitored and what threshold triggers alerting?
- [ ] What is the recovery procedure if Event Hub reaches capacity and starts throttling?
- [ ] Should geo-disaster recovery be configured for cross-region failover?
- [ ] How are poison messages (malformed events) handled to prevent consumer blocking?

#### Network & Security

- [ ] Is the Event Hub namespace deployed with Premium tier for private endpoint support?
- [ ] What DNS resolution strategy is used for private endpoints (Azure Private DNS Zone, custom DNS)?
- [ ] Are there existing firewall rules or service endpoints that could conflict with private endpoint routing?
- [ ] What Azure Policy constraints apply to Event Hub deployment (e.g., require private endpoints, disable public access)?

---

### 2. Azure AD / Entra ID Log Export

#### Scope & Coverage

- [ ] Which Azure AD log categories are required (SignInLogs, AuditLogs, RiskyUsers, etc.)?
- [ ] Are there multiple Azure AD tenants that need log export configured?
- [ ] What Entra ID license tier is in place (Free, P1, P2)? Sign-in logs require P1+.
- [ ] Are Conditional Access and Identity Protection logs required (P2 license)?

#### Performance & Volume

- [ ] What is the expected volume of sign-in events per day (interactive + non-interactive)?
- [ ] During peak authentication periods, what is the burst rate?
- [ ] Are there service principals or managed identities with high-frequency authentication that may dominate log volume?

#### Error Handling

- [ ] How will failed diagnostic setting exports be detected and alerted?
- [ ] What is the procedure if Azure AD logs stop flowing (license expiry, permission change)?
- [ ] How are schema changes in Azure AD log format handled by downstream processing?

---

### 3. Azure Activity Log Export

#### Scope & Coverage

- [ ] How many Azure subscriptions require Activity Log export?
- [ ] Is there a management group-level diagnostic setting, or per-subscription configuration?
- [ ] Which Activity Log categories are required (Administrative, Security, ServiceHealth, Policy, etc.)?

#### Performance & Volume

- [ ] What is the expected volume of control plane operations per subscription?
- [ ] Are there automation or IaC pipelines that generate high volumes of ARM operations?
- [ ] Should Resource Health events be included (can be high volume for large deployments)?

#### Error Handling

- [ ] How is diagnostic setting drift detected (settings removed or modified)?
- [ ] What is the procedure when new subscriptions are created—manual or automated diagnostic setup?

---

### 4. Azure Resource Log Export

#### Scope & Coverage

- [ ] Which Azure resource types require diagnostic log export?
- [ ] What log categories are required per resource type (e.g., AKS: kube-apiserver, kube-audit)?
- [ ] Is there a policy-driven approach to enforce diagnostic settings on all resources?
- [ ] How are newly deployed resources automatically configured for log export?

#### Performance & Volume

- [ ] Which resources generate the highest log volume (AKS, Application Gateway, Firewall)?
- [ ] Are verbose log categories (e.g., kube-audit-admin) required, or can they be filtered?
- [ ] What sampling strategy, if any, should be applied to high-volume resource logs?

#### Error Handling

- [ ] How is diagnostic setting compliance monitored across all resources?
- [ ] What is the remediation process for resources missing diagnostic settings?
- [ ] How are transient export failures (e.g., Event Hub throttling) surfaced?

---

### 5. Azure Blob Storage (Log Staging)

#### Capacity & Performance

- [ ] What is the expected storage consumption for staged logs (based on batch size and retention)?
- [ ] Is Standard storage sufficient, or is Premium required for low-latency access?
- [ ] What is the expected blob count in staging container during normal operation?
- [ ] Should soft delete be enabled for accidental deletion recovery?

#### Reliability & Error Handling

- [ ] What is the lifecycle policy retention for staged logs (24 hours proposed)?
- [ ] What is the retention for failed delivery logs (7 days proposed)?
- [ ] How is blob write failure handled in the Function (retry, dead-letter)?
- [ ] Is immutable storage required for compliance (WORM)?

#### Network & Security

- [ ] Is the storage account configured with private endpoint only (no public access)?
- [ ] What is the minimum TLS version enforced?
- [ ] Are customer-managed keys (CMK) required for encryption at rest?
- [ ] What RBAC roles are assigned to the Function App managed identity?

---

### 6. Azure Function App (Log Forwarder)

#### Capacity & Performance

- [ ] What Function App plan tier is appropriate (Premium EP1/EP2/EP3)?
- [ ] What is the minimum instance count for high availability (2+ recommended)?
- [ ] What is the maximum instance count to handle burst scenarios?
- [ ] What batch size optimizes throughput vs. latency (100, 500, 1000 events)?
- [ ] What is the maximum execution duration before timeout (default 5 min, max 10 min)?

#### Reliability & Error Handling

- [ ] What retry policy is implemented for ActiveGate delivery failures?
- [ ] How many retry attempts before moving to failed container (3 proposed)?
- [ ] What is the exponential backoff schedule (1s, 5s, 30s proposed)?
- [ ] How are failed logs reprocessed from the failed container (timer trigger, manual)?
- [ ] Is there a dead-letter mechanism for permanently failed logs?
- [ ] How is idempotency ensured to prevent duplicate log ingestion on retries?

#### Network & Security

- [ ] Is the Function App VNet-integrated with route-all enabled?
- [ ] What outbound IP addresses are used (NAT Gateway, Azure Firewall)?
- [ ] Are all secrets stored in Key Vault with managed identity access?
- [ ] What is the Key Vault firewall configuration for Function access?

#### Monitoring & Alerting

- [ ] What Application Insights configuration is in place for Function telemetry?
- [ ] What metrics trigger alerts (execution failures, duration, throttling)?
- [ ] How is consumer lag from Event Hub monitored?
- [ ] What is the alerting threshold for failed blob count?

---

### 7. Dynatrace ActiveGate

#### Capacity & Performance

- [ ] What VM size is appropriate for expected log volume (D2s_v5, D4s_v5, D8s_v5)?
- [ ] How many ActiveGate nodes are required for high availability (2 minimum)?
- [ ] What is the expected CPU and memory utilization under peak load?
- [ ] Is the Log Ingest module enabled and configured?
- [ ] What is the ActiveGate version, and is it compatible with the Dynatrace SaaS cluster?

#### Reliability & Error Handling

- [ ] How is ActiveGate health monitored (Dynatrace built-in, Azure VM metrics)?
- [ ] What is the failover behavior when one ActiveGate node fails?
- [ ] How are ActiveGate restarts handled (automatic, manual intervention)?
- [ ] What disk space is allocated for local buffering during outages?
- [ ] What is the retention period for locally buffered data?

#### Network & Security

- [ ] Is the internal load balancer configured with appropriate health probes?
- [ ] What TLS certificate is used for the ActiveGate endpoint (internal CA, public CA)?
- [ ] How is certificate renewal managed (automated, manual)?
- [ ] What NSG rules restrict inbound traffic to the ActiveGate subnet?
- [ ] What outbound connectivity is required to Dynatrace SaaS (HTTPS/443)?
- [ ] Is egress controlled via NAT Gateway, Azure Firewall, or direct internet?

#### Dynatrace Configuration

- [ ] What API token scopes are required (`logs.ingest`, `metrics.ingest`)?
- [ ] What is the token rotation policy (90 days recommended)?
- [ ] Is a network zone configured for the ActiveGate?
- [ ] What Dynatrace environment (production, non-production) receives the logs?

---

### 8. VNet Peering & Network Connectivity

#### Architecture & Design

- [ ] What are the CIDR ranges for VNET-A and VNET-B (ensure no overlap)?
- [ ] Is bidirectional peering required, or is one-way sufficient?
- [ ] Are there existing hub-spoke or transit network designs that affect routing?
- [ ] Is Azure Firewall or NVA in the path between VNets?

#### Performance & Reliability

- [ ] What is the expected bandwidth between VNET-A and VNET-B?
- [ ] Is peering latency acceptable for real-time log forwarding (<5ms typical)?
- [ ] How is peering health monitored and alerted?
- [ ] What is the failover path if peering fails (no alternative proposed)?

#### Security & Compliance

- [ ] Are NSG rules configured to restrict traffic to required ports only (9999)?
- [ ] Is NSG flow logging enabled for audit and troubleshooting?
- [ ] Are there Azure Policy constraints on peering (e.g., require approval)?
- [ ] Is traffic inspection required between VNets (Azure Firewall)?

---

### 9. Dynatrace SaaS Integration

#### Configuration

- [ ] What is the Dynatrace environment ID and cluster URL?
- [ ] What log retention policy applies in Dynatrace (7 days, 35 days, custom)?
- [ ] Are log processing rules required for parsing and enrichment?
- [ ] What log attributes should be extracted and indexed?

#### Performance & Limits

- [ ] What is the Dynatrace log ingest rate limit for the environment?
- [ ] Is the current DDU (Davis Data Units) quota sufficient for expected volume?
- [ ] What is the expected DDU consumption for Azure logs?

#### Error Handling

- [ ] How are Dynatrace API errors (429, 500) handled by the pipeline?
- [ ] What is the notification mechanism for Dynatrace ingestion failures?
- [ ] How is log delivery completeness validated?

---

### 10. Cross-Cutting Concerns

#### Disaster Recovery

- [ ] What is the target RTO (Recovery Time Objective) for the pipeline?
- [ ] What is the target RPO (Recovery Point Objective) for log data?
- [ ] Is there a secondary region deployment for geo-redundancy?
- [ ] What is the procedure for failing over to secondary region?

#### Compliance & Governance

- [ ] What data classification applies to the logs (confidential, internal, public)?
- [ ] Are there data residency requirements restricting region deployment?
- [ ] What audit logging is required for the pipeline itself?
- [ ] Is PII masking required before logs reach Dynatrace?

#### Cost Management

- [ ] What is the estimated monthly cost for each component?
- [ ] Are reserved instances appropriate for predictable workloads (VMs, Functions)?
- [ ] What cost alerts and budgets are configured?
- [ ] How is log volume growth projected and budgeted?

#### Operational Readiness

- [ ] What runbooks are required for common operational tasks?
- [ ] Who is responsible for monitoring and maintaining each component?
- [ ] What is the escalation path for pipeline failures?
- [ ] What is the change management process for pipeline modifications?
- [ ] Is there a staging/test environment for validating changes before production?
