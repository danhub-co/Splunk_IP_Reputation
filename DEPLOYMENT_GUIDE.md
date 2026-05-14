# Splunk IP Reputation - Deployment Guide

Complete step-by-step guide to deploy the IP Reputation enrichment workflow in your environment.

## 📋 Table of Contents

1. [Prerequisites](#prerequisites)
2. [Architecture Overview](#architecture-overview)
3. [Phase 1: Pre-Deployment Setup](#phase-1-pre-deployment-setup)
4. [Phase 2: N8N Workflow Import](#phase-2-n8n-workflow-import)
5. [Phase 3: Credential Configuration](#phase-3-credential-configuration)
6. [Phase 4: Workflow Customization](#phase-4-workflow-customization)
7. [Phase 5: Testing & Validation](#phase-5-testing--validation)
8. [Phase 6: Production Activation](#phase-6-production-activation)
9. [Monitoring & Maintenance](#monitoring--maintenance)
10. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Software & Services
- **N8N Instance** (self-hosted, Docker, or cloud)
- **Splunk** (with alert capability)
- **API Accounts** (VirusTotal, AlienVault OTX, Slack, Gmail)
- **Network Access** to external APIs

### Minimum Permissions
- N8N: Admin access to create workflows and credentials
- Splunk: Alert creation and webhook configuration
- Gmail: Account with 2FA app password (if enabled)
- Slack: Workspace admin or app creation permission

### Estimated Setup Time
- Prerequisites: 30-45 minutes
- N8N Setup: 15 minutes
- Credential Configuration: 30-45 minutes
- Testing & Validation: 20-30 minutes
- **Total: ~2-3 hours**

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Splunk SIEM                             │
│              (Security Event Processing)                     │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       │ Alert Webhook (POST)
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                   N8N AUTOMATION                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 1. Extract IOCs (IP, Reason)                           │  │
│  └─────────────┬──────────────────────────────────────────┘  │
│                │                                              │
│  ┌─────────────┴──────────────┬──────────────────────┐       │
│  │                            │                      │       │
│  ▼                            ▼                      ▼       │
│ VT API                    AlienVault               Metadata  │
│ Check                      OTX API                   Cache    │
│  │                            │                      │       │
│  └─────────────┬──────────────┴──────────────────────┘       │
│                │                                              │
│  ┌─────────────▼──────────────────────────────────────────┐  │
│  │ 2. Merge & Process Threat Data                         │  │
│  └─────────────┬──────────────────────────────────────────┘  │
│                │                                              │
│  ┌─────────────▼──────────────────────────────────────────┐  │
│  │ 3. Generate Summary & Filter by Status                 │  │
│  └─────────────┬──────────────────────────────────────────┘  │
│                │                                              │
│  ┌─────────────┴──────────────┬──────────────────────┐       │
│  │                            │                      │       │
│  ▼                            ▼                      ▼       │
│ HTML Report              Slack Alert               Gmail    │
│ (All IPs)             (Suspicious Only)         (Summary)   │
│                                                              │
└──────────────┬───────────────────────────────┬───────────────┘
               │                               │
               ▼                               ▼
        ┌──────────────┐              ┌──────────────┐
        │   Slack      │              │    Gmail     │
        │ Notifications│              │    Reports   │
        └──────────────┘              └──────────────┘
```

---

## Phase 1: Pre-Deployment Setup

### Step 1.1: Get VirusTotal API Key

1. Go to https://www.virustotal.com/gui/home/upload
2. Sign up or log in with your account
3. Navigate to **Settings → API Key**
4. Copy your API key (keep it secure)
5. Note the API key for later use

**Free Tier Limits:**
- 500 requests/day
- 4 requests/minute

### Step 1.2: Get AlienVault OTX API Key

1. Visit https://otx.alienvault.com/api
2. Create or log in to your account
3. Go to **Settings → API Key**
4. Create a new API key
5. Copy the key for later use

**Free Tier Limits:**
- Unlimited requests
- No authentication required for public data

### Step 1.3: Create Slack OAuth2 App

1. Go to https://api.slack.com/apps
2. Click **Create New App → From scratch**
3. **App Name:** "Splunk IP Reputation Bot"
4. **Workspace:** Select your target workspace
5. In **OAuth & Permissions:**
   - Add scopes: `chat:write`, `chat:write.public`
6. Install to workspace
7. Copy **Bot User OAuth Token** (starts with `xoxb-`)
8. Note the **Channel ID** where alerts should post (format: `C0B23UX2T32`)

**To find Channel ID:**
- Right-click channel → View details → Channel ID in URL

### Step 1.4: Enable Gmail API

1. Go to https://console.cloud.google.com
2. Create a new project or select existing
3. Enable **Gmail API**
4. Create OAuth2 credentials (Desktop app):
   - Application type: Desktop
   - Download credentials JSON
5. For 2FA-enabled accounts, create an [App Password](https://myaccount.google.com/apppasswords)

### Step 1.5: Prepare Splunk Alert Configuration

**In Splunk:**
1. Navigate to **Alerts & Actions**
2. Create new alert or edit existing
3. Go to **Alert Actions → Webhook**
4. Enable webhook and set URL:
   ```
   https://[YOUR_N8N_INSTANCE]/webhook/e645d98e-f80c-47e5-b96e-762c96f3db76
   ```
5. Set payload (JSON) with at least:
   ```json
   {
     "result": {
       "src_ip": "$result.src_ip$",
       "reason": "$result.reason$"
     }
   }
   ```

---

## Phase 2: N8N Workflow Import

### Step 2.1: Access N8N Dashboard

1. Open your N8N instance: `https://[YOUR_N8N_INSTANCE]`
2. Log in with admin credentials
3. Navigate to **Workflows**

### Step 2.2: Import the Workflow

**Option A: From GitHub URL**
1. Click **+ New → Import from URL**
2. Paste: `https://raw.githubusercontent.com/danhub-co/Splunk_IP_Reputation/main/✅Splunk%20Alert%20-%20IP%20Reputation%20Check(1).json`
3. Click **Import**

**Option B: From File Upload**
1. Click **+ New → Import from File**
2. Upload `✅Splunk Alert - IP Reputation Check(1).json`
3. Click **Open**

### Step 2.3: Verify Workflow Import

- ✅ All nodes visible
- ✅ Connections intact
- ✅ 11 nodes loaded:
  - Splunk Alert (webhook)
  - Extract IOCs
  - VirusTotal IP reputation check
  - AlienVault Lookup
  - Merge Threat Data
  - Process Intel Data
  - Generate IP Summary
  - Filter Suspicious IPs
  - IP summary display
  - Slack IP Alert
  - Gmail

---

## Phase 3: Credential Configuration

### Step 3.1: Configure VirusTotal Credentials

1. In N8N, click the **VirusTotal IP reputation check** node
2. In **Authentication** dropdown, select **Create new credential**
3. Credential type: **VirusTotal**
4. **API Key:** Paste your VirusTotal API key
5. Click **Create**

### Step 3.2: Configure AlienVault Credentials

1. Click the **AlienVault Lookup** node
2. In **Authentication** dropdown, select **Create new credential**
3. Credential type: **AlienVault**
4. **API Key:** Paste your AlienVault API key
5. Click **Create**

### Step 3.3: Configure Slack OAuth2

1. Click the **Slack IP Alert** node
2. Under **Slack OAuth2 API**, click **Create new credential**
3. Click **Connect my account** or **Authenticate**
4. Grant permissions to your Slack workspace
5. Select the target channel from dropdown
6. Click **Create**

### Step 3.4: Configure Gmail OAuth2

1. Click the **Gmail** node
2. Under **Gmail OAuth2**, click **Create new credential**
3. Click **Connect my account**
4. Authenticate with your Gmail account
5. Grant Gmail API permissions
6. Click **Create**

---

## Phase 4: Workflow Customization

### Step 4.1: Update Gmail Recipient

1. Click the **Gmail** node
2. In **Send To** field, change:
   ```
   FROM: user@example.com
   TO: your-soc-team@company.com
   ```
3. Update **Subject** if desired:
   ```
   [New Alert] IP Reputation Check Summary
   ```

### Step 4.2: Update Slack Channel (if different)

1. Click the **Slack IP Alert** node
2. In **Channel ID** dropdown, select your target channel
3. (Or keep current if already configured)

### Step 4.3: Customize HTML Report Template (Optional)

1. Click the **IP summary display** node
2. Modify HTML template in **html** parameter to match your branding
3. Add company logo, colors, or additional fields

### Step 4.4: Adjust Threat Severity Levels (Optional)

1. Click the **Filter Suspicious IPs** node
2. Modify conditions to change alert trigger:
   ```
   Current: Status equals "Suspicious"
   Options: Change to "High Risk", "Medium Risk", etc.
   ```

---

## Phase 5: Testing & Validation

### Step 5.1: Test VirusTotal Integration

1. Click **VirusTotal IP reputation check** node
2. Click **Test step**
3. Input test data:
   ```json
   {
     "ip_address": "8.8.8.8"
   }
   ```
4. Verify response contains:
   - `data.attributes.tags`
   - `data.attributes.last_analysis_stats`
5. ✅ Should return successful response

### Step 5.2: Test AlienVault Integration

1. Click **AlienVault Lookup** node
2. Click **Test step**
3. Input test data:
   ```json
   {
     "ip_address": "8.8.8.8"
   }
   ```
4. Verify response contains threat data
5. ✅ Should return 200 OK

### Step 5.3: Test Full Workflow

1. Click **Listen for Webhook** button on **Splunk Alert** node
2. Send test alert from Splunk:
   ```bash
   curl -X POST "https://[N8N_INSTANCE]/webhook/e645d98e-f80c-47e5-b96e-762c96f3db76" \
   -H "Content-Type: application/json" \
   -d '{
     "result": {
       "src_ip": "192.0.2.1",
       "reason": "Suspicious outbound connection"
     }
   }'
   ```
3. Monitor workflow execution
4. Verify all nodes execute successfully

### Step 5.4: Verify Email Delivery

1. Check inbox for:
   - **From:** N8N instance
   - **Subject:** "[New Alert] IP Reputation Check Summary"
   - **Content:** HTML report with IP reputation data
2. ✅ Email should arrive within 30 seconds

### Step 5.5: Verify Slack Notification

1. If IP marked as "Suspicious", check Slack channel for:
   - **Message source:** Slack bot integration
   - **Content:** IP details and threat classification
2. ✅ Notification should appear within 30 seconds

### Test Data (Known Safe IPs)

Use these IPs for testing:
- `8.8.8.8` (Google DNS - Clean)
- `1.1.1.1` (Cloudflare DNS - Clean)
- `192.0.2.1` (Documentation - Unknown)

---

## Phase 6: Production Activation

### Step 6.1: Review Security Settings

- ✅ Credentials secured (not exposed in workflow)
- ✅ Webhook path is unique
- ✅ API keys rotated if necessary
- ✅ Email recipients verified
- ✅ Slack channel restricted to authorized users

### Step 6.2: Enable Workflow

**Method 1: UI**
1. Click workflow name → **Settings**
2. Toggle **Active** to ON
3. Click **Save**

**Method 2: Edit JSON**
1. Change `"active": false` to `"active": true`
2. Save and deploy

### Step 6.3: Configure Splunk Alert

1. In Splunk, create/update alert with:
   ```
   Webhook URL: https://[YOUR_N8N_INSTANCE]/webhook/e645d98e-f80c-47e5-b96e-762c96f3db76
   Method: POST
   Execution: Real-time or scheduled
   ```
2. Test alert trigger
3. Verify N8N receives webhook

### Step 6.4: Set Up Logging & Monitoring

1. Enable N8N logging:
   - Level: **Info** (production) or **Debug** (troubleshooting)
2. Configure alerts for workflow failures:
   - Set up email notifications on failed executions
3. Create N8N dashboard for monitoring

---

## Monitoring & Maintenance

### Daily Tasks
- Check workflow execution logs for errors
- Monitor failed webhook deliveries
- Review alert accuracy in Slack/Email

### Weekly Tasks
- Analyze threat detection patterns
- Review false positive rates
- Validate API response times

### Monthly Tasks
- Audit API quota usage (VirusTotal)
- Rotate credentials (recommended)
- Review and optimize workflow performance
- Check for N8N updates

### Key Metrics to Monitor
```
- Workflow execution time: Target <5 seconds
- API success rate: Target >99%
- Email delivery rate: Target >99%
- Slack notification latency: Target <1 minute
- Alert accuracy: Track false positives
```

### N8N Execution Logs Location
- **UI:** Workflows → [Workflow Name] → Executions
- **Log Format:** Timestamp, Node Name, Status, Duration, Error Message

---

## Troubleshooting

### Issue: Webhook Not Receiving Data

**Symptoms:**
- Workflow not triggering on Splunk alerts
- No execution logs in N8N

**Solutions:**
1. Verify webhook URL in Splunk matches exactly
2. Check N8N webhook path is listening
3. Test with curl command:
   ```bash
   curl -X POST "https://[N8N_INSTANCE]/webhook/e645d98e-f80c-47e5-b96e-762c96f3db76" \
   -H "Content-Type: application/json" \
   -d '{"test": "data"}'
   ```
4. Check firewall rules allow outbound traffic from Splunk
5. Verify N8N instance is running

### Issue: VirusTotal API Fails

**Symptoms:**
- Error: "401 Unauthorized" or "429 Too Many Requests"
- Node shows red error indicator

**Solutions:**
1. Verify API key is correct (no spaces/typos)
2. Check API quota: https://www.virustotal.com/gui/settings/api-key
3. If rate limited, upgrade to paid tier or increase delay between requests
4. Add retry logic in VirusTotal node:
   - Set **onError:** "Wait then Retry"
   - Set **Wait Time:** 2-5 seconds

### Issue: AlienVault API Timeout

**Symptoms:**
- Error: "Request timeout" or "ECONNREFUSED"
- Slow workflow execution

**Solutions:**
1. Check AlienVault API status: https://otx.alienvault.com/api
2. Increase timeout in node settings to 30 seconds
3. Add error handling: Set **onError: "Continue Regular Output"** (already configured)
4. Test connectivity: `ping otx.alienvault.com`

### Issue: Slack Notification Not Sending

**Symptoms:**
- Slack node shows error
- No messages in target channel
- Error: "channel_not_found"

**Solutions:**
1. Verify Channel ID is correct (not channel name)
2. Confirm Slack app is installed to workspace
3. Check bot has permission to post in channel
4. Test Slack OAuth token validity
5. Verify target IP status is "Suspicious" (required for notification)

### Issue: Gmail Not Sending Email

**Symptoms:**
- Gmail node error or no delivery
- Error: "Invalid credentials" or "SMTP failed"

**Solutions:**
1. Verify Gmail OAuth token is current
2. If using 2FA, use [App Password](https://myaccount.google.com/apppasswords) instead
3. Check email recipient address is valid
4. Verify Gmail account has IMAP enabled
5. Check spam folder if email arrives late

### Issue: Workflow Execution Slow

**Symptoms:**
- Execution time >10 seconds
- API responses delayed

**Solutions:**
1. Check VirusTotal/AlienVault API status
2. Review N8N instance resource usage (CPU, Memory)
3. Reduce HTML report complexity
4. Parallelize API calls (already implemented)
5. Add caching for repeated IPs

### Issue: Data Not Merging Correctly

**Symptoms:**
- Merge Threat Data node shows unexpected output
- Missing fields in summary

**Solutions:**
1. Verify all three inputs reach merge node
2. Check Process Intel Data code for parsing errors
3. Add debug nodes to inspect intermediate data
4. Review N8N logs for JavaScript errors

---

## Rollback Procedure

If deployment fails or issues occur:

### Quick Rollback
1. Disable workflow: Toggle **Active** to OFF
2. Disable Splunk alert webhook
3. Verify Splunk alerts continue normally (without N8N)

### Full Rollback
1. Disable workflow in N8N
2. Remove N8N webhook from Splunk alert
3. Delete N8N credentials (optional)
4. Archive workflow version in N8N

### Recovery Steps
1. Review error logs in N8N and Splunk
2. Identify failed component (API, credential, data format)
3. Fix configuration issue
4. Re-enable and test thoroughly

---

## Support & Resources

### Documentation Links
- **N8N Documentation:** https://docs.n8n.io
- **VirusTotal API:** https://developers.virustotal.com
- **AlienVault OTX:** https://otx.alienvault.com/api
- **Slack API:** https://api.slack.com
- **Gmail API:** https://developers.google.com/gmail

### GitHub Repository
- **Repo:** https://github.com/danhub-co/Splunk_IP_Reputation
- **Issues:** Report bugs or request features

### Contact
- For questions, create a GitHub issue
- For urgent issues, contact the SOC team

---

## Deployment Checklist

Use this checklist to track deployment progress:

```
PRE-DEPLOYMENT
☐ VirusTotal API key obtained
☐ AlienVault API key obtained
☐ Slack OAuth2 configured
☐ Gmail OAuth2 configured
☐ Splunk alert webhook prepared
☐ N8N instance deployed and running

IMPORT & SETUP
☐ Workflow imported into N8N
☐ All 11 nodes visible and connected
☐ VirusTotal credentials added
☐ AlienVault credentials added
☐ Slack OAuth2 connected
☐ Gmail OAuth2 connected

CUSTOMIZATION
☐ Gmail recipient email updated
☐ Slack channel verified
☐ HTML template reviewed (optional)
☐ Threat severity levels configured (optional)

TESTING
☐ VirusTotal API test successful
☐ AlienVault API test successful
☐ Full workflow end-to-end test passed
☐ Email delivery verified
☐ Slack notification verified
☐ Test with "Suspicious" IP successful

PRODUCTION
☐ Security settings reviewed
☐ Workflow activated
☐ Splunk alert configured
☐ Logging enabled
☐ Monitoring dashboard set up
☐ Team trained on alert process

MAINTENANCE
☐ Backup credentials documented (secure location)
☐ Runbook created for on-call team
☐ Escalation procedures documented
☐ Monthly maintenance schedule set
```

---

**Deployment Date:** _______________  
**Deployed By:** _______________  
**Status:** ☐ Complete ☐ In Progress ☐ Rollback

**Last Updated:** 2026-05-14
