**Azure Monitor practice lab**: you’ll stand up a Log Analytics workspace, attach a VM via **Azure Monitor Agent (AMA)** using a **Data Collection Rule (DCR)**, enable platform diagnostics, build a couple of alert rules (metric + log), and run quick **KQL** queries. I’m using **centralindia** but feel free to swap regions.

---

# 1) Prereqs & Variables

```bash
# Login & subscription
az login
az account set --subscription "<SUBSCRIPTION_ID>"

# Variables
LOCATION="centralindia"
RG="rg-azmon-lab"
LAW="log-azmon-lab"
VMNAME="vm-azmon-lab"
VNET="vnet-azmon-lab"
SUBNET="snet-azmon-lab"
ADMINUSER="azureuser"
AG_NAME="ag-azmon-email"          # Action Group (email)
EMAIL_TO="<your.email@example.com>"
DCR_NAME="dcr-azmon-ama"
DCE_NAME="dce-azmon"              # Data Collection Endpoint (optional but recommended)
STACCT="stazmon$RANDOM"           # storage account to demo diagnostics
```

---

# 2) Resource Group, Workspace, VM

```bash
# RG
az group create -n $RG -l $LOCATION

# Log Analytics Workspace
az monitor log-analytics workspace create -g $RG -n $LAW -l $LOCATION
LAW_ID=$(az monitor log-analytics workspace show -g $RG -n $LAW --query id -o tsv)

# Networking
az network vnet create -g $RG -n $VNET --address-prefixes 10.10.0.0/16 \
  --subnet-name $SUBNET --subnet-prefixes 10.10.1.0/24

# VM (Linux) – small & quick
az vm create -g $RG -n $VMNAME --image Ubuntu2204 --size Standard_B1s \
  --admin-username $ADMINUSER --generate-ssh-keys \
  --vnet-name $VNET --subnet $SUBNET \
  --public-ip-sku Standard

VM_ID=$(az vm show -g $RG -n $VMNAME --query id -o tsv)
```

---

# 3) Data Collection Endpoint & Rule (AMA)

```bash
# (Optional) Data Collection Endpoint (good for scale/centralization)
az monitor data-collection endpoint create \
  -g $RG -n $DCE_NAME -l $LOCATION --public-network-access Enabled
DCE_ID=$(az monitor data-collection endpoint show -g $RG -n $DCE_NAME --query id -o tsv)

# DCR for performance/heartbeat (Linux perf counters)
cat > dcr.json <<'JSON'
{
  "location": "$LOCATION",
  "properties": {
    "dataFlows": [
      {
        "destinations": [ "lawDest" ],
        "streams": [ "Microsoft-Perf", "Microsoft-Syslog", "Microsoft-InsightsMetrics", "Microsoft-Heartbeat" ]
      }
    ],
    "dataSources": {
      "syslog": [
        {
          "name": "syslogAll",
          "streams": [ "Microsoft-Syslog" ],
          "facilityNames": [ "daemon", "kern", "syslog", "auth", "authpriv", "user" ],
          "logLevels": [ "Error", "Critical", "Alert", "Emergency", "Warning", "Notice", "Info" ]
        }
      ],
      "performanceCounters": [
        {
          "name": "perfLinux",
          "streams": [ "Microsoft-Perf" ],
          "samplingFrequencyInSeconds": 60,
          "counterSpecifiers": [
            "Processor(*)\\% Processor Time",
            "Memory\\Available MBytes",
            "LogicalDisk(*)\\% Free Space"
          ]
        }
      ]
    },
    "destinations": {
      "logAnalytics": [
        {
          "workspaceResourceId": "$LAW_ID",
          "name": "lawDest"
        }
      ]
    }
  }
}
JSON

# Replace tokens in the JSON (simple env expansion)
sed -i.bak "s|\$LOCATION|$LOCATION|g; s|\$LAW_ID|$LAW_ID|g" dcr.json

# Create DCR
az monitor data-collection rule create \
  -g $RG -n $DCR_NAME --rule-file dcr.json

DCR_ID=$(az monitor data-collection rule show -g $RG -n $DCR_NAME --query id -o tsv)

# Associate (scope) DCR to the VM
az monitor data-collection rule association create \
  --name "${DCR_NAME}-assoc" --rule-id $DCR_ID --resource $VM_ID

# Install Azure Monitor Agent on the VM
az vm extension set -g $RG -n AzureMonitorLinuxAgent --publisher Microsoft.Azure.Monitor --vm-name $VMNAME
```

> Within \~5–10 minutes you should see **Heartbeat** and **Perf** data in the workspace.

---

# 4) Enable Platform Logs/Metrics (Diagnostics) on a Resource

We’ll create a Storage Account and send its diagnostics to the workspace.

```bash
# Storage account (for diagnostics demo)
az storage account create -g $RG -n $STACCT -l $LOCATION --sku Standard_LRS --kind StorageV2
ST_ID=$(az storage account show -g $RG -n $STACCT --query id -o tsv)

# Turn on diagnostic settings -> send to Log Analytics (LAW)
az monitor diagnostic-settings create \
  --name "diag-to-law" --resource $ST_ID \
  --workspace $LAW_ID \
  --metrics '[{"category": "Transaction", "enabled": true}]' \
  --logs '[{"category": "StorageRead", "enabled": true},{"category":"StorageWrite","enabled":true},{"category":"StorageDelete","enabled":true}]'
```

---

# 5) Action Group (Email) & Alerts

## 5a) Action Group

```bash
az monitor action-group create -g $RG -n $AG_NAME \
  --action email email01 "$EMAIL_TO"
AG_ID=$(az monitor action-group show -g $RG -n $AG_NAME --query id -o tsv)
```

## 5b) Metric Alert (VM CPU > 80% for 5 mins)

```bash
az monitor metrics alert create \
  -g $RG -n "alert-vm-cpu-high" \
  --scopes $VM_ID \
  --condition "avg Percentage CPU > 80" \
  --description "CPU above 80% for 5 min" \
  --window-size 5m --evaluation-frequency 1m \
  --action $AG_ID
```

## 5c) Log Alert (No heartbeat in 10 minutes)

```bash
# KQL query: no heartbeat for the VM in last 10m
cat > heartbeat-noisy.kql <<'KQL'
Heartbeat
| where TimeGenerated > ago(10m)
| summarize LastBeat=max(TimeGenerated) by Computer
| where datetime_diff('minute', now(), LastBeat) > 10
KQL

# Create a scheduled query alert
az monitor scheduled-query create \
  -g $RG -n "alert-no-heartbeat" \
  --scopes $LAW_ID \
  --description "VM heartbeat missing (10m)" \
  --condition "count 'Heartbeat | where TimeGenerated > ago(10m) | where Computer =~ \"$VMNAME\"' < 1" \
  --evaluation-frequency "PT5M" \
  --window-size "PT10M" \
  --action-groups $AG_ID \
  --severity 2
```

> Tip: For precise KQL in log alerts you can use **–condition-query** / **–query** flavors; the above uses a simple count condition for quick setup.

---

# 6) Quick KQL (copy into Logs in the workspace)

**a) Heartbeat (VM up/down signal):**

```kusto
Heartbeat
| where TimeGenerated > ago(1h)
| summarize LastSeen=max(TimeGenerated) by Computer, _ResourceId
| order by LastSeen desc
```

**b) CPU over time (Perf table):**

```kusto
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor" and CounterName == "% Processor Time" and InstanceName == "_Total"
| summarize AvgCPU=avg(CounterValue) by bin(TimeGenerated, 5m), Computer
| order by TimeGenerated asc
```

**c) Activity Logs (tenant-wide):**

```kusto
AzureActivity
| where TimeGenerated > ago(6h)
| project TimeGenerated, ResourceGroup, Resource, OperationName, ActivityStatus, Caller
| order by TimeGenerated desc
```

**d) Storage diagnostics (if you enabled above):**

```kusto
StorageBlobLogs
| where TimeGenerated > ago(1h)
| summarize count() by OperationName, ResponseType
| order by count_ desc
```

---

# 7) VM Insights (Optional, Nice to Have)

```bash
# Turn on VM Insights (maps, perf)
az monitor vm-insights enable --resource $VM_ID --workspace $LAW_ID
```

You’ll get performance charts and dependency maps in Azure Monitor > **VM Insights**.

---

# 8) Workbook (One-click Starter)

1. Portal → **Azure Monitor** → **Workbooks** → **New** → **Add query**
2. Paste the **Perf** query (CPU) and render as time chart.
3. Add a second query panel with **Heartbeat** to show last seen.
4. Save as “VM Health (Lab)”.

> Workbooks are just KQL + visualizations; you can export JSON for version control.

---

# 9) Cost/Best-Practice Guardrails

* Use **B1s**/**B2s** test VMs, short retention (e.g., 7–14 days) for the workspace while practicing:

  ```bash
  az monitor log-analytics workspace update -g $RG -n $LAW --retention-time 14
  ```
* Restrict DCR streams to only what you need (Heartbeat + select Perf).
* Prefer **Action Groups** that point to email/webhook/ITSM instead of SMS for labs.

---

# 10) Cleanup

```bash
az group delete -n $RG --yes --no-wait
```

---

## What you practiced

* Creating a **Log Analytics workspace** and wiring **AMA** via **DCR**
* Enabling **diagnostic settings** for platform logs/metrics
* Building **metric** and **log** alerts with **Action Groups**
* Running core **KQL** for heartbeat, performance, activity logs
* (Optional) **VM Insights** and simple **Workbook**
