# App Request Portal

A self-service app request solution for Microsoft Intune. Enable your users to browse and request software deployments through an intuitive web portal, with configurable approval workflows and automatic Azure AD group management.

---

## Deploy to Azure

Click the button below to deploy the App Request Portal to your Azure subscription:

<p>
  <a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FPowerStacks-BI%2FApp-Request-Portal-Releases%2Fmain%2Fazuredeploy.json">
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
| SQL Server | `Microsoft.Sql/servers` | - | Database server |
| SQL Database | `Microsoft.Sql/servers/databases` | Basic (5 DTU) | Application data storage |
| Application Insights | `Microsoft.Insights/components` | - | Application monitoring and logging |
| Log Analytics Workspace | `Microsoft.OperationalInsights/workspaces` | PerGB2018 | Centralized logging |

**Estimated Monthly Cost**: ~$50-80 USD (varies by region and usage)

> **Note**: You can scale the App Service Plan and SQL Database up or down after deployment based on your organization's needs.

---

## Features

- **Self-Service App Catalog** - Users browse and request apps from your Intune library
- **Approval Workflows** - Configurable multi-stage approvals with manager and group-based routing
- **Automatic Group Management** - Creates Azure AD groups and Intune assignments automatically
- **Email Notifications** - Notify requesters and approvers at each workflow stage
- **Dark Mode** - Admin-configurable with user override option
- **Reports & Analytics** - ROI calculator, usage reports, and approval metrics
- **Branding** - Customize logo, colors, and text to match your organization
- **Winget Integration** - Browse and publish apps from the Windows Package Manager catalog

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

### Entra ID permissions

The user creating the app registrations must be able to:
- Create app registrations
- Grant admin consent for API permissions

---

## Deployment Wizard Input Summary

During deployment you will be prompted for:

| Parameter | Description | Where to find it |
|-----------|-------------|------------------|
| **Tenant ID** | Your Azure AD tenant ID | API app → Overview |
| **API Client ID** | Application ID of the backend app | API app → Overview |
| **API Client Secret** | Secret value from the backend app | API app → Certificates & secrets |
| **Frontend Client ID** | Application ID of the frontend app | Frontend app → Overview |
| **SQL Admin Password** | Password for the SQL database | Create a strong password (12+ chars) |
| **Admin Group ID** | (Optional) Azure AD group for admins | Azure AD → Groups → Object ID |
| **Approver Group ID** | (Optional) Azure AD group for approvers | Azure AD → Groups → Object ID |

---

## After Deployment

### 1. Copy the Portal URL

From the **Deployment Outputs**, copy the **Portal URL** (e.g., `https://app-apprequest-prod-abc123.azurewebsites.net`)

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
