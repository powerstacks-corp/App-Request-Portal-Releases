# Admin Guide

This guide walks administrators through setting up and managing the App Request Portal using the web-based admin interface.

## Getting Started

After initial deployment and Azure AD configuration (see [SETUP.md](SETUP.md)), most portal configuration can be done directly through the Admin Dashboard.

### Accessing the Admin Dashboard

1. Sign in to the portal with an account that is a member of the Admin Group
2. Click **Admin** in the navigation menu
3. You'll see six tabs:
   - **App Management** - Manage apps synced from Intune
   - **Pending Approvals** - Review and approve/reject requests
   - **Settings** - Configure portal-wide options
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
   - **Access Groups** - Configure Admin and Approver Azure AD groups
   - **Email Notifications** - Set up email settings for notifications
   - **Sync Apps** - Import apps from your Intune tenant

The wizard saves your settings as you progress through each step. You can skip the wizard at any time and configure settings manually.

> **Note:** The License step is required for the portal to be fully operational. Without a valid license, users will see warning banners and some features may be restricted.

**Quick Reference Commands** (shown in wizard completion):
| Component | Command |
|-----------|---------|
| API | `cd src/AppRequestPortal.API && dotnet run` |
| Web | `cd src/AppRequestPortal.Web && npm start` |
| Packager | `cd src/AppRequestPortal.PackagingAgent && dotnet run` |

## Portal Settings

The Settings tab allows you to configure portal-wide options without editing configuration files.

### Email Notifications

Configure how the portal sends email notifications for request submissions and approvals.

| Setting | Description |
|---------|-------------|
| **Enable email notifications** | Toggle to turn email notifications on or off |
| **Send As User ID** | The Azure AD Object ID of the user or shared mailbox that will send emails. Find this in Azure Portal > Azure AD > Users > [select user] > Object ID |
| **From Address** | The email address displayed in the From field (should match the mailbox) |
| **Portal URL** | The URL of your portal, used in email links to direct users back to the portal |

> **Note:** The app registration must have the `Mail.Send` Microsoft Graph permission with admin consent granted.

### Microsoft Teams Notifications

Send notifications to a Microsoft Teams channel when app requests are submitted, approved, or rejected. This uses Teams Incoming Webhooks for simple, secure integration.

| Setting | Description |
|---------|-------------|
| **Enable Teams notifications** | Toggle to turn Teams channel notifications on or off |
| **Webhook URL** | The Incoming Webhook URL from your Teams channel |
| **Test** | Send a test notification to verify the webhook is configured correctly |
| **New request submitted** | Notify the channel when a new app request is submitted |
| **Request approved** | Notify the channel when a request is approved |
| **Request rejected** | Notify the channel when a request is rejected |

#### Setting Up Teams Notifications

**Step 1: Create a Teams Channel (or use existing)**

1. Open **Microsoft Teams**
2. Navigate to the Team where you want notifications
3. Create a new channel (e.g., "App Approvals") or use an existing one
4. Ensure approvers/admins who need to see notifications are members of this channel

**Step 2: Add an Incoming Webhook Connector**

1. In Teams, right-click the channel name and select **Manage channel**
2. Click the **Connectors** tab (or **Edit** > **Connectors** in newer versions)
3. Search for **Incoming Webhook** and click **Configure**
4. Enter a name (e.g., "App Request Portal") - this name appears on notifications
5. Optionally upload a custom icon for the webhook
6. Click **Create**
7. **Copy the webhook URL** - you'll need this for the portal settings

> **Important:** The webhook URL is a secret. Anyone with this URL can post to your channel. Store it securely and don't commit it to source control.

**Step 3: Configure the Portal**

1. Go to **Admin** > **Settings** tab
2. Scroll to **Microsoft Teams Notifications**
3. Enable **Enable Teams notifications**
4. Paste the webhook URL in the **Webhook URL** field
5. Click **Test** to verify the connection - you should see a test message in Teams
6. Configure which events should trigger notifications
7. Click **Save Settings**

#### How It Works

- Notifications are sent as **Adaptive Cards** - rich, formatted messages in Teams
- Each notification includes:
  - Title (e.g., "New App Request Submitted")
  - Requestor name and email
  - App name and publisher
  - Timestamp
  - Action button linking to the portal
- The notification appears to come from the webhook connector name you configured
- Everyone in the Teams channel sees the notification

#### Limitations

- **Channel-based only**: Notifications go to a channel, not direct messages to individuals
- **One-way**: Users cannot approve/reject directly from Teams - they click through to the portal
- **Single channel**: All notification types go to the same channel (the configured webhook)

> **Tip:** For a dedicated approvals workflow, create a private channel with only approvers as members, and use that channel's webhook URL.

### Group-Based Authorization

Control who has admin and approver access to the portal.

| Setting | Description |
|---------|-------------|
| **Admin Group** | Azure AD group Object ID. Members have full admin access to sync apps, manage settings, and view all requests |
| **Approver Group** | Azure AD group Object ID. Members can approve/reject requests (in addition to workflow-specific approvers) |

> **Tip:** Leave these empty during development to allow all authenticated users admin access. For production, always configure security groups.

### Recommended Conditional Access Policy

Since the App Request Portal is used to request apps for Intune-managed devices, we recommend protecting access to the portal with a Conditional Access policy that requires:
- **Managed device** - The device accessing the portal must be enrolled in Intune
- **Compliant device** - The device must meet your organization's compliance policies

This ensures users can only request apps from trusted, compliant devices.

#### Prerequisites

Before creating the policy:
1. You must have **Azure AD Premium P1** or **P2** license (or Microsoft 365 E3/E5, etc.)
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
| **Auto-create Azure AD groups** | Automatically create a security group when an app doesn't have a target group configured |

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

If auto-update is not configured, you can still update by redeploying from the [releases repository](https://github.com/PowerStacks-BI/App-Request-Portal-Releases):

1. Go to the releases repository
2. Click the **Deploy to Azure** button
3. Use the same resource group and settings as your original deployment
4. The ARM template will update the existing deployment with the latest version

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
| **Group Name Prefix** | Prefix used when auto-creating Azure AD groups (default: `AppPortal-`). Groups are named `{prefix}{AppName}-Required`. Use this to identify portal-managed groups in your tenant. |

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

1. **Update Azure AD Redirect URIs** - Add your custom domain URLs to your App Registration
2. **Update Portal URL** - In Settings > Email Notifications, update the Portal URL to use your custom domain
3. **Test Authentication** - Sign out and sign back in to verify authentication works

> **Note:** For detailed DNS configuration, certificate options, and troubleshooting, see [CUSTOM-DOMAINS.md](CUSTOM-DOMAINS.md).

## App Management

The App Management tab is your central hub for managing which Intune apps are available in the portal and how they behave.

### Syncing Apps from Intune

1. Click the **Sync Apps from Intune** button
2. The portal fetches all mobile apps from your Intune tenant
3. Apps are imported with their name, publisher, description, icon, and category
4. New apps are hidden by default - you must make them visible for users to see them

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

**Automatic Deployment Setup:**

When you toggle an app's visibility to **Yes** for the first time, the portal automatically:

1. **Creates an Azure AD Security Group** named `{GroupNamePrefix}{AppName}-Required`
   - Example: `AppPortal-Microsoft Teams-Required`
   - The prefix is configurable in Settings (default: `AppPortal-`)

2. **Creates an Intune App Assignment**
   - The app is assigned as **Required** to the new security group
   - Assignment type (User or Device) is based on the app's Assignment setting

This automation ensures that when a user's request is approved, they simply need to be added to the group and Intune handles the deployment.

> **Note:** If group or assignment creation fails, the error is logged but the visibility toggle still succeeds. You can manually create the assignment in Intune if needed.

#### Requires Approval Toggle

- **Yes**: Requests go through the configured approval workflow before completion
- **No**: Requests are auto-approved and the user/device is immediately added to the target group

Use "No" for low-risk apps that don't need oversight. Use "Yes" for apps that need manager or IT approval.

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
| **Target Group** | Azure AD security group for app assignment. Click "Search Groups" to browse, or use "Clear" to remove. |
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

#### Assignment Type

Determines what gets added to the Azure AD group when a request is approved:

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

See [APPROVAL-WORKFLOWS.md](APPROVAL-WORKFLOWS.md) for detailed workflow configuration options.

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
2. **Consistent Naming**: Groups follow the pattern `{GroupNamePrefix}{AppName}-Required` (e.g., `AppPortal-Microsoft Teams-Required`)
3. **Custom Prefix**: Configure the Group Name Prefix in Settings to match your organization's naming conventions
4. **Manual Override**: You can still manually set a Target Group on an app if you prefer to use an existing group

> **Tip:** To use existing groups instead of auto-created ones, set the Target Group before making the app visible.

### Email Notifications

1. Create a dedicated shared mailbox for notifications
2. Grant the app `Mail.Send` permission on that mailbox
3. Use a recognizable From address like `apprequest@company.com`
4. Include your portal URL so users can click through to view status

## Winget Catalog & Cloud Packaging

The **Winget Catalog** tab in the Admin Dashboard allows you to browse over 10,000 apps from the Windows Package Manager repository and publish them directly to Intune as Win32 apps.

### Browsing the Winget Catalog

1. Navigate to **Admin** > **Winget Catalog** tab
2. Browse popular packages or use the search box to find specific apps
3. Each package shows: name, publisher, version, and description
4. Icons are loaded automatically where available

### Publishing to Intune

1. Find the app you want to publish
2. Click **Publish to Intune** button on the package card
3. A packaging job is created and queued for processing
4. Monitor job status in the **Packaging Jobs** section below the catalog

### Packaging Jobs

The Packaging Jobs section shows all queued and completed packaging operations:

| Status | Description |
|--------|-------------|
| **Pending** | Job is queued, waiting for processing |
| **Downloading** | Cloud agent is downloading the package from Winget |
| **Packaging** | Creating .intunewin package with IntuneWinAppUtil |
| **Uploading** | Uploading package to Intune via Graph API |
| **Creating** | Creating Win32LobApp in Intune |
| **Completed** | App successfully created in Intune |
| **Failed** | Error occurred - click Retry to requeue |

### Cloud Packaging Architecture

The packaging process runs entirely in the cloud using Azure Container Instance:

1. **Queue Message**: When you click "Publish to Intune", a job is added to Azure Storage Queue
2. **Azure Function**: A queue-triggered function detects the new message
3. **Container Instance**: The function starts a Windows container with IntuneWinAppUtil.exe
4. **Package Creation**: The container downloads the app, creates the .intunewin package
5. **Intune Upload**: The package is uploaded to Intune and a Win32 app is created

No on-premises infrastructure is required. Containers start on-demand and terminate when complete.

### Setting Up Cloud Packaging

If you deployed with `enablePackagingAgent=true` (default), the infrastructure is already in place. You just need to:

1. **Build and push the container image**:
   ```bash
   # Authenticate to your Azure Container Registry
   az acr login --name <your-acr-name>

   # Build and push the packaging agent image
   cd src/AppRequestPortal.PackagingAgent
   docker build -t <your-acr>.azurecr.io/packaging-agent:latest .
   docker push <your-acr>.azurecr.io/packaging-agent:latest
   ```

2. **Deploy the Azure Function**:
   ```bash
   cd src/AppRequestPortal.PackagingTrigger
   func azure functionapp publish <your-function-app-name>
   ```

3. **Verify connectivity**:
   - Go to Admin > Winget Catalog
   - Try publishing a test app (e.g., "7-Zip")
   - Check Packaging Jobs for status

### Winget Integration Settings

In **Admin** > **Settings** > **Winget Integration**:

| Setting | Description |
|---------|-------------|
| **Winget Repository API URL** | Custom Winget API endpoint. Leave empty to use the public api.winget.run. Use this to point to a private/trusted repository. |

### Local Development Testing

For developers testing the packaging feature locally (without Azure Container Instance):

1. **Prerequisites**:
   - Download [IntuneWinAppUtil.exe](https://github.com/microsoft/Microsoft-Win32-Content-Prep-Tool) to `C:\Tools\IntuneWinAppUtil.exe`
   - Ensure Winget CLI is installed on your machine
   - Azure Storage account configured (uses the same storage queue as production)

2. **Configure the Packaging Agent** (`src/AppRequestPortal.PackagingAgent/appsettings.json`):
   ```json
   {
     "PortalApi": {
       "BaseUrl": "http://localhost:5000",
       "ApiKey": ""
     },
     "IntuneWinAppUtil": {
       "Path": "C:\\Tools\\IntuneWinAppUtil.exe"
     }
   }
   ```

3. **Configure the API** (`src/AppRequestPortal.API/appsettings.json`):
   - Ensure `AzureStorage:ConnectionString` points to your Azure Storage account
   - Optionally set `PackagingAgent:ApiKey` to secure the callback endpoints

4. **Run all services**:
   ```bash
   # API (port 5000)
   cd src/AppRequestPortal.API && dotnet run

   # Frontend (port 3000)
   cd src/AppRequestPortal.Web && npm start

   # Packaging Agent (polls queue)
   cd src/AppRequestPortal.PackagingAgent && dotnet run
   ```

5. **Test**: Go to Admin > Winget Catalog, search for an app, click "Publish to Intune". The local agent will pick up the job and process it.

### Troubleshooting Cloud Packaging

**Jobs stuck in Pending:**
- Verify the Azure Function is deployed and running (or the local Packaging Agent is running)
- Check Function App logs in Azure Portal
- Ensure the storage queue connection is configured

**Jobs failing with download errors:**
- The agent tries multiple download methods (Winget, Chocolatey, direct download)
- Check if the package exists and is accessible
- For local testing: ensure Winget CLI is installed and working

**Jobs failing during upload:**
- Verify the API has Graph API permissions for Intune (`DeviceManagementApps.ReadWrite.All`)
- Check the `PortalApi:BaseUrl` setting points to your API
- Review API logs for Graph API errors (detailed error messages are logged)

**Jobs failing with "Request body too large":**
- This was fixed in recent updates. Ensure your API is running the latest code
- The `/api/packaging/jobs/{id}/complete` endpoint now has `[DisableRequestSizeLimit]` attribute

**Container not starting (Azure):**
- Verify the container image is pushed to ACR
- Check ACR credentials in Function App settings
- Ensure the subscription has Windows container quota available

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
  --settings WEBSITE_RUN_FROM_PACKAGE="https://github.com/PowerStacks-BI/App-Request-Portal-Releases/releases/download/v1.5.5/AppRequestPortal.zip"
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
- Confirm **Enable email notifications** is toggled on
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
