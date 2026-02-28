# Troubleshooting Guide

This guide helps you diagnose and resolve common issues with the App Portal for Intune.

---

## Table of Contents

- [How to View Logs](#how-to-view-logs)
  - [Application Insights (Recommended)](#application-insights-recommended)
  - [App Service Log Stream](#app-service-log-stream)
  - [Packaging Job Status](#packaging-job-status)
  - [Background Service Logs](#background-service-logs)
- [Common Issues](#common-issues)
  - [All Admin Pages Return 403 Forbidden](#all-admin-pages-return-403-forbidden)
  - [WinGet Package Publishing Errors](#winget-package-publishing-errors)
  - [Migration Issues](#migration-issues)

---

## How to View Logs

The App Portal for Intune uses multiple logging destinations. Each serves a different purpose.

### Application Insights (Recommended)

**Best for:** Detailed error traces, background service logs, performance monitoring

**Location:** Azure Portal → Your App Service → Application Insights → Logs

**Common Queries:**

**1. View Recent Errors (Last 1 Hour)**
```kusto
traces
| where severityLevel >= 3  // Warning, Error, Critical
| where timestamp > ago(1h)
| order by timestamp desc
| project timestamp, severityLevel, message, operation_Name
| take 100
```

**2. Packaging Service Logs**
```kusto
traces
| where message contains "PackagingQueueService"
| order by timestamp desc
| take 100
```

**3. Search for Specific Package Errors**
Replace `Google.GoogleDrive` with your package ID:
```kusto
traces
| where message contains "Google.GoogleDrive"
| order by timestamp desc
| take 50
```

**4. View All Logs for a Specific Job ID**
Replace `abc123` with your job ID:
```kusto
traces
| where message contains "abc123"
| order by timestamp desc
| take 100
```

**5. WinGet Cache Background Service**
```kusto
traces
| where message contains "WingetCacheStreamingService"
| order by timestamp desc
| take 100
```

**6. Failed Requests (500 errors)**
```kusto
requests
| where resultCode >= 500
| where timestamp > ago(1h)
| order by timestamp desc
| take 50
```

---

### App Service Log Stream

**Best for:** Real-time monitoring, startup issues, general web app logs

**Azure CLI:**
```bash
# Enable logging (one-time setup)
az webapp log config \
  --name <your-app-name> \
  --resource-group <your-resource-group> \
  --application-logging filesystem \
  --level verbose

# View live logs
az webapp log tail \
  --name <your-app-name> \
  --resource-group <your-resource-group>
```

**Azure Portal:**
1. Go to your App Service
2. Select **Log stream** from the left menu
3. Choose **Application Logs**

**Note:** Background services (packaging, cache refresh) may not appear in log stream. Use Application Insights for those.

---

### Packaging Job Status

**Best for:** Quick status check, user-facing error messages

**Location:** Admin Dashboard → Packaging Jobs tab

**What you see:**
- Job ID
- Package ID
- Status (Pending, Downloading, Packaging, Uploading, Completed, Failed)
- Error message (if failed)
- Start/completion time

**Limitations:**
- Error messages may be truncated
- Detailed stack traces not shown
- For full error details, use Application Insights

**Example Job Statuses:**

| Status | Description |
|--------|-------------|
| Pending | Job queued, waiting to be processed |
| Downloading | Downloading installer from WinGet repository |
| Packaging | Creating .intunewin package |
| Uploading | Uploading package to Intune |
| Completed | Successfully published to Intune |
| Failed | Error occurred (see error message) |

---

### Background Service Logs

Background services run independently of HTTP requests. Their logs appear in Application Insights, not the live log stream.

**Services that run in the background:**
1. **PackagingQueueService** - Processes WinGet packaging jobs
2. **WingetCacheStreamingService** - Refreshes cached packages daily
3. **RequestEscalationService** - Escalates stale approval requests
4. **AppSyncService** - Syncs apps from Intune
5. **LicenseValidationService** - Validates PowerStacks license

**How to view:**

Use Application Insights with service-specific queries:

```kusto
// Packaging service
traces
| where message contains "PackagingQueueService"
| order by timestamp desc
| take 100

// Cache refresh service
traces
| where message contains "WingetCacheStreamingService"
| order by timestamp desc
| take 100

// Request escalation service
traces
| where message contains "RequestEscalationService"
| order by timestamp desc
| take 100
```

---

### Verifying Application Insights is Enabled

Application Insights is critical for monitoring and troubleshooting. Follow these steps to verify it's enabled and working:

**1. Check if Application Insights Resource Exists**

Azure Portal → Resource Group → Look for resource named like `apprequest-insights-{uniqueId}`

If missing, the ARM template deployment may have failed. Redeploy using the latest template.

**2. Verify Connection String is Set**

Azure CLI:
```bash
# View all app settings
az webapp config appsettings list \
  --name <your-app-name> \
  --resource-group <your-resource-group> \
  --query "[?name=='APPLICATIONINSIGHTS_CONNECTION_STRING']"
```

Azure Portal:
1. Go to your App Service
2. Settings → Environment variables
3. Look for `APPLICATIONINSIGHTS_CONNECTION_STRING`
4. Value should start with `InstrumentationKey=...`

**If missing:** The ARM template sets this automatically. If deploying manually, copy the connection string from the Application Insights resource.

**3. Verify Application Insights is Not Disabled**

Azure Portal:
1. Go to your App Service
2. Settings → Application Insights
3. Ensure it shows "Connected" or "Enabled"
4. If it says "Disabled":
   - Click "Turn on Application Insights"
   - Select your existing Application Insights resource
   - Click "Apply"

**4. Test if Logs are Flowing**

**A. Check startup logs:**
```kusto
traces
| where message contains "Application Insights"
| order by timestamp desc
| take 10
```

You should see a message like:
```
Application Insights is configured with connection string: InstrumentationKey=...
```

**B. Generate test traffic:**
1. Visit your portal homepage
2. Navigate to a few pages
3. Wait 2-3 minutes for logs to appear

**C. Query for recent activity:**
```kusto
requests
| where timestamp > ago(10m)
| order by timestamp desc
| take 20
```

If you see results, Application Insights is working correctly.

**5. Common Issues**

**No logs appearing:**
- **Wait 2-5 minutes** - There's a delay before logs appear
- Check if the connection string is set correctly
- Restart the App Service to reinitialize Application Insights
- Verify the Application Insights resource isn't disabled or deleted

**"Application Insights was disabled" message:**
- This can happen if you manually disabled it in the portal
- Follow step 3 above to re-enable it
- The ARM template configures it automatically on deployment

**Connection string present but no data:**
- Verify the connection string matches the Application Insights resource
- Check if the Application Insights resource has any data (Portal → Application Insights → Logs)
- Restart the App Service

**6. Manual Configuration (if needed)**

If Application Insights isn't working after deployment:

```bash
# Get the connection string from Application Insights
az monitor app-insights component show \
  --app <insights-name> \
  --resource-group <resource-group> \
  --query "connectionString" -o tsv

# Set it in App Service
az webapp config appsettings set \
  --name <app-name> \
  --resource-group <resource-group> \
  --settings APPLICATIONINSIGHTS_CONNECTION_STRING="<connection-string>"

# Restart the app
az webapp restart \
  --name <app-name> \
  --resource-group <resource-group>
```

---

## Common Issues

### All Admin Pages Return 403 Forbidden

**Cause (v1.10.6+):** The portal requires `AdminGroupId` to be configured. If no Admin Group ID is set in either portal settings (database) or `appsettings.json` / environment variables, all admin endpoints return 403 Forbidden.

**Fix:**

1. Set the `AdminGroupId` via environment variable and restart:
   - **Azure App Service:** Go to Configuration → Application Settings → Add `AppSettings__AdminGroupId` with the Entra ID group Object ID → Save → Restart
   - **Local/IIS:** Set `AppSettings__AdminGroupId` environment variable, or add to `appsettings.json`:
     ```json
     { "AppSettings": { "AdminGroupId": "your-group-object-id" } }
     ```
2. Restart the application
3. Sign in with an account that is a member of that group
4. Navigate to Admin → Settings and save the Admin Group ID in portal settings

**Prevention:** Always keep `AdminGroupId` set in `appsettings.json` or environment variables as a fallback, even after configuring it in the portal UI.

### WinGet Package Publishing Errors

#### Error: "Failed to download installer manifest from GitHub: NotFound"

**Cause:** Package doesn't exist in Microsoft's winget-pkgs repository, or the path structure is different.

**Diagnosis:**
1. Check Application Insights for the full error message:
   ```kusto
   traces
   | where message contains "Failed to download installer manifest"
   | order by timestamp desc
   | take 10
   ```

2. Look for the manifest URL in the error message
3. Try opening the URL in a browser to verify if it exists

**Solution:**
- Verify the package ID is correct
- Check if the package exists in https://github.com/microsoft/winget-pkgs
- Some packages may not be available in the Microsoft repository
- Try searching for the package at https://winget.run

---

#### Error: "File {filename}.exe can not be found"

**Cause:** Downloaded installer file is missing or failed to download.

**Diagnosis:**
1. Check Application Insights for download logs:
   ```kusto
   traces
   | where message contains "Downloading to:"
   | order by timestamp desc
   | take 20
   ```

2. Look for:
   - "Download complete: {FilePath} ({Size} bytes)" - file downloaded successfully
   - "File verified: {FilePath} exists" - file validation passed
   - Any errors between these log entries

**Possible Causes:**
- Download failed (network issue, installer URL invalid)
- File was downloaded but is 0 bytes
- Path or filename too long (Windows 260 character limit)
- Temporary directory cleanup happened before packaging

**Solution:**
- Check the packaging job status for more details
- Verify the installer URL is valid and accessible
- Try publishing the package again
- Check App Service disk space

---

#### Error: "Failed to create .intunewin package"

**Cause:** SvRooij.ContentPrep library couldn't create the .intunewin file.

**Diagnosis:**
1. Check Application Insights for packaging logs:
   ```kusto
   traces
   | where message contains "Creating .intunewin package"
   | order by timestamp desc
   | take 20
   ```

2. Look for the error message from the packager

**Common Causes:**
- Installer file doesn't exist (see previous section)
- Installer file is corrupted or 0 bytes
- Insufficient disk space
- Invalid file format

**Solution:**
- Verify the installer downloaded successfully
- Check App Service disk space and scaling
- Try a different package to isolate the issue

---

### Install Status Issues

#### Install Status Stuck on "Pending Install" — Graph API URL Issue

**Cause (fixed in v1.11.10):** Prior to v1.11.10, the Intune `retrieveDeviceAppInstallationStatusReport` API was called without the required `microsoft.graph.` namespace prefix. The Graph beta endpoint silently returned empty data instead of erroring, so the polling service saw no install data for any app/user and statuses remained stuck at PendingInstall or NotApplicable.

**Solution:** Upgrade to v1.11.10. After deploying, the next polling cycle (within 15 minutes) will query Intune with the correct URL and update all statuses.

---

#### Install Status Stuck on "Pending Install" — General

**Cause:** The Intune install status polling service hasn't detected a status change yet, or the device hasn't checked in with Intune.

**Diagnosis:**
1. Check if the background polling service is running:
   ```kusto
   traces
   | where message contains "Checking install status"
   | order by timestamp desc
   | take 10
   ```

2. Verify the user/device is in the correct deployment group in Entra ID

3. Check if the device is online and syncing with Intune

**Common Causes:**
- Device hasn't synced with Intune since the assignment was created (can take up to 8 hours)
- User/device was not successfully added to the deployment group
- App assignment filter is excluding the device
- The polling service runs every 15 minutes — wait for the next cycle

**Solution:**
- On the target device, open Company Portal and trigger a sync
- Verify group membership in Entra ID
- Check the Intune device status report for the specific app

---

#### Install Status Shows "Pending Install" but Intune Shows "Failed"

**Cause (fixed in v1.11.4):** Prior to v1.11.4, the install status parser had incorrect numeric mappings — Intune's `resultantAppState` value `"2"` (Failed) was mapped to Installing, and `"5"` (PendingInstall) was mapped to Failed. This caused failures to be displayed as "Installing" or to remain stuck at "Pending Install".

**Solution:** Upgrade to v1.11.4 or later. The fix corrects all numeric mappings to match Microsoft's `resultantAppState` enum. After upgrading, the next polling cycle (within 15 minutes) will update all affected requests to the correct status.

---

#### Install Status Stuck at "Not Applicable" / Deployment Status Shows All Zeros

**Cause (fixed in v1.11.9):** Prior to v1.11.9, requests that were auto-approved (apps without an approval workflow) never initialized `InstallStatus` to `PendingInstall`. The status defaulted to `NotApplicable`, so the install status polling service never tracked their deployment progress. This only affected auto-approved requests — manually-approved requests via the approval workflow were set correctly.

**Solution:** Upgrade to v1.11.9. After upgrading, run the following SQL against your database to fix existing requests that are stuck:

```sql
UPDATE AppRequests
SET InstallStatus = 1  -- PendingInstall
WHERE Status = 4       -- Completed
  AND InstallStatus = 0 -- NotApplicable
  AND AppId IN (
    SELECT Id FROM Apps
    WHERE IntuneAppId IS NOT NULL
      AND IntuneAppId NOT LIKE 'winget-%'
  );
```

The polling service will then pick up these requests and check Intune for their actual install status within 15 minutes.

---

#### PSADT-Wrapped App Fails to Install (Error 0x8007EA68)

**Cause (fixed in v1.11.5):** Prior to v1.11.5, the generated `Deploy-Application.ps1` for MSI installers passed `-Parameters '/qn /norestart'` to `Execute-MSI`. PSADT v4 renamed this parameter and already applies `/qn /norestart` automatically in Silent deploy mode, causing a parameter conflict.

**Diagnosis:**
1. Check the Intune device install report for the app — look for `HexErrorCode: 0x8007EA68` (decimal 60008)
2. Check PSADT logs on the target device: `C:\Windows\Logs\Software\`
3. Error codes in the 60000 range indicate PSADT framework errors

**PSADT Error Codes:**

| Exit Code | Hex Code | Description |
|-----------|----------|-------------|
| 60001 | 0x8007EA61 | General PSADT error |
| 60002 | 0x8007EA62 | Pre-installation phase error |
| 60003 | 0x8007EA63 | Installation phase error |
| 60008 | 0x8007EA68 | Generic non-zero exit from wrapped installer |

**Solution:** Upgrade to v1.11.5 or later. Then delete the affected app from Intune using the Delete button in the portal and republish it. The regenerated PSADT script will no longer pass conflicting silent switches.

---

### Migration Issues

#### Error: "Invalid column name 'PopularityRank'"

**Cause:** Database migration hasn't been applied.

**Diagnosis:**
1. Check which migrations have been applied:
   ```sql
   SELECT MigrationId, ProductVersion
   FROM __EFMigrationsHistory
   ORDER BY MigrationId DESC
   ```

2. Look for migration `20260219003512_AddPopularityRankAndCategoryToWingetCache`

**Solution:**

**Option 1: Apply migration manually**
```sql
-- Add missing columns
ALTER TABLE WingetPackageCache ADD PopularityRank INT NULL;
ALTER TABLE WingetPackageCache ADD Category NVARCHAR(MAX) NULL;

-- Record migration
INSERT INTO __EFMigrationsHistory (MigrationId, ProductVersion)
VALUES ('20260219003512_AddPopularityRankAndCategoryToWingetCache', '8.0.0');
```

**Option 2: Restart App Service**
Migrations run automatically on startup. Restarting may trigger the migration.

---

## Getting Help

If you can't resolve an issue:

1. **Gather information:**
   - Error message from Application Insights (full text)
   - Job ID or Request ID if applicable
   - Steps to reproduce the issue
   - Timestamp when the error occurred

2. **Check existing issues:**
   - GitHub Issues: https://github.com/powerstacks-corp/app-portal-for-intune/issues

3. **Open a new issue:**
   - Include all gathered information
   - Paste Application Insights query results
   - Include screenshots if helpful

---

## Useful Azure CLI Commands

**View App Service Configuration**
```bash
az webapp config show \
  --name <your-app-name> \
  --resource-group <your-resource-group>
```

**Restart App Service**
```bash
az webapp restart \
  --name <your-app-name> \
  --resource-group <your-resource-group>
```

**View Environment Variables**
```bash
az webapp config appsettings list \
  --name <your-app-name> \
  --resource-group <your-resource-group>
```

**View Connection Strings**
```bash
az webapp config connection-string list \
  --name <your-app-name> \
  --resource-group <your-resource-group>
```

**Check App Service Health**
```bash
az webapp show \
  --name <your-app-name> \
  --resource-group <your-resource-group> \
  --query "state"
```

---

## Additional Resources

- [Admin Guide](ADMIN-GUIDE.md) - Complete administration documentation
- [Setup Guide](SETUP.md) - Deployment and configuration
- [Security Guide](SECURITY.md) - Security best practices
- [Architecture Decisions](ARCHITECTURE_DECISIONS.md) - Why we made certain design choices
