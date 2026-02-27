# Admin Guide

This guide walks administrators through setting up and managing the App Request Portal using the web-based admin interface.

## Getting Started

After initial deployment and Entra ID configuration (see [SETUP.md](SETUP.md)), most portal configuration can be done directly through the Admin Dashboard.

### Accessing the Admin Dashboard

1. Sign in to the portal with an account that is a member of the Admin Group
2. Click **Admin** in the navigation menu
3. You'll see seven tabs:
   - **App Management** - Manage apps synced from Intune
   - **Pending Approvals** - Review and approve/reject requests
   - **Settings** - Configure portal-wide options (authorization, display, deployment)
   - **Communications** - Notification settings, company branding, and Terms of Service
   - **Branding** - Customize portal appearance
   - **Winget Catalog** - Browse and publish apps from Winget
   - **Reports** - View analytics, trends, and deployment status

### Setup Wizard

For first-time setup or to reconfigure the portal, use the **Setup Wizard**:

1. Go to **Admin** > **Settings** tab
2. Click the **🚀 Setup Wizard** button
3. Follow the guided steps:
   - **Welcome** - Overview of setup steps
   - **License** - Enter and validate your PowerStacks license key
   - **Access Groups** - Configure Admin and Approver Entra ID groups
   - **Email Notifications** - Set up email settings for notifications
   - **Sync Apps** - Import apps from your Intune tenant

The wizard saves your settings as you progress through each step. You can skip the wizard at any time and configure settings manually.

> **Note:** The License step is required for the portal to be fully operational. Without a valid license, users will see warning banners and some features may be restricted.

**Quick Reference Commands** (shown in wizard completion):
| Component | Command |
|-----------|---------|
| API (includes packaging service) | `cd src/AppRequestPortal.API && dotnet run` |
| Web | `cd src/AppRequestPortal.Web && npm start` |

## Portal Settings

The Settings tab allows you to configure portal-wide options including authorization, display settings, deployment configuration, and version management. Notification and messaging settings are on the **Communications** tab (see below).

### Group-Based Authorization

Control who has admin and approver access to the portal.

| Setting | Description |
|---------|-------------|
| **Admin Group** | **(Required)** Entra ID group Object ID. Members have full admin access to sync apps, manage settings, and view all requests. **If not configured, all admin endpoints return 403 Forbidden.** |
| **Approver Group** | Entra ID group Object ID. Members can approve/reject requests (in addition to workflow-specific approvers) |

> **Important (v1.10.6+):** The Admin Group is **required**. If no Admin Group ID is configured (in either portal settings or `appsettings.json`), all users are denied admin access. See [SETUP.md](SETUP.md#step-6-configure-admin-access-required) for initial configuration instructions.
>
> **Lost admin access?** If the Admin Group ID is accidentally cleared from portal settings, the `appsettings.json` / environment variable value is used as a fallback. If neither is set, you must set `AppSettings__AdminGroupId` as an environment variable (or in `appsettings.json`) and restart the application to regain access.

### Recommended Conditional Access Policy

Since the App Request Portal is used to request apps for Intune-managed devices, we recommend protecting access to the portal with a Conditional Access policy that requires:
- **Managed device** - The device accessing the portal must be enrolled in Intune
- **Compliant device** - The device must meet your organization's compliance policies

This ensures users can only request apps from trusted, compliant devices.

#### Prerequisites

Before creating the policy:
1. You must have **Entra ID Premium P1** or **P2** license (or Microsoft 365 E3/E5, etc.)
2. You need the **Conditional Access Administrator** or **Global Administrator** role
3. Have at least one compliance policy configured in Intune

#### Creating the Conditional Access Policy

1. **Navigate to Conditional Access**
   - Go to [Azure Portal](https://portal.azure.com)
   - Navigate to **Microsoft Entra ID** > **Security** > **Conditional Access**
   - Click **+ New policy**

2. **Name the Policy**
   - Enter a descriptive name: `App Request Portal - Require Compliant Device`

3. **Configure Assignments - Users**
   - Under **Users**, click **0 users and groups selected**
   - Select **Include** > **All users**
   - (Optional) Under **Exclude**, add a break-glass admin account for emergency access

4. **Configure Assignments - Target Resources**
   - Under **Target resources**, click **No target resources selected**
   - Select **Cloud apps**
   - Click **Include** > **Select apps**
   - Search for and select your App Request Portal app registrations:
     - `App Request Portal API` (or your API app registration name)
     - `App Request Portal Frontend` (or your frontend app registration name)
   - Click **Select**

5. **Configure Conditions (Optional)**
   - Under **Conditions** > **Device platforms**
   - Click **Not configured**
   - Set **Configure** to **Yes**
   - Select **Include** > **Select device platforms**
   - Check: **Windows**, **iOS**, **Android** (the platforms you manage)
   - Click **Done**

6. **Configure Access Controls - Grant**
   - Under **Grant**, click **0 controls selected**
   - Select **Grant access**
   - Check **Require device to be marked as compliant**
   - Check **Require Microsoft Entra hybrid joined device** (optional, for hybrid environments)
   - Select **Require one of the selected controls** (OR) or **Require all the selected controls** (AND) based on your requirements
   - Click **Select**

7. **Configure Session Controls (Optional)**
   - Under **Session**, you can configure:
     - **Sign-in frequency**: Require re-authentication periodically
     - **Persistent browser session**: Disable persistent sessions for extra security

8. **Enable the Policy**
   - Set **Enable policy** to **Report-only** first to test
   - Click **Create**

9. **Test and Enable**
   - Monitor the Sign-in logs for a few days in Report-only mode
   - Verify legitimate users can access the portal from compliant devices
   - Verify access is blocked from non-compliant/unmanaged devices
   - Once verified, edit the policy and change to **On**

#### Policy Summary

| Setting | Value |
|---------|-------|
| **Name** | App Request Portal - Require Compliant Device |
| **Users** | All users (exclude break-glass account) |
| **Cloud apps** | App Request Portal API, App Request Portal Frontend |
| **Conditions** | Device platforms: Windows, iOS, Android |
| **Grant** | Require device to be marked as compliant |
| **Enable policy** | Report-only (then On after testing) |

#### Troubleshooting Access Issues

If users report they cannot access the portal:

1. **Check Sign-in Logs**
   - Go to **Microsoft Entra ID** > **Sign-in logs**
   - Filter by the user and application
   - Look for **Failure** entries and check the **Conditional Access** tab
   - The tab shows which policies applied and why access was denied

2. **Common Issues**
   | Issue | Solution |
   |-------|----------|
   | Device not enrolled | User needs to enroll their device in Intune |
   | Device not compliant | User needs to resolve compliance issues (updates, encryption, etc.) |
   | Using personal device | User needs to use their work-managed device |
   | Policy excluding wrong users | Review the Exclude settings in the CA policy |

3. **Verify Device Status**
   - Go to **Microsoft Intune admin center** > **Devices**
   - Search for the user's device
   - Check **Compliance** status and any failed compliance policies

#### Alternative: Allow Browser Access with App Protection

If you need to allow browser access from unmanaged devices (less secure), you can create an alternative policy:

1. Create a second CA policy for browser access
2. Target the same apps
3. Under **Conditions** > **Client apps**, select **Browser** only
4. Under **Grant**, require **Approved client app** or **App protection policy**
5. This allows access from unmanaged devices but with some protection

> **Recommendation:** For maximum security, require compliant managed devices. The App Request Portal is designed for employees requesting apps on their managed devices, so this policy aligns with the intended use case.

### General Settings

| Setting | Description |
|---------|-------------|
| **Require manager approval by default** | When enabled, new approval workflows include manager approval as the first stage |
| **Auto-create Entra ID groups** | Automatically create a security group when an app doesn't have a target group configured |

### Version & Updates

The Settings tab displays version information and update settings:

| Setting | Description |
|---------|-------------|
| **Current Version** | Displays the installed portal version, build date, and environment |
| **Automatically check for updates** | When enabled, the portal periodically checks for new versions |
| **Show update notifications** | When enabled, displays a notification banner when updates are available |
| **Check for Updates** | Manual button to check for available updates |
| **Install Update** | One-click button to download and install updates (requires configuration) |

When an update is available, you'll see:
- Update badge with the new version number
- Link to release notes
- **Install Update** button (if auto-update is configured)

#### Enabling In-App Updates

The portal supports one-click updates directly from the Admin Dashboard. This feature downloads the latest release and deploys it via Azure's Kudu ZIP deploy API.

**Prerequisites:**
- Portal must be running in Azure App Service
- Deployment credentials must be configured

**Configuration Steps:**

1. **Get Deployment Credentials** from Azure Portal:
   - Go to your App Service → **Deployment Center** → **FTPS credentials**
   - Copy the **Username** (starts with `$`, e.g., `$app-apprequest-prod-abc123`)
   - Copy the **Password**

2. **Add App Settings** in Azure Portal:
   - Go to your App Service → **Configuration** → **Application settings**
   - Add these settings:

   | Name | Value |
   |------|-------|
   | `Deployment__PublishUser` | Your FTPS username (e.g., `$app-apprequest-prod-abc123`) |
   | `Deployment__PublishPassword` | Your FTPS password |

3. **Using the Update Feature**:
   - Go to **Admin** > **Settings** > **Version & Updates**
   - Click **Check for Updates** to see if a new version is available
   - If configured correctly, an **Install Update** button appears
   - Click to download and deploy the update automatically
   - The application will restart during the update process

> **Note**: The Install Update button only appears when:
> - The portal is running in Azure App Service (not locally)
> - Deployment credentials are properly configured
> - An update is available

#### Manual Updates

If auto-update is not configured, you can manually update using either method below:

**Method 1: Kudu ZIP Deploy (Recommended for existing installations)**

For existing deployments, use the Kudu ZIP deployment feature:

1. Go to the [releases repository](https://github.com/powerstacks-corp/App-Request-Portal-Releases/releases)
2. Download the latest `AppRequestPortal-X.X.X.zip` file (not the source code)
3. In Azure Portal, navigate to your App Service
4. Click **Advanced Tools** → **Go** (opens Kudu)
5. Click **Tools** → **Zip Push Deploy**
6. Drag and drop the downloaded ZIP file into the deployment area
7. Wait for deployment to complete (watch the logs)
8. Restart your App Service if needed
9. Database migrations will run automatically on next startup

> **Note:** This method preserves your existing configuration and database. The ZIP contains only application files.

**Method 2: Deploy to Azure Button (New installations only)**

For fresh installations on an empty resource group:

1. Go to the [releases repository](https://github.com/powerstacks-corp/App-Request-Portal-Releases)
2. Click the **Deploy to Azure** button
3. Select an **empty resource group** or create a new one
4. Configure deployment parameters

> **Important:** The Deploy to Azure button will fail on resource groups containing existing resources. Use Method 1 for existing deployments.

### License Management

The portal requires a valid PowerStacks license to operate. License status is displayed in the Admin Dashboard and affects portal functionality.

#### Viewing License Status

1. Go to **Admin** > **Settings** tab
2. The **License** section shows:
   - Current license status (Valid, Expired, Over Device Limit, etc.)
   - License type and expiration date
   - Device count vs. licensed limit
   - Last validation timestamp

#### License Validation

The portal automatically validates your license:
- On application startup
- Every 24 hours
- When you manually click **Validate License**

To force a validation check, click the **Validate License** button in the License section.

#### Updating License Key

1. Go to **Admin** > **Settings** tab
2. In the **License** section, enter your new license key
3. Click **Save License Key**
4. The portal validates the new key and displays the result

Alternatively, use the **Setup Wizard** to enter or update your license key.

#### License Warnings

Users see warning banners in the following situations:

| Condition | Banner Message |
|-----------|----------------|
| License expiring soon (≤30 days) | "License expires in X days. Please contact your IT administrator to renew." |
| Device count in grace period (up to 3% over limit) | "Device count exceeds license limit by X devices. Please contact your IT administrator to upgrade." |
| License invalid/expired | Warning message explaining the issue |

> **Note:** When device count exceeds the license limit by more than 3%, new app requests are blocked until the license is upgraded or device count is reduced.

#### Device Count

The portal tracks managed devices from Intune that have checked in within the last 30 days. Device count is updated:
- During each app sync from Intune
- When you click **Update Device Count** in the License section

### Display Settings

Configure the portal's visual appearance for all users.

| Setting | Description |
|---------|-------------|
| **Enable dark mode** | Toggle dark mode on/off for all portal users (default setting) |
| **Max featured apps on home page** | Maximum number of featured apps to display in the home page carousel (default: 8) |
| **Hero App** | Select one app to feature prominently at the top of the home page |

**Dark Mode Behavior:**

The portal supports multiple dark mode sources with the following priority:
1. **User preference** - Users can click the sun/moon icon (☀️/🌙) in the header to toggle dark mode for themselves
2. **System preference** - If the user hasn't set a preference, the portal auto-detects the operating system's dark mode setting
3. **Admin default** - Falls back to the admin-configured dark mode setting

User preferences are stored in localStorage and persist across sessions. Users can always override the admin setting for their own viewing preference.

**Dark Mode Styling:**
When dark mode is enabled, the portal uses a vignette-style design inspired by Microsoft Learn and Intune admin center:
- **Main content area**: Darkest (#1a1a1a) with subtle inset shadow for depth
- **Header/Footer**: Medium dark (#252525) with subtle borders
- **Outer edges**: Lighter dark gray (#2d2d2d)

This creates a professional look where the center content draws focus while the periphery provides visual framing.

> **Note:** Dark mode settings persist across page refreshes and login/logout cycles. The admin setting is loaded via a public API endpoint so it applies even before the user authenticates.

### App Deployment Settings

| Setting | Description |
|---------|-------------|
| **Group Name Prefix** | Prefix used when auto-creating Entra ID groups (default: `AppPortal-`). Groups are named `{prefix}{AppName}-Required`. Use this to identify portal-managed groups in your tenant. |

### Custom Domain Configuration

The Settings tab includes a **Custom Domain** section for configuring a custom domain (e.g., `apps.yourdomain.com`) for your portal.

#### Prerequisites

Before configuring a custom domain:
1. Your DNS must be configured with the appropriate CNAME or A record pointing to your Azure App Service
2. Your Azure App Service must be on the Basic tier or higher (required for custom domains with SSL)

#### Configuring via Admin Dashboard

1. Go to **Admin** > **Settings** tab
2. Scroll to the **Custom Domain** section
3. Read the prerequisites and ensure DNS is configured
4. Click **Configure Custom Domain in Azure**
5. This opens the Azure Portal with a pre-configured ARM template that:
   - Adds your custom domain to the App Service
   - Creates a free Azure-managed SSL certificate
   - Binds the certificate to your domain

#### After Configuration

Once your custom domain is configured:

1. **Update Entra ID Redirect URIs** - Add your custom domain URLs to your App Registration
2. **Update Portal URL** - In Communications > Email Notifications, update the Portal URL to use your custom domain
3. **Test Authentication** - Sign out and sign back in to verify authentication works

> **Note:** For detailed DNS configuration, certificate options, and troubleshooting, see [CUSTOM-DOMAINS.md](CUSTOM-DOMAINS.md).

## Communications

The Communications tab (formerly "Terms of Service") consolidates all notification, messaging, and company branding settings in one place.

### Company Information

Configure your organization's branding details. These fields are used in notification messages and will support further message customization in future releases.

| Setting | Description |
|---------|-------------|
| **Company Name** | Your organization's name, displayed in notifications |
| **Company Logo** | Upload a logo image (PNG/JPEG). Displayed in branded communications |
| **Support Email** | Contact email for support inquiries |
| **Support Phone** | Contact phone number for support |

### Email Notifications

Configure how the portal sends email notifications for request submissions and approvals.

| Setting | Description |
|---------|-------------|
| **Enable email notifications** | Toggle to turn email notifications on or off |
| **Send As User ID** | The Entra ID Object ID of the user or shared mailbox that will send emails. Find this in Azure Portal > Entra ID > Users > [select user] > Object ID |
| **From Address** | The email address displayed in the From field (should match the mailbox) |
| **Portal URL** | The URL of your portal, used in email links to direct users back to the portal |

**Email Events** — When email notifications are enabled, you can toggle individual events:

| Event | Description |
|-------|-------------|
| **Request Submitted** | Notify requestor when their request is submitted |
| **Approval Required** | Notify approvers when their approval is needed |
| **Request Approved** | Notify requestor when their request is approved |
| **Request Rejected** | Notify requestor when their request is rejected |
| **App Installed** | Notify requestor when their app is installed on their device |
| **App Published** | Notify admin when a WinGet app is published to Intune |

> **Note:** The app registration must have the `Mail.Send` Microsoft Graph permission with admin consent granted.

### Microsoft Teams Bot Notifications

Send personal Teams notifications to approvers and requestors via a Teams Bot. Each user receives individual Adaptive Card messages in their Teams chat. This uses Microsoft Bot Framework proactive messaging.

| Setting | Description |
|---------|-------------|
| **Enable Teams bot notifications** | Toggle to turn Teams bot notifications on or off |
| **Bot App ID** | Your API Client ID (the bot reuses the API app registration) |
| **Test** | Send a test notification to yourself to verify the bot is working |
| **Approval Required** | Notify approvers when their approval is needed |
| **Request Approved** | Notify requestor when their request is approved |
| **Request Rejected** | Notify requestor when their request is rejected |
| **App Installed** | Notify requestor when their app is installed on their device |
| **App Published** | Notify admin when a WinGet app is published to Intune |

#### Prerequisites

1. An **Azure Bot** resource registered in Azure Portal using your API app registration's Client ID (see [SETUP.md](SETUP.md#step-10-configure-microsoft-teams-bot-notifications-optional))
2. The **Microsoft Teams** channel enabled on the Azure Bot resource
3. The bot **pre-installed for users** via Teams Admin Center setup policies

> No separate `Bot__` environment variables are needed — the bot uses `AzureAd__ClientId`, `AzureAd__ClientSecret`, and `AzureAd__TenantId` directly.

#### Configuring the Portal

1. Go to **Admin** > **Communications** tab
2. Scroll to **Microsoft Teams Bot Notifications**
3. Enable **Enable Teams bot notifications**
4. Enter the **Bot App ID** (your API Client ID — the bot reuses the same app registration)
5. Click **Test** to send a test notification to yourself
6. Select which events should trigger notifications
7. Click **Save Settings**

#### How It Works

- The bot is pre-installed for users via Teams Admin Center setup policies
- When the bot is installed for a user, Teams sends a `conversationUpdate` event — the portal stores a conversation reference for that user
- To send a notification, the portal retrieves the stored conversation reference and uses Bot Framework proactive messaging
- For **pooled approvals** (group-based), the portal expands the group membership and sends individual messages to each group member
- For **sequential approvals**, only the current stage approvers are notified
- Notifications are sent as **Adaptive Cards** with request details and action buttons

#### Troubleshooting

- **Bot not sending messages**: Verify the bot is installed for the user by checking the `BotConversationReferences` table
- **Test notification fails**: Ensure the bot is installed for your user account first
- **401 errors**: Verify the Azure Bot resource's Microsoft App ID matches your `AzureAd__ClientId` and the client secret is valid
- **Some users don't receive notifications**: The Teams Admin Center setup policy may take up to 24 hours to propagate

> **Note:** No additional Microsoft Graph API permissions are required for Teams bot notifications. Bot Framework handles its own authentication.

### Approval Reminders

Automatically send reminder emails for pending approvals.

| Setting | Description |
|---------|-------------|
| **Enable approval reminders** | Toggle to enable/disable automatic reminders |
| **Reminder interval (days)** | Days before the first reminder is sent (default: 2) |
| **Max reminders** | Maximum number of reminders per request (default: 3) |

### Stale Request Escalation

Automatically escalate requests that have been pending too long.

| Setting | Description |
|---------|-------------|
| **Enable escalation** | Toggle to enable/disable automatic escalation |
| **Escalation threshold (hours)** | Hours before a request is escalated (default: 48) |
| **Recipient email(s)** | Comma-separated email addresses for escalation notifications |
| **Recipient group** | Entra ID group whose members receive escalation notifications |

### Terms of Service

The Terms of Service section remains within the Communications tab, allowing admins to create and manage TOS versions that users must accept.

## App Management

The App Management tab is your central hub for managing which Intune apps are available in the portal and how they behave.

### Syncing Apps from Intune

1. Click the **Sync Apps from Intune** button
2. The portal fetches all mobile apps from your Intune tenant
3. Apps are imported with their name, publisher, description, icon, and category
4. New apps are hidden by default - you must make them visible for users to see them

**Incremental Sync:** After the first full sync, subsequent syncs are incremental — only new and modified apps are fetched from Intune. The portal tracks the last sync date and uses `lastModifiedDateTime` filtering to minimize Graph API calls. Icons are fetched in parallel (5 concurrent requests) for faster sync times.

**Scheduled Background Sync:** A background service automatically syncs apps on a schedule:
- **Nightly full sync** at 2:00 AM UTC — catches deletions and ensures consistency
- **Hourly incremental sync** — picks up new and changed apps quickly
- The manual **Sync Apps from Intune** button is still available for on-demand syncs

**Pagination:** The portal follows `@odata.nextLink` pagination from the Graph API, so tenants with more than 999 apps are fully supported.

### App Table Columns

| Column | Description |
|--------|-------------|
| **App** | App name, icon, and publisher |
| **Type** | App platform and type from Intune (e.g., Win32, Microsoft Store, iOS Store) |
| **Category** | App category (editable via Edit modal) |
| **Cost** | Optional cost to display to users (editable) |
| **Assignment** | User or Device - determines what gets added to the AAD group |
| **Approval** | Whether this app requires approval workflow |
| **Visible** | Whether users can see this app in the portal |
| **Actions** | Edit details or configure approval workflow |

### Supported App Types

The portal supports self-service deployment for specific app types. When syncing apps from Intune, each app is classified by platform and support status:

| Platform | Supported Types | Unsupported Types |
|----------|-----------------|-------------------|
| **Windows** | Win32, Microsoft Store (New) | Windows Universal (LoB), MSI (LoB), AppX (LoB) |
| **iOS** | iOS Store, iOS VPP | iOS (LoB) |
| **Android** | Android Store, Managed Google Play, Android for Work | Android (LoB) |
| **Web** | Web Apps | - |
| **macOS** | - | All macOS app types |

**Why some types are unsupported:**
- **Line of Business (LoB) apps** require special handling during deployment that the portal cannot automate
- **macOS apps** have different deployment mechanisms that are not yet implemented

Unsupported apps appear in the admin UI with their type displayed, but the **Visible** toggle is disabled. Admins can see these apps exist but cannot make them available for self-service.

### Configuring Individual Apps

#### Visibility Toggle

- **Yes** (green): App appears in the user-facing app catalog
- **No** (red): App is hidden from users but still tracked in the system
- **Disabled** (grayed out): App type is not supported for self-service deployment

Use this to control which Intune apps are available for self-service requests. Apps that shouldn't be requested (system apps, dependencies, etc.) should remain hidden.

**Hiding Apps with Active Deployments:**

When you uncheck the "Visible" toggle on an app that has an active Intune deployment and Azure AD group, a confirmation dialog appears asking:

> Are you sure you want to hide "[App Name]"?
>
> This app has an Intune deployment (GroupName).
>
> Do you want to also delete the deployment group and assignment?
>
> • Click OK to hide the app AND delete the deployment/group
> • Click Cancel to hide the app but keep the deployment/group

This gives you two options:
- **OK**: Hides the app from the catalog AND removes the Intune assignment and deletes the Azure AD deployment group
- **Cancel**: Only hides the app from the catalog but preserves the deployment infrastructure (useful if you plan to make it visible again later)

> **Note:** Deleting the deployment and group is permanent. Users who currently have the app will lose access when group membership is removed.

**Automatic Deployment Setup:**

When you toggle an app's visibility to **Yes** for the first time, the portal automatically:

1. **Creates an Entra ID Security Group** named `{GroupNamePrefix}{AppName}-{arch}-{locale}-v{version}-Required`
   - Example: `AppPortal-Microsoft Teams-x64-en-US-v1-0-0-Required`
   - The prefix is configurable in Settings (default: `AppPortal-`)
   - Dots in version numbers are replaced with dashes (e.g., `1.0.0` becomes `v1-0-0`)

2. **Creates an Intune App Assignment**
   - The app is assigned as **Required** to the new security group
   - Assignment type (User or Device) is based on the app's Assignment setting

This automation ensures that when a user's request is approved, they simply need to be added to the group and Intune handles the deployment.

> **Note:** If group or assignment creation fails, the error is logged but the visibility toggle still succeeds. You can manually create the assignment in Intune if needed.

#### Requires Approval Toggle

- **Yes**: Requests go through the configured approval workflow before completion
- **No**: Requests are auto-approved and the user/device is immediately added to the target group
- **Disabled** (grayed out): App is not visible - approval can only be enabled for visible apps

Use "No" for low-risk apps that don't need oversight. Use "Yes" for apps that need manager or IT approval.

**Approval Requires Visibility:**

The approval toggle is automatically disabled if the app is not visible. This validation ensures that:
- You cannot enable approval for apps that users cannot see in the catalog
- You cannot hide an app that currently requires approval (you must disable approval first)

If you try to enable approval for a hidden app, you'll see the tooltip: "App must be visible to enable approval".

#### Delete from Intune

A red **Delete** button appears in the Actions column for any app with a real Intune deployment (not synced apps with a `winget-` prefix). Clicking it removes the app from both Intune and the portal entirely.

**Confirmation flow:**

1. **Step 1** — "Are you sure you want to delete *[app name]* from Intune?"
   - Click **Delete** to proceed, or **Cancel** to abort
2. **Step 2** *(only for portal-published apps with a deployment group)* — "This app has a deployment group (*group name*). Do you also want to delete the deployment group?"
   - Click **Yes** to delete the group, or **No** to keep it for reuse when republishing

**What gets deleted:**

| Action | Always | Only if "Yes" to group deletion |
|--------|--------|-------------------------------|
| Win32 app removed from Intune | Yes | — |
| Intune assignment removed | Yes | — |
| All associated requests removed | Yes | — |
| App record removed from portal | Yes | — |
| Entra ID security group deleted | — | Yes |

After deletion, the app is completely removed from the portal. To restore it, republish from the WinGet catalog.

**When to use Delete from Intune:**
- You need to republish an app with updated settings (e.g., different silent switches, updated installer)
- An app deployment is in a bad state and needs to be recreated
- You want to permanently remove an app from both Intune and the portal

### Edit App Modal

Click **Edit** on any app row to open the Edit App Modal, which provides access to all configurable app properties:

#### App Details Section

| Field | Description |
|-------|-------------|
| **Cost** | Optional cost to display on app cards (informational only, no billing). Enter a decimal value or leave empty for "Free". |
| **Category** | App category displayed on app cards. Start typing to see suggestions from existing categories in your catalog. You can enter any category name. |

#### Assignment Settings Section

| Field | Description |
|-------|-------------|
| **Assignment Type** | User (add requester to group) or Device (add requester's device to group) |
| **Target Group** | Entra ID security group for app assignment. Click "Search Groups" to browse, or use "Clear" to remove. |
| **Assignment Filter** | Optional Intune assignment filter. Select filter type (Include/Exclude) and choose a filter. |

#### Win32 Deployment Options Section

These options appear only for Win32 apps and control Intune deployment behavior:

| Field | Description |
|-------|-------------|
| **Install Context** | System (machine-wide) or User (per-user) installation |
| **Device Restart** | How to handle restart after install: Based on Return Code, Allow, Suppress, or Force |
| **End User Notification** | What users see: Show All, Show Reboot Only, or Hide All |
| **Allow Available Uninstall** | Whether users can uninstall from Company Portal |
| **Restart Grace Period** | Enable/configure grace period before forced restart |

Click **Save** to apply changes or **Cancel** to discard.

#### App Icon

Each app displays an icon in the catalog. Icons are typically synced from Intune, but some apps may be missing icons. The Edit App modal provides two ways to set an icon:

- **Upload Icon** — Upload a PNG or JPEG image file from your computer
- **Browse Library** — Opens the Icon Library picker, which shows all unique icons already used by other apps in the portal. Search by app name or publisher, then click an icon to apply it instantly. This is useful when multiple apps share the same publisher icon or when you want to reuse an existing icon without re-uploading.

#### Assignment Type

Determines what gets added to the Entra ID group when a request is approved:

- **User**: The requesting user is added to the group (most common)
- **Device**: The user's device is added to the group (useful for device-targeted deployments)

For Device assignment:
- The portal **auto-detects** the user's current device based on browser/OS information
- Devices matching the detected OS are pre-selected in the dropdown
- The user can override the selection if they want to install on a different device
- If a primary device is set in Intune and matches the current OS, it takes priority
- The selected device is added to the target AAD group
- Intune deploys the app to that specific device

**Device Detection Logic:**
1. Browser detects the operating system (Windows, macOS, iOS, Android)
2. Portal matches against the user's registered Intune devices
3. Matching devices are marked as "(Detected)" in the dropdown
4. A helpful message confirms the auto-detection: "This device was automatically detected based on your current browser."

### Configuring Approval Workflows

Click the **Workflow** button on any app to open the Approval Workflow Editor.

#### No Approval Required

For apps with "Approval" set to "No", requests are auto-approved immediately. No workflow configuration is needed.

#### Simple Approval

For apps that just need someone from the Approver Group to sign off:
1. Set "Approval" to "Yes"
2. Leave the workflow with no stages
3. Any member of the configured Approver Group can approve

#### Manager + Additional Approvers

1. Enable **Require Manager Approval**
2. Choose workflow type:
   - **Linear**: Specific users approve in sequence
   - **Pooled**: Any member of specified groups can approve
3. Add approval stages as needed

#### Conditional Stages (v1.8.2)

Each approval stage can be made conditional, so it only applies when specific criteria are met:

1. Open the Approval Workflow Editor for an app
2. Add a stage and assign an approver or group
3. Toggle "Make this stage conditional"
4. Add conditions using the condition builder:
   - Select a condition type (Department, Cost Center, Job Title, Location, Platform, Request Count)
   - Choose an operator (Equals, Not Equals, Contains, etc.)
   - Enter the comparison value
5. Add multiple conditions with AND/OR logic
6. A human-readable summary of the conditions is shown on the stage card

See [APPROVAL-WORKFLOWS.md](APPROVAL-WORKFLOWS.md#conditional-workflows-v160-ui-in-v182) for detailed condition types and operators.

See [APPROVAL-WORKFLOWS.md](APPROVAL-WORKFLOWS.md) for detailed workflow configuration options.

### Version History and Rollback

The portal tracks version history for apps published from the Winget Catalog, enabling you to rollback to previous versions if a new version introduces issues.

#### Viewing Version History

1. Go to **Admin** > **App Management** tab
2. Find the app in the app table
3. Click the **History** button in the Actions column
4. The Version History modal displays all recorded versions

#### Understanding Version Status

Each version in the history shows a status badge:

| Status | Description |
|--------|-------------|
| **Current** (blue) | The version currently deployed in Intune |
| **Archived** (gray) | Previous version that was deployed, available for rollback |
| **Failed** (red) | Deployment or publish attempt that failed |

#### Version Information Displayed

For each version, you'll see:

- **Version number** - The semantic version (e.g., "1.2.3")
- **Recorded date** - When this version was detected/published
- **Updated by** - User who published this version (if available)
- **Installer size** - Size of the installer package
- **Release notes** - Change summary for this version
- **Package ID** - WinGet package identifier (e.g., "7zip.7zip")

#### Rolling Back to a Previous Version

If a new app version causes issues, you can rollback to a previous version:

1. Open the **Version History** modal for the app
2. Find the version you want to rollback to (must have "Archived" status)
3. Click the **Rollback** button on that version
4. Click **Confirm Rollback** to proceed
5. The portal will:
   - Re-publish the selected version to Intune
   - Update the app assignment with the older version
   - Mark the rolled-back version as "Current"
   - Archive the previously current version

> **Important:** Rollback is only available for versions marked as "Archived". Failed deployments and the current version cannot be rolled back to.

#### Version History Limitations

- **Winget apps only**: Version history is only tracked for apps published from the Winget Catalog
- **Manual Intune apps**: Apps added directly to Intune (not via portal) do not have version history
- **Retention policy**: By default, the portal keeps all versions indefinitely. Admins can configure retention limits in Settings

#### Troubleshooting Rollback Issues

**Rollback button is grayed out:**
- Only "Archived" versions can be rolled back to
- Ensure the version's installer is still available

**Rollback fails:**
- Check Graph API permissions (`DeviceManagementApps.ReadWrite.All`)
- Verify the installer URL is still accessible
- Review API logs for detailed error messages

**Version history is empty:**
- Version history only tracks apps published through the portal
- If you synced an existing Intune app, its pre-sync versions are not tracked
- Only new publishes/updates will appear in history

## Pending Approvals

The Pending Approvals tab shows all requests waiting for your approval.

### Viewing Requests

Each request shows:
- **Requestor**: Name and email of the person requesting the app
- **App**: The app being requested
- **Requested**: When the request was submitted
- **Stage**: Current approval stage number
- **Justification**: Reason provided by the requestor (if any)

### Approving Requests

1. Review the request details
2. Click **Approve** to advance the request
3. The request moves to the next approval stage (or completes if this was the final stage)

### Rejecting Requests

1. Click **Reject**
2. Enter a reason for rejection (required)
3. The requestor is notified of the rejection with your reason

## User Experience

### Browsing Apps

The portal provides a Microsoft Store-style browsing experience:

**Home Page:**
- Hero section featuring a prominently displayed app
- Featured apps carousel with navigation controls
- Category sections showing apps grouped by category
- Quick links to Browse Apps and My Requests
- Platform badges (Windows, iOS, Android, macOS, Web) on app cards
- "New" badges on apps added within the last 14 days

**Browse Apps Page:**
- Search apps by name, publisher, or description
- Filter apps by category using the dropdown
- Filter apps by platform (Windows, iOS, Android, etc.)
- Featured apps section at the top
- Apps organized by category with platform badges

**Visual Indicators:**
- **Platform badges** - Show the app's target platform with icons (🪟 Windows, 🍎 iOS, 🤖 Android, 🍏 macOS, 🌐 Web)
- **"New" badge** - Green badge on apps added within the last 14 days
- **"Featured" badge** - Gold badge on featured apps
- **Price indicator** - Shows cost or "Free" label

**App Detail Page:**
When users click on any app card, they see a detailed view including:
- Large hero banner with app icon and blurred background
- App name, publisher, and category badges
- "Featured" badge if applicable
- **Get** button to request the app
- Price or "Free" indicator
- Full description
- App information (Publisher, Version, Category, Platform, Approval status)

### How Users Request Apps

Users can request apps in two ways:

**Quick Request (Get button):**
1. Click the **Get** button on any app card (Home, Browse Apps, or App Detail page)
2. The request modal opens directly
3. If Device assignment, select the target device
4. Enter optional justification
5. Click **Submit Request**

**From App Detail Page:**
1. Click on an app card to open the App Detail page
2. Review the app description and information
3. Click the **Get** button
4. Complete the request form and submit

### Request Status Flow

| Status | Description |
|--------|-------------|
| **Pending** | Request is awaiting approval |
| **Approved** | All approvals complete, processing assignment |
| **Rejected** | Request was rejected by an approver |
| **Processing** | System is adding user/device to AAD group |
| **Completed** | User/device successfully added to group |
| **Failed** | Error occurred during processing |

### My Requests

Users can view their request history by clicking **My Requests** in the navigation. This shows all requests they've submitted with current status.

### Request New App

Users can request apps that aren't in the catalog by clicking the **+ Request New App** button on the Browse Apps page.

#### How It Works

1. User fills out the form with app name, publisher, description, and optional download URL
2. The portal sends an email notification to **all members of the Admin Group**
3. The email includes:
   - Requestor name and email
   - App name and publisher
   - Business justification provided by the user
   - Download URL (if provided)
   - Suggestions for how to add the app (Winget catalog or manual upload)
4. The request is logged in the audit trail

#### Admin Actions

When you receive a new app request email:

1. **Evaluate the request**: Is this app appropriate for your organization?
2. **Find the app**:
   - Check the Winget Catalog in Admin Dashboard for easy publishing
   - Search for the app in Intune if it's already available
   - Download from the vendor if needed
3. **Add to portal**:
   - Use Winget Catalog to publish directly to Intune, or
   - Manually add the app to Intune and sync
4. **Configure visibility**: Make the app visible in the portal
5. **Notify the user**: Reply to the email or notify the user directly

#### Configuration

The Request New App feature uses your existing email notification settings:

- **Admin Group**: Members receive the notification emails
- **Email Settings**: Uses the same `Mail.Send` configuration as other notifications

No additional configuration is required. If email notifications are disabled, the feature will return an error to the user.

## Reports & Analytics

The Admin Dashboard includes comprehensive reporting capabilities to help you understand app request patterns and deployment status.

### Accessing Reports

1. Navigate to **Admin** in the navigation menu
2. The dashboard displays summary tiles at the top
3. Use the tabs to navigate between report views:
   - **Summary** - Overview statistics with install status
   - **Trends** - Visual charts showing request patterns over time
   - **By Person** - Detailed request history by user
   - **Install Status** - Deployment status for approved requests

### Summary Dashboard

The Summary view shows key metrics as clickable tiles:

| Tile | Description |
|------|-------------|
| **Total Requests** | Total number of app requests in the system |
| **Pending** | Requests awaiting approval |
| **Approved** | Requests that have been approved |
| **Rejected** | Requests that were rejected |
| **Pending Install** | Approved requests waiting for Intune deployment |
| **Installing** | Apps currently being installed on devices |
| **Installed** | Successfully deployed apps |
| **Install Failed** | Apps that failed to install |

The install status tiles (Pending Install, Installing, Installed, Install Failed) are color-coded for quick identification:
- **Pending Install** (blue) - Deployment pending
- **Installing** (orange) - Installation in progress
- **Installed** (green) - Successfully deployed
- **Install Failed** (red) - Deployment failed

### Trends Tab

The Trends tab provides visual analytics to help identify patterns and popular applications.

#### Request Trends Chart

The main trends chart shows:
- **Requested** (blue line) - Number of new requests per day
- **Completed** (green line) - Number of completed requests per day
- Visual area fills under each line for easy comparison

Use the time range dropdown to view:
- Last 7 days
- Last 14 days
- Last 30 days
- Last 90 days

#### Top Requested Apps

A horizontal bar chart showing the most frequently requested applications, helping you identify:
- Popular apps that might benefit from auto-approval
- Apps that may need better visibility or promotion
- Patterns in user requests

#### Status Distribution

A breakdown showing the distribution of request statuses:
- Total count and percentage for each status
- Visual progress bars for comparison
- Separate sections for request status and install status

### Install Status Tracking

The portal automatically tracks the deployment status of approved requests for Intune apps.

#### How It Works

1. **Initial Status**: When a request is approved for an Intune-managed app, the install status is set to "Pending Install"
2. **Background Polling**: A background service checks Intune for deployment status every 15 minutes
3. **Status Updates**: The portal updates the install status based on Intune's reported deployment state
4. **Final Status**: Once installed (or failed), the status stops being polled

#### Install Status Values

| Status | Description |
|--------|-------------|
| **Not Applicable** | App is not tracked for install status (e.g., Winget apps) |
| **Pending Install** | Request approved, waiting for Intune to begin deployment |
| **Installing** | Intune is actively installing the app on the device |
| **Installed** | App successfully installed and detected on the device |
| **Install Failed** | Installation failed - check the error message for details |
| **Uninstalled** | App was installed but has since been removed |

#### Viewing Install Status

**Admin Dashboard:**
1. Go to **Admin** > Reports section
2. View install status counts in the summary tiles
3. Click on any status tile to filter by that status
4. Use the **Install Status** tab for detailed view

**Install Status Tab:**
The dedicated Install Status tab shows:
- Summary counts for each install state
- List of all requests with their current install status
- Last checked timestamp for each request
- Error messages for failed installations

#### Troubleshooting Install Status

**Status stuck on "Pending Install":**
- Verify the device is online and connected to Intune
- Check that the user/device is correctly added to the target AAD group
- Review Intune device sync status in the Intune admin center

**Status shows "Install Failed":**
- Check the error message in the Install Status column
- Common issues:
  - Disk space insufficient
  - App dependencies not met
  - User cancelled the installation
  - Device compliance issues blocking deployment

**Status not updating:**
- The polling service runs every 15 minutes
- Check API logs for any errors in the InstallStatusPollingService
- Verify the app registration has `DeviceManagementApps.Read.All` permission

### By Person Report

The **By Person** tab allows you to search for a specific user and view all their app requests:

1. Enter a user's name or email in the search box
2. View their complete request history
3. See status, install status, and timestamps for each request
4. Use the **Retry** button on failed requests to re-attempt group membership

### Audit Trail

The **Audit Trail** tab provides a comprehensive log of all portal activity for compliance and security monitoring.

#### Accessing the Audit Trail

1. Go to **Admin** > **Reports** section
2. Click the **Audit Trail** button in the report navigation
3. Use filters to narrow down results

#### Available Filters

| Filter | Description |
|--------|-------------|
| **Search** | Free-text search across user email, action, entity, and details |
| **Action Type** | Filter by specific action (e.g., Request.Submitted, App.Suggested) |
| **Entity Type** | Filter by entity type (e.g., Request, App, Settings) |
| **Start Date** | Show events from this date onwards |
| **End Date** | Show events up to this date |

#### Audit Log Information

Each audit entry includes:

| Field | Description |
|-------|-------------|
| **Timestamp** | When the action occurred |
| **User** | Email address of the user who performed the action |
| **Action** | The type of action performed |
| **Entity Type** | The type of object affected |
| **Entity ID** | Identifier of the affected object |
| **Details** | Additional context (varies by action type) |
| **IP Address** | The IP address of the user |

#### Common Actions Logged

| Action | Description |
|--------|-------------|
| `Request.Submitted` | User submitted an app request |
| `Request.Approved` | Approver approved a request |
| `Request.Rejected` | Approver rejected a request |
| `Request.Completed` | Request was fulfilled (user added to group) |
| `App.Suggested` | User submitted a new app request via Request New App form |
| `Settings.Updated` | Admin changed portal settings |
| `Apps.Synced` | Admin synced apps from Intune |

#### Exporting Audit Logs

1. Apply any desired filters
2. Click the **Export CSV** button
3. A CSV file downloads with all matching audit entries
4. Use this for compliance reporting or external analysis

#### Audit Log Retention

Audit logs are stored indefinitely in the SQL database. There is no automatic purge. For organizations with high volume, consider:

- Periodic export to long-term storage
- Database scaling if performance is affected
- Custom retention policies via direct database management

## Best Practices

### App Visibility Strategy

1. **Sync all apps** to get your full Intune catalog
2. **Keep system apps hidden** (dependencies, frameworks, required apps)
3. **Make user-requestable apps visible** (productivity tools, optional software)
4. **Use categories** from Intune to help users find apps

### Approval Configuration Strategy

| App Type | Recommended Setting |
|----------|---------------------|
| Free productivity tools | No approval required |
| Licensed software | Manager approval |
| Admin/privileged tools | Multi-stage with IT Security |
| Developer tools | Manager + IT approval |

### Group Management

With automatic group and assignment creation, the portal handles most group management for you:

1. **Automatic Setup**: When you make an app visible, the portal creates a security group and Intune assignment automatically
2. **Consistent Naming**: Groups follow the pattern `{GroupNamePrefix}{AppName}-{arch}-{locale}-v{version}-Required` (e.g., `AppPortal-Microsoft Teams-x64-en-US-v1-0-0-Required`). Dots in version numbers are replaced with dashes.
3. **Custom Prefix**: Configure the Group Name Prefix in Settings to match your organization's naming conventions
4. **Manual Override**: You can still manually set a Target Group on an app if you prefer to use an existing group

> **Tip:** To use existing groups instead of auto-created ones, set the Target Group before making the app visible.

### Email Notifications

1. Create a dedicated shared mailbox for notifications
2. Grant the app `Mail.Send` permission on that mailbox
3. Use a recognizable From address like `apprequest@company.com`
4. Include your portal URL so users can click through to view status

## Winget Catalog & Cloud Packaging

The **Winget Catalog** tab in the Admin Dashboard allows you to browse over 9,000 apps from Microsoft's official WinGet repository ([microsoft/winget-pkgs](https://github.com/microsoft/winget-pkgs)) and publish them directly to Intune as Win32 apps. Package manifests are fetched directly from GitHub (no third-party services), ensuring zero supply chain attack risk.

### Browsing the Winget Catalog

1. Navigate to **Admin** > **Winget Catalog** tab
2. Popular packages are displayed 24 per page with pagination controls
3. Use the search box to find specific apps (search uses "Load More" infinite scroll)
4. Each package shows: name, publisher, version, and description
5. Icons are loaded automatically where available

### Publishing to Intune

1. Find the app you want to publish
2. **Select architecture and locale** (if available):
   - Each package card displays available architectures (e.g., x64, x86, arm64) and locales (e.g., en-US, de-DE, fr-FR)
   - If multiple options are available, they appear as dropdown selectors
   - If only one option is available, it appears as a label showing what will be published
   - Default selections: x64 for architecture, en-US for locale
3. Click **Publish to Intune** button on the package card
4. A packaging job is created with your selected architecture and locale
5. Monitor job status in the **Packaging Jobs** section below the catalog

**Architecture, Locale, and Version Tracking:**

When apps are published from the WinGet catalog:
- The selected architecture, locale, and version are stored in the packaging job
- When the app is synced from Intune, it automatically inherits these values
- Deployment groups are named to include all three: `AppPortal-AppName-x64-en-US-v25-10-186-0-Required`
- Dots in version numbers are replaced with dashes (e.g., `25.10.186.0` becomes `v25-10-186-0`)
- This helps identify which variant and version of multi-architecture apps is deployed, and prevents duplicate deployments of the same version

### Packaging Jobs

The Packaging Jobs section shows all queued and completed packaging operations:

| Status | Description |
|--------|-------------|
| **Pending** | Job is queued, waiting for processing |
| **Downloading** | Downloading the installer from WinGet manifest URL |
| **Packaging** | Wrapping in PSADT v4 and creating .intunewin package |
| **Uploading** | Uploading package to Intune via Graph API |
| **Creating** | Creating Win32LobApp in Intune |
| **Completed** | App successfully created in Intune |
| **Failed** | Error occurred - click Retry to requeue |

### Packaging Architecture

The packaging process runs in-process on the App Service — no separate containers, agents, or functions needed:

1. **Queue Message**: When you click "Publish to Intune", a job is added to Azure Storage Queue
2. **Background Service**: The in-process `PackagingQueueService` picks up the job
3. **Download**: Installer is downloaded from the URL in the WinGet manifest
4. **Wrapping** (PSADT mode only): Installer is wrapped in PSAppDeployToolkit v4 for standardized deployment
5. **Package Creation**: `.intunewin` package is created using the cross-platform SvRooij.ContentPrep library
6. **Intune Upload**: The package is uploaded and a Win32LobApp is created via Graph API

### Packaging Methods: Raw vs PSADT

When publishing an app from the Winget Catalog, you can choose between two packaging methods using the dropdown on each package card:

| Method | Description |
|--------|-------------|
| **Raw** (default) | Packages the installer directly. The installer executable is the entry point — Intune runs it with silent switches. Simpler, smaller package size. |
| **PSADT** | Wraps the installer in [PSAppDeployToolkit v4](https://psappdeploytoolkit.com/). Provides standardized logging, pre/post-install hooks, and consistent exit codes across all installer types. Larger package size due to framework files. |

**When to use PSADT:**
- You need standardized deployment logging across all apps
- You want consistent install/uninstall behavior regardless of installer type
- You need the pre-installation and post-installation hooks (e.g., closing running applications before install)

**When to use Raw:**
- You want the simplest, most direct installation
- Package size matters (PSADT adds ~5 MB of framework files)
- The app's native installer already handles silent installation well

#### Raw Packaging Details

In Raw mode, the installer is packaged directly into the `.intunewin` file. The install and uninstall commands sent to Intune depend on the installer type:

**Install commands by file type:**

| File Type | Install Command | Uninstall Command |
|-----------|----------------|-------------------|
| `.msi` | `msiexec /i "installer.msi" {silent switch}` | `msiexec /x "installer.msi" {silent switch}` |
| `.exe` | `"installer.exe" {silent switch}` | Registry lookup (see below) |
| `.msix` / `.appx` | `powershell.exe Add-AppxPackage -Path 'installer.msix'` | Registry lookup (see below) |

**Registry-based uninstall (for non-MSI):** For EXE and other non-MSI installers, the uninstall command searches the Windows registry (`HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall` and the `Wow6432Node` equivalent) for an entry matching the app's display name. It uses the `QuietUninstallString` if available, otherwise appends `/S /silent /quiet` to the `UninstallString`.

#### PSADT Packaging Details

In PSADT mode, the installer is placed inside a PSAppDeployToolkit v4 folder structure. The PSADT framework provides a standardized wrapper around the installer with logging, error handling, and consistent exit codes.

**Package structure created:**

```
Package/
+-- Invoke-AppDeployToolkit.exe       <-- Entry point (what Intune runs)
+-- Deploy-Application.ps1            <-- Generated install/uninstall logic
+-- AppDeployToolkit/                 <-- Framework files (from PSADT v4 template)
|   +-- AppDeployToolkitMain.ps1
|   +-- AppDeployToolkitHelpers.ps1
|   +-- ...
+-- Files/
    +-- installer.exe (or .msi)       <-- Your actual installer
```

**Intune commands (all PSADT packages use the same commands):**
- **Install:** `Invoke-AppDeployToolkit.exe -DeploymentType Install -DeployMode Silent`
- **Uninstall:** `Invoke-AppDeployToolkit.exe -DeploymentType Uninstall -DeployMode Silent`

**Generated Deploy-Application.ps1 script variables:**

| Variable | Value |
|----------|-------|
| `$appVendor` | Publisher name from the Winget manifest |
| `$appName` | Package name from the Winget manifest |
| `$appVersion` | (empty — version tracked via Intune metadata) |
| `$appLang` | `EN` |
| `$appRevision` | `01` |
| `$appScriptVersion` | `1.0.0` |
| `$appScriptAuthor` | `AppRequestPortal` |

**Install section — what runs during installation:**

For **MSI** installers:
```powershell
Execute-MSI -Action Install -Path "$dirFiles\installer.msi"
```

PSADT automatically adds `/qn /norestart` when running in Silent deploy mode. If a custom silent switch was provided that includes MSI properties (e.g., `TRANSFORMS=...`, `PROPERTY=VALUE`), those are passed via `-AddParameters`.

For **EXE** installers:
```powershell
Execute-Process -Path "$dirFiles\installer.exe" -Parameters '{silent switch}' -WaitForMsiExec
```

The `-WaitForMsiExec` parameter ensures PSADT waits if another MSI installation is already in progress. EXE installers always get explicit silent switches since PSADT doesn't know the correct flags for each installer.

**Uninstall section — what runs during uninstallation:**

For **MSI** installers:
```powershell
Execute-MSI -Action Uninstall -Path "$dirFiles\installer.msi"
```

For **non-MSI** installers, the script searches the registry for the app's uninstall string:
```powershell
$regPaths = @(
    'HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*',
    'HKLM:\Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*'
)
$app = Get-ItemProperty $regPaths |
    Where-Object { $_.DisplayName -like '*AppName*' } |
    Select-Object -First 1

if ($app.QuietUninstallString) {
    Execute-Process -Path 'cmd.exe' -Parameters "/c $($app.QuietUninstallString)"
} elseif ($app.UninstallString) {
    Execute-Process -Path 'cmd.exe' -Parameters "/c $($app.UninstallString) /S /silent /quiet"
}
```

**Repair section:** Runs the same commands as the install section. PSADT supports `Invoke-AppDeployToolkit.exe -DeploymentType Repair -DeployMode Silent` for reinstallation.

**Error handling:** If the PSADT script encounters a fatal error, it exits with an error code in the 60000 range and logs the error details. PSADT's built-in logging writes to `C:\Windows\Logs\Software\` on the target device.

The following PSADT error codes are mapped as "Failed" return codes in Intune:

| Exit Code | Description |
|-----------|-------------|
| `60001` | General PSADT error (catch block in Deploy-Application.ps1) |
| `60002` | Error during pre-installation phase |
| `60003` | Error during installation phase |
| `60008` | Generic non-zero exit from the wrapped installer |

These are registered in the Intune Win32 app return code mappings so that PSADT failures are properly reported in Intune and reflected in the portal's install status tracking.

**PSADT fallback:** If PSADT wrapping fails for any reason (e.g., template download failure, extraction error), the system automatically falls back to Raw packaging. The job will still complete successfully — just without the PSADT wrapper.

**PSADT template caching:** The PSADT v4 template is downloaded once from the [official GitHub releases](https://github.com/PSAppDeployToolkit/PSAppDeployToolkit/releases) and cached in Azure Blob Storage (`psadt-template/PSAppDeployToolkit_Template_v4.zip`). Subsequent packaging jobs reuse the cached template.

#### Silent Switches

Silent switches tell the installer to run without displaying any UI. The portal resolves the silent switch in this order:

1. **Winget manifest value** — if the manifest specifies `InstallerSwitches.Silent`, that value is used
2. **Installer type default** — based on the `InstallerType` field in the Winget manifest
3. **File extension fallback** — based on the installer's file extension
4. **Last resort** — `/S /silent`

**Default silent switches by installer type:**

| Installer Type | Silent Switch | Used By |
|---------------|--------------|---------|
| **MSI** / **WiX** | `/qn /norestart` | Windows Installer packages |
| **InnoSetup** | `/VERYSILENT /SUPPRESSMSGBOXES /NORESTART /SP-` | Inno Setup installers |
| **Nullsoft** / **NSIS** | `/S` | NSIS installers |
| **Burn** | `/quiet /norestart` | WiX Burn bootstrapper bundles |
| **EXE** (generic) | `/S /silent` | Generic executables |

These defaults are applied in both Raw and PSADT modes.

#### Detection Scripts

Every app published to Intune includes an auto-generated PowerShell detection script. Intune runs this script on target devices to determine whether the app is already installed.

**Detection strategy (in order of preference):**

1. **ProductCode detection (most reliable)** — If the Winget manifest includes `AppsAndFeaturesEntries` with a `ProductCode` (MSI GUID), the script checks for that exact GUID in the registry uninstall paths. This is the most precise detection method.

2. **DisplayName detection (fallback)** — If no ProductCode is available, the script searches the registry for an entry where `DisplayName` matches the app name (using wildcard matching). Publisher name is also checked if available in the manifest.

**Version checking:** When a version number is available from the Winget manifest (via `AppsAndFeaturesEntries.DisplayVersion` or the package version), the detection script compares the installed version against the required minimum version. Apps with older versions are reported as "Not Installed" so Intune will upgrade them.

**Registry paths checked:**
- `HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*`
- `HKLM:\Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*`

**Script execution context:** Detection scripts run as 64-bit PowerShell (`runAs32Bit: false`) without signature enforcement (`enforceSignatureCheck: false`). The script exits with code `0` if the app is detected, or `1` if not installed.

#### Intune Win32 App Configuration

When the app is created in Intune via the Graph API, the following settings are applied:

| Setting | Value |
|---------|-------|
| **Applicable architectures** | x64, x86 |
| **Minimum OS** | Windows 10 1903 |
| **Install context** | System (runs as SYSTEM account) |
| **Device restart behavior** | Based on return code |

**Return codes configured:**

| Code | Type | Meaning |
|------|------|---------|
| `0` | Success | Installation completed successfully |
| `1707` | Success | Installation completed successfully (MSI) |
| `3010` | Soft reboot | Installation succeeded, reboot recommended |
| `1641` | Hard reboot | Installation succeeded, reboot initiated |
| `1618` | Retry | Another installation is in progress, retry later |

**App icon:** If available, the app icon is fetched automatically (via Google S2 Favicon API or GitHub publisher avatar) and included as a PNG or JPEG in the Intune app metadata.

### Setting Up Packaging

Packaging works out of the box with the default deployment. No additional setup is required beyond:

1. **Azure Storage Account** (deployed by default via Bicep template)
2. **Graph API permissions** for Intune app management (see Prerequisites)
3. **Verify connectivity**:
   - Go to Admin > Winget Catalog
   - Try publishing a test app (e.g., "7-Zip")
   - Check Packaging Jobs for status

The PSADT v4 template is automatically downloaded from the [official GitHub repository](https://github.com/PSAppDeployToolkit/PSAppDeployToolkit) on first use and cached in Azure Blob Storage.

### App Updates

The **App Updates** section on the Winget Catalog tab shows all apps that have been published from WinGet and are being tracked for version updates.

**Tracked Apps Table:**

| Column | Description |
|--------|-------------|
| **App** | App name and publisher |
| **WinGet Package ID** | The WinGet identifier (e.g., `Microsoft.VisualStudioCode`) |
| **Published Version** | Version currently in your portal/Intune |
| **Latest Version** | Latest version available in the WinGet repository |
| **Status** | "Update Available" (green) or "Up to Date" |
| **Last Checked** | When the version was last compared |
| **Actions** | Apply or Dismiss update (only shown when update is available) |

**Automatic Tracking:**
- Apps published from the WinGet catalog are automatically tracked for updates
- The `SourceWingetPackageId` is backfilled automatically by the background service (no manual Intune sync required)
- Update checks run on a configurable schedule (default: every hour when enabled)

**Actions:**
- **Check for Updates**: Manually trigger an update check for all tracked apps
- **Apply**: Updates the portal record with the latest version information
- **Dismiss**: Hides the update notification for that app

### Winget Integration Settings

In **Admin** > **Settings** > **Winget Integration**:

| Setting | Description |
|---------|-------------|
| **WinGet Repository URL** | GitHub repository URL for WinGet packages. Default: `https://github.com/microsoft/winget-pkgs` (Microsoft's official repository). Format: `https://github.com/owner/repo` or `owner/repo`. Organizations with custom/private WinGet repositories can point to their internal GitLab/GitHub mirror. **Warning**: Changing this will clear the entire package cache. |
| **GitHub Personal Access Token** | Recommended. Required for the "Show More Results" live search feature (GitHub's Code Search API requires authentication). Also increases API rate limits from 60/hour to 5,000/hour for faster cache syncs. Create a classic token at [https://github.com/settings/tokens](https://github.com/settings/tokens) with `public_repo` scope. |

#### Why Use a GitHub Token?

**Without token** (unauthenticated):
- 60 API requests per hour
- Initial cache sync takes 2-3 hours
- May hit rate limits during heavy use
- **"Show More Results" live search will not work** — GitHub's Code Search API requires authentication. Only cached results and exact package ID lookups (e.g., `Google.Chrome`) will return results.

**With token** (authenticated):
- 5,000 API requests per hour
- Initial cache sync takes 30-60 minutes
- Reliable operation even with many users
- **Full live search** — "Show More Results" queries the entire WinGet repository in real time via GitHub Code Search, finding packages that may not yet be in the local cache

**Creating a GitHub Personal Access Token:**

1. Go to [GitHub Settings → Personal Access Tokens → Tokens (classic)](https://github.com/settings/tokens)
2. Click "Generate new token" → "Generate new token (classic)"
3. Give it a descriptive name: "App Request Portal WinGet Integration"
4. Select scope: **public_repo** (Access public repositories)
5. Click "Generate token"
6. Copy the token (starts with `ghp_...`)
7. Paste into Admin Settings → WinGet Integration → GitHub Personal Access Token
8. Click Save Settings

**Note**: Tokens are stored securely in Azure Key Vault and never exposed in logs or UI.

### Local Development Testing

For developers testing the packaging feature locally:

1. **Prerequisites**:
   - Azure Storage account configured (uses the same storage queue and blob container as production)
   - Graph API credentials configured for Intune access

2. **Configure the API** (`src/AppRequestPortal.API/appsettings.json`):
   - Ensure `AzureStorage:ConnectionString` points to your Azure Storage account

3. **Run the services**:
   ```bash
   # API with background packaging service (port 5000)
   cd src/AppRequestPortal.API && dotnet run

   # Frontend (port 3000)
   cd src/AppRequestPortal.Web && npm start
   ```

4. **Test**: Go to Admin > Winget Catalog, search for an app, click "Publish to Intune". The in-process background service will pick up the job and process it.

### Troubleshooting Packaging

**Jobs stuck in Pending:**
- Verify the App Service is running and the background service started (check Application Insights logs for `PackagingQueueService`)
- Ensure the `AzureStorage:ConnectionString` is configured correctly
- Check that the storage queue `packaging-jobs` exists

**Jobs failing during download:**
- The service downloads directly from the installer URL in the WinGet manifest
- Check if the package exists in the Microsoft winget-pkgs repository
- Review logs for HTTP errors or GitHub API rate limits

**Jobs failing during PSADT wrapping:**
- PSADT wrapping has graceful fallback — if it fails, the raw installer is packaged instead
- Check logs for `PsadtService` entries
- First-time use requires downloading the PSADT template from GitHub; ensure outbound HTTPS is allowed

**Jobs failing during upload:**
- Verify the API has Graph API permissions for Intune (`DeviceManagementApps.ReadWrite.All`)
- Review API logs for Graph API errors (detailed error messages are logged)

## Database Maintenance

The App Request Portal uses Azure SQL Database Basic tier, which includes comprehensive automatic maintenance features. **No manual database maintenance is required.**

### Why No Manual Maintenance is Needed

Azure SQL Database handles all maintenance tasks automatically, including:

| Feature | Description |
|---------|-------------|
| **Automatic Index Tuning** | Azure monitors query patterns and automatically creates, drops, or rebuilds indexes to optimize performance |
| **Automatic Plan Correction** | Detects and fixes query plan regression issues automatically |
| **Automatic Backups** | Point-in-Time Restore (PITR) with 7-day retention (Basic tier), up to 35 days on higher tiers |
| **Geo-Redundant Storage** | Backups are stored redundantly across Azure regions for disaster recovery |
| **Automatic Updates** | Database engine patches and security updates applied automatically with no downtime |
| **Automatic Statistics** | Query statistics are automatically updated to ensure optimal query plans |

### What About Maintenance Scripts?

You may be familiar with on-premises SQL Server maintenance solutions like [Ola Hallengren's Maintenance Solution](https://ola.hallengren.com/) that schedule index rebuilds, integrity checks, and backup jobs. **These are not needed for Azure SQL Database** because:

1. **Index Maintenance**: Azure's automatic tuning handles index optimization. For a Basic tier database under 2GB, index fragmentation has minimal impact on performance.

2. **Integrity Checks (DBCC CHECKDB)**: Azure runs these automatically. You cannot schedule them yourself on Azure SQL Database.

3. **Backup Jobs**: Azure manages all backups automatically. You cannot create your own backup jobs—instead, you use Azure's built-in Point-in-Time Restore feature.

4. **Statistics Updates**: Azure automatically updates statistics as needed. Manual `UPDATE STATISTICS` commands are rarely necessary.

### Backup and Recovery

Azure SQL Database provides built-in backup and recovery capabilities:

| Tier | PITR Retention | Backup Storage |
|------|---------------|----------------|
| **Basic** | 7 days | Geo-redundant (GRS) |
| **Standard** | 35 days | Geo-redundant (GRS) |
| **Premium** | 35 days | Geo-redundant (GRS) |

**To restore your database:**

1. Go to **Azure Portal** > **SQL databases** > your database
2. Click **Restore** in the toolbar
3. Select a restore point (any time within retention period)
4. Azure creates a new database with data as of that point in time

> **Note:** Restoring creates a *new* database—it does not overwrite the existing one. You would rename databases after verifying the restore if needed.

### When to Consider Scaling Up

The Basic tier (2GB, 5 DTUs) is suitable for the App Request Portal's typical workload. Consider scaling up if you observe:

- Consistent high DTU usage (>80%) in Azure metrics
- Slow query response times
- Timeouts during peak usage

To scale up:
1. Go to **Azure Portal** > **SQL databases** > your database
2. Click **Compute + storage**
3. Select a higher tier (Standard, Premium) or increase DTUs
4. Changes take effect within minutes with minimal downtime

### Manual Maintenance (If Ever Needed)

In rare cases where you need to manually optimize, you can:

```sql
-- View index fragmentation (informational only)
SELECT
    OBJECT_NAME(ips.object_id) AS TableName,
    i.name AS IndexName,
    ips.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 30
ORDER BY ips.avg_fragmentation_in_percent DESC;

-- Rebuild a specific index if needed (usually not necessary)
ALTER INDEX [IX_AppRequests_UserId] ON [AppRequests] REBUILD;
```

However, for the App Request Portal's typical data volumes (hundreds to thousands of app requests), this manual intervention is almost never necessary.

## Disaster Recovery & Backups

The App Request Portal includes built-in disaster recovery features to protect your data.

### What's Automatically Protected

| Component | Protection | Recovery |
|-----------|------------|----------|
| **SQL Database** | Automated backups (7-day PITR) + geo-redundant storage | Restore to any point in time |
| **Storage Account** | Geo-redundant (GRS) with 6 copies across 2 regions | Automatic failover available |
| **Key Vault Secrets** | Soft delete (7-day recovery) | Recover deleted secrets |
| **Application Code** | GitHub repository + immutable release packages | Redeploy from ARM template |

### Quick Recovery Actions

**Restore deleted data (SQL):**
```bash
az sql db restore --resource-group <rg> --server <server> \
  --name AppRequestPortal --dest-name AppRequestPortal-Restored \
  --time "2026-02-14T10:00:00Z"
```

**Recover deleted Key Vault secret:**
```bash
az keyvault secret recover --vault-name <vault> --name AzureAdClientSecret
```

**Rollback to previous app version:**
```bash
az webapp config appsettings set --resource-group <rg> --name <app> \
  --settings WEBSITE_RUN_FROM_PACKAGE="https://github.com/powerstacks-corp/App-Request-Portal-Releases/releases/download/v1.5.5/AppRequestPortal.zip"
```

### High Availability (Optional)

For organizations requiring higher uptime, see the [Disaster Recovery Guide](DISASTER-RECOVERY.md) for:
- SQL Active Geo-Replication setup
- Traffic Manager / Azure Front Door configuration
- Multi-region deployment patterns

### Monthly Backup Verification

We recommend testing your recovery capability monthly:
1. Restore SQL database to a point 24 hours ago (test environment)
2. Verify data integrity
3. Delete test database
4. Document actual recovery time

## Troubleshooting

### Apps Not Syncing

- Verify the app registration has `DeviceManagementApps.Read.All` permission
- Check that admin consent is granted
- Look at API logs for Graph API errors

### Users Not Seeing Apps

- Verify the app's **Visible** toggle is set to Yes
- Check that the user is authenticated
- Confirm the app has synced (check Last Sync Date)

### Approvals Not Working

- Verify the approver is in the correct AAD group
- For Linear workflows, ensure the correct person is approving
- Check that the workflow is properly configured on the app

### User Not Added to Group

- Verify the app's **Target Group** is configured
- Check the app registration has `Group.ReadWrite.All` permission
- Look at API logs for Graph API errors

### Email Notifications Not Sending

- Verify `Mail.Send` permission has admin consent
- Check the **Send As User ID** is a valid Object ID
- Confirm **Enable email notifications** is toggled on in the **Communications** tab
- Look at API logs for email sending errors

### Request Shows "Failed" Status

When a request shows as "Failed" after approval, it means the system couldn't add the user to the target group. Common causes:

1. **User already in group** - Fixed in v1.2.0; now treated as success
2. **Group doesn't exist** - The target group may have been deleted
3. **Permission issue** - The app lacks `GroupMember.ReadWrite.All` permission
4. **Invalid group ID** - The group ID configured for the app is incorrect

**To retry a failed request:**
1. Go to **Admin > Reports > By Person**
2. Search for the user
3. Find the failed request and click **Retry**

### Viewing Azure App Service Logs

When troubleshooting issues, you can view detailed logs in Azure. There are three main options:

#### Option 1: Log Stream (Easiest - Real-time)

View logs as they happen. This is the quickest way to see what's happening:

1. Go to the [Azure Portal](https://portal.azure.com)
2. Navigate to your **App Service** (e.g., `apprequest-prod-xxxxx`)
3. In the left menu, under **Monitoring**, click **Log stream**
4. Logs will appear in real-time as requests are made

> **Tip:** Keep Log Stream open in one tab while reproducing the issue in another tab. Reproduce the error and watch the logs appear.

#### Option 2: Application Insights (Historical Queries)

View historical logs and run queries. This is useful for investigating past issues.

> **Setup Required:** Application Insights requires a connection string and (optionally) API credentials to be configured in your Azure App Service. If you haven't set this up yet, see **[Step 12: Configure Application Insights](SETUP.md#step-12-configure-application-insights-optional)** in the Setup Guide. The key settings are:
> - `APPLICATIONINSIGHTS_CONNECTION_STRING` — enables telemetry collection (logs, traces, exceptions)
> - `ApplicationInsights__AppId` and `ApplicationInsights__ApiKey` — enables the metrics dashboard in the Admin panel

> **Important:** You must navigate to the **Application Insights resource directly** (e.g., `ai-apprequest-prod`), NOT the "Logs" blade inside App Service. The App Service > Logs blade queries Log Analytics tables (`AppServiceConsoleLogs`), not Application Insights tables (`traces`).

**Steps:**
1. Go to the [Azure Portal](https://portal.azure.com)
2. Go to your **Resource Group**
3. Click on the **Application Insights** resource (e.g., `ai-apprequest-prod`)
4. In the left menu, click **Logs**
5. Close the "Queries" popup if it appears
6. Use these queries:

**View recent errors (severity 3+):**
```kusto
traces
| where severityLevel >= 3
| order by timestamp desc
| take 100
```

**View all logs from the last hour:**
```kusto
traces
| where timestamp > ago(1h)
| order by timestamp desc
```

**Search for group operations:**
```kusto
traces
| where message contains "AddUserToGroup" or message contains "group"
| order by timestamp desc
| take 50
```

**View exceptions:**
```kusto
exceptions
| order by timestamp desc
| take 50
```

**If you see "No tables" or "Failed to resolve table 'traces'":**
- You may be in the wrong location. Make sure you're in **Application Insights > Logs**, not **App Service > Logs**
- If Application Insights was just deployed, wait 5-10 minutes for data to appear
- Ensure the portal is running version 1.2.0+ (which includes Application Insights integration)

#### Option 3: App Service Logs (Log Analytics)

If you're in the App Service > Logs blade, it uses different tables. Use these queries instead:

**View console output:**
```kusto
AppServiceConsoleLogs
| order by TimeGenerated desc
| take 100
```

**View HTTP logs:**
```kusto
AppServiceHTTPLogs
| order by TimeGenerated desc
| take 100
```

**Check what tables are available:**
```kusto
search *
| summarize count() by $table
```

#### Option 4: Enable Detailed Filesystem Logging

For even more detail, enable filesystem logging:

1. Go to your **App Service** in Azure Portal
2. Under **Monitoring**, click **App Service logs**
3. Enable **Application Logging (Filesystem)** and set level to **Information** or **Verbose**
4. Enable **Detailed error messages**
5. Click **Save**

Logs are then available in Log Stream and can be downloaded from **Advanced Tools (Kudu)** > **Debug Console** > **LogFiles**
