# Approval Workflows

This document describes how to configure flexible approval workflows for app requests.

## Overview

Each app can be configured to either require approval or not. For apps that require approval, you can configure a custom workflow that determines how requests are processed.

### Requires Approval Setting

In the Admin Dashboard's **App Management** tab, each app has an **Approval** toggle:

- **No**: Requests are auto-approved immediately. The user/device is added to the target AAD group without any approval steps.
- **Yes**: Requests go through the configured approval workflow before completion.

Use auto-approval for low-risk apps like free productivity tools. Use approval workflows for licensed software, admin tools, or sensitive applications.

### Workflow Types

For apps that require approval, the system supports two types of workflows:

- **Linear**: Specific users must approve in a defined sequence
- **Pooled**: Any member of specified Azure AD groups can approve at each stage

Additionally, you can require manager approval before the workflow stages begin.

## Workflow Configuration

### Settings Per App

| Setting | Description |
|---------|-------------|
| `requireManagerApproval` | If `true`, the requesting user's manager must approve first before the workflow stages |
| `workflowType` | Either `Linear` or `Pooled` |
| `stages` | Ordered list of approval stages |

### Linear Workflow Stages

For Linear workflows, each stage specifies a specific person who must approve:

```json
{
  "stageOrder": 1,
  "approverUserId": "user-azure-ad-object-id",
  "approverEmail": "approver@company.com",
  "approverDisplayName": "John Smith"
}
```

The request moves through each stage in order. Each specified person must approve before moving to the next stage.

### Pooled Workflow Stages

For Pooled workflows, each stage specifies an Azure AD security group:

```json
{
  "stageOrder": 1,
  "approverGroupId": "group-azure-ad-object-id",
  "approverGroupName": "IT Approvers"
}
```

Any member of the specified group can approve the request at that stage. Once approved, it moves to the next stage.

## Request Flow

### With Manager Approval Required

1. User submits request
2. Request is sent to user's manager (from Azure AD)
3. Manager approves/rejects
4. If approved, request proceeds to workflow stage 1
5. Continue through all workflow stages
6. If all stages approve, request is completed

### Without Manager Approval

1. User submits request
2. Request proceeds directly to workflow stage 1
3. Continue through all workflow stages
4. If all stages approve, request is completed

## API Endpoints

### Get Workflow for an App

```
GET /api/apps/{appId}/approval-workflow
```

Returns the current workflow configuration for the app, or a default empty configuration if none exists.

### Create/Update Workflow

```
PUT /api/apps/{appId}/approval-workflow
```

**Request Body (Linear Example):**

```json
{
  "requireManagerApproval": true,
  "workflowType": "Linear",
  "stages": [
    {
      "stageOrder": 1,
      "approverUserId": "abc-123-user-id",
      "approverEmail": "first.approver@company.com",
      "approverDisplayName": "First Approver"
    },
    {
      "stageOrder": 2,
      "approverUserId": "def-456-user-id",
      "approverEmail": "second.approver@company.com",
      "approverDisplayName": "Second Approver"
    }
  ]
}
```

**Request Body (Pooled Example):**

```json
{
  "requireManagerApproval": false,
  "workflowType": "Pooled",
  "stages": [
    {
      "stageOrder": 1,
      "approverGroupId": "group-id-1",
      "approverGroupName": "Department Approvers"
    },
    {
      "stageOrder": 2,
      "approverGroupId": "group-id-2",
      "approverGroupName": "IT Security Team"
    }
  ]
}
```

### Delete Workflow

```
DELETE /api/apps/{appId}/approval-workflow
```

Removes the workflow configuration. Apps without a workflow will use the default behavior (immediate approval or rejection based on admin settings).

## Setting Up Workflows

### Prerequisites

1. **For Pooled workflows**: Create Azure AD security groups for each approval stage
2. **For Linear workflows**: Know the Azure AD Object IDs of the specific approvers
3. Admin access to the App Request Portal

### Using the UI (Recommended)

1. Navigate to the **Admin Dashboard** > **App Management** tab
2. Find the app you want to configure
3. Set the **Approval** toggle to **Yes** (if not already)
4. Click the **Workflow** button to open the Approval Workflow Editor
5. Configure the workflow:
   - Toggle **Require Manager Approval** if needed
   - Choose the **Workflow Type** (Linear or Pooled)
   - Click **Add Stage** to add approval stages
   - For Linear: Search and select specific users
   - For Pooled: Search and select Azure AD groups
6. Click **Save**

See [ADMIN-GUIDE.md](ADMIN-GUIDE.md) for detailed UI instructions.

### Using the API

You can also configure workflows programmatically using the API endpoints described below.

### Finding Azure AD Object IDs

**For Users (Linear):**
1. Go to Azure Portal > Azure Active Directory > Users
2. Select the user
3. Copy the "Object ID" from the Overview page

**For Groups (Pooled):**
1. Go to Azure Portal > Azure Active Directory > Groups
2. Select the group
3. Copy the "Object ID" from the Overview page

## Example Scenarios

### Scenario 0: No Approval Required (Auto-Approve)

For low-risk apps that don't need any oversight:

1. In Admin Dashboard, find the app
2. Set the **Approval** toggle to **No**

Requests are immediately approved and the user/device is added to the target group. No workflow configuration needed.

**Best for:** Free productivity tools, browser extensions, utilities

### Scenario 1: IT Software - Manager + IT Approval

Apps that require both manager approval and IT department review:

```json
{
  "requireManagerApproval": true,
  "workflowType": "Pooled",
  "stages": [
    {
      "stageOrder": 1,
      "approverGroupId": "it-approvers-group-id",
      "approverGroupName": "IT Software Approvers"
    }
  ]
}
```

### Scenario 2: Sensitive Software - Multi-Level Approval

High-security applications requiring department head and security team approval:

```json
{
  "requireManagerApproval": true,
  "workflowType": "Pooled",
  "stages": [
    {
      "stageOrder": 1,
      "approverGroupId": "dept-heads-group-id",
      "approverGroupName": "Department Heads"
    },
    {
      "stageOrder": 2,
      "approverGroupId": "security-team-group-id",
      "approverGroupName": "Security Team"
    }
  ]
}
```

### Scenario 3: VIP Approval Chain

Specific executives must approve in order:

```json
{
  "requireManagerApproval": false,
  "workflowType": "Linear",
  "stages": [
    {
      "stageOrder": 1,
      "approverUserId": "cto-user-id",
      "approverEmail": "cto@company.com",
      "approverDisplayName": "CTO"
    },
    {
      "stageOrder": 2,
      "approverUserId": "ciso-user-id",
      "approverEmail": "ciso@company.com",
      "approverDisplayName": "CISO"
    }
  ]
}
```

### Scenario 4: Simple Manager Approval Only

Standard software that just needs manager sign-off:

```json
{
  "requireManagerApproval": true,
  "workflowType": "Pooled",
  "stages": []
}
```

## Database Schema

The approval workflow system uses these tables:

- **ApprovalWorkflows**: Stores the workflow configuration per app
- **ApprovalStages**: Stores the ordered stages for each workflow
- **RequestApprovals**: Tracks the approval status at each stage for each request

### Running Migrations

After updating to this version, run migrations to create the new tables:

```powershell
cd src/AppRequestPortal.API
dotnet ef migrations add AddApprovalWorkflows --project ../AppRequestPortal.Infrastructure --startup-project .
dotnet ef database update --project ../AppRequestPortal.Infrastructure --startup-project .
```

## Notifications

The approval workflow integrates with multiple notification channels to keep approvers and requestors informed.

### Email Notifications

Email notifications are sent automatically when:
- A new request is submitted (to approvers)
- A request is approved (to requestor)
- A request is rejected (to requestor)
- A request moves to the next approval stage (to next-stage approvers)

See [SETUP.md](SETUP.md) Step 9 for email configuration.

### Microsoft Teams Notifications

Teams channel notifications provide real-time visibility into approval activity. When enabled:
- **New Request Submitted**: Notification with requestor name, app details, and link to review
- **Request Approved**: Notification with approval details
- **Request Rejected**: Notification with rejection reason

Teams notifications are sent to a channel via Incoming Webhook, so all channel members see the activity. Approvers click through to the portal to take action.

See [ADMIN-GUIDE.md](ADMIN-GUIDE.md#microsoft-teams-notifications) for Teams setup instructions.
