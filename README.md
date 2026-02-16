# App Request Portal

A self-service app request solution for Microsoft Intune. Enable your users to browse and request software deployments through an intuitive web portal, with configurable approval workflows and automatic Azure AD group management.

---

## Deploy to Azure

Click the button below to deploy the App Request Portal to your Azure subscription:

<p>
  <a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FPowerStacks-BI%2FApp-Request-Portal-Releases%2Fmain%2Fazuredeploy.json/createUIDefinitionUri/https%3A%2F%2Fraw.githubusercontent.com%2FPowerStacks-BI%2FApp-Request-Portal-Releases%2Fmain%2FcreateUiDefinition.json">
    <img src="https://aka.ms/deploytoazurebutton" alt="Deploy to Azure">
  </a>
</p>

> **Important**
> Before clicking **Deploy to Azure**, review the **Prerequisites** section below.
> You must create two Entra ID app registrations before deployment.

---

## What This Deployment Creates

The deployment creates the following Azure resources:

| Resource | Type | SKU/Tier | Purpose |
|----------|------|----------|---------|
| App Service Plan | `Microsoft.Web/serverfarms` | B2 Basic | Hosts the web application |
| Web App | `Microsoft.Web/sites` | Linux, .NET 8 | The App Request Portal application |
| Azure Key Vault | `Microsoft.KeyVault/vaults` | Standard | Secure secret storage |
| SQL Server | `Microsoft.Sql/servers` | - | Database server |
| SQL Database | `Microsoft.Sql/servers/databases` | Basic (5 DTU) | Application data storage with geo-redundant backups |
| Storage Account | `Microsoft.Storage/storageAccounts` | Standard GRS | Geo-redundant Queue and Blob storage for Winget packaging |
| Application Insights | `Microsoft.Insights/components` | - | Application monitoring and logging |
| Log Analytics Workspace | `Microsoft.OperationalInsights/workspaces` | PerGB2018 | Centralized logging |

**Estimated Monthly Cost**: ~$200-300 USD (varies by region and usage)

> **Cost Details**: This includes B2 App Service Plan ($150-220/month), SQL Database with geo-redundant backups ($20-30/month), geo-redundant Storage Account ($10/month), Key Vault ($2-10/month), and Application Insights (pay-as-you-go, ~$10-20/GB ingested). Production deployments may require Standard or Premium tier App Service for higher availability.

> **Note**: You can scale the App Service Plan and SQL Database up or down after deployment based on your organization's needs.

---

## Features

### Core Capabilities
- **Self-Service App Catalog** - Users browse and request apps from your Intune library
- **User & Device Targeting** - Deploy apps to user profiles or specific devices
- **Approval Workflows** - Configurable multi-stage approvals with manager and group-based routing
- **Conditional Workflows** - Route approvals based on cost, category, platform, publisher, or department
- **Actionable Email Notifications** - Approve/reject directly from email without visiting the portal (Office 365 MessageCard format)
- **Auto-Escalation** - Automatically escalate stale approval requests with configurable thresholds
- **Per-App Acknowledgments** - Require users to acknowledge terms before requesting specific apps
- **Automatic Group Management** - Creates Entra ID groups and Intune assignments automatically

### Mobile & Platform Support
- **iOS & Android Apps** - Full support for iOS App Store, Google Play Store, and VPP apps
- **Windows, macOS, Linux** - Multi-platform app deployment

### Winget Integration
- **Winget Package Browser** - Browse and publish apps from the Windows Package Manager catalog to Intune
- **Automatic Updates** - Detect and apply Winget package updates to Intune apps with one click
- **Version History & Rollback** - Track app versions and rollback to previous versions if needed
- **Bulk Publishing** - Publish multiple Winget packages to Intune in one operation

### Reporting & Analytics
- **Intune Install Status Tracking** - Real-time deployment status monitoring (Pending, Installing, Installed, Failed)
- **Dashboard Charts & Trends** - Visualize request patterns, top apps, and completion trends over time
- **ROI Calculator** - Quantify help desk cost savings with automated app deployment
- **Detailed Reports** - Export analytics by app, user, approval activity, and installation status
- **Application Insights Integration** - Performance metrics, error tracking, and usage analytics

### Advanced Features
- **SLA Tracking** - Monitor request processing times with configurable SLA targets and breach alerts
- **Bulk Operations** - Request multiple apps at once or process multiple approvals simultaneously
- **Request on Behalf** - IT staff can submit requests for other users
- **Branding** - Customize logo, colors, and text to match your organization
- **Dark Mode** - Admin-configurable with user override option

---

## Prerequisites (Required Before Deployment)

### 1. Create the API Application (Backend)

This application handles backend authentication and Microsoft Graph API access.

#### How to create it in the Azure portal

1. Go to **https://entra.microsoft.com**
2. Navigate to **Identity → Applications → App registrations**
3. Click **New registration**
4. Configure the registration:
   - **Name**: `App Request Portal API`
   - **Supported account types**: Accounts in this organizational directory only (Single tenant)
   - **Redirect URI**: Leave blank for now
5. Click **Register**
6. From the **Overview** page, record:
   - **Application (client) ID**
   - **Directory (tenant) ID**

#### Add a client secret

1. In your new app registration, go to **Certificates & secrets**
2. Click **New client secret**
3. Add a description (e.g., `App Request Portal`) and select an expiration
4. Click **Add**
5. **Immediately copy the secret value** - it won't be shown again

> **Important**
> Store the client secret securely. You'll need it during deployment.

#### Configure API permissions

1. Go to **API permissions**
2. Click **Add a permission → Microsoft Graph → Application permissions**
3. Add the following permissions:

| Permission | Purpose |
|------------|---------|
| `DeviceManagementApps.Read.All` | Read Intune apps |
| `DeviceManagementApps.ReadWrite.All` | Create app assignments |
| `DeviceManagementManagedDevices.Read.All` | Read user devices |
| `Group.ReadWrite.All` | Create and manage Azure AD groups |
| `User.Read.All` | Read user profiles and managers |
| `Directory.Read.All` | Read directory data |
| `Mail.Send` | Send email notifications (optional) |

4. Click **Grant admin consent for [Your Organization]**

> **Important**
> Admin consent is required. Without it, the portal cannot access Microsoft Graph APIs.

#### Expose an API (for frontend authentication)

1. Go to **Expose an API**
2. Click **Set** next to Application ID URI, accept the default (e.g., `api://<client-id>`)
3. Click **Add a scope**:
   - **Scope name**: `access_as_user`
   - **Who can consent**: Admins and users
   - **Admin consent display name**: `Access App Request Portal`
   - **Admin consent description**: `Allow the application to access App Request Portal on behalf of the signed-in user`
4. Click **Add scope**

---

### 2. Create the Frontend Application (SPA)

This application handles user authentication in the browser.

#### How to create it in the Azure portal

1. Go to **https://entra.microsoft.com**
2. Navigate to **Identity → Applications → App registrations**
3. Click **New registration**
4. Configure the registration:
   - **Name**: `App Request Portal`
   - **Supported account types**: Accounts in this organizational directory only (Single tenant)
   - **Redirect URI**: Select **Single-page application (SPA)** and enter `https://placeholder.azurewebsites.net`
5. Click **Register**
6. From the **Overview** page, record:
   - **Application (client) ID**

> **Note**
> You'll update the Redirect URI after deployment with your actual portal URL.

#### Configure API permissions

1. Go to **API permissions**
2. Click **Add a permission → My APIs**
3. Select **App Request Portal API** (the app you created in Step 1)
4. Check **access_as_user** and click **Add permissions**

---

## Required Permissions

### Azure permissions

The user deploying the template must have:
- **Owner** role on the target resource group, OR
- **Contributor + User Access Administrator** roles

### Resource provider registration

Your subscription must have the following resource providers registered:
- `Microsoft.AlertsManagement` (required for Application Insights)

To check and register:
1. Go to **Azure Portal** → **Subscriptions** → select your subscription
2. Click **Resource providers** (under Settings)
3. Search for `Microsoft.AlertsManagement`
4. If not registered, select it and click **Register**

### Entra ID permissions

The user creating the app registrations must be able to:
- Create app registrations
- Grant admin consent for API permissions

---

## Deployment Wizard Input Summary

During deployment you will be prompted for:

| Parameter | Description | Where to find it |
|-----------|-------------|------------------|
| **API Client ID** | Application ID of the backend app | API app → Overview |
| **API Client Secret** | Secret value from the backend app | API app → Certificates & secrets |
| **Frontend Client ID** | Application ID of the frontend app | Frontend app → Overview |
| **SQL Admin Password** | Password for the SQL database | Create a strong password (12+ chars) |
| **Admin Group ID** | (Optional) Azure AD group for admins | Entra ID → Groups → Object ID |

> **Note**: The Tenant ID is automatically detected from your Azure session.

---

## After Deployment

### 1. Copy the Portal URL

After the deployment completes, you need to find your portal URL:

1. In the Azure Portal, you should see **"Your deployment is complete"**
2. Click **Outputs** in the left sidebar (or click **Deployment details** then **Outputs**)
3. Find **portalUrl** in the list and copy the value (e.g., `https://app-apprequest-prod-abc123.azurewebsites.net`)

> **Tip**: If you navigated away, go to your **Resource Group** → **Deployments** → select the deployment → **Outputs**

### 2. Update the Frontend Redirect URI

1. Go to **https://entra.microsoft.com**
2. Navigate to **Identity → Applications → App registrations**
3. Open **App Request Portal** (the frontend app)
4. Go to **Authentication**
5. Under **Single-page application**, update the Redirect URI to your portal URL
6. Click **Save**

### 3. Complete the Setup Wizard

1. Visit your portal URL
2. Sign in with an Azure AD account
3. Complete the **Setup Wizard**:
   - Enter your PowerStacks license key
   - Configure admin and approver groups
   - Set up email notifications (optional)
   - Sync apps from Intune

---

## Custom Domain (Optional)

To use a custom domain like `apps.yourdomain.com`:

1. Configure DNS (CNAME pointing to your App Service)
2. Add the custom domain in Azure Portal → App Service → Custom domains
3. Update your frontend app's Redirect URI to include the custom domain

---

## Documentation

- [Admin Guide](https://github.com/PowerStacks-BI/AppRequestPortal/blob/main/docs/ADMIN-GUIDE.md)
- [Deployment Guide](https://github.com/PowerStacks-BI/AppRequestPortal/blob/main/docs/DEPLOYMENT.md)
- [Custom Domains](https://github.com/PowerStacks-BI/AppRequestPortal/blob/main/docs/CUSTOM-DOMAINS.md)
- [Approval Workflows](https://github.com/PowerStacks-BI/AppRequestPortal/blob/main/docs/APPROVAL-WORKFLOWS.md)

---

## Support

- **Issues**: [GitHub Issues](https://github.com/PowerStacks-BI/AppRequestPortal/issues)
- **License**: Contact [PowerStacks](https://powerstacks.com) for licensing

---

## License

This software requires a valid PowerStacks license. Contact [powerstacks.com](https://powerstacks.com) for licensing information.

---

Built by [PowerStacks](https://powerstacks.com)
