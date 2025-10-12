# Wazuh Daily Analysis & Reporting Workflow

> AI-powered security alert analysis and reporting system for Wazuh SIEM

![n8n](https://img.shields.io/badge/n8n-workflow-orange)
![License](https://img.shields.io/badge/license-MIT-blue)
![Version](https://img.shields.io/badge/version-1.0.0-green)

## 🚀 Quick Start

**TL;DR - Get started in 5 steps:**

1. **Import workflow** into n8n
2. **Create Data Table** named `wazuh_alerts` with required columns (see [Installation](#installation))
3. **Configure credentials**: OpenRouter API + Gmail OAuth2
4. **Set up Wazuh integration** to send alerts level 7+ to n8n webhook (see [Wazuh Configuration](#wazuh-configuration))
5. **Activate workflow** and you're done! 🎉

**First time?** Read the full guide below for detailed setup instructions.

---

## 📋 Table of Contents

- [Quick Start](#quick-start)
- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Wazuh Configuration](#wazuh-configuration)
- [n8n Configuration](#n8n-configuration)
- [Workflow Configuration](#workflow-configuration)
- [Data Structure](#data-structure)
- [How It Works](#how-it-works)
- [Usage](#usage)
- [Troubleshooting](#troubleshooting)
- [Advanced Configuration](#advanced-configuration)
- [FAQ](#frequently-asked-questions)
- [Best Practices](#best-practices)
- [Contributing](#contributing)
- [License](#license)

## 🎯 Overview

This n8n workflow automates the analysis of Wazuh security alerts using AI (Claude 3.5 Sonnet via OpenRouter). It processes alerts from your Wazuh deployment, groups them by agent and rule type, analyzes each group for true/false positives, and generates a comprehensive HTML email report.

### Why This Workflow?

- **Reduces Alert Fatigue**: AI helps identify false positives automatically
- **Granular Analysis**: Each rule type on each agent is analyzed separately for better accuracy
- **Actionable Reports**: Get specific recommendations per alert type
- **No Database Required**: Uses n8n's built-in Data Table for storage
- **Fully Automated**: Runs on schedule (default: every 4 hours)

## ✨ Features

### Core Features

- ✅ **AI-Powered Analysis**: Uses Claude 3.5 Sonnet to analyze security alerts
- ✅ **Smart Grouping**: Groups alerts by Agent + Rule ID for focused analysis
- ✅ **Multi-Strategy Parsing**: Robust JSON parser handles truncated/malformed AI responses
- ✅ **HTML Email Reports**: Professional, scannable email reports with statistics
- ✅ **Automatic Cleanup**: Deletes processed alerts older than 7 days
- ✅ **Loop Processing**: Analyzes each group independently (not just the first one)

### Analysis Capabilities

The AI analyzes each group for:
- Alert patterns and correlations
- Frequency and timing analysis
- True Positive vs False Positive determination
- Known false positive patterns for specific Wazuh rules
- Security posture assessment
- Specific recommendations per rule type

### Report Statistics

- Unique agents with alerts
- Unique rule types triggered
- Total alert count
- True Positives count
- False Positives count
- Mixed/Uncertain count

## 🏗️ Architecture

### Complete System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Wazuh SIEM                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│  │  Agent   │  │  Agent   │  │  Agent   │                       │
│  │    1     │  │    2     │  │    3     │                       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                       │
│       │             │             │                             │
│       └─────────────┴─────────────┘                             │
│                     │                                           │
│              ┌──────▼──────┐                                    │
│              │   Wazuh     │                                    │
│              │   Manager   │                                    │
│              │             │                                    │
│              │ Integration │ (Level 7+ filter)                  │
│              └──────┬──────┘                                    │
└───────────────────┬─┴───────────────────────────────────────────┘
                    │ HTTPS Webhook
                    │
┌───────────────────▼─────────────────────────────────────────────┐
│                       n8n Platform                              │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │          Webhook Receiver Workflow (Separate)              │ │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────────┐          │ │
│  │  │ Webhook  │──▶ │Transform │──▶│ Insert to    │          │ │
│  │  │ Trigger  │    │  Data    │    │ Data Table   │          │ │
│  │  └──────────┘    └──────────┘    └──────────────┘          │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Data Table                              │ │
│  │              (wazuh_alerts storage)                        │ │
│  └───────────────────────┬────────────────────────────────────┘ │
│                          │                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │      Analysis & Reporting Workflow (Main - This One)       │ │
│  │                                                            │ │
│  │  ┌──────────┐    ┌──────────┐   ┌──────────┐               │ │
│  │  │Schedule  │──▶│  Fetch   │──▶│  Group   │               │ │
│  │  │ Trigger  │    │ Alerts   │   │by Agent  │               │ │
│  │  │(4 hours) │    │          │   │ & Rule   │               │ │
│  │  └──────────┘    └──────────┘   └─────┬────┘               │ │
│  │                                       │                    │ │
│  │                               ┌───────▼────────┐           │ │
│  │                               │  AI Analysis   │           │ │
│  │                               │     Loop       │◀─────┐   │ │
│  │                               │  (per group)   │      │    │ │
│  │                               └───────┬────────┘      │    │ │
│  │                                       │            Multiple│ │
│  │                               ┌───────▼────────┐   Groups  │ │
│  │                               │   Aggregate    │      │    │ │
│  │                               │    Results     │──────┘    │ │
│  │                               └───────┬────────┘           │ │
│  │                                       │                    │ │
│  │                               ┌───────▼────────┐           │ │
│  │                               │   Generate     │           │ │
│  │                               │  HTML Report   │           │ │
│  │                               └───────┬────────┘           │ │
│  │                                       │                    │ │
│  │                               ┌───────▼────────┐           │ │
│  │                               │  Send Email    │           │ │
│  │                               └───────┬────────┘           │ │
│  │                                       │                    │ │
│  │                               ┌───────▼────────┐           │ │
│  │                               │  Mark as       │           │ │
│  │                               │  Processed     │           │ │
│  │                               │  & Cleanup     │           │ │
│  │                               └────────────────┘           │ │
│  └────────────────────────────────────────────────────────────┘ │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                    ┌───────▼────────┐
                    │   OpenRouter   │
                    │  (Claude 3.5)  │
                    └────────────────┘
```

### Analysis Workflow Details

```
┌─────────────────┐
│  Schedule       │  Runs every 4 hours
│  Trigger        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Fetch Alerts   │  Get all alerts from Data Table
│  from Data      │
│  Table          │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Group by       │  Group by agent_name + agent_ip + rule_id
│  Agent & Rule   │  Filters unprocessed (processed = "0")
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Prepare AI     │  Create analysis prompt for each group
│  Prompt         │  (runs once per group)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  AI Analysis    │  Call OpenRouter API (Claude 3.5 Sonnet)
│  Loop           │  One API call per group
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Parse AI       │  Multi-strategy JSON parser
│  Response       │  Handles truncated/malformed responses
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Aggregate      │  Collect all analyses into one item
│  Results        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Generate       │  Create HTML email report
│  HTML Report    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Send Email     │  Send via Gmail
│  Report         │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Mark as        │  Update processed = "1"
│  Processed      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Cleanup Old    │  Delete alerts > 7 days old
│  Alerts         │  (processed = "1" only)
└─────────────────┘
```

## 📦 Prerequisites

### Required Services

1. **n8n** (self-hosted or cloud)
   - Version: 1.0.0 or higher
   - Data Tables enabled (built-in feature)

2. **OpenRouter Account**
   - Sign up at [openrouter.ai](https://openrouter.ai)
   - Get API key
   - Supports Claude 3.5 Sonnet model

3. **Gmail Account** (or other SMTP email service)
   - OAuth2 authentication configured in n8n
   - OR use SMTP credentials

4. **Wazuh SIEM**
   - Source of security alerts
   - Need a way to populate the Data Table (separate workflow or integration)

### Optional

- **Wazuh to n8n Integration**: Create a webhook or API integration to push Wazuh alerts to the Data Table

## 🚀 Installation

### Step 1: Import Workflow

1. Download the workflow JSON file
2. In n8n, go to **Workflows** → **Import from File**
3. Select the downloaded JSON file
4. The workflow will be imported (inactive by default)

### Step 2: Create Data Table

Before running the workflow, create the required Data Table:

1. Go to **n8n Settings** → **Data Tables**
2. Click **"Create New Table"**
3. Name it: `wazuh_alerts`
4. Add the following columns:

| Column Name       | Type          | Description                          |
|-------------------|---------------|--------------------------------------|
| `id`              | Auto-ID       | Auto-generated unique ID             |
| `agent_name`      | Text          | Wazuh agent name                     |
| `agent_ip`        | Text          | Wazuh agent IP address               |
| `agent_id`        | Text          | Wazuh agent ID                       |
| `rule_id`         | Text          | Wazuh rule ID (e.g., "60110")        |
| `rule_level`      | Number        | Rule severity level (0-15)           |
| `rule_description`| Text          | Rule description                     |
| `rule_groups`     | Text          | Rule groups (comma-separated)        |
| `alert_data`      | Text          | Alert message/description            |
| `raw_body`        | Text          | Full JSON alert data                 |
| `timestamp`       | Date & Time   | Alert timestamp                      |
| `location`        | Text          | Alert source location                |
| `processed`       | Text          | "0" = unprocessed, "1" = processed   |
| `created_at`      | Date & Time   | When alert was added to table        |
| `source`          | Text          | Always "wazuh"                       |

### Step 3: Configure Credentials

#### OpenRouter API

1. In n8n, go to **Credentials**
2. Click **"Add Credential"**
3. Search for "OpenRouter"
4. Add your OpenRouter API key
5. Test the credential
6. Save as "OpenRouter account"

#### Gmail OAuth2

1. In n8n, go to **Credentials**
2. Click **"Add Credential"**
3. Search for "Gmail OAuth2"
4. Follow the OAuth setup wizard
5. Authorize n8n to send emails
6. Save as "Gmail account"

### Step 4: Update Workflow Settings

Open the imported workflow and update:

1. **Send Email Report** node:
   - Change `sendTo` to your email address
   - Example: `user@example.com`

2. **Schedule every 4 hours** node:
   - Adjust schedule if needed (default: every 4 hours)
   - Can change to daily, hourly, etc.

3. **All nodes with credentials**:
   - Ensure they're using your configured credentials
   - OpenRouter API credential
   - Gmail credential

### Step 5: Activate Workflow

1. Click **"Active"** toggle in top-right
2. Workflow will now run on schedule

## 🔐 Wazuh Configuration

To send alerts from Wazuh to this n8n workflow, you need to configure Wazuh to forward alerts via webhook or integration.

### Option 1: Wazuh Integration (Recommended)

Configure Wazuh to send alerts to n8n via webhook integration. This requires setting up a webhook receiver in n8n and configuring Wazuh to send alerts to it.

#### Step 1: Create Webhook Receiver in n8n

1. Create a new workflow in n8n
2. Add a **Webhook** trigger node
3. Configure as follows:
   - **Webhook Path**: `wazuh-alerts` (or custom path)
   - **HTTP Method**: POST
   - **Response Mode**: "On Received"
   
4. Add a **Data Table** node after the webhook
5. Configure to insert data into `wazuh_alerts` table
6. Map incoming fields appropriately

7. Copy the webhook URL (e.g., `https://your-n8n-instance.com/webhook/wazuh-alerts`)

#### Step 2: Configure Wazuh Integration

Edit the Wazuh integration configuration file on your Wazuh manager:

**File Location**: `/var/ossec/etc/ossec.conf`

Add the following integration block:

```xml
<ossec_config>
  <integration>
    <name>custom-webhook</name>
    <hook_url>https://your-n8n-instance.com/webhook/wazuh-alerts</hook_url>
    <level>7</level>
    <alert_format>json</alert_format>
    <options>{"Content-Type":"application/json"}</options>
  </integration>
</ossec_config>
```

**Configuration Parameters:**

- `<name>`: Integration name (use `custom-webhook`)
- `<hook_url>`: Your n8n webhook URL
- `<level>`: Minimum alert level to forward (7 = medium-high severity)
  - Levels: 0-15 (higher = more severe)
  - Level 7+ includes: account changes, privilege escalations, suspicious activity
  - Adjust based on your needs (e.g., `<level>5</level>` for more alerts)
- `<alert_format>`: Use `json` for structured data
- `<options>`: HTTP headers (keep Content-Type as application/json)

**Optional: Filter by Rule or Group**

To only send specific rule types:

```xml
<integration>
  <name>custom-webhook</name>
  <hook_url>https://your-n8n-instance.com/webhook/wazuh-alerts</hook_url>
  <level>7</level>
  <rule_id>60110,60120,60227</rule_id>  <!-- Specific rules -->
  <alert_format>json</alert_format>
</integration>
```

Or filter by group:

```xml
<integration>
  <name>custom-webhook</name>
  <hook_url>https://your-n8n-instance.com/webhook/wazuh-alerts</hook_url>
  <level>7</level>
  <group>windows_security,authentication_failed</group>  <!-- Specific groups -->
  <alert_format>json</alert_format>
</integration>
```

#### Step 3: Restart Wazuh Manager

After editing the configuration:

```bash
# Restart Wazuh manager to apply changes
systemctl restart wazuh-manager

# Verify the service is running
systemctl status wazuh-manager

# Check logs for integration errors
tail -f /var/ossec/logs/ossec.log | grep integration
```

#### Step 4: Verify Alert Flow

1. Trigger a test alert in Wazuh (e.g., failed SSH login, file change)
2. Check n8n webhook executions
3. Verify data appears in `wazuh_alerts` Data Table
4. Check that `processed` field is set to `"0"`

### Option 2: Wazuh API Polling (Alternative)

If webhooks are not feasible, you can poll the Wazuh API:

1. Create a scheduled workflow in n8n
2. Use HTTP Request node to query Wazuh API
3. Filter for alerts with level >= 7
4. Insert into Data Table

**Example API Query:**

```
GET https://wazuh-manager:55000/security/alerts
Authorization: Bearer <your-wazuh-api-token>
```

### Option 3: Filebeat to n8n (Advanced)

For high-volume environments:

1. Configure Filebeat to read Wazuh alerts
2. Send to Logstash or directly to n8n webhook
3. Use Logstash filters to process and enrich data
4. Forward to n8n Data Table

### Alert Level Guidelines

**Wazuh Alert Levels:**

| Level | Severity | Description | Examples |
|-------|----------|-------------|----------|
| 0-3   | Low      | Informational | System events, routine operations |
| 4-6   | Medium   | Suspicious activity | Multiple failed logins, file changes |
| 7-9   | High     | Security concern | Account changes, privilege escalation |
| 10-12 | Critical | Active attack | Multiple attack patterns, known exploits |
| 13-15 | Severe   | Confirmed breach | Rootkit detected, system compromise |

**Recommended Settings:**

- **Production**: `<level>7</level>` (High and above)
- **Development**: `<level>5</level>` (Medium and above)
- **High Security**: `<level>4</level>` (All suspicious activity)
- **Custom**: Use rule_id or group filters for specific alerts

## 🔧 n8n Configuration

### Webhook Receiver Workflow

Create a separate workflow to receive and store Wazuh alerts:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Webhook    │────▶│  Transform   │────▶│  Insert to   │
│   Trigger    │     │    Data      │     │  Data Table  │
└──────────────┘     └──────────────┘     └──────────────┘
```

#### Webhook Trigger Node Configuration

- **Path**: `wazuh-alerts`
- **Method**: POST
- **Response**: Return 200 OK immediately

#### Transform Data Node (Code Node)

```javascript
// Extract and transform Wazuh alert data
const alert = $input.item.json;

return {
  json: {
    agent_name: alert.agent?.name || 'Unknown',
    agent_ip: alert.agent?.ip || 'Unknown',
    agent_id: alert.agent?.id || 'Unknown',
    rule_id: alert.rule?.id || '',
    rule_level: alert.rule?.level || 0,
    rule_description: alert.rule?.description || '',
    rule_groups: alert.rule?.groups?.join(',') || '',
    alert_data: alert.data?.win?.system?.message || 
                alert.full_log || 
                JSON.stringify(alert.data),
    raw_body: JSON.stringify(alert),
    timestamp: alert.timestamp || new Date().toISOString(),
    location: alert.location || 'Unknown',
    processed: "0",  // Important: string "0" not number
    created_at: new Date().toISOString(),
    source: "wazuh"
  }
};
```

#### Insert to Data Table Node Configuration

- **Operation**: Insert
- **Data Table**: `wazuh_alerts`
- **Columns**: Map all fields from transformed data
- **On Conflict**: Skip duplicate (optional)

### Testing the Integration

1. **Send a test webhook** from command line:

```bash
curl -X POST https://your-n8n-instance.com/webhook/wazuh-alerts \
  -H "Content-Type: application/json" \
  -d '{
    "timestamp": "2025-01-15T10:30:00.000Z",
    "agent": {
      "name": "test-agent",
      "ip": "x.x.x.x",
      "id": "001"
    },
    "rule": {
      "id": "60110",
      "level": 8,
      "description": "User account changed",
      "groups": ["windows", "windows_security"]
    },
    "data": {
      "win": {
        "system": {
          "message": "Test alert message"
        }
      }
    },
    "location": "EventChannel"
  }'
```

2. **Check n8n executions**: Verify the webhook received the data
3. **Check Data Table**: Confirm the alert was inserted
4. **Trigger analysis workflow**: Run the main workflow to process the test alert

### Complete Webhook Receiver Example

Here's a complete n8n workflow JSON for receiving Wazuh alerts:

```json
{
  "name": "Wazuh Alert Receiver",
  "nodes": [
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "wazuh-alerts",
        "responseMode": "onReceived",
        "options": {}
      },
      "name": "Webhook",
      "type": "n8n-nodes-base.webhook",
      "position": [250, 300]
    },
    {
      "parameters": {
        "jsCode": "const alert = $input.item.json;\n\nreturn {\n  json: {\n    agent_name: alert.agent?.name || 'Unknown',\n    agent_ip: alert.agent?.ip || 'Unknown',\n    agent_id: alert.agent?.id || 'Unknown',\n    rule_id: alert.rule?.id || '',\n    rule_level: alert.rule?.level || 0,\n    rule_description: alert.rule?.description || '',\n    rule_groups: alert.rule?.groups?.join(',') || '',\n    alert_data: alert.data?.win?.system?.message || alert.full_log || JSON.stringify(alert.data),\n    raw_body: JSON.stringify(alert),\n    timestamp: alert.timestamp || new Date().toISOString(),\n    location: alert.location || 'Unknown',\n    processed: '0',\n    created_at: new Date().toISOString(),\n    source: 'wazuh'\n  }\n};"
      },
      "name": "Transform Wazuh Alert",
      "type": "n8n-nodes-base.code",
      "position": [450, 300]
    },
    {
      "parameters": {
        "operation": "create",
        "dataTable": "wazuh_alerts"
      },
      "name": "Insert to Data Table",
      "type": "n8n-nodes-base.dataTable",
      "position": [650, 300]
    }
  ],
  "connections": {
    "Webhook": {
      "main": [[{"node": "Transform Wazuh Alert", "type": "main", "index": 0}]]
    },
    "Transform Wazuh Alert": {
      "main": [[{"node": "Insert to Data Table", "type": "main", "index": 0}]]
    }
  }
}
```

**To use this workflow:**
1. Import it into n8n
2. Activate the workflow
3. Copy the webhook URL
4. Configure it in Wazuh (as shown above)
5. Test by sending a sample alert

## ⚙️ Workflow Configuration

### Adjusting AI Analysis

In the **"Code in JavaScript"** node, you can adjust:

```javascript
{
  model: "anthropic/claude-3.5-sonnet",  // AI model
  temperature: 0.3,                       // Creativity (0.0-1.0, lower = more focused)
  max_tokens: 4000                        // Response length
}
```

### Changing Schedule

In the **"Schedule every 4 hours"** node:
- Change `hoursInterval` to desired frequency
- Or switch to cron expression for custom schedules

### Customizing Report

Edit the **"Generate HTML Report"** node to customize:
- Email styling (CSS)
- Card layouts
- Table columns
- Summary statistics

### Cleanup Retention Period

In the **"Filter Old Alerts"** node, adjust retention:

```javascript
const sevenDaysAgo = new Date();
sevenDaysAgo.setDate(sevenDaysAgo.getDate() - 7);  // Change 7 to desired days
```

## 📊 Data Structure

### Input: Wazuh Alert Format

The workflow expects alerts in the following format in the Data Table:

```json
{
  "id": 1,
  "agent_name": "agent-hostname",
  "agent_ip": "x.x.x.x",
  "agent_id": "001",
  "rule_id": "60110",
  "rule_level": 8,
  "rule_description": "User account changed",
  "rule_groups": "windows,windows_security,account_changed",
  "alert_data": "A user account was changed...",
  "raw_body": "{...full wazuh alert JSON...}",
  "timestamp": "2025-10-12T12:39:45.030Z",
  "location": "EventChannel",
  "processed": "0",
  "created_at": "2025-10-12T13:04:03.943Z",
  "source": "wazuh"
}
```

### Output: AI Analysis Format

The AI returns analysis in this format:

```json
{
  "agent_name": "agent-hostname",
  "agent_ip": "x.x.x.x",
  "rule_id": "60110",
  "rule_description": "User account changed",
  "total_alerts": 5,
  "verdict": "False Positive",
  "confidence": "High",
  "summary": "Brief summary of findings for this specific rule",
  "detailed_analysis": "Detailed analysis of alert patterns...",
  "recommendations": "Specific recommendations for this agent and rule"
}
```

## 🔄 How It Works

### 1. Data Collection (You Need to Set This Up)

**Not included in this workflow** - you need to create a separate integration to populate the Data Table:

Options:
- Wazuh webhook → n8n webhook trigger → Insert to Data Table
- Scheduled API polling from Wazuh
- Filebeat/Logstash → API → n8n

### 2. Grouping Logic

Alerts are grouped by a composite key:
```
Key = agent_name + agent_ip + rule_id
```

**Example:**
- "agent-hostname_x.x.x.x_60110" → Group 1 (5 alerts)
- "agent-hostname_x.x.x.x_60120" → Group 2 (3 alerts)
- "server-hostname_y.y.y.y_60110" → Group 3 (2 alerts)

This creates **3 separate analyses** instead of lumping all alerts together.

### 3. AI Analysis Loop

For each group:
1. Creates a focused prompt with rule information
2. Calls OpenRouter API (Claude 3.5 Sonnet)
3. Parses response with multi-strategy parser
4. Validates and extracts all required fields

### 4. Robust Parsing

The parser uses 3 strategies:

**Strategy 1: Standard JSON with Auto-Fix**
- Removes markdown code blocks
- Adds missing closing braces
- Removes trailing commas
- Fixes line breaks in strings

**Strategy 2: Progressive Parsing**
- If JSON is truncated, finds the longest valid JSON substring
- Tries parsing from full length down in 10-char increments

**Strategy 3: Regex Field Extraction**
- If JSON completely fails, extracts fields individually using regex
- Ensures no field is left as "Unable to determine" if it exists in the text

### 5. Report Generation

Aggregates all analyses and generates:
- Summary cards with key metrics
- Detailed table (one row per agent+rule combo)
- Professional HTML styling
- Mobile-responsive design

### 6. Cleanup

After sending the report:
1. Marks all analyzed alerts as `processed = "1"`
2. Fetches all alerts from the table
3. Filters for alerts that are:
   - Processed (`processed = "1"`)
   - Older than 7 days
4. Deletes those old processed alerts

## 📖 Usage

### Manual Trigger

1. Open the workflow
2. Click **"Test workflow"** or **"Execute Workflow"**
3. Workflow will process all unprocessed alerts immediately

### Scheduled Execution

- Workflow runs automatically every 4 hours (default)
- Processes new alerts since last run
- Sends email report

### Viewing Results

Check your email for the HTML report containing:
- Summary of total alerts, agents, and rules
- Verdicts breakdown (True/False Positives)
- Detailed table with analysis per agent+rule combination
- Specific recommendations

### Reviewing Raw Data

1. Go to **Settings** → **Data Tables** → **wazuh_alerts**
2. View all alerts and their processing status
3. Filter by `processed` field to see what's pending

## 🔧 Troubleshooting

### Wazuh Integration Issues

**Alerts Not Reaching n8n**

**Check:**
- Wazuh manager is running: `systemctl status wazuh-manager`
- Integration is configured in `/var/ossec/etc/ossec.conf`
- Wazuh manager was restarted after config changes
- Check Wazuh logs: `tail -f /var/ossec/logs/ossec.log | grep integration`
- Webhook URL is correct and accessible from Wazuh server
- n8n webhook workflow is active

**Test connectivity from Wazuh server:**
```bash
curl -X POST https://your-n8n-instance.com/webhook/wazuh-alerts \
  -H "Content-Type: application/json" \
  -d '{"test": "alert"}'
```

**Check Wazuh integration status:**
```bash
# View integration configuration
grep -A 10 '<integration>' /var/ossec/etc/ossec.conf

# Check if alerts are being generated
tail -f /var/ossec/logs/alerts/alerts.json

# Verify integration is active
/var/ossec/bin/wazuh-integratord -t
```

**Common Wazuh Issues:**

1. **Integration not triggering**
   - Verify alert level meets threshold (e.g., level >= 7)
   - Check if rule_id or group filters are too restrictive
   - Ensure alerts are actually being generated in Wazuh

2. **SSL/TLS errors**
   - Wazuh requires HTTPS for external webhooks
   - Install proper SSL certificate on n8n
   - Or use self-signed cert (not recommended for production)

3. **Timeout errors**
   - n8n webhook taking too long to respond
   - Set webhook to "On Received" response mode
   - Check n8n server resources

**Wrong Data Format in Data Table**

**Check:**
- Transform node is correctly parsing Wazuh JSON
- Field names match Data Table column names exactly
- `processed` is string "0" not number 0
- `timestamp` is in ISO format

**Debug the transform node:**
```javascript
// Add console.log to see raw data
console.log('Received alert:', JSON.stringify($input.item.json, null, 2));
```

### No Email Received

**Check:**
- Gmail credentials are valid
- Email address in "Send Email Report" node is correct
- Check spam folder
- Check n8n execution logs for errors

### "Paired item data unavailable" Error

**Solution:**
- Ensure all Code nodes have `mode: "runOnceForEachItem"`
- Check that HTTP Request node has `batchSize: 1`
- This has been fixed in the latest version

### AI Returns "Unable to determine"

**Solutions:**
- Increase `max_tokens` in "Code in JavaScript" node (currently 4000)
- Check OpenRouter API key is valid and has credits
- Review the `raw_ai_response` field to see what AI actually returned
- The multi-strategy parser should handle most issues automatically

### No Alerts Being Processed

**Check:**
- Data Table `wazuh_alerts` exists
- Alerts have `processed = "0"` (string, not number)
- Alerts exist in the table
- "Fetch Unprocessed Alerts" node is configured correctly

### Workflow Doesn't Run on Schedule

**Check:**
- Workflow is **Active** (toggle in top-right)
- Schedule trigger is configured correctly
- n8n instance is running (for self-hosted)

### Memory/Performance Issues

**Solutions:**
- Reduce `max_tokens` if needed
- Limit number of alerts processed per run
- Adjust cleanup retention period (delete older alerts more frequently)
- Process fewer rule types at once

## 🎛️ Advanced Configuration

### Custom AI Models

Edit the "Code in JavaScript" node to use different models:

```javascript
{
  model: "anthropic/claude-3-opus",      // More powerful, slower
  model: "openai/gpt-4-turbo",           // Alternative
  model: "anthropic/claude-3-haiku",     // Faster, cheaper
}
```

### Multi-Language Support

Edit the AI prompt in "Prepare prompt for AI Analysis" to request responses in other languages:

```javascript
const prompt = `You are a cybersecurity analyst... 
Please respond in Spanish/French/German, etc.`
```

### Custom Verdict Categories

Modify the parser to accept custom verdicts:

```javascript
// In "Parse AI Response" node
analysis.verdict = extractField(aiResponse, 'verdict') || 'Needs Review';
// Add support for: "Suspicious", "Critical", "Informational", etc.
```

### Integration with Other Tools

**Send to Slack:**
- Add a Slack node after "Generate HTML Report"
- Format the summary for Slack

**Create Tickets:**
- Add Jira/ServiceNow node
- Create tickets for True Positives automatically

**Store in Database:**
- Add PostgreSQL/MySQL node
- Store analysis results for long-term tracking

## 📝 Best Practices

### Wazuh Integration

1. **Alert Level Tuning**: Start with level 7, adjust based on your environment
   - Too low = noise and unnecessary API costs
   - Too high = might miss important security events
   
2. **Use Rule/Group Filters**: Instead of processing all alerts, filter for specific types:
   ```xml
   <group>authentication_failed,privilege_escalation,malware</group>
   ```

3. **Monitor Integration Health**: 
   - Check Wazuh logs daily for integration errors
   - Set up alerts for failed webhook deliveries
   - Monitor n8n webhook execution success rate

4. **Rate Limiting**: 
   - Implement rate limiting on n8n webhook
   - Prevent overload from alert storms
   - Use n8n's "Execute Once" option if needed

5. **Data Retention**: 
   - Wazuh generates lots of alerts
   - Clean up processed alerts regularly (workflow does this automatically)
   - Archive important alerts before deletion if needed

### Workflow Optimization

1. **Start Small**: Test with a few alerts first before processing hundreds
2. **Monitor Costs**: OpenRouter API calls cost money - monitor your usage
3. **Review False Positives**: Periodically review AI verdicts for accuracy
4. **Tune Prompts**: Adjust the AI prompt based on your specific environment
5. **Data Retention**: Adjust cleanup period based on your compliance requirements
6. **Backup Data**: Export Data Table periodically as backup
7. **Version Control**: Keep track of workflow changes in git

### Security

1. **Webhook Authentication**: 
   - Add header-based authentication to webhook
   - Use n8n's built-in authentication options
   - Rotate API keys regularly

2. **Network Security**:
   - Use HTTPS only (required)
   - Restrict webhook access by IP (firewall rules)
   - Place n8n behind VPN if possible

3. **Data Privacy**:
   - Be aware that alerts are sent to OpenRouter (third-party)
   - Review your organization's data policies
   - Consider using self-hosted AI models for sensitive environments

4. **Access Control**:
   - Limit who can modify the workflows
   - Use n8n's user management features
   - Audit workflow changes regularly

## 🤝 Contributing

Contributions are welcome! To contribute:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

## 📄 License

This project is licensed under the MIT License.

## 📋 Changelog

### Version 1.0.0 (Current)

**Features:**
- ✅ AI-powered alert analysis using Claude 3.5 Sonnet
- ✅ Smart grouping by Agent + Rule ID
- ✅ Multi-strategy JSON parser for robust AI response handling
- ✅ HTML email reports with statistics
- ✅ Automatic cleanup of old alerts (7+ days)
- ✅ Loop processing for all alert groups
- ✅ Wazuh integration via webhook
- ✅ Support for alert level filtering (level 7+ recommended)

**Known Issues:**
- Large volumes of alerts (>1000 per run) may exceed API rate limits
- Very long alert messages may cause AI response truncation (mitigated by 4000 token limit)
- Data Table performance may degrade with >10,000 total alerts (use cleanup)

**Roadmap:**
- [ ] Support for custom AI models (Ollama, local LLMs)
- [ ] Slack integration template
- [ ] Dashboard for analysis metrics
- [ ] Ticket creation for True Positives (Jira, ServiceNow)
- [ ] Historical trend analysis
- [ ] Alert correlation across multiple agents

## 🙏 Acknowledgments

- **n8n** - Workflow automation platform
- **Anthropic** - Claude AI models
- **OpenRouter** - AI API gateway
- **Wazuh** - Open source SIEM platform

## 📞 Support

For issues and questions:
- Open an issue on GitHub
- Check n8n community forum
- Review n8n documentation: https://docs.n8n.io

---

**Built with ❤️ using n8n and AI**

*Last Updated: October 2025*

