# App Request Portal - Setup Guide

This guide walks you through setting up the App Request Portal from scratch.

## Prerequisites

- Azure subscription with appropriate permissions
- Entra ID tenant with Global Administrator or Application Administrator role
- .NET 8.0 SDK
- Node.js 18+ and npm
- Visual Studio 2022 or VS Code
- Azure CLI installed
- SQL Server (LocalDB, Express, or Developer Edition) or Azure SQL Database

### Install Required .NET Tools

Install the Entity Framework Core CLI tools:

```powershell
# Add NuGet source if not already configured
dotnet nuget add source https://api.nuget.org/v3/index.json -n nuget.org

# Install EF Core tools globally
dotnet tool install --global dotnet-ef
```

## Step 1: Entra ID App Registrations

You need to create two app registrations in Entra ID:

### Backend API App Registration

1. Navigate to Azure Portal > Microsoft Entra ID > App registrations
2. Click "New registration"
3. Name: `App Request Portal - API`
4. Supported account types: "Accounts in this organizational directory only"
5. Redirect URI: Leave empty for now
6. Click "Register"

7. Note the **Application (client) ID** and **Directory (tenant) ID**

8. Configure API permissions:
   - Click "API permissions" > "Add a permission"
   - Select "Microsoft Graph" > "Application permissions"
   - Add the following permissions:
     - `DeviceManagementApps.Read.All` - Read Intune apps
     - `DeviceManagementApps.ReadWrite.All` - Manage Intune apps and create assignments
     - `DeviceManagementManagedDevices.Read.All` - Read user devices
     - `Group.ReadWrite.All` - Create and manage security groups
     - `User.Read.All` - Read user profiles, managers, and group memberships
     - `Directory.Read.All` - Read directory data
     - `Mail.Send` - Send email notifications (optional, see Step 9)
     - `Chat.Create` - Create Teams chats for direct approver notifications (optional, see Step 11)
     - `Chat.ReadWrite.All` - Send Teams chat messages (optional, see Step 11)
   - Click "Grant admin consent"

   > **Note:** `DeviceManagementApps.ReadWrite.All` is required to automatically create Intune app assignments when apps are made visible in the portal.

   > **Note:** The Teams chat permissions (`Chat.Create` and `Chat.ReadWrite.All`) are optional. Without them, the portal will function normally but Teams direct chat notifications will not be available. You can still use webhook-based Teams channel notifications (Step 10) which don't require these permissions.

9. Create a client secret:
   - Click "Certificates & secrets" > "New client secret"
   - Description: `API Secret`
   - Expires: Choose appropriate duration
   - Click "Add"
   - **Copy the secret value immediately** (you won't be able to see it again)

10. Expose an API:
    - Click "Expose an API" > "Add a scope"
    - Application ID URI: Accept default or use `api://your-api-client-id`
    - Scope name: `access_as_user`
    - Who can consent: Admins and users
    - Display name: `Access API as user`
    - Description: `Allow the application to access the API as the signed-in user`
    - Click "Add scope"

### Frontend SPA App Registration

1. Click "New registration"
2. Name: `App Request Portal - Frontend`
3. Supported account types: "Accounts in this organizational directory only"
4. Redirect URI:
   - Type: "Single-page application (SPA)"
   - URI: `http://localhost:3000`
5. Click "Register"

6. Note the **Application (client) ID**

7. Configure API permissions:
   - Click "API permissions" > "Add a permission"
   - Select "APIs my organization uses" > Select your backend API app
   - Check `access_as_user`
   - Click "Add permissions"
   - Also add "Microsoft Graph" > "Delegated permissions" > `User.Read` (used for profile photo)
   - Click "Grant admin consent"

8. Configure authentication:
   - Click "Authentication"
   - Settings
   - Under "Implicit grant and hybrid flows", check:
     - Access tokens
     - ID tokens
   - Click "Save"

## Step 2: Configure Application Settings

### Backend API Configuration

1. Open [AppRequestPortal/src/AppRequestPortal.API/appsettings.json](../src/AppRequestPortal.API/appsettings.json)

2. Update the following values:
```json
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "Domain": "yourdomain.onmicrosoft.com",
    "TenantId": "your-tenant-id",
    "ClientId": "your-api-client-id",
    "ClientSecret": "your-client-secret"
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=your-sql-server;Database=AppRequestPortal;..."
  }
}
```

### Frontend Configuration

1. Copy [AppRequestPortal/src/AppRequestPortal.Web/.env.example](../src/AppRequestPortal.Web/.env.example) to `.env`

2. Update the values:
```
REACT_APP_CLIENT_ID=your-frontend-client-id
REACT_APP_TENANT_ID=your-tenant-id
REACT_APP_API_CLIENT_ID=your-api-client-id
REACT_APP_REDIRECT_URI=http://localhost:3000
REACT_APP_API_URL=https://localhost:7001/api
```

## Step 3: Database Setup

1. Ensure you have SQL Server running locally or use Azure SQL Database
   - **LocalDB** (included with Visual Studio): Connection string uses `(localdb)\MSSQLLocalDB`
   - **SQL Server Express**: Connection string uses `localhost\SQLEXPRESS`
   - **Azure SQL**: Use the full server FQDN from Azure Portal

2. Update the connection string in [appsettings.json](../src/AppRequestPortal.API/appsettings.json)

   Example for LocalDB:
   ```json
   "ConnectionStrings": {
     "DefaultConnection": "Server=(localdb)\\MSSQLLocalDB;Database=AppRequestPortal;Trusted_Connection=True;MultipleActiveResultSets=true"
   }
   ```

3. Restore NuGet packages:
```powershell
cd src/AppRequestPortal.API
dotnet restore
```

4. Run database migrations (from the API project directory):
```powershell
dotnet ef migrations add InitialCreate --project ../AppRequestPortal.Infrastructure --startup-project .
dotnet ef database update --project ../AppRequestPortal.Infrastructure --startup-project .
```

> **Note:** The `--project` flag points to where the DbContext lives (Infrastructure), and `--startup-project` points to the API project which has the configuration.

## Step 4: Run the Application Locally

### Start the Backend API

```powershell
cd src/AppRequestPortal.API
dotnet restore
dotnet run
```

The API will be available at `https://localhost:7001`

### Start the Frontend

```powershell
cd src/AppRequestPortal.Web
npm install
npm start
```

The web app will be available at `http://localhost:3000`

## Step 5: Test the Application

1. Navigate to `http://localhost:3000`
2. Click "Sign In with Microsoft"
3. Authenticate with your Entra ID account
4. You should see the home page

## Step 6: Configure Admin Access (REQUIRED)

> **CRITICAL (v1.10.6+):** You **must** configure the `AdminGroupId` before anyone can access admin functions. The portal uses a fail-closed security model — if no admin group is configured, **all admin endpoints return 403 Forbidden** for every user, including the person who deployed the portal. There is no first-run setup wizard; if you skip this step, the only way to recover admin access is to set the `AdminGroupId` in `appsettings.json` or the `AppSettings__AdminGroupId` environment variable and restart the application.

### Step 6a: Create Security Groups

1. Navigate to Azure Portal > Microsoft Entra ID > Groups
2. Create a new security group:
   - Group type: **Security**
   - Group name: `AppPortal-Admins` (or your preferred name)
   - Group description: `Administrators for the App Request Portal`
   - Membership type: **Assigned**
3. Click **Create**
4. Add users who should have admin access to this group
5. Copy the **Object ID** of the group (found on the group's Overview page)

### Step 6b: Set AdminGroupId in Configuration (Required)

You **must** set `AdminGroupId` before the first admin can log in. Choose one of the following methods:

**Method 1: appsettings.json** — Update [appsettings.json](../src/AppRequestPortal.API/appsettings.json):

```json
{
  "AppSettings": {
    "AdminGroupId": "your-admin-group-object-id",
    "ApproverGroupId": "your-approver-group-object-id"
  }
}
```

**Method 2: Environment variable** (recommended for Azure App Service):

```
AppSettings__AdminGroupId=your-admin-group-object-id
AppSettings__ApproverGroupId=your-approver-group-object-id
```

**Configuration options:**

| Setting | Description |
|---------|-------------|
| `AdminGroupId` | **(Required)** Object ID of the Entra ID group for administrators. Admins can sync apps from Intune and manage all settings. **Without this, no user can access admin features.** |
| `ApproverGroupId` | Object ID of the Entra ID group for approvers. Approvers can approve/reject app requests. |

You can use the same group for both settings, or create separate groups for more granular control.

### Step 6c: Configure Additional Settings via Portal UI

Once you have initial admin access (via the configuration above):

1. Navigate to **Admin** > **Settings** tab
2. Under **Group-Based Authorization**:
   - Enter the **Admin Group** Object ID (this saves it to the database so it persists independently of `appsettings.json`)
   - Enter the **Approver Group** Object ID
3. Under **App Deployment Settings**:
   - Set the **Group Name Prefix** (default: `AppPortal-`) - this prefix is used when auto-creating Entra ID security groups for app deployments
4. Click **Save Settings**

See [ADMIN-GUIDE.md](ADMIN-GUIDE.md) for detailed instructions on using the Portal Settings UI.

> **Note:** The portal checks the database settings first, then falls back to `appsettings.json`. Once you configure the admin group via the UI, the `appsettings.json` value serves as a fallback only. It is recommended to keep the `appsettings.json` value as a recovery mechanism in case the database value is accidentally cleared.

### App Deployment Settings

The portal automatically creates Entra ID security groups and Intune app assignments when apps are made visible. Configure the group naming:

| Setting | Default | Description |
|---------|---------|-------------|
| `GroupNamePrefix` | `AppPortal-` | Prefix for auto-created security groups. Groups are named `{prefix}{AppName}-Required`. |

This setting is configured via the Portal Settings UI (Admin > Settings > App Deployment Settings).

> **Tip:** Settings configured in the UI take precedence over appsettings.json values. The initial values from appsettings.json are used to seed the database on first run

## Step 7: Deploy to Azure

See [DEPLOYMENT.md](DEPLOYMENT.md) for detailed deployment instructions.

## Troubleshooting

### Authentication Issues

- Verify that all Entra ID app registration settings are correct
- Ensure client IDs and tenant IDs match in configuration files
- Check that admin consent has been granted for all required permissions

### API Connection Issues

- Verify the API URL in the frontend configuration
- Check CORS settings in the API
- Ensure the API is running and accessible

### Database Issues

- Verify the connection string is correct
- Ensure the SQL Server is running and accessible
- Check that migrations have been applied

### Graph API Permissions

- Ensure admin consent has been granted for all required permissions
- Verify the client secret is valid and not expired
- Check that the managed identity (when deployed to Azure) has appropriate permissions

## Step 8: Configure Approval Workflows

The portal supports flexible per-app approval workflows. See [APPROVAL-WORKFLOWS.md](APPROVAL-WORKFLOWS.md) for detailed configuration options including:

- Manager approval requirements
- Linear workflows (specific users approve in sequence)
- Pooled workflows (any member of a group can approve at each stage)
- Multi-stage approval chains

## Step 9: Configure Email Notifications (Optional)

The portal can send email notifications for:
- Request submitted confirmations
- Approval required notifications to approvers
- Request approved/rejected notifications to requestors

### Prerequisites

1. **Mail.Send permission** must be added to your API app registration (see Step 1, item 8)
2. A **user mailbox or shared mailbox** to send emails from

### Add Mail.Send Permission

If you didn't add it during initial setup:

1. Navigate to Azure Portal > Microsoft Entra ID > App registrations
2. Select your **backend API** app registration
3. Click "API permissions" > "Add a permission"
4. Select "Microsoft Graph" > "Application permissions"
5. Search for `Mail.Send` and check it
6. Click "Add permissions"
7. Click **"Grant admin consent for [your tenant]"** (requires Global Admin or Privileged Role Administrator)

### Get the User Object ID

You need the Object ID of the user or shared mailbox that will send emails:

1. Navigate to Azure Portal > Microsoft Entra ID > Users
2. Search for and select the user (or shared mailbox)
3. Copy the **Object ID** from the Overview page

> **Tip:** You can create a dedicated shared mailbox like `apprequest-noreply@yourdomain.com` for this purpose.

### Option A: Configure via Portal Settings UI (Recommended)

1. Navigate to **Admin** > **Communications** tab
2. Under **Email Notifications**:
   - Toggle **Enable email notifications** on
   - Enter the **Send As User ID** (Object ID of mailbox)
   - Enter the **From Address** (email address)
   - Enter the **Portal URL** (for email links)
3. Click **Save Settings**

### Option B: Configure via appsettings.json

Update [appsettings.json](../src/AppRequestPortal.API/appsettings.json):

```json
{
  "EmailSettings": {
    "SendAsUserId": "user-object-id-here",
    "FromAddress": "apprequest-noreply@yourdomain.com",
    "PortalUrl": "https://your-portal-url.com"
  }
}
```

| Setting | Description |
|---------|-------------|
| `SendAsUserId` | The Object ID of the user or shared mailbox to send emails from. If empty, email notifications are disabled. |
| `FromAddress` | The email address shown in the From field (should match the mailbox). |
| `PortalUrl` | The URL of your portal, used for links in email notifications.

> **Tip:** Settings configured in the UI take precedence over appsettings.json values |

### Test Email Notifications

1. Submit an app request
2. Check that the requestor receives a confirmation email
3. Check that approvers receive an approval request email
4. Approve or reject the request and verify the requestor receives the result notification

### Troubleshooting Email Issues

- **403 Forbidden**: Ensure `Mail.Send` permission has admin consent granted
- **User not found**: Verify the `SendAsUserId` is a valid Object ID
- **Email not sent**: Check the API logs for detailed error messages

## Step 10: Configure Microsoft Teams Channel Notifications (Optional)

Send notifications to a Microsoft Teams channel when app requests are submitted, approved, or rejected.

### Create an Incoming Webhook

1. In **Microsoft Teams**, navigate to the channel where you want notifications
2. Click the three dots (...) next to the channel name > **Manage channel**
3. Go to **Connectors** tab > search for **Incoming Webhook** > **Configure**
4. Enter a name (e.g., "App Request Portal") and optionally upload an icon
5. Click **Create** and **copy the webhook URL**

### Configure in Portal

1. Navigate to **Admin** > **Communications** tab
2. Under **Microsoft Teams Channel Notifications**:
   - Toggle **Enable Teams notifications** on
   - Paste the **Webhook URL**
   - Click **Test** to verify the connection
   - Select which events should trigger notifications
3. Click **Save Settings**

### What Gets Notified

| Event | Card Content |
|-------|-------------|
| **New Request** | Requestor name, app name, publisher, justification, link to review |
| **Approved** | Requestor, app, who approved, link to portal |
| **Rejected** | Requestor, app, who rejected, rejection reason |

See [ADMIN-GUIDE.md](ADMIN-GUIDE.md#microsoft-teams-notifications) for detailed setup instructions with screenshots.

## Step 11: Configure Microsoft Teams Direct Chat (Optional)

Send direct Teams chat messages to approvers when their approval is required. Unlike channel notifications, direct chat messages are sent individually to each approver, ensuring they receive personal notifications in their Teams chat.

### Prerequisites

1. **Chat.Create permission** must be added to your API app registration (see Step 1, item 8)
2. **Chat.ReadWrite.All permission** must be added to your API app registration (see Step 1, item 8)

### Add Teams Chat Permissions

If you didn't add them during initial setup:

1. Navigate to Azure Portal > Microsoft Entra ID > App registrations
2. Select your **backend API** app registration
3. Click "API permissions" > "Add a permission"
4. Select "Microsoft Graph" > "Application permissions"
5. Search for and add:
   - `Chat.Create` - Create chats with users
   - `Chat.ReadWrite.All` - Read and write all chat messages
6. Click "Add permissions"
7. Click **"Grant admin consent for [your tenant]"** (requires Global Admin or Privileged Role Administrator)

### Configure in Portal

1. Navigate to **Admin** > **Communications** tab
2. Under **Microsoft Teams Direct Chat**:
   - Toggle **Enable Teams direct chat** on
   - Select which events should trigger chat messages:
     - **Approval Required** - Notify approvers when their approval is needed
     - **Request Approved** - Notify requestor when their request is approved
     - **Request Rejected** - Notify requestor when their request is rejected
3. Click **Save Settings**

### How It Works

- When approval is required, each approver receives a personal Teams chat message
- For **pooled approvals** (group-based), the portal expands the group membership and sends individual messages to each group member
- For **sequential approvals**, only the current stage approvers are notified
- Messages include Adaptive Cards with request details and a link to the portal

### Troubleshooting Teams Chat Issues

- **403 Forbidden**: Ensure `Chat.Create` and `Chat.ReadWrite.All` permissions have admin consent granted
- **User not found**: Verify the user has a valid Teams license and can receive chats
- **Messages not appearing**: Check that the app has been granted tenant-wide admin consent

## Step 12: Configure Application Insights (Optional)

Application Insights provides telemetry logging and the in-portal **Application Insights Dashboard** (performance metrics, error tracking, usage analytics, health monitoring). There are two parts to configure:

### Part A: Connection String (Telemetry & Logging)

The `APPLICATIONINSIGHTS_CONNECTION_STRING` environment variable enables telemetry collection (logs, traces, exceptions). If you deployed via the ARM/Bicep template, this is already set automatically.

**To verify or set manually:**

1. Go to [Azure Portal](https://portal.azure.com) > your **Resource Group**
2. Click on the **Application Insights** resource (e.g., `ai-apprequest-prod`)
3. On the **Overview** page, copy the **Connection String** (click the copy icon)
4. Go to your **App Service** > **Configuration** > **Application settings**
5. Add or verify the setting:
   - **Name:** `APPLICATIONINSIGHTS_CONNECTION_STRING`
   - **Value:** The connection string you copied (starts with `InstrumentationKey=...`)
6. Click **Save** and restart the App Service

### Part B: App ID & API Key (Metrics Dashboard)

The in-portal Application Insights Dashboard tab uses the Application Insights REST API. This requires an **App ID** and **API Key** which are separate from the connection string.

**Without these settings**, the dashboard still works but uses database-only metrics (no direct Application Insights API queries).

**To configure:**

1. Go to [Azure Portal](https://portal.azure.com) > your **Resource Group**
2. Click on the **Application Insights** resource

**Get the App ID:**
3. In the left menu, click **API Access** (under Configure)
4. Copy the **Application ID** (this is the App ID, NOT the Instrumentation Key)

**Create an API Key:**
5. On the same **API Access** page, click **Create API key**
6. Enter a description: `App Request Portal Metrics Dashboard`
7. Check **Read telemetry** under permissions
8. Click **Generate key**
9. **Copy the key immediately** — it won't be shown again

**Add to App Service:**
10. Go to your **App Service** > **Configuration** > **Application settings**
11. Add two settings:
    - **Name:** `ApplicationInsights__AppId` — **Value:** The Application ID from step 4
    - **Name:** `ApplicationInsights__ApiKey` — **Value:** The API key from step 9
12. Click **Save** and restart the App Service

> **Note:** Use double underscores (`__`) in the setting names — this is how ASP.NET Core maps environment variables to the `ApplicationInsights:AppId` configuration path.

> **Note:** For local development, add these to `appsettings.json` under an `ApplicationInsights` section:
> ```json
> {
>   "ApplicationInsights": {
>     "AppId": "your-app-id",
>     "ApiKey": "your-api-key"
>   }
> }
> ```

### Verifying Application Insights

After configuration, verify it's working:

1. Navigate to **Admin** > **Application Insights** tab in the portal
2. You should see performance metrics, error counts, and usage data
3. If you see "Application Insights is not configured", check that both `ApplicationInsights__AppId` and `ApplicationInsights__ApiKey` are set correctly

For detailed troubleshooting, see [TROUBLESHOOTING.md](TROUBLESHOOTING.md#verifying-application-insights-is-enabled).

---

## Troubleshooting Deployment Issues

### Key Vault Reference Failures (Red X Marks)

After deploying to Azure App Service, you may see red X marks next to Key Vault references in the Configuration settings, indicating the app cannot access secrets. This is usually caused by Azure AD identity propagation delays.

**Symptoms:**
- App Service Configuration shows red X marks next to Key Vault source settings
- `/health` endpoint returns 503 Service Unavailable
- `/health/migrations` returns error: `Keyword not supported: '@microsoft.keyvault'`
- App logs show "ArgumentException" related to connection strings

**Root Cause:**
When deploying with Managed Identity and Key Vault for the first time, there's a 5-15 minute propagation delay for the identity to sync across Azure services. During this time, the App Service cannot resolve Key Vault references.

**Solution:**

1. **Wait for propagation (recommended for new deployments):**
   - After deployment completes, wait 10-15 minutes
   - Refresh the Configuration page to check if red X marks turn to green checkmarks
   - Restart the App Service
   - Verify `/health` and `/health/migrations` endpoints return successful responses

2. **Verify Managed Identity is enabled:**
   - Go to App Service → **Identity** → **System assigned**
   - Ensure Status is **On**
   - Note the **Object (principal) ID**

3. **Verify Key Vault access policy:**
   - Go to Key Vault → **Access policies**
   - Verify an access policy exists for your App Service's Managed Identity
   - Required permissions: **Get** and **List** under Secret permissions
   - If missing, click **+ Create** and add the App Service principal

4. **Check Key Vault networking:**
   - Go to Key Vault → **Networking**
   - Ensure either:
     - "Allow public access from all networks" is selected, OR
     - App Service outbound IPs are added to the firewall allowlist

5. **Verify secret names match:**
   - Go to Key Vault → **Secrets**
   - Verify these secrets exist:
     - `AzureAdClientSecret` (or similar name for the client secret)
     - `SqlConnectionString` (or the name referenced in connection strings)
     - `StorageConnectionString` (if using Azure Storage)
   - In App Service → Configuration, click "Show value" on each Key Vault reference
   - The secret name in the URL must **exactly match** the secret name in Key Vault (case-sensitive)

6. **Force refresh:**
   - Go to App Service → **Restart**
   - Wait 2-3 minutes for cold start
   - Check Configuration again for green checkmarks

**Prevention for Future Deployments:**
- After deploying ARM template, wait 10 minutes before testing the app
- Use `/health/migrations` endpoint to verify database connectivity before troubleshooting further
- Always check Configuration page for green checkmarks on Key Vault references

### Database Migration Issues

**Symptoms:**
- `/health/migrations` shows `"pendingCount" > 0`
- Portal returns "Error loading settings" or "Error saving license key"
- Database tables are missing

**Solution:**
1. Verify `/health/migrations` endpoint shows pending migrations
2. Restart the App Service (migrations auto-apply on startup)
3. Wait 2 minutes and check `/health/migrations` again
4. If still pending, check Application Insights logs for migration errors
5. Common errors:
   - **SQL firewall:** Add App Service IP to SQL Server firewall rules
   - **Connection timeout:** Increase timeout in connection string
   - **Permission denied:** Ensure SQL user has db_owner role

See [.claude/DEPLOYMENT-CHECKLIST.md](.claude/DEPLOYMENT-CHECKLIST.md) for comprehensive post-deployment verification steps.

---

## Next Steps

- Read the [Admin Guide](ADMIN-GUIDE.md) to learn how to configure the portal through the web UI
- Configure Conditional Access policies for enhanced security
- Set up Azure Monitor and Application Insights for monitoring
- Customize the UI to match your organization's branding
