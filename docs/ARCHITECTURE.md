# Architecture Overview

This document provides a detailed overview of the App Portal for Intune architecture.

## System Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                              Azure Cloud                               │
│                                                                        │
│  ┌──────────────┐         ┌──────────────┐                             │
│  │   Entra ID   │◄────────┤   Frontend   │                             │
│  │              │         │  (React SPA) │                             │
│  └──────┬───────┘         └──────┬───────┘                             │
│         │                        │                                     │
│         │                        │ HTTPS/JWT                           │
│         │                        │                                     │
│         │                 ┌──────▼───────┐      ┌──────────────┐       │
│         │                 │   API Layer  │◄─────┤  Azure Bot   │       │
│         │                 │ (ASP.NET 8)  │      │ (Teams Bot)  │       │
│         │                 └──────┬───────┘      └──────┬───────┘       │
│         │                        │                     │               │
│         │    ┌───────────┬───────┼──────────┬──────────┘               │
│         │    │           │       │          │                          │
│         │  ┌─▼────────┐ │ ┌─────▼────┐ ┌───▼──────┐ ┌──────────┐     │
│         └──► Graph API │ │ │   SQL    │ │  Azure   │ │   App    │     │
│            │           │ │ │ Database │ │ Storage  │ │ Insights │     │
│            └─────┬─────┘ │ └──────────┘ └──────────┘ └──────────┘     │
│                  │       │                                             │
│            ┌─────▼─────┐ │  ┌──────────────┐                          │
│            │  Intune   │ └──► Key Vault    │                          │
│            │ & Groups  │    │ (Secrets)    │                          │
│            └───────────┘    └──────────────┘                          │
└────────────────────────────────────────────────────────────────────────┘
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
- Entra ID authentication with SSO
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

The database is managed by Entity Framework Core with automatic migrations applied on startup. The schema contains **19 tables** organized into the following categories:

#### Table Summary

| Table | Purpose | Type |
|-------|---------|------|
| `Apps` | Intune app catalog synced from Microsoft Intune | Core |
| `AppRequests` | User app requests with status tracking | Core |
| `AppApprovers` | Per-app approver assignments | Core |
| `AuditLogs` | Complete audit trail of all portal actions | Core |
| `ApprovalWorkflows` | Per-app approval workflow configuration | Approval |
| `ApprovalStages` | Ordered stages within a workflow | Approval |
| `WorkflowConditions` | Conditional logic per approval stage | Approval |
| `RequestApprovals` | Per-request approval decisions at each stage | Approval |
| `PortalSettings` | Global portal configuration (singleton) | Config |
| `BrandingSettings` | Portal branding/theming (singleton) | Config |
| `LicenseInfo` | PowerStacks license status (singleton) | Config |
| `VendorLicenseAcceptances` | Vendor license agreement acceptance (singleton) | Config |
| `CategorySettings` | Custom category colors and icons | Config |
| `TermsOfService` | Terms of Service versions | Compliance |
| `TermsAcceptances` | User acceptance records for TOS | Compliance |
| `PackagingJobs` | WinGet-to-Intune packaging pipeline jobs | Packaging |
| `WingetPackageCache` | Cached WinGet package catalog | Packaging |
| `AppVersionHistories` | App version tracking for updates | Updates |
| `BotConversationReferences` | Teams bot conversation references for proactive messaging | Notifications |

#### Core Tables

```sql
-- Apps: Intune application catalog
CREATE TABLE Apps (
    Id NVARCHAR(450) PRIMARY KEY,
    IntuneAppId NVARCHAR(255) NOT NULL UNIQUE,
    DisplayName NVARCHAR(255) NOT NULL,
    Description NVARCHAR(MAX),
    Publisher NVARCHAR(255),
    Version NVARCHAR(50),
    Architecture NVARCHAR(MAX),           -- x64, x86, arm64
    Locale NVARCHAR(MAX),                 -- e.g. en-US
    IconUrl NVARCHAR(MAX),
    IconBase64 NVARCHAR(MAX),
    IconContentType NVARCHAR(MAX),
    Category NVARCHAR(100),
    Cost DECIMAL(18,2),
    IsVisible BIT NOT NULL DEFAULT 1,
    RequiresApproval BIT NOT NULL DEFAULT 0,
    IsFeatured BIT NOT NULL DEFAULT 0,
    AssignmentType INT NOT NULL DEFAULT 0,       -- 0=User, 1=Device
    TargetGroupId NVARCHAR(100),
    TargetGroupName NVARCHAR(255),
    AzureADGroupId NVARCHAR(MAX),
    AzureADGroupName NVARCHAR(MAX),
    IntuneAppType NVARCHAR(MAX),                 -- e.g. win32LobApp
    Platform INT NOT NULL DEFAULT 0,              -- 0=Unknown, 1=Windows, 2=iOS, 3=Android, 4=macOS
    IsSupportedAppType BIT NOT NULL DEFAULT 0,
    IntuneAssignmentId NVARCHAR(MAX),
    FilterId NVARCHAR(MAX),
    FilterName NVARCHAR(MAX),
    FilterType INT NOT NULL DEFAULT 0,
    InstallBehavior NVARCHAR(MAX) DEFAULT 'System',
    DeviceRestartBehavior NVARCHAR(MAX) DEFAULT 'BasedOnReturnCode',
    EndUserNotification NVARCHAR(MAX) DEFAULT 'ShowAll',
    AllowAvailableUninstall BIT NOT NULL DEFAULT 1,
    DeliveryOptimizationPriority INT NOT NULL DEFAULT 0,
    RestartGracePeriodEnabled BIT NOT NULL DEFAULT 1,
    RestartGracePeriodMinutes INT NOT NULL DEFAULT 1440,
    RestartCountdownMinutes INT NOT NULL DEFAULT 15,
    AllowSnoozeRestart BIT NOT NULL DEFAULT 1,
    SnoozeDurationMinutes INT NOT NULL DEFAULT 240,
    -- Mobile app fields
    IsVppApp BIT NOT NULL DEFAULT 0,
    VppTokenId NVARCHAR(MAX),
    VppLicenseCount INT,
    VppUsedLicenseCount INT,
    BundleId NVARCHAR(MAX),
    MinimumOsVersion NVARCHAR(MAX),
    StoreUrl NVARCHAR(MAX),
    -- Update tracking
    SourceWingetPackageId NVARCHAR(255),
    LatestAvailableVersion NVARCHAR(MAX),
    LastUpdateCheckDate DATETIME2,
    UpdateAvailable BIT NOT NULL DEFAULT 0,
    -- Acknowledgment
    RequiresAcknowledgment BIT NOT NULL DEFAULT 0,
    AcknowledgmentText NVARCHAR(MAX),
    CreatedDate DATETIME2 NOT NULL,
    LastSyncDate DATETIME2 NOT NULL
);

-- AppRequests: User app requests
CREATE TABLE AppRequests (
    Id NVARCHAR(450) PRIMARY KEY,
    AppId NVARCHAR(450) NOT NULL,
    UserId NVARCHAR(450) NOT NULL,
    UserEmail NVARCHAR(255) NOT NULL,
    UserDisplayName NVARCHAR(255) NOT NULL,
    DeviceId NVARCHAR(MAX),
    DeviceName NVARCHAR(MAX),
    Status INT NOT NULL DEFAULT 0,           -- 0=Pending, 1=Approved, 2=Rejected, 3=Completed, 4=Cancelled
    Justification NVARCHAR(1000),
    RequestedDate DATETIME2 NOT NULL,
    ReviewedDate DATETIME2,
    ReviewedBy NVARCHAR(MAX),
    ReviewerEmail NVARCHAR(MAX),
    ReviewComments NVARCHAR(1000),
    RejectedReason NVARCHAR(MAX),
    CompletedDate DATETIME2,
    CurrentApprovalStage INT NOT NULL DEFAULT 0,
    LastReminderSentDate DATETIME2,
    ReminderCount INT NOT NULL DEFAULT 0,
    -- On-behalf-of fields
    RequestedOnBehalfOfUserId NVARCHAR(MAX),
    RequestedOnBehalfOfEmail NVARCHAR(MAX),
    RequestedOnBehalfOfDisplayName NVARCHAR(MAX),
    -- Install status tracking
    InstallStatus INT NOT NULL DEFAULT 0,
    InstallStatusLastChecked DATETIME2,
    InstallStatusErrorMessage NVARCHAR(MAX),
    IntuneAssignmentId NVARCHAR(MAX),
    -- Escalation
    LastEscalationDate DATETIME2,
    EscalationCount INT NOT NULL DEFAULT 0,
    -- Actionable emails
    ActionToken NVARCHAR(MAX),
    FOREIGN KEY (AppId) REFERENCES Apps(Id) ON DELETE RESTRICT
);

-- AppApprovers: Per-app approver assignments
CREATE TABLE AppApprovers (
    Id NVARCHAR(450) PRIMARY KEY,
    AppId NVARCHAR(450) NOT NULL,
    UserId NVARCHAR(450) NOT NULL,
    UserEmail NVARCHAR(255) NOT NULL,
    UserDisplayName NVARCHAR(255) NOT NULL,
    ApproverType INT NOT NULL DEFAULT 0,     -- 0=Designated, 1=Manager
    CreatedDate DATETIME2 NOT NULL,
    FOREIGN KEY (AppId) REFERENCES Apps(Id) ON DELETE CASCADE,
    UNIQUE (AppId, UserId)
);

-- AuditLogs: Complete audit trail
CREATE TABLE AuditLogs (
    Id NVARCHAR(450) PRIMARY KEY,
    UserId NVARCHAR(450) NOT NULL,
    UserEmail NVARCHAR(255) NOT NULL,
    Action NVARCHAR(100) NOT NULL,
    EntityType NVARCHAR(100) NOT NULL,
    EntityId NVARCHAR(450) NOT NULL,
    Details NVARCHAR(MAX),                   -- JSON payload
    Timestamp DATETIME2 NOT NULL,
    IpAddress NVARCHAR(50) NOT NULL
);
```

#### Approval Workflow Tables

```sql
-- ApprovalWorkflows: Per-app workflow configuration
CREATE TABLE ApprovalWorkflows (
    Id NVARCHAR(450) PRIMARY KEY,
    AppId NVARCHAR(450) NOT NULL UNIQUE,     -- One workflow per app
    RequireManagerApproval BIT NOT NULL DEFAULT 0,
    WorkflowType INT NOT NULL DEFAULT 0,     -- 0=Pooled, 1=Linear
    CreatedDate DATETIME2 NOT NULL,
    ModifiedDate DATETIME2,
    ModifiedBy NVARCHAR(255),
    FOREIGN KEY (AppId) REFERENCES Apps(Id) ON DELETE CASCADE
);

-- ApprovalStages: Ordered stages within a workflow
CREATE TABLE ApprovalStages (
    Id NVARCHAR(450) PRIMARY KEY,
    WorkflowId NVARCHAR(450) NOT NULL,
    StageOrder INT NOT NULL,                 -- 1-based ordering
    -- Linear workflow fields
    ApproverUserId NVARCHAR(MAX),
    ApproverEmail NVARCHAR(255),
    ApproverDisplayName NVARCHAR(255),
    -- Pooled workflow fields
    ApproverGroupId NVARCHAR(MAX),
    ApproverGroupName NVARCHAR(255),
    -- Conditional logic
    HasConditions BIT NOT NULL DEFAULT 0,
    ConditionDescription NVARCHAR(500),
    CreatedDate DATETIME2 NOT NULL,
    FOREIGN KEY (WorkflowId) REFERENCES ApprovalWorkflows(Id) ON DELETE CASCADE
);

-- WorkflowConditions: Conditional logic per approval stage
CREATE TABLE WorkflowConditions (
    Id NVARCHAR(450) PRIMARY KEY,
    StageId NVARCHAR(100) NOT NULL,
    Type NVARCHAR(50) NOT NULL DEFAULT 'Always',     -- Always, Cost, Category, Platform, Publisher, Department
    Operator NVARCHAR(50) NOT NULL DEFAULT 'Equals',  -- Equals, NotEquals, GreaterThan, etc.
    Value NVARCHAR(500),
    Value2 NVARCHAR(500),                             -- For range conditions
    LogicalOperator NVARCHAR(10) DEFAULT 'And',       -- And, Or
    FOREIGN KEY (StageId) REFERENCES ApprovalStages(Id) ON DELETE CASCADE
);

-- RequestApprovals: Per-request approval decisions at each stage
CREATE TABLE RequestApprovals (
    Id NVARCHAR(450) PRIMARY KEY,
    RequestId NVARCHAR(450) NOT NULL,
    StageOrder INT NOT NULL,                 -- 0=manager, 1+=workflow stages
    StageType INT NOT NULL,                  -- 0=Manager, 1=Linear, 2=Pooled
    Status INT NOT NULL DEFAULT 0,           -- 0=Pending, 1=Approved, 2=Rejected, 3=Skipped
    ApprovedByUserId NVARCHAR(MAX),
    ApprovedByEmail NVARCHAR(255),
    ApprovedByDisplayName NVARCHAR(255),
    DecisionDate DATETIME2,
    Comments NVARCHAR(1000),
    -- Pooled stage fields
    GroupId NVARCHAR(MAX),
    GroupName NVARCHAR(255),
    -- Linear stage fields
    ExpectedApproverUserId NVARCHAR(MAX),
    ExpectedApproverEmail NVARCHAR(255),
    CreatedDate DATETIME2 NOT NULL,
    FOREIGN KEY (RequestId) REFERENCES AppRequests(Id) ON DELETE CASCADE
);
```

#### Configuration Tables (Singletons)

```sql
-- PortalSettings: Global portal configuration (always Id=1)
CREATE TABLE PortalSettings (
    Id INT PRIMARY KEY,                      -- Always 1
    -- Email notification settings
    EmailSendAsUserId NVARCHAR(100),
    EmailFromAddress NVARCHAR(255),
    EmailPortalUrl NVARCHAR(500),
    EmailNotificationsEnabled BIT NOT NULL DEFAULT 0,
    EmailNotifyOnSubmitted BIT NOT NULL DEFAULT 1,
    EmailNotifyOnApproved BIT NOT NULL DEFAULT 1,
    EmailNotifyOnRejected BIT NOT NULL DEFAULT 1,
    EmailNotifyOnApprovalRequired BIT NOT NULL DEFAULT 1,
    EmailNotifyOnInstalled BIT NOT NULL DEFAULT 1,
    EmailNotifyOnAppPublished BIT NOT NULL DEFAULT 1,
    -- Actionable email settings
    ActionableEmailsEnabled BIT NOT NULL DEFAULT 1,
    ActionableEmailApiBaseUrl NVARCHAR(MAX),
    ActionableEmailOriginatorId NVARCHAR(MAX),
    -- Teams Bot notification settings
    TeamsBotEnabled BIT NOT NULL DEFAULT 0,
    TeamsBotAppId NVARCHAR(MAX),
    TeamsBotNotifyOnApprovalRequired BIT NOT NULL DEFAULT 1,
    TeamsBotNotifyOnApproved BIT NOT NULL DEFAULT 1,
    TeamsBotNotifyOnRejected BIT NOT NULL DEFAULT 1,
    TeamsBotNotifyOnInstalled BIT NOT NULL DEFAULT 1,
    TeamsBotNotifyOnAppPublished BIT NOT NULL DEFAULT 1,
    -- Approval reminders
    ApprovalRemindersEnabled BIT NOT NULL DEFAULT 0,
    ReminderIntervalDays INT NOT NULL DEFAULT 2,
    MaxReminderCount INT NOT NULL DEFAULT 3,
    -- Group authorization
    AdminGroupId NVARCHAR(100),
    AdminGroupName NVARCHAR(255),
    ApproverGroupId NVARCHAR(100),
    ApproverGroupName NVARCHAR(255),
    UserAccessGroupId NVARCHAR(MAX),
    UserAccessGroupName NVARCHAR(MAX),
    RequestOnBehalfGroupId NVARCHAR(MAX),
    RequestOnBehalfGroupName NVARCHAR(MAX),
    -- General settings
    RequireManagerApproval BIT NOT NULL DEFAULT 1,
    AutoCreateGroups BIT NOT NULL DEFAULT 1,
    GroupNamePrefix NVARCHAR(MAX) DEFAULT 'AppPortal-',
    -- Display settings
    MaxFeaturedApps INT NOT NULL DEFAULT 0,
    DarkModeEnabled BIT NOT NULL DEFAULT 0,
    HeroAppId NVARCHAR(MAX),
    -- WinGet settings
    WingetRepoUrl NVARCHAR(MAX),
    GitHubPersonalAccessToken NVARCHAR(MAX),
    WingetUpdateCheckEnabled BIT NOT NULL DEFAULT 0,
    WingetUpdateCheckIntervalHours INT NOT NULL DEFAULT 24,
    WingetUpdateNotificationsEnabled BIT NOT NULL DEFAULT 0,
    -- Auto-update settings
    AutoCheckUpdates BIT NOT NULL DEFAULT 1,
    ShowUpdateNotification BIT NOT NULL DEFAULT 1,
    ReleaseChannel NVARCHAR(MAX) DEFAULT 'Latest',
    -- Reports & ROI settings
    HelpDeskCostPerTicket DECIMAL(18,2) NOT NULL DEFAULT 22.00,
    CurrencyCode NVARCHAR(10) DEFAULT 'USD',
    CurrencySymbol NVARCHAR(10) DEFAULT '$',
    -- SLA settings
    SlaTrackingEnabled BIT NOT NULL DEFAULT 0,
    SlaTargetApprovalHours INT NOT NULL DEFAULT 24,
    SlaWarningThresholdHours INT NOT NULL DEFAULT 18,
    SlaSendBreachAlerts BIT NOT NULL DEFAULT 1,
    SlaSendWarningAlerts BIT NOT NULL DEFAULT 1,
    SlaBusinessHoursOnly BIT NOT NULL DEFAULT 0,
    SlaBusinessDayStartHour INT NOT NULL DEFAULT 9,
    SlaBusinessDayEndHour INT NOT NULL DEFAULT 17,
    -- Request escalation
    RequestEscalationEnabled BIT NOT NULL DEFAULT 0,
    EscalationThresholdHours INT NOT NULL DEFAULT 48,
    EscalationRecipientEmail NVARCHAR(MAX),
    EscalationRecipientGroupId NVARCHAR(MAX),
    EscalationRecipientGroupName NVARCHAR(MAX),
    -- Company info
    CompanyName NVARCHAR(MAX),
    CompanyLogoBase64 NVARCHAR(MAX),
    CompanyLogoContentType NVARCHAR(MAX),
    SupportEmail NVARCHAR(MAX),
    SupportPhone NVARCHAR(MAX),
    -- Intune sync
    LastIntuneSyncDate DATETIME2,
    -- Metadata
    LastModified DATETIME2 NOT NULL,
    ModifiedBy NVARCHAR(255)
);

-- BrandingSettings: Portal theming (always Id=1)
CREATE TABLE BrandingSettings (
    Id INT PRIMARY KEY,                      -- Always 1
    LogoBase64 NVARCHAR(MAX),
    LogoContentType NVARCHAR(50) DEFAULT 'image/webp',
    FaviconBase64 NVARCHAR(MAX),
    FaviconContentType NVARCHAR(50),
    PrimaryColor NVARCHAR(20) DEFAULT '#0078d4',
    PrimaryColorHover NVARCHAR(20) DEFAULT '#106ebe',
    SecondaryColor NVARCHAR(20) DEFAULT '#6c757d',
    HeaderTextColor NVARCHAR(20) DEFAULT '#ffffff',
    BackgroundColor NVARCHAR(20) DEFAULT '#f5f5f5',
    CardBackgroundColor NVARCHAR(20) DEFAULT '#ffffff',
    PortalTitle NVARCHAR(100) DEFAULT 'App Portal for Intune',
    Tagline NVARCHAR(MAX),
    WelcomeMessage NVARCHAR(500),
    FooterText NVARCHAR(500),
    SupportContactInfo NVARCHAR(500),
    LastModified DATETIME2 NOT NULL,
    ModifiedBy NVARCHAR(255)
);

-- LicenseInfo: PowerStacks license status (always Id=1)
CREATE TABLE LicenseInfo (
    Id INT PRIMARY KEY,                      -- Always 1
    Status NVARCHAR(50) DEFAULT 'unknown',
    Enabled BIT NOT NULL DEFAULT 0,
    LicenseType NVARCHAR(50),
    StartDate DATETIME2,
    EndDate DATETIME2,
    LicensedDeviceCount INT,
    CurrentDeviceCount INT NOT NULL DEFAULT 0,
    AuthorizedTenants NVARCHAR(MAX),         -- JSON array
    TenantId NVARCHAR(100),
    IsTenantAuthorized BIT NOT NULL DEFAULT 0,
    IsWithinDeviceLimit BIT NOT NULL DEFAULT 1,
    LastValidated DATETIME2,
    LastDeviceCountUpdate DATETIME2,
    ErrorMessage NVARCHAR(500),
    ApiKey NVARCHAR(MAX)
);

-- VendorLicenseAcceptances: Vendor license agreement (always Id=1)
CREATE TABLE VendorLicenseAcceptances (
    Id INT PRIMARY KEY,                      -- Always 1
    AcceptedVersion NVARCHAR(20) NOT NULL,
    AcceptedByUserId NVARCHAR(100) NOT NULL,
    AcceptedByEmail NVARCHAR(255) NOT NULL,
    AcceptedByDisplayName NVARCHAR(255),
    AcceptedDate DATETIME2 NOT NULL,
    IpAddress NVARCHAR(50),
    UserAgent NVARCHAR(500),
    TenantId NVARCHAR(100)
);

-- CategorySettings: Custom category colors and icons
CREATE TABLE CategorySettings (
    Id INT PRIMARY KEY IDENTITY,
    CategoryName NVARCHAR(100) NOT NULL UNIQUE,
    Color NVARCHAR(20),                      -- Hex e.g. '#0078d4'
    Icon NVARCHAR(255)
);
```

#### Compliance Tables

```sql
-- TermsOfService: TOS versions
CREATE TABLE TermsOfService (
    Id NVARCHAR(450) PRIMARY KEY,
    Version NVARCHAR(20) NOT NULL DEFAULT '1.0',
    Title NVARCHAR(255) NOT NULL,
    Content NVARCHAR(MAX) NOT NULL,
    ChangesSummary NVARCHAR(1000),
    IsActive BIT NOT NULL DEFAULT 0,
    IsRequired BIT NOT NULL DEFAULT 1,
    CreatedDate DATETIME2 NOT NULL,
    PublishedDate DATETIME2,
    CreatedBy NVARCHAR(255),
    ModifiedBy NVARCHAR(255),
    ModifiedDate DATETIME2
);

-- TermsAcceptances: User acceptance records
CREATE TABLE TermsAcceptances (
    Id NVARCHAR(450) PRIMARY KEY,
    TermsOfServiceId NVARCHAR(450) NOT NULL,
    UserId NVARCHAR(100) NOT NULL,
    UserEmail NVARCHAR(255) NOT NULL,
    UserDisplayName NVARCHAR(255),
    AcceptedDate DATETIME2 NOT NULL,
    IpAddress NVARCHAR(50),
    UserAgent NVARCHAR(500),
    FOREIGN KEY (TermsOfServiceId) REFERENCES TermsOfService(Id) ON DELETE CASCADE,
    UNIQUE (TermsOfServiceId, UserId)
);
```

#### Packaging & Update Tables

```sql
-- PackagingJobs: WinGet-to-Intune packaging pipeline
CREATE TABLE PackagingJobs (
    Id INT PRIMARY KEY IDENTITY,
    JobId NVARCHAR(50) NOT NULL UNIQUE,
    WingetPackageId NVARCHAR(255) NOT NULL,
    PackageName NVARCHAR(255) NOT NULL,
    PackageVersion NVARCHAR(50),
    Publisher NVARCHAR(255),
    Description NVARCHAR(MAX),
    IconBase64 NVARCHAR(MAX),
    Status NVARCHAR(50) NOT NULL DEFAULT 'Pending',
    IntunewinBlobUrl NVARCHAR(500),
    IntuneAppId NVARCHAR(100),
    ErrorMessage NVARCHAR(MAX),
    CreatedBy NVARCHAR(255) NOT NULL,
    CreatedAt DATETIME2 NOT NULL,
    CompletedAt DATETIME2,
    StatusMessage NVARCHAR(500),
    PackageSize BIGINT,
    Architecture NVARCHAR(MAX),
    Locale NVARCHAR(MAX),
    InstallCommand NVARCHAR(1000),
    UninstallCommand NVARCHAR(1000),
    SilentInstallSwitch NVARCHAR(MAX),
    AppsAndFeaturesEntriesJson NVARCHAR(MAX),
    UsePsadt BIT NOT NULL DEFAULT 1
);

-- WingetPackageCache: Cached WinGet catalog
CREATE TABLE WingetPackageCache (
    Id INT PRIMARY KEY IDENTITY,
    PackageId NVARCHAR(MAX) NOT NULL,
    Name NVARCHAR(MAX) NOT NULL,
    Publisher NVARCHAR(MAX) NOT NULL,
    LatestVersion NVARCHAR(MAX) NOT NULL,
    Description NVARCHAR(MAX),
    Homepage NVARCHAR(MAX),
    License NVARCHAR(MAX),
    IconUrl NVARCHAR(MAX),
    TagsJson NVARCHAR(MAX),                  -- JSON array
    InstallersJson NVARCHAR(MAX),            -- JSON array
    CreatedAt DATETIME2 NOT NULL,
    LastUpdated DATETIME2 NOT NULL,
    ExpiresAt DATETIME2 NOT NULL,
    PopularityRank INT,
    Category NVARCHAR(MAX)
);

-- AppVersionHistories: Version tracking for app updates
CREATE TABLE AppVersionHistories (
    Id NVARCHAR(450) PRIMARY KEY,
    AppId NVARCHAR(100) NOT NULL,
    Version NVARCHAR(50) NOT NULL,
    RecordedAt DATETIME2 NOT NULL,
    UpdatedByUserId NVARCHAR(100),
    UpdatedByUserName NVARCHAR(255),
    WingetPackageId NVARCHAR(255),
    InstallerUrl NVARCHAR(1000),
    InstallerHash NVARCHAR(100),             -- SHA256
    IntuneAppId NVARCHAR(100),
    ReleaseNotes NVARCHAR(2000),
    IsRollbackAvailable BIT NOT NULL DEFAULT 1,
    InstallerSizeBytes BIGINT,
    Status NVARCHAR(50) DEFAULT 'Archived',  -- Current, Archived, RolledBack
    FOREIGN KEY (AppId) REFERENCES Apps(Id) ON DELETE CASCADE,
    UNIQUE (AppId, Version)
);
```

#### Notification Tables

```sql
-- BotConversationReferences: Teams bot proactive messaging references
CREATE TABLE BotConversationReferences (
    Id INT PRIMARY KEY IDENTITY,
    UserId NVARCHAR(100) NOT NULL UNIQUE,    -- Entra ID object ID
    UserEmail NVARCHAR(255),
    ConversationId NVARCHAR(255) NOT NULL,
    ServiceUrl NVARCHAR(500) NOT NULL,
    ConversationReferenceJson NVARCHAR(MAX) NOT NULL,
    InstalledDate DATETIME2 NOT NULL,
    LastActivityDate DATETIME2 NOT NULL
);
```

### 4. Microsoft Graph API Integration

**Purpose**:
- Retrieve apps from Intune
- Manage Entra ID groups
- Add/remove users and devices from groups
- Get user information and manager hierarchy

**Permissions Required**:
- `DeviceManagementApps.Read.All`
- `DeviceManagementApps.ReadWrite.All`
- `DeviceManagementManagedDevices.Read.All` (for device count)
- `Group.ReadWrite.All`
- `User.Read.All`
- `Directory.Read.All`
- `Mail.Send` (optional — for email notifications)

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

### 5. Azure Key Vault

**Purpose**: Secure storage for application secrets

**Secrets Stored**:
- `AzureAdClientSecret` — Entra ID client secret
- `SqlConnectionString` — SQL Database connection string
- `StorageConnectionString` — Azure Storage connection string

**Access**: App Service Managed Identity with Get and List permissions only. Secrets are referenced via Key Vault references in App Settings (e.g., `@Microsoft.KeyVault(SecretUri=...)`).

### 6. Azure Storage Account

**Purpose**: Queue and blob storage for the cloud packaging pipeline

**Containers/Queues**:
- `packaging-jobs` queue — Job messages for the packaging agent
- `intunewin-packages` blob container — Packaged `.intunewin` files and PSADT templates

### 7. Azure Bot (Optional)

**Purpose**: Send personal Teams notifications to users via Bot Framework proactive messaging

**Configuration**:
- Reuses the Backend API Entra ID app registration (same Client ID/Secret)
- Messaging endpoint: `https://{app-url}/api/messages`
- SingleTenant, Teams channel enabled
- No additional Microsoft Graph permissions required

**Key Files**:
- [/src/AppRequestPortal.API/Bot/AppRequestBot.cs](../src/AppRequestPortal.API/Bot/AppRequestBot.cs) — Handles install/uninstall events
- [/src/AppRequestPortal.API/Controllers/BotController.cs](../src/AppRequestPortal.API/Controllers/BotController.cs) — Bot messaging endpoint
- [/src/AppRequestPortal.Infrastructure/Services/TeamsBotService.cs](../src/AppRequestPortal.Infrastructure/Services/TeamsBotService.cs) — Proactive notification service

## Data Flow

### User Request Flow

```
1. User browses apps
   Frontend → API → Database → Apps list

2. User submits request
   Frontend → API → Database (save request)
                  → Email service (notify approvers)
                  → Teams bot (notify users)

3. Approver reviews request
   Frontend → API → Database (update status)
                  → Email service (notify requester)
                  → Teams bot (notify users)

4. System processes approved request
   API → Graph API (create/get AD group)
       → Graph API (add user/device to group)
       → Intune (app deployment triggered automatically)
       → Database (update request status)
       → Email service (notify requester)
       → Teams bot (notify users)
```

### Notification Services

The portal uses two notification channels:

| Service | Purpose | Technology |
|---------|---------|------------|
| **EmailNotificationService** | Direct user notifications | Microsoft Graph Mail.Send API |
| **TeamsBotService** | Personal Teams notifications | Bot Framework proactive messaging with Adaptive Cards |

Both services are optional and can be enabled/disabled independently in the Admin Communications tab. The Teams Bot sends personal 1:1 messages to users via stored conversation references (the bot must be pre-installed for users via Teams Admin Center setup policies).

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
2. MSAL redirects to Entra ID login
3. User authenticates
4. Entra ID returns ID token and access token
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
2. **JWT Authentication**: Stateless authentication with Entra ID
3. **RBAC**: Role-based access control for granular permissions
4. **Conditional Access**: Integration with Entra ID Conditional Access policies
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
2. **Access Groups** - Configure admin and approver Entra ID groups
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
