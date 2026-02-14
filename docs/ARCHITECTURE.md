# Architecture Overview

This document provides a detailed overview of the App Request Portal architecture.

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Azure Cloud                              │
│                                                                   │
│  ┌──────────────┐         ┌──────────────┐                      │
│  │   Azure AD   │◄────────┤   Frontend   │                      │
│  │              │         │  (React SPA) │                      │
│  └──────┬───────┘         └──────┬───────┘                      │
│         │                        │                               │
│         │                        │ HTTPS/JWT                     │
│         │                        │                               │
│         │                 ┌──────▼───────┐                      │
│         │                 │   API Layer  │                      │
│         │                 │ (ASP.NET 8)  │                      │
│         │                 └──────┬───────┘                      │
│         │                        │                               │
│         │         ┌──────────────┼──────────────┐               │
│         │         │              │              │               │
│         │    ┌────▼─────┐  ┌────▼─────┐  ┌────▼─────┐         │
│         └────►  Graph    │  │   SQL    │  │   App    │         │
│              │   API     │  │ Database │  │ Insights │         │
│              └───────────┘  └──────────┘  └──────────┘         │
│                    │                                             │
│              ┌─────▼──────┐                                     │
│              │   Intune   │                                     │
│              │  & Groups  │                                     │
│              └────────────┘                                     │
└─────────────────────────────────────────────────────────────────┘
```

## Components

### 1. Frontend (React SPA)

**Technology Stack**:
- React 18
- TypeScript
- Azure MSAL for authentication
- Axios for HTTP requests
- React Router for navigation

**Key Features**:
- Single-page application for optimal performance
- Azure AD authentication with SSO
- Responsive design for desktop and mobile
- Real-time status updates for app requests

**Files**:
- [/src/AppRequestPortal.Web/src/App.tsx](../src/AppRequestPortal.Web/src/App.tsx)
- [/src/AppRequestPortal.Web/src/authConfig.ts](../src/AppRequestPortal.Web/src/authConfig.ts)
- [/src/AppRequestPortal.Web/src/services/apiClient.ts](../src/AppRequestPortal.Web/src/services/apiClient.ts)

### 2. Backend API (ASP.NET Core)

**Technology Stack**:
- ASP.NET Core 8.0
- Entity Framework Core
- Microsoft.Identity.Web
- Microsoft Graph SDK

**Architecture Pattern**: Clean Architecture / Layered Architecture

**Layers**:
1. **API Layer** ([AppRequestPortal.API](../src/AppRequestPortal.API/))
   - Controllers
   - Authentication/Authorization
   - Middleware
   - API endpoints

2. **Core Layer** ([AppRequestPortal.Core](../src/AppRequestPortal.Core/))
   - Domain models
   - Business interfaces
   - Business logic (services)

3. **Infrastructure Layer** ([AppRequestPortal.Infrastructure](../src/AppRequestPortal.Infrastructure/))
   - Data access (Entity Framework)
   - External service integrations (Graph API)
   - Repository implementations

### 3. Database (Azure SQL)

**Schema**:

```sql
-- Apps table
CREATE TABLE Apps (
    Id NVARCHAR(450) PRIMARY KEY,
    IntuneAppId NVARCHAR(450) NOT NULL UNIQUE,
    DisplayName NVARCHAR(255) NOT NULL,
    Description NVARCHAR(1000),
    Publisher NVARCHAR(255),
    Version NVARCHAR(50),
    IconUrl NVARCHAR(MAX),
    IsVisible BIT NOT NULL DEFAULT 1,
    RequiresApproval BIT NOT NULL DEFAULT 1,
    AzureADGroupId NVARCHAR(450),
    AzureADGroupName NVARCHAR(255),
    CreatedDate DATETIME2 NOT NULL,
    LastSyncDate DATETIME2 NOT NULL
);

-- AppRequests table
CREATE TABLE AppRequests (
    Id NVARCHAR(450) PRIMARY KEY,
    AppId NVARCHAR(450) NOT NULL,
    UserId NVARCHAR(450) NOT NULL,
    UserEmail NVARCHAR(255) NOT NULL,
    UserDisplayName NVARCHAR(255) NOT NULL,
    DeviceId NVARCHAR(450),
    DeviceName NVARCHAR(255),
    Status INT NOT NULL,
    Justification NVARCHAR(1000),
    RequestedDate DATETIME2 NOT NULL,
    ReviewedDate DATETIME2,
    ReviewedBy NVARCHAR(450),
    ReviewerEmail NVARCHAR(255),
    ReviewComments NVARCHAR(1000),
    FOREIGN KEY (AppId) REFERENCES Apps(Id)
);

-- AppApprovers table
CREATE TABLE AppApprovers (
    Id NVARCHAR(450) PRIMARY KEY,
    AppId NVARCHAR(450) NOT NULL,
    UserId NVARCHAR(450) NOT NULL,
    UserEmail NVARCHAR(255) NOT NULL,
    UserDisplayName NVARCHAR(255) NOT NULL,
    ApproverType INT NOT NULL,
    CreatedDate DATETIME2 NOT NULL,
    FOREIGN KEY (AppId) REFERENCES Apps(Id) ON DELETE CASCADE
);

-- AuditLogs table
CREATE TABLE AuditLogs (
    Id NVARCHAR(450) PRIMARY KEY,
    UserId NVARCHAR(450) NOT NULL,
    UserEmail NVARCHAR(255) NOT NULL,
    Action NVARCHAR(100) NOT NULL,
    EntityType NVARCHAR(100) NOT NULL,
    EntityId NVARCHAR(450) NOT NULL,
    Details NVARCHAR(MAX),
    Timestamp DATETIME2 NOT NULL,
    IpAddress NVARCHAR(50) NOT NULL
);

-- PortalSettings table (singleton)
CREATE TABLE PortalSettings (
    Id INT PRIMARY KEY,
    -- Email settings
    EmailSendAsUserId NVARCHAR(100),
    EmailFromAddress NVARCHAR(255),
    EmailPortalUrl NVARCHAR(500),
    EmailNotificationsEnabled BIT NOT NULL DEFAULT 0,
    -- Teams notification settings
    TeamsNotificationsEnabled BIT NOT NULL DEFAULT 0,
    TeamsWebhookUrl NVARCHAR(MAX),
    TeamsNotifyOnNewRequest BIT NOT NULL DEFAULT 1,
    TeamsNotifyOnApproval BIT NOT NULL DEFAULT 1,
    TeamsNotifyOnRejection BIT NOT NULL DEFAULT 1,
    -- Group authorization
    AdminGroupId NVARCHAR(100),
    AdminGroupName NVARCHAR(255),
    ApproverGroupId NVARCHAR(100),
    ApproverGroupName NVARCHAR(255),
    UserAccessGroupId NVARCHAR(100),
    UserAccessGroupName NVARCHAR(255),
    -- General settings
    RequireManagerApproval BIT NOT NULL DEFAULT 1,
    AutoCreateGroups BIT NOT NULL DEFAULT 1,
    GroupNamePrefix NVARCHAR(50) DEFAULT 'AppPortal-',
    -- Display settings
    MaxFeaturedApps INT DEFAULT 0,
    DarkModeEnabled BIT NOT NULL DEFAULT 0,
    HeroAppId NVARCHAR(450),
    -- Integration settings
    WingetRepoUrl NVARCHAR(500),
    -- Reports & ROI settings
    HelpDeskCostPerTicket DECIMAL(18,2) DEFAULT 22.00,
    CurrencyCode NVARCHAR(10) DEFAULT 'USD',
    CurrencySymbol NVARCHAR(10) DEFAULT '$',
    -- Metadata
    LastModified DATETIME2 NOT NULL,
    ModifiedBy NVARCHAR(255)
);
```

### 4. Microsoft Graph API Integration

**Purpose**:
- Retrieve apps from Intune
- Manage Azure AD groups
- Add/remove users and devices from groups
- Get user information and manager hierarchy

**Permissions Required**:
- `DeviceManagementApps.Read.All`
- `DeviceManagementApps.ReadWrite.All`
- `DeviceManagementManagedDevices.Read.All` (for device count)
- `Group.ReadWrite.All`
- `User.Read.All`
- `Directory.Read.All`

**Key Operations**:
```csharp
// Get Intune apps
GET /deviceAppManagement/mobileApps

// Create AD group
POST /groups

// Add user to group
POST /groups/{group-id}/members/$ref

// Get user's manager
GET /users/{user-id}/manager

// Get user's devices
GET /users/{user-id}/managedDevices
```

## Data Flow

### User Request Flow

```
1. User browses apps
   Frontend → API → Database → Apps list

2. User submits request
   Frontend → API → Database (save request)
                  → Email service (notify approvers)
                  → Teams service (notify channel)

3. Approver reviews request
   Frontend → API → Database (update status)
                  → Email service (notify requester)
                  → Teams service (notify channel)

4. System processes approved request
   API → Graph API (create/get AD group)
       → Graph API (add user/device to group)
       → Intune (app deployment triggered automatically)
       → Database (update request status)
       → Email service (notify requester)
       → Teams service (notify channel)
```

### Notification Services

The portal uses two notification channels:

| Service | Purpose | Technology |
|---------|---------|------------|
| **EmailNotificationService** | Direct user notifications | Microsoft Graph Mail.Send API |
| **TeamsNotificationService** | Channel-wide notifications | Teams Incoming Webhooks with Adaptive Cards |

Both services are optional and can be enabled/disabled independently in Admin Settings.

### App Sync Flow

```
1. Admin triggers sync
   Frontend → API → Graph API (get Intune apps)
                  → Graph API (get managed device count, last 30 days)
                  → PowerStacks License API (validate license)
                  → Database (update apps, device count, license info)

2. Scheduled sync (background job)
   Background Service → Graph API (get Intune apps)
                     → Graph API (get managed device count)
                     → PowerStacks License API (validate license)
                     → Database (update apps, device count, license info)
```

## Security Architecture

### Authentication Flow

```
1. User visits frontend
2. MSAL redirects to Azure AD login
3. User authenticates
4. Azure AD returns ID token and access token
5. Frontend stores tokens in session storage
6. Frontend includes access token in API requests
7. API validates JWT token
8. API authorizes based on user roles/claims
```

### Authorization Levels

- **User**: Can browse apps and submit requests
- **Approver**: Can approve/reject requests for assigned apps
- **Admin**: Can manage apps, approvers, and view all requests

### Security Features

1. **HTTPS Only**: All communication encrypted in transit
2. **JWT Authentication**: Stateless authentication with Azure AD
3. **RBAC**: Role-based access control for granular permissions
4. **Conditional Access**: Integration with Azure AD Conditional Access policies
5. **Managed Identity**: Service-to-service authentication without secrets
6. **Key Vault**: Secure storage of secrets and certificates
7. **Audit Logging**: Complete audit trail of all actions

## Scalability Considerations

### Horizontal Scaling

- **App Services**: Scale out to multiple instances
- **Database**: Use Azure SQL elastic pools or scale up tiers
- **Frontend**: Served via CDN for global distribution

### Caching Strategy

```csharp
// Cache app list (1 hour TTL)
[ResponseCache(Duration = 3600)]
public async Task<IActionResult> GetApps()

// Cache user's devices (30 minutes TTL)
[ResponseCache(Duration = 1800, VaryByQueryKeys = new[] { "userId" })]
public async Task<IActionResult> GetUserDevices(string userId)
```

### Performance Optimizations

1. **Database Indexing**: Indexes on frequently queried columns
2. **Connection Pooling**: Efficient database connection management
3. **Async/Await**: Non-blocking I/O operations
4. **Lazy Loading**: Load data only when needed
5. **Pagination**: Limit result sets for large queries

## Monitoring and Observability

### Application Insights Integration

```csharp
// Custom telemetry
telemetryClient.TrackEvent("AppRequestApproved", new Dictionary<string, string>
{
    { "AppId", appId },
    { "UserId", userId },
    { "ProcessingTime", processingTime.ToString() }
});

// Dependency tracking
telemetryClient.TrackDependency("GraphAPI", "GetIntuneApps", startTime, duration, success);
```

### Key Metrics

- Request rate and response times
- Failure rates and error types
- Graph API call latency
- Database query performance
- User authentication success rate
- App request approval time

### Logging

```csharp
// Structured logging with Serilog or built-in ILogger
_logger.LogInformation(
    "App request {RequestId} approved by {ReviewerId} for user {UserId}",
    requestId, reviewerId, userId
);

_logger.LogError(
    exception,
    "Failed to add user {UserId} to group {GroupId}",
    userId, groupId
);
```

## Disaster Recovery

The default deployment includes Tier 2 disaster recovery capabilities with geo-redundant backups across Azure regions.

### Built-in Protection (Default)

| Component | Protection | RPO | RTO |
|-----------|------------|-----|-----|
| **SQL Database** | Automated backups + geo-redundant storage | 5 min | 1-2 hours |
| **Storage Account** | Geo-redundant (GRS) - 6 copies across 2 regions | 0 | 1-2 hours |
| **Key Vault** | Soft delete (7-day recovery window) | 0 | 15 min |
| **Application** | GitHub releases + ARM template | 0 | 30 min |

### Backup Details

- **SQL Point-in-Time Restore**: 7 days (Basic tier), up to 35 days (Standard+)
- **SQL Geo-Backup**: Cross-region backup for regional disaster recovery
- **Storage GRS**: Synchronous replication to paired Azure region
- **Key Vault Soft Delete**: 7-day retention for accidental deletion recovery

### High Availability Options (Tier 3)

For organizations requiring higher uptime, these can be configured manually:

```
┌─────────────────────────────────────────────────────────────┐
│                    Traffic Manager                           │
│                  (DNS-based failover)                        │
└──────────────────────┬───────────────────┬──────────────────┘
                       │                   │
           ┌───────────▼───────┐ ┌────────▼────────┐
           │   Primary Region  │ │ Secondary Region │
           │  ┌─────────────┐  │ │ ┌─────────────┐  │
           │  │ App Service │  │ │ │ App Service │  │
           │  └──────┬──────┘  │ │ └──────┬──────┘  │
           │         │         │ │        │         │
           │  ┌──────▼──────┐  │ │ ┌──────▼──────┐  │
           │  │  SQL (R/W)  │←─┼─┼─┤ SQL (Read)  │  │
           │  └─────────────┘  │ │ └─────────────┘  │
           └───────────────────┘ └─────────────────┘
                 Active              Standby
```

- **SQL Active Geo-Replication**: ~$5/month (Basic replica)
- **Traffic Manager**: ~$0.75/million queries
- **Secondary App Service**: ~$55/month (B2)

### Recovery Procedures

See the [Disaster Recovery Guide](DISASTER-RECOVERY.md) for detailed runbooks covering:
- Accidental data deletion recovery
- Key Vault secret recovery
- Complete region failure recovery
- Application rollback procedures

## Admin Features

### Setup Wizard
A guided setup wizard helps administrators configure the portal on first use:
1. **License** - Enter and validate PowerStacks license key
2. **Access Groups** - Configure admin and approver Azure AD groups
3. **Email Notifications** - Set up email sender identity and notification preferences
4. **Sync Apps** - Import apps from Intune catalog

### Branding Customization
Administrators can fully customize the portal appearance:
- **Logo & Favicon** - Upload custom images (PNG, JPG, SVG, ICO)
- **Colors** - Primary color, hover color, secondary color, header text color, background color, card background
- **Text** - Portal title, tagline, welcome message, footer text, support contact info

### Dark Mode
- Admin-controlled default dark mode setting
- User toggle to override admin preference (persisted in localStorage)
- Respects system preference when user hasn't set a preference
- Full dark mode styling for all components

### App Categories
Apps can be organized into categories for easier browsing:
- Categories are synced from Intune app metadata
- Apps display their category on the Browse Apps page
- Filter apps by category

### Multi-Stage Approval Workflows
Custom approval workflows can be configured per app:
- **Multiple stages** - Define sequential approval stages
- **Stage types** - Manager approval, specific user approval, or group approval
- **Notifications** - Email notifications at each stage
- **Tracking** - View current approval stage in pending approvals list

### Winget Catalog Integration
Import apps from the Windows Package Manager (Winget) repository:
- Search the Winget catalog by name
- View app details including publisher, version, and description
- Create Intune Win32 apps from Winget packages (requires packaging service)

## Reports & Analytics

The portal includes a comprehensive Reports tab for administrators with the following capabilities:

### ROI Calculator
- Calculates cost savings based on completed app requests
- Formula: `completedRequests × costPerTicket = totalSavings`
- Default help desk cost: $22/ticket (industry benchmark for Tier 1 support)
- Configurable currency (USD, EUR, GBP, CAD, AUD, JPY, CHF, INR)
- Time period filters: Last 30 days, 90 days, Year, All time

### App-Based Reports
- View total requests per app
- Breakdown by status (completed, pending, rejected)
- User history for each app

### Person-Based Reports
- Search users who have made requests
- View all apps requested by a specific user
- Request dates and status tracking

### Approval Analytics
- Top 10 approvers by approval count
- Pending approvals count
- Average approval time (hours)
- Common rejection reasons

## Licensing System

The portal integrates with PowerStacks License API for device-based licensing:

### License Validation
- API endpoint: `https://api.powerstacks.com/powerbi-app/auth`
- Validates license status, device count limits, and tenant authorization
- Auto-validates every 24 hours or on-demand via admin panel

### Device Count Enforcement
- Counts managed devices from Intune that have checked in within last 30 days
- Device count is updated during each app sync
- **3% Grace Period**: If device count exceeds the licensed limit by up to 3%, the portal remains operational but displays a warning banner to users
- If device count exceeds the licensed limit plus the 3% grace period, new app requests are blocked

### Security Measures
- License validation forces API check for critical operations (not just cached data)
- Prevents database tampering from bypassing license enforcement
- Displays warning banners to end users when license issues occur

### License Configuration

The license API key can be configured in two ways:

1. **Via Admin Dashboard** (Recommended): Navigate to Admin Dashboard → License tab and enter the API key in the "License Key" field. This stores the key in the database.

2. **Via Setup Wizard**: During initial setup, enter the API key in the License step of the Setup Wizard.

3. **Via `appsettings.json`** (Fallback): Add to your configuration file:
```json
{
  "License": {
    "ApiKey": "your-powerstacks-api-key"
  }
}
```

The system checks for the API key in the following order:
1. Database (LicenseInfo.ApiKey)
2. Configuration file (License:ApiKey)

### Database Tables

```sql
-- LicenseInfo table (singleton)
CREATE TABLE LicenseInfo (
    Id INT PRIMARY KEY,
    Status NVARCHAR(50) NOT NULL DEFAULT 'unknown',
    Enabled BIT NOT NULL DEFAULT 0,
    LicenseType NVARCHAR(50),
    StartDate DATETIME2,
    EndDate DATETIME2,
    LicensedDeviceCount INT,
    CurrentDeviceCount INT NOT NULL DEFAULT 0,
    AuthorizedTenants NVARCHAR(MAX),
    TenantId NVARCHAR(100),
    IsTenantAuthorized BIT NOT NULL DEFAULT 0,
    IsWithinDeviceLimit BIT NOT NULL DEFAULT 1,
    LastValidated DATETIME2,
    LastDeviceCountUpdate DATETIME2,
    ErrorMessage NVARCHAR(500),
    ApiKey NVARCHAR(500)
);
```

## Future Enhancements

1. **Power Automate Integration**: Visual workflow designer for approvals
2. **Real-time Notifications**: SignalR for live updates
3. ~~**Advanced Analytics**: Usage reports and insights dashboard~~ ✓ Implemented
4. **Mobile App**: Native mobile app for iOS/Android
5. **Self-Service Group Management**: Let users manage their own app groups
6. ~~**App Categories**: Organize apps by category/department~~ ✓ Implemented
7. **Scheduled Deployments**: Schedule app installations for specific times
8. **Multi-language Support**: Internationalization (i18n)
9. ~~**Teams Notifications**: Notify Teams channels on request events~~ ✓ Implemented
10. **Teams Approvals**: Approve/reject directly from Teams Adaptive Cards
