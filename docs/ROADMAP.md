# App Request Portal - Product Roadmap

This document outlines the planned features and current development status of the App Request Portal.

**Last Updated:** February 2026

---

## Current Status

The App Request Portal is a self-service application catalog for Microsoft Intune, allowing users to request apps and administrators to manage approvals and deployments.

### Core Features (Available Now)

- **User Self-Service**
  - Browse app catalog with search and filtering
  - Request apps with justification
  - Track request status
  - View installation instructions

- **Approval Workflows**
  - Multi-stage approval workflows
  - Manager and group-based approval
  - Email notifications for approvers
  - Automatic follow-up reminders

- **Admin Management**
  - Sync apps from Microsoft Intune
  - Configure app visibility and approval requirements
  - Manage approval workflows per app
  - Portal branding and customization
  - Terms of Service management

- **Integrations**
  - Microsoft Entra ID (Azure AD) authentication
  - Microsoft Intune for app deployment
  - Winget package catalog for new app discovery
  - Email notifications via Microsoft Graph

- **Reporting**
  - ROI calculator (help desk cost savings)
  - Request reports by app, user, and approval activity
  - CSV export for all reports
  - Trend charts showing request patterns over time
  - Install status tracking with deployment metrics

- **Winget Auto-Updates**
  - Track apps published from Winget for available updates
  - Admin dashboard showing apps with available updates
  - One-click "Apply Update" to update app versions

---

## Planned Features

### Near-Term (Next 1-2 Sprints)

| Feature | Description |
|---------|-------------|
| **Audit Trail Viewer** | Searchable UI for viewing all portal activity |
| **Enhanced Approval Reports** | Show what was approved (not just who approved) |
| **Duplicate Request Prevention** | Check if user already has app access, show "You have access" badge |
| **Application Insights Dashboard** | Surface key metrics (latency, errors, active users) in Admin panel |

### Medium-Term (3-6 Months)

| Feature | Description |
|---------|-------------|
| **Approval Delegation** | Delegate approvals when out of office |
| **Escalation Rules** | Auto-escalate pending approvals after configurable time |
| **Self-Service Winget Requests** | Allow users to request apps from Winget catalog |
| **PSADT Integration** | PowerShell App Deployment Toolkit wrapper generation for enterprise deployments |
| ~~**Disaster Recovery**~~ | ~~DR documentation, geo-redundancy options, backup/restore procedures~~ (Completed) |
| **Rate Limiting** | API rate limiting for security and performance |

### Long-Term (6+ Months)

| Feature | Description |
|---------|-------------|
| **Microsoft Teams Approvals** | Approve/reject directly from Teams Adaptive Cards |
| **Scheduled Update Checking** | Automatic Winget update detection (daily, weekly, Patch Tuesday) |
| **Ring Deployments** | Staged rollouts for app updates |
| **iOS/Android Store Apps** | Full support for mobile app deployment |
| **ServiceNow Integration** | Bi-directional ticket sync |

---

## Recently Completed

### February 2026

- **Azure Key Vault Integration**
  - All secrets (Azure AD, SQL, Storage) stored in Key Vault
  - Managed Identity access (no plaintext secrets in App Settings)
  - Soft delete enabled for accidental deletion recovery
  - Automatic deployment via ARM template

- **Disaster Recovery & Geo-Redundancy**
  - Geo-redundant storage (GRS) enabled by default
  - Geo-redundant SQL backups for cross-region recovery
  - Comprehensive DR documentation with recovery runbooks
  - Tier 3 (High Availability) manual setup guide

- **Unit Test Infrastructure**
  - xUnit test framework with Moq and FluentAssertions
  - Test data factories for Apps, Users, Requests
  - Mock services for IIntuneService, IAzureADService, INotificationService
  - In-memory database context for isolated testing
  - GitHub Actions CI workflow for automated testing

- **Microsoft Teams Notifications**
  - Send notifications to Teams channels via Incoming Webhooks
  - Rich Adaptive Cards with requestor, app details, and action buttons
  - Configurable notifications for new requests, approvals, and rejections
  - Test button to verify webhook configuration
  - Full documentation in Admin Guide

- **User Details in Reports**
  - Display department and job title from Entra ID in approval reports
  - Show manager information for requestors
  - Enhanced CSV export with user details

- **AppsAndFeaturesEntries from Winget**
  - Fetch ProductCode and UpgradeCode from Winget manifests
  - Improved detection rules for Win32 apps
  - Better uninstall detection using registry entries

- **Unsupported App Type Filtering**
  - Filter out unsupported app types during Intune sync
  - Cleaner app list without Windows MSI LoB, macOS apps, etc.
  - Automatically removes existing unsupported apps

- **Intune-Style Toggle Switches**
  - New pill-shaped toggle buttons with Yes/No labels
  - Consistent styling across the admin interface

- **Performance Improvements**
  - Memory caching for Graph API group membership checks (5-minute cache)
  - Faster loading of packaging jobs page
  - Always-visible cancel button for packaging jobs

- **Intune Install Status Tracking**
  - Real-time deployment status (Pending Install, Installing, Installed, Failed)
  - Background polling service checks Intune every 15 minutes
  - Install status tiles on reports dashboard
  - Error message display for failed installations

- **Dashboard Analytics & Charts**
  - Trend chart showing requests vs completions over time
  - Top requested apps horizontal bar chart
  - Status distribution breakdown
  - Configurable time ranges (7, 14, 30, 90 days)

- **Winget Auto-Updates Phase A+B**
  - Apps published from Winget are tracked for updates
  - "Check for Updates" dashboard in Admin panel
  - One-click "Apply Update" to update app versions

- **Terms of Service**
  - Click-through TOS with version tracking
  - Admin management for creating and activating TOS versions
  - User acceptance tracking for compliance

- **Follow-Up Reminder Emails**
  - Configurable reminder interval and count
  - Background service for automated reminders

- **CSV Export for Reports**
  - Export all report views to CSV
  - Failed requests tile added to dashboard

- **Portal Access Enforcement**
  - Restrict portal access to specific Entra ID group

- **Microsoft-Style Header UI**
  - User profile dropdown with sign-out
  - Settings gear with dark mode toggle

### January 2026

- Winget catalog browser and Win32 packaging
- Microsoft Store-style home page with hero app
- Portal branding customization
- Dark mode with user preference
- Basic reports and ROI calculator

---

## Feature Requests

To request a new feature:
1. Check if it's already on the roadmap above
2. Open a GitHub issue with the "enhancement" label
3. Describe the use case and expected behavior

We review and prioritize feature requests based on customer impact and alignment with our product vision.

---

## Not Currently Planned

| Feature | Reason |
|---------|--------|
| Multi-Tenant Support | Portal is designed for single-tenant deployments |
| Container Deployment | App Service works well with in-app updates |
| On-Premises Deployment | Cloud-first design with Azure dependencies |

---

## Documentation

- [Admin Guide](ADMIN-GUIDE.md) - Complete administration documentation
- [Setup Guide](SETUP.md) - Deployment and configuration instructions
- [User Guide](USER-GUIDE.md) - End user documentation
