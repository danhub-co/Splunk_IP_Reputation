# Splunk IP Reputation

An automated threat intelligence workflow that enriches Splunk security alerts with IP reputation data from multiple threat intelligence sources.

## 📋 Overview

This N8N workflow integrates Splunk alerts with VirusTotal and AlienVault to perform automated IP reputation checks and route suspicious IPs to security teams via Slack and email notifications.

## 🚀 Features

- **Alert Ingestion**: Receives security alerts from Splunk containing IP addresses
- **Multi-Source Enrichment**: Queries VirusTotal and AlienVault APIs for reputation data
- **Threat Intelligence Merging**: Combines data from multiple threat intelligence platforms
- **Suspicious IP Filtering**: Automatically identifies and routes suspicious IPs
- **Multi-Channel Notifications**: Sends alerts via Slack and email
- **HTML Reporting**: Generates comprehensive threat summary reports

## 🏗️ Architecture

### Workflow Nodes

| Node | Type | Function |
|------|------|----------|
| **Splunk Alert** | Webhook | Entry point - receives alerts from Splunk |
| **Extract IOCs** | Code | Extracts source IP and event reason |
| **VirusTotal IP Reputation Check** | HTTP Request | Queries VirusTotal API for IP reputation |
| **AlienVault Lookup** | HTTP Request | Queries AlienVault OTX API for threat data |
| **Merge Threat Data** | Merge | Combines results from multiple threat sources |
| **Process Intel Data** | Code | Processes and categorizes threat intelligence |
| **Generate IP Summary** | Code | Creates comprehensive threat summary with tags |
| **Filter Suspicious IPs** | Switch | Routes suspicious IPs for alerting |
| **IP Summary Display** | HTML | Generates HTML report visualization |
| **Slack IP Alert** | Slack | Sends alert notification to Slack |
| **Gmail** | Gmail | Sends email summary report |

## 📊 Data Flow

```
Splunk Alert
    ↓
Extract IOCs
    ↓
├─→ VirusTotal IP Reputation Check
├─→ AlienVault Lookup
    ↓
Merge Threat Data
    ↓
Process Intel Data
    ↓
Generate IP Summary
    ↓
├─→ IP Summary Display (Email)
└─→ Filter Suspicious IPs
        ↓
    Slack IP Alert
```

## 🔧 Configuration

### Required Credentials

- **VirusTotal API Key** - For IP reputation queries
- **AlienVault API Key** - For threat intelligence lookups
- **Slack OAuth2 Credentials** - For Slack notifications
- **Gmail OAuth2 Credentials** - For email reports

### Environment Setup

1. Deploy the N8N workflow configuration
2. Configure API credentials for each threat intelligence source
3. Set up Slack channel and Gmail recipient email
4. Configure Splunk webhook to send alerts to N8N

## 📤 Input

Splunk alert webhook payload containing:
```json
{
  "result": {
    "src_ip": "192.168.1.1",
    "reason": "Suspicious outbound connection"
  }
}
```

## 📥 Output

### Data Extracted
- **IP Address** - Source IP from alert
- **VirusTotal Tags** - Threat classifications and malware families
- **AlienVault Pulses** - Related threat intelligence pulses
- **Reputation Score** - Combined reputation assessment
- **Status** - Classification (Suspicious/Clean)

### Notifications
- **Slack Alert**: Direct message to security team for suspicious IPs
- **Email Report**: HTML-formatted threat summary report

## ⚙️ Key Features

### Threat Enrichment
- Extracts malware families and threat tags from VirusTotal
- Retrieves related threat pulses from AlienVault
- Compiles WHOIS and metadata information
- Generates threat status classification

### Alert Routing
- Automatically filters IPs marked as "Suspicious"
- Sends real-time Slack notifications to SOC team
- Generates detailed HTML email reports
- Maintains audit trail in workflow logs

## 📝 Workflow Steps

1. **Alert Ingestion**: Splunk sends alert via webhook with IP address
2. **IOC Extraction**: Parse IP and event reason from alert
3. **Threat Intelligence Query**: Simultaneously query VirusTotal and AlienVault
4. **Data Merging**: Combine results from all threat sources
5. **Intelligence Processing**: Extract relevant tags, pulses, and metadata
6. **Summary Generation**: Create comprehensive threat analysis
7. **Decision Point**: Filter by threat status
8. **Alert Dispatch**: Send notifications to Slack and email

## 🔐 Security Considerations

- All API credentials stored securely in N8N
- Webhook endpoint secured with unique path
- OAuth2 authentication for Slack and Gmail
- Predefined credential types for threat intelligence APIs

## 📚 Integration Points

- **Splunk**: Source of security alerts
- **VirusTotal**: IP reputation and threat intelligence
- **AlienVault OTX**: Open threat exchange data
- **Slack**: Real-time team notifications
- **Gmail**: Email reporting

## 🚨 Use Cases

- Automated response to suspicious IP connections
- Real-time threat enrichment for security alerts
- Centralized IP reputation assessment
- SOC team alerting and escalation
- Compliance reporting and audit trails

## 📄 Files

- `README.md` - This documentation
- `✅Splunk Alert - IP Reputation Check(1).json` - N8N workflow configuration

## 📞 Support

For issues, questions, or contributions, please refer to the repository's issue tracker.

---

**Last Updated**: 2026-05-14  
**Workflow Status**: Configured for deployment
