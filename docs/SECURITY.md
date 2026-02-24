# Security Overview

This document provides comprehensive security documentation for the App Request Portal, including all Azure resources created, permissions granted, and security configurations. This is intended to assist security teams with reviews and compliance requirements.

## Table of Contents

1. [Azure Resources Created](#azure-resources-created)
2. [Identity and Access Management](#identity-and-access-management)
3. [Entra ID App Registrations](#azure-ad-app-registrations)
4. [Managed Identity Configuration](#managed-identity-configuration)
5. [Network Security](#network-security)
6. [Data Protection](#data-protection)
7. [Authentication Flow](#authentication-flow)
8. [Authorization Model](#authorization-model)
9. [Audit Logging](#audit-logging)
10. [Security Checklist](#security-checklist)

---

## Azure Resources Created

The ARM template deploys the following resources:

| Resource Type | Name Pattern | Purpose |
|--------------|--------------|---------|
| App Service Plan | `asp-apprequest-{env}` | Hosts the web application |
| App Service | `apprequest-{env}-{unique}` | ASP.NET Core API + React SPA |
| Azure Key Vault | `kv-appreq-{unique}` | Secure secret storage |
| SQL Server | `sql-apprequest-{env}-{unique}` | Database server |
| SQL Database | `AppRequestPortal` | Application data storage |
| Storage Account | `stappreq{unique}` | Queue + blob storage for packaging |
| Application Insights | `ai-apprequest-{env}` | Application monitoring |
| Log Analytics Workspace | `la-apprequest-{env}` | Centralized logging |

### Resource Configuration Details

#### App Service
- **Runtime**: .NET 8.0 on Linux
- **TLS Version**: Minimum TLS 1.2 enforced
- **HTTPS Only**: Enabled (HTTP redirects to HTTPS)
- **FTPS State**: Disabled (no FTP access)
- **Client Affinity**: Disabled (stateless application)
- **Always On**: Enabled (prevents cold starts)

#### SQL Server
- **TLS Version**: Minimum TLS 1.2
- **Firewall**: Only Azure services allowed (0.0.0.0-0.0.0.0)
- **Authentication**: SQL authentication (username/password)

---

## Identity and Access Management

### System-Assigned Managed Identity

The App Service is configured with a **System-Assigned Managed Identity**. This identity is automatically created and managed by Azure.

```json
"identity": {
  "type": "SystemAssigned"
}
```

**Purpose**: The Managed Identity allows the application to authenticate to Azure services without storing credentials in code or configuration.

### Role Assignments

The following permissions are granted to the Managed Identity:

| Permission Type | Scope | Purpose |
|----------------|-------|---------|
| Website Contributor Role | App Service | Enables in-app updates via Azure Resource Manager |
| Key Vault Access Policy (Get, List Secrets) | Key Vault | Read application secrets from Key Vault |

**Why Website Contributor?**

The in-app update feature allows administrators to update the application from within the portal. To do this, the application must be able to modify its own `WEBSITE_RUN_FROM_PACKAGE` app setting. The Website Contributor role grants this permission.

**Why Key Vault Access?**

All sensitive secrets (Entra ID client secret, SQL connection string, storage connection string) are stored in Azure Key Vault. The Managed Identity needs read access to retrieve these secrets at runtime.

**Scope Limitation**: All permissions are scoped to the minimum required resources (not the resource group or subscription), following the principle of least privilege.

```json
{
  "type": "Microsoft.Authorization/roleAssignments",
  "scope": "[concat('Microsoft.Web/sites/', variables('appName'))]",
  "properties": {
    "roleDefinitionId": "[subscriptionResourceId('Microsoft.Authorization/roleDefinitions', 'de139f84-1756-47ae-9be6-808fbbe84772')]",
    "principalId": "[reference(resourceId('Microsoft.Web/sites', variables('appName')), '2022-09-01', 'full').identity.principalId]",
    "principalType": "ServicePrincipal"
  }
}
```

---

## Entra ID App Registrations

Two Entra ID App Registrations are required:

### 1. Backend API Application (Confidential Client)

**Application Type**: Web application / API

**Authentication**:
- Client ID (stored in App Settings)
- Client Secret (stored in App Settings)

**API Permissions (Application Permissions)**:

| API | Permission | Type | Purpose |
|-----|-----------|------|---------|
| Microsoft Graph | `DeviceManagementApps.Read.All` | Application | Read Intune apps |
| Microsoft Graph | `DeviceManagementApps.ReadWrite.All` | Application | Create/update Intune apps |
| Microsoft Graph | `DeviceManagementManagedDevices.Read.All` | Application | Count managed devices for licensing |
| Microsoft Graph | `Group.ReadWrite.All` | Application | Create and manage Entra ID groups |
| Microsoft Graph | `User.Read.All` | Application | Read user profiles, managers, and group memberships |
| Microsoft Graph | `Directory.Read.All` | Application | Read directory data |
| Microsoft Graph | `Mail.Send` | Application | Send email notifications (optional) |

**Admin Consent**: Required for all application permissions.

**Token Configuration**:
- Access tokens issued for: `api://{client-id}`
- Scopes exposed: `access_as_user`

### 2. Frontend SPA Application (Public Client)

**Application Type**: Single-page application (SPA)

**Authentication**:
- Client ID only (no secret - public client)
- MSAL.js with PKCE flow

**API Permissions (Delegated Permissions)**:

| API | Permission | Type | Purpose |
|-----|-----------|------|---------|
| Microsoft Graph | `User.Read` | Delegated | Read signed-in user's profile and photo |
| Backend API | `access_as_user` | Delegated | Call backend API on behalf of user |

**Redirect URIs**:
- `https://{app-name}.azurewebsites.net/`
- `https://{custom-domain}/` (if custom domain configured)

---

## Managed Identity Configuration

### How the Managed Identity Works

1. **Creation**: When the ARM template deploys, Azure automatically creates a service principal in Entra ID for the App Service.

2. **Environment Variables**: Azure injects these environment variables into the application:
   - `MSI_ENDPOINT` or `IDENTITY_ENDPOINT`: Token endpoint URL
   - `IDENTITY_HEADER`: Secret header for authentication
   - `WEBSITE_SITE_NAME`: App Service name
   - `WEBSITE_RESOURCE_GROUP`: Resource group name
   - `WEBSITE_OWNER_NAME`: Subscription ID

3. **Token Acquisition**: The application uses `DefaultAzureCredential` from Azure.Identity SDK to acquire tokens:
   ```csharp
   var credential = new DefaultAzureCredential();
   var armClient = new ArmClient(credential);
   ```

4. **Authorization**: The Managed Identity authenticates to Azure Resource Manager and is authorized via the Website Contributor role assignment.

### What the Managed Identity CAN Do

- Read and update the App Service's own configuration settings
- Read the App Service's properties
- Restart the App Service
- Read secrets from the provisioned Key Vault (Get, List permissions only)

### What the Managed Identity CANNOT Do

- Access other resources in the subscription
- Modify other App Services
- Write, delete, or manage secrets in Key Vault
- Access SQL Database directly (uses SQL authentication via connection string from Key Vault)
- Deploy or delete resources

---

## Network Security

### Inbound Traffic

| Protocol | Port | Source | Destination | Purpose |
|----------|------|--------|-------------|---------|
| HTTPS | 443 | Internet | App Service | User access |
| HTTPS | 443 | App Service | Azure SQL | Database connections |
| HTTPS | 443 | App Service | Microsoft Graph | API calls |

### App Service Network Configuration

- **Public Access**: Enabled (internet-accessible)
- **TLS 1.2**: Minimum version enforced
- **HTTPS Only**: HTTP requests redirected to HTTPS
- **IP Restrictions**: None by default (can be configured)

### SQL Server Firewall

The SQL Server is configured with the following security controls:

- **Azure Services**: Allowed (firewall rule `AllowAllWindowsAzureIps`)
  - This allows the App Service to connect to the database
  - Other Azure services in the subscription cannot access unless explicitly allowed
- **Public IP Access**: Denied by default
  - No client IP addresses are whitelisted
  - Database is not accessible from the internet
- **Private Endpoint**: Not configured by default (can be added for enhanced security)

**Important:** The database is NOT accessible from the public internet. Only Azure services (like the App Service) can connect.

To verify SQL firewall rules:
```bash
az sql server firewall-rule list --server <server-name> --resource-group <rg-name>
```

To add a client IP for temporary admin access:
```bash
az sql server firewall-rule create --server <server-name> --resource-group <rg-name> \
  --name "AdminAccess" --start-ip-address <your-ip> --end-ip-address <your-ip>
```

**Remember to remove temporary rules after use.**

### Recommendations for Enhanced Security

1. **Virtual Network Integration**: Deploy App Service into a VNet
2. **Private Endpoints**: Use private endpoints for SQL and storage
3. **IP Restrictions**: Limit access to known IP ranges
4. **Azure Front Door**: Add WAF protection

---

## Data Protection

### Data at Rest

| Data Type | Storage Location | Encryption |
|-----------|-----------------|------------|
| Application data | Azure SQL Database | TDE (Transparent Data Encryption) |
| User settings | Azure SQL Database | TDE |
| Branding assets | Azure SQL Database | TDE |
| Logs | Log Analytics | Azure-managed encryption |

### Data in Transit

- All communication uses TLS 1.2+
- HTTPS enforced at App Service level
- SQL connections use encrypted connections

### Sensitive Data Handling

| Data | Storage Method | Notes |
|------|---------------|-------|
| Entra ID Client Secret | Azure Key Vault | Referenced via Key Vault reference in App Settings |
| SQL Connection String | Azure Key Vault | Referenced via Key Vault reference in App Settings |
| Storage Connection String | Azure Key Vault | Referenced via Key Vault reference in App Settings |
| User tokens | Session storage (browser) | Not persisted server-side |
| Audit logs | SQL Database | Retained indefinitely |

### Secrets Management

All sensitive secrets are stored in Azure Key Vault and accessed via Key Vault references:

```
AzureAd__ClientSecret = @Microsoft.KeyVault(SecretUri=https://{vault}.vault.azure.net/secrets/AzureAdClientSecret/)
ConnectionStrings__DefaultConnection = @Microsoft.KeyVault(SecretUri=https://{vault}.vault.azure.net/secrets/SqlConnectionString/)
AzureStorage__ConnectionString = @Microsoft.KeyVault(SecretUri=https://{vault}.vault.azure.net/secrets/StorageConnectionString/)
```

**Key Vault Security Features:**
- Secrets encrypted at rest with Azure-managed keys
- Soft delete enabled (7-day retention for accidental deletion recovery)
- Access restricted to the App Service's Managed Identity only
- No direct secret access for users or administrators (must use Azure Portal/CLI with appropriate permissions)
- All secret access is logged in Key Vault diagnostic logs

### Secret and Key Rotation

**Entra ID Client Secret Rotation:**

The Entra ID client secret has an expiration date (typically 1-2 years). To rotate:

1. **Create new secret** in Azure Portal → Entra ID → App Registrations → Your API App → Certificates & secrets
2. **Update Key Vault secret**:
   ```bash
   az keyvault secret set --vault-name <vault-name> --name AzureAdClientSecret --value "<new-secret>"
   ```
3. **Restart App Service** to pick up the new secret:
   ```bash
   az webapp restart --name <app-name> --resource-group <rg-name>
   ```
4. **Delete old secret** from Entra ID after confirming the app works

**SQL Password Rotation:**

1. **Reset password** in Azure Portal → SQL Server → Reset admin password
2. **Update connection string** in Key Vault:
   ```bash
   az keyvault secret set --vault-name <vault-name> --name SqlConnectionString --value "Server=tcp:..."
   ```
3. **Restart App Service**

**Emergency Key Rotation:**

If you suspect a secret has been compromised:
1. Immediately rotate the affected secret using the steps above
2. Review Key Vault access logs for unauthorized access
3. Review App Service logs for unusual activity
4. Consider rotating all secrets if compromise scope is unknown

### Certificate Management

**SSL Certificates:**

| Certificate Type | Management | Renewal |
|-----------------|------------|---------|
| Azure-managed (default) | Automatic | Auto-renewed by Azure |
| Custom domain with Azure certificate | Automatic | Auto-renewed by Azure |
| Custom uploaded certificate | Manual | Upload new certificate before expiration |

**Entra ID App Certificates (Optional):**

If using certificate authentication instead of client secrets:
- Monitor expiration dates in Entra ID → App Registrations → Certificates & secrets
- Upload new certificate before expiration
- Update Key Vault with new certificate thumbprint

**Monitoring Expiration:**

Set up Azure Monitor alerts for:
- Key Vault secret expiration (90 days before)
- Entra ID client secret expiration
- SSL certificate expiration (if custom)

```bash
# Check Key Vault secret expiration dates
az keyvault secret list --vault-name <vault-name> --query "[].{name:name, expires:attributes.expires}"
```

---

## Authentication Flow

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   Browser    │         │   Entra ID   │         │  App Service │
└──────┬───────┘         └──────┬───────┘         └──────┬───────┘
       │                        │                        │
       │  1. Navigate to app    │                        │
       │───────────────────────────────────────────────>│
       │                        │                        │
       │  2. Redirect to login  │                        │
       │<───────────────────────────────────────────────│
       │                        │                        │
       │  3. User authenticates │                        │
       │───────────────────────>│                        │
       │                        │                        │
       │  4. Return tokens      │                        │
       │<───────────────────────│                        │
       │                        │                        │
       │  5. API call with token                         │
       │───────────────────────────────────────────────>│
       │                        │                        │
       │                        │  6. Validate token     │
       │                        │<───────────────────────│
       │                        │                        │
       │                        │  7. Token valid        │
       │                        │───────────────────────>│
       │                        │                        │
       │  8. API response       │                        │
       │<───────────────────────────────────────────────│
```

### Token Validation

The API validates JWT tokens from Entra ID:

1. **Issuer**: Must match Entra ID tenant
2. **Audience**: Must match API Client ID
3. **Signature**: Validated using Entra ID public keys
4. **Expiration**: Token must not be expired
5. **Claims**: User identity extracted from token

---

## Authorization Model

### Role-Based Access Control

| Role | How Assigned | Capabilities |
|------|--------------|--------------|
| User | Any authenticated user | Browse apps, submit requests, view own requests |
| Approver | Member of Approver Entra ID Group | Approve/reject requests, view pending approvals |
| Admin | Member of Admin Entra ID Group | Full access, manage apps, settings, users |

### Group-Based Authorization

Authorization groups are configured in the database (`PortalSettings` table):

```sql
AdminGroupId      -- Entra ID Group Object ID for admins
ApproverGroupId   -- Entra ID Group Object ID for approvers
UserAccessGroupId -- (Optional) Restrict access to specific group
```

### Authorization Check Flow

```csharp
// Check if user is admin
var userId = User.FindFirstValue("oid");
var isAdmin = await _azureADService.IsUserInGroupAsync(userId, adminGroupId);
```

---

## Audit Logging

### What is Logged

| Event | Data Captured |
|-------|--------------|
| App request submitted | User ID, App ID, Device ID, Timestamp |
| Request approved/rejected | Reviewer ID, Decision, Comments, Timestamp |
| Settings changed | User ID, Setting changed, Old/New values |
| App sync | User ID, Apps added/updated, Timestamp |
| User login | User ID, IP address, Timestamp |

### Audit Log Storage

- **Location**: SQL Database (`AuditLogs` table)
- **Retention**: Indefinite (no automatic purge)
- **Access**: Admin-only via Reports section

### Log Schema

```sql
CREATE TABLE AuditLogs (
    Id NVARCHAR(450) PRIMARY KEY,
    UserId NVARCHAR(450) NOT NULL,
    UserEmail NVARCHAR(255) NOT NULL,
    Action NVARCHAR(100) NOT NULL,
    EntityType NVARCHAR(100) NOT NULL,
    EntityId NVARCHAR(450) NOT NULL,
    Details NVARCHAR(MAX),  -- JSON payload
    Timestamp DATETIME2 NOT NULL,
    IpAddress NVARCHAR(50) NOT NULL
);
```

---

## Security Checklist

### Pre-Deployment

- [ ] Entra ID App Registrations created with minimum required permissions
- [ ] Admin consent granted for application permissions
- [ ] Strong SQL administrator password configured
- [ ] Admin Group ID configured (**required** — admin access is denied without it)
- [ ] Approver Group ID configured (restricts approver access)

### Post-Deployment

- [ ] Verify HTTPS-only access working
- [ ] Test authentication flow
- [ ] Verify role-based access (admin vs. user)
- [ ] Review Entra ID sign-in logs
- [ ] Enable Azure Security Center recommendations

### Ongoing Operations

- [ ] Rotate Entra ID client secret annually
- [ ] Review audit logs regularly
- [ ] Monitor failed authentication attempts
- [ ] Update application when security patches released
- [ ] Review and remove unused app permissions

### Optional Enhancements

- [ ] Enable Entra ID Conditional Access policies
- [ ] Configure Entra ID Identity Protection
- [ ] Implement Private Endpoints for SQL and Key Vault
- [ ] Add Azure Front Door with WAF
- [ ] Enable Azure Defender for App Service
- [x] Azure Key Vault for secrets (enabled by default)

---

## Compliance Considerations

### Data Residency

- All data stored in the Azure region selected during deployment
- No cross-region data replication by default

### Data Retention

- Audit logs: Retained indefinitely
- Application data: Retained until manually deleted
- App Insights logs: 90 days default (configurable)

### GDPR Considerations

- User data limited to: Name, Email, User ID (from Entra ID)
- No sensitive personal data stored
- Data can be exported/deleted upon request via database access

---

## Incident Response

### Security Event Indicators

Monitor for these events in Entra ID and App Insights:

1. **Multiple failed logins** from same IP
2. **Unusual geographic access** patterns
3. **Bulk data access** (many API calls in short time)
4. **Privilege escalation** attempts
5. **Configuration changes** outside business hours

### Response Actions

1. **Disable App Service** if compromise suspected
2. **Rotate secrets** (Entra ID client secret, SQL password)
3. **Review audit logs** for unauthorized access
4. **Revoke user access** if account compromised
5. **Contact Azure Support** for assistance

---

## Contact

For security questions or to report vulnerabilities:
- Email: security@powerstacks.com
