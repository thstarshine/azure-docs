﻿---
title: Collect Azure service logs and metrics for Log Analytics | Microsoft Docs
description: Configure diagnostics on Azure resources to write logs and metrics to Log Analytics.
services: log-analytics
documentationcenter: ''
author: bandersmsft
manager: carmonm
editor: ''
ms.assetid: 84105740-3697-4109-bc59-2452c1131bfe
ms.service: log-analytics
ms.workload: na
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 04/12/2017
ms.author: banders
ms.custom: H1Hack27Feb2017

---
# Collect Azure service logs and metrics for use in Log Analytics

There are four different ways of collecting logs and metrics for Azure services:

1. Azure diagnostics direct to Log Analytics (*Diagnostics* in the following table)
2. Azure diagnostics to Azure storage to Log Analytics (*Storage* in the following table)
3. Connectors for Azure services (*Connectors* in the following table)
4. Scripts to collect and then post data into Log Analytics (blanks in the following table and for services that are not listed)


| Service                 | Resource Type                           | Logs        | Metrics     | Solution |
| --- | --- | --- | --- | --- |
| Application gateways    | Microsoft.Network/applicationGateways   | Diagnostics | Diagnostics | [Azure Application Gateway Analytics](log-analytics-azure-networking-analytics.md#azure-application-gateway-analytics-solution-in-log-analytics) |
| Application insights    |                                         | Connector   | Connector   | [Application Insights Connector](https://blogs.technet.microsoft.com/msoms/2016/09/26/application-insights-connector-in-oms/) (Preview) |
| Automation accounts     | Microsoft.Automation/AutomationAccounts | Diagnostics |             | [More information](../automation/automation-manage-send-joblogs-log-analytics.md)|
| Batch accounts          | Microsoft.Batch/batchAccounts           | Diagnostics | Diagnostics | |
| Classic cloud services  |                                         | Storage     |             | [More information](log-analytics-azure-storage-iis-table.md) |
| Cognitive services      | Microsoft.CognitiveServices/accounts    |             | Diagnostics | |
| Data Lake analytics     | Microsoft.DataLakeAnalytics/accounts    | Diagnostics |             | |
| Data Lake store         | Microsoft.DataLakeStore/accounts        | Diagnostics |             | |
| Event Hub namespace     | Microsoft.EventHub/namespaces           | Diagnostics | Diagnostics | |
| IoT Hubs                | Microsoft.Devices/IotHubs               |             | Diagnostics | |
| Key Vault               | Microsoft.KeyVault/vaults               | Diagnostics |             | [KeyVault Analytics](log-analytics-azure-key-vault.md) |
| Load Balancers          | Microsoft.Network/loadBalancers         | Diagnostics |             |  |
| Logic Apps              | Microsoft.Logic/workflows <br> Microsoft.Logic/integrationAccounts | Diagnostics | Diagnostics | |
| Network Security Groups | Microsoft.Network/networksecuritygroups | Diagnostics |             | [Azure Network Security Group Analytics](log-analytics-azure-networking-analytics.md#azure-network-security-group-analytics-solution-in-log-analytics) |
| Recovery vaults         | Microsoft.RecoveryServices/vaults       |             |             | [Azure Recovery Services Analytics (Preview)](https://github.com/krnese/AzureDeploy/blob/master/OMS/MSOMS/Solutions/recoveryservices/)|
| Search services         | Microsoft.Search/searchServices         | Diagnostics | Diagnostics | |
| Service Bus namespace   | Microsoft.ServiceBus/namespaces         | Diagnostics | Diagnostics | [Service Bus Analytics (Preview)](https://github.com/Azure/azure-quickstart-templates/tree/master/oms-servicebus-solution)|
| Service Fabric          |                                         | Storage     |             | [Service Fabric Analytics (Preview)](log-analytics-service-fabric.md) |
| SQL (v12)               | Microsoft.Sql/servers/databases <br> Microsoft.Sql/servers/elasticPools |             | Diagnostics | [Azure SQL Analytics (Preview)](log-analytics-azure-sql.md) |
| Storage                 |                                         |             | Script      | [Azure Storage Analytics (Preview)](https://github.com/Azure/azure-quickstart-templates/tree/master/oms-azure-storage-analytics-solution) |
| Virtual Machines        | Microsoft.Compute/virtualMachines       | Extension   | Extension <br> Diagnostics  | |
| Virtual Machines scale sets | Microsoft.Compute/virtualMachines <br> Microsoft.Compute/virtualMachineScaleSets/virtualMachines |             | Diagnostics | |
| Web Server farms        | Microsoft.Web/serverfarms               |             | Diagnostics | |
| Web Sites               | Microsoft.Web/sites <br> Microsoft.Web/sites/slots |             | Diagnostics | [Azure Web Apps Analytics (Preview)](https://azuremarketplace.microsoft.com/marketplace/apps/Microsoft.AzureWebAppsAnalyticsOMS?tab=Overview) |


> [!NOTE]
> For monitoring Azure virtual machines (both Linux and Windows), we recommend installing the [Log Analytics VM extension](log-analytics-azure-vm-extension.md). The agent provides you with insights collected from within your virtual machines. You can also use the extension for Virtual machine scale sets.
>
>

## Azure diagnostics direct to Log Analytics
Many Azure resources are able to write diagnostic logs and metrics directly to Log Analytics and this is the preferred way of collecting the data for analysis. When using Azure diagnostics, data is written immediately to Log Analytics and there is no need to first write the data to storage.

Azure resources that support [Azure monitor](../monitoring-and-diagnostics/monitoring-overview.md) can send their logs and metrics directly to Log Analytics.

* For the details of the available metrics, refer to [supported metrics with Azure Monitor](../monitoring-and-diagnostics/monitoring-supported-metrics.md).
* For the details of the available logs, refer to [supported services and schema for diagnostic logs](../monitoring-and-diagnostics/monitoring-overview-of-diagnostic-logs.md#supported-services-and-schema-for-diagnostic-logs).

### Enable diagnostics with PowerShell
You need the November 2016 (v2.3.0) or later release of [Azure PowerShell](/powershell/azure/overview).

The following PowerShell example shows how to use [Set-AzureRmDiagnosticSetting](/powershell/module/azurerm.insights/set-azurermdiagnosticsetting) to enable diagnostics on a network security group. The same approach works for all supported resources - set `$resourceId` to the resource id of the resource you want to enable diagnostics for.

```powershell
$workspaceId = "/subscriptions/d2e37fee-1234-40b2-5678-0b2199de3b50/resourcegroups/oi-default-east-us/providers/microsoft.operationalinsights/workspaces/rollingbaskets"

$resourceId = "/SUBSCRIPTIONS/ec11ca60-1234-491e-5678-0ea07feae25c/RESOURCEGROUPS/DEMO/PROVIDERS/MICROSOFT.NETWORK/NETWORKSECURITYGROUPS/DEMO"

Set-AzureRmDiagnosticSetting -ResourceId $ResourceId  -WorkspaceId $workspaceId -Enabled $true
```

### Enable diagnostics with Resource Manager templates

To enable diagnostics on a resource when it is created, and have the diagnostics sent to your Log Analytics workspace you can use a template similar to the one below. This example is for an Automation account but works for all supported resource types.

```json
        {
            "type": "Microsoft.Automation/automationAccounts/providers/diagnosticSettings",
            "name": "[concat(parameters('omsAutomationAccountName'), '/', 'Microsoft.Insights/service')]",
            "apiVersion": "2015-07-01",
            "dependsOn": [
                "[concat('Microsoft.Automation/automationAccounts/', parameters('omsAutomationAccountName'))]",
                "[concat('Microsoft.OperationalInsights/workspaces/', parameters('omsWorkspaceName'))]"
            ],
            "properties": {
                "workspaceId": "[resourceId('Microsoft.OperationalInsights/workspaces', parameters('omsWorkspaceName'))]",
                "logs": [
                    {
                        "category": "JobLogs",
                        "enabled": true
                    },
                    {
                        "category": "JobStreams",
                        "enabled": true
                    }
                ]
            }
        }
```

[!INCLUDE [log-analytics-troubleshoot-azure-diagnostics](../../includes/log-analytics-troubleshoot-azure-diagnostics.md)]

## Azure diagnostics to storage then to Log Analytics

For collecting logs from within some resources, it is possible to send the logs to Azure storage and then configure Log Analytics to read the logs from storage.

Log Analytics can use this approach to collect diagnostics from Azure storage for the following resources and logs:

| Resource | Logs |
| --- | --- |
| Service Fabric |ETWEvent <br> Operational Event <br> Reliable Actor Event <br> Reliable Service Event |
| Virtual Machines |Linux Syslog <br> Windows Event <br> IIS Log <br> Windows ETWEvent |
| Web Roles <br> Worker Roles |Linux Syslog <br> Windows Event <br> IIS Log <br> Windows ETWEvent |

> [!NOTE]
> You are charged normal Azure data rates for storage and transactions when you send diagnostics to a storage account and for when Log Analytics reads the data from your storage account.
>
>

See [Use blob storage for IIS and table storage for events](log-analytics-azure-storage-iis-table.md) to learn more about how Log Analytics can collect these logs.

## Connectors for Azure services

There is a connector for Application Insights, which allows data collected by Application Insights to be sent to Log Analytics.

Learn more about the [Application Insights connector](https://blogs.technet.microsoft.com/msoms/2016/09/26/application-insights-connector-in-oms/).

## Scripts to collect and post data to Log Analytics

For Azure services that do not provide a direct way to send logs and metrics to Log Analytics you can use an Azure Automation script to collect the log and metrics. The script can then send the data to Log Analytics using the [data collector API](log-analytics-data-collector-api.md)

The Azure template gallery has [examples of using Azure Automation](https://azure.microsoft.com/en-us/resources/templates/?term=OMS) to collect data from services and sending it to Log Analytics.

## Next steps

* [Use blob storage for IIS and table storage for events](log-analytics-azure-storage-iis-table.md) to read the logs for Azure services that write diagnostics to table storage or IIS logs written to blob storage.
* [Enable Solutions](log-analytics-add-solutions.md) to provide insight into the data.
* [Use search queries](log-analytics-log-searches.md) to analyze the data.
