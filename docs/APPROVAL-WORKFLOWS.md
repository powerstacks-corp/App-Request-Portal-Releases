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
- **Pooled**: Any member of specified Entra ID groups can approve at each stage

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

For Pooled workflows, each stage specifies an Entra ID security group:

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
2. Request is sent to user's manager (from Entra ID)
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

1. **For Pooled workflows**: Create Entra ID security groups for each approval stage
2. **For Linear workflows**: Know the Entra ID Object IDs of the specific approvers
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
   - For Pooled: Search and select Entra ID groups
6. Click **Save**

See [ADMIN-GUIDE.md](ADMIN-GUIDE.md) for detailed UI instructions.

### Using the API

You can also configure workflows programmatically using the API endpoints described below.

### Finding Entra ID Object IDs

**For Users (Linear):**
1. Go to Azure Portal > Microsoft Entra ID > Users
2. Select the user
3. Copy the "Object ID" from the Overview page

**For Groups (Pooled):**
1. Go to Azure Portal > Microsoft Entra ID > Groups
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

### Actionable Email Messages (v1.6.1)

Approvers can now approve or reject requests **directly from their email** without visiting the portal. When enabled, approval emails include Approve/Reject buttons using Office 365 MessageCard format.

**Features:**
- Approve/Reject buttons embedded in email
- Secure token-based authentication
- Works in Outlook, Outlook Web, and other Office 365 email clients
- Idempotency checks prevent double-approval
- Automatic fallback to standard HTML emails if not configured

**Configuration:**
1. Navigate to **Admin Dashboard** > **Settings** > **Email Notifications**
2. Enable **"Enable actionable email messages"** checkbox
3. Set **"API Base URL for email actions"** (e.g., `https://apprequest.company.com`)
4. Save settings

**How it works:**
1. User submits request
2. Approver receives email with Approve/Reject buttons
3. Approver clicks button in email (no login required)
4. Request is immediately approved/rejected
5. Approver sees confirmation message
6. Requestor receives notification of the decision

**Security:**
- Each email contains a unique, single-use action token
- Tokens expire after 7 days or when request is closed
- All email-based actions are logged in audit trail
- Only intended approver can use the token

---

## Advanced Workflow Features

### Conditional Workflows (v1.6.0)

Configure approval requirements that vary based on app characteristics, requestor department, or other conditions.

**Available Conditions:**
- **Cost Threshold**: Route expensive apps ($500+) to finance team
- **Category**: Different approval chains for "Development Tools" vs "Productivity"
- **Platform**: Separate workflows for Windows, iOS, Android apps
- **Publisher**: Special approval for apps from untrusted publishers
- **Department**: Route requests based on requestor's department (from Entra ID)

**Operators:**
- `Equals`, `NotEquals`
- `GreaterThan`, `LessThan`, `GreaterThanOrEqual`, `LessThanOrEqual`
- `Contains`, `In`

**Logical Combinations:**
- Combine multiple conditions with `AND` or `OR`
- Example: "Cost > $1000 AND Category = Security Tools"

**Example: Route by Cost**

```json
{
  "workflowConditions": [
    {
      "conditionType": "Cost",
      "operator": "GreaterThan",
      "value": "1000",
      "resultWorkflowId": "expensive-app-workflow-id"
    },
    {
      "conditionType": "Cost",
      "operator": "LessThanOrEqual",
      "value": "1000",
      "resultWorkflowId": "standard-app-workflow-id"
    }
  ]
}
```

**Use Cases:**
- **High-Cost Apps**: Apps over $500 require finance + IT approval
- **Security Tools**: Apps in "Security" category need CISO approval
- **Mobile Apps**: iOS/Android apps follow mobile device management approval path
- **Department-Specific**: Engineering department gets expedited approval for dev tools

See [ADMIN-GUIDE.md](ADMIN-GUIDE.md#conditional-workflows) for UI configuration instructions.

---

### Per-App Acknowledgment Requirement (v1.6.1)

Require users to explicitly acknowledge specific terms before requesting certain apps. Useful for:
- Expensive subscriptions ("I understand this is $500/month")
- Compliance requirements ("I agree to complete security training")
- Usage agreements ("I will only use this for business purposes")

**Configuration (Per-App):**
1. Navigate to **Admin Dashboard** > **App Management**
2. Edit the app
3. Enable **"Require acknowledgment before requesting"** checkbox
4. Set **"Acknowledgment Text"** (e.g., "I understand this is a $500/month subscription")
5. Save

**User Experience:**
- When requesting the app, users see a required checkbox with your acknowledgment text
- Submit button is disabled until checkbox is checked
- Acknowledgment is recorded in the request audit trail

**Server-Side Validation:**
- Backend validates that acknowledgment was provided
- Prevents submission without acknowledgment (even if client-side check is bypassed)

---

### SLA Tracking (v1.6.0)

Monitor and enforce Service Level Agreements for request processing times.

**SLA Status Indicators:**
- **On Track**: Request is within SLA target (green)
- **Warning**: Request is approaching SLA breach threshold (yellow)
- **Breached**: Request exceeded SLA target (red)

**Configurable Settings:**
1. **SLA Target Hours**: How long requests should be approved (default: 48 hours)
2. **Warning Threshold**: Percentage of SLA at which to show warning (default: 75%)
3. **Business Hours Only**: Whether to count only business hours (M-F, 9am-5pm)

**Email Alerts:**
- Sent when requests enter Warning state
- Sent when requests enter Breached state
- Configurable recipient(s)

**Reports:**
- SLA compliance rate (% of requests meeting SLA)
- Average approval time
- Median approval time
- List of at-risk requests
- Breach history

See [ADMIN-GUIDE.md](ADMIN-GUIDE.md#sla-tracking) for configuration details.

---

### Auto-Escalation of Stale Requests (v1.6.1)

Automatically escalate approval requests that have been pending for too long, ensuring nothing falls through the cracks.

**How It Works:**
1. Background service checks every 30 minutes for pending requests
2. Requests older than configured threshold (default: 48 hours) are escalated
3. Email notifications sent to designated escalation contacts
4. Re-escalates every 24 hours if still pending
5. Tracks escalation count on each request

**Configuration:**
1. Navigate to **Admin Dashboard** > **Settings** > **Request Escalation**
2. Enable **"Enable automatic escalation of stale requests"**
3. Set **"Escalation Threshold (Hours)"** (default: 48)
4. Set **"Escalation Recipient Email(s)"** (comma-separated) OR **"Escalation Group ID"** (Entra ID group)
5. Save settings

**Escalation Email Contents:**
- **Subject**: "⚠️ ESCALATION: Stale App Request - [App Name]"
- **Details**: Requestor, app, days pending, justification, escalation number
- **Action Link**: Direct link to review the request in portal

**Use Cases:**
- Ensure approvers don't miss urgent requests
- Alert IT managers of bottlenecks
- Provide visibility to executive team
- Meet SLA commitments

**Best Practices:**
- Set threshold to 50% of your SLA target (e.g., 24 hours if SLA is 48 hours)
- Use an Entra ID group for escalation recipients (easier to manage)
- Monitor escalation frequency - high rates may indicate approval process issues

---

## Troubleshooting

### Approver Doesn't Receive Email
- Verify email notifications are enabled in Admin Settings
- Check spam/junk folder
- Verify approver is in the correct group (for Pooled workflows)
- Check Application Insights logs for email send errors

### Actionable Buttons Don't Work in Email
- Verify "API Base URL" is set correctly in Email Notifications settings
- Check that email client supports Office 365 MessageCards (Outlook, OWA)
- Verify action token hasn't expired (7-day limit)
- Check request hasn't already been approved/rejected

### Request Stuck in Approval
- Check if approver is on vacation (use approval delegation when available)
- Verify manager is set in Entra ID (for manager approval workflows)
- Check if request has been escalated (see Request Escalation settings)
- Review audit logs for approval history

### Escalation Emails Not Sending
- Verify Request Escalation is enabled
- Check escalation recipient email/group ID is valid
- Verify background service is running (check Application Insights)
- Confirm threshold has been reached (check request age)

### SLA Status Incorrect
- Verify SLA settings (target hours, business hours mode)
- Check timezone configuration in Admin Settings
- Review SLA calculation in reports dashboard

---

## See Also

- [Admin Guide](ADMIN-GUIDE.md) - Detailed UI instructions
- [Setup Guide](SETUP.md) - Initial configuration
- [User Guide](USER-GUIDE.md) - End-user perspective
