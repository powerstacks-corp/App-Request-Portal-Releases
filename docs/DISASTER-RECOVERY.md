# Disaster Recovery Guide

This document provides disaster recovery procedures for the App Request Portal, including backup strategies, recovery runbooks, and high availability options.

**Last Updated:** February 2026

---

## Table of Contents

1. [Recovery Objectives](#recovery-objectives)
2. [Built-in Protection (Tier 2)](#built-in-protection-tier-2)
3. [Recovery Procedures](#recovery-procedures)
4. [High Availability Setup (Tier 3)](#high-availability-setup-tier-3)
5. [Backup Verification](#backup-verification)
6. [Emergency Contacts](#emergency-contacts)

---

## Recovery Objectives

### Tier 2 (Default Deployment)

| Metric | Target |
|--------|--------|
| **RTO (Recovery Time Objective)** | 1-2 hours |
| **RPO (Recovery Point Objective)** | 5 minutes (SQL), 0 minutes (Storage with GRS) |
| **Data Loss Risk** | Minimal - geo-redundant backups |

### Tier 3 (High Availability)

| Metric | Target |
|--------|--------|
| **RTO (Recovery Time Objective)** | 15-30 minutes |
| **RPO (Recovery Point Objective)** | Near-zero with active geo-replication |
| **Data Loss Risk** | Near-zero |

---

## Built-in Protection (Tier 2)

The default ARM template deployment includes these disaster recovery features:

### SQL Database Backups

| Feature | Configuration |
|---------|---------------|
| **Automated Backups** | Enabled by default |
| **Point-in-Time Restore** | 7 days (Basic tier) |
| **Backup Storage** | Geo-redundant (cross-region) |
| **Full Backups** | Weekly |
| **Differential Backups** | Every 12-24 hours |
| **Transaction Log Backups** | Every 5-10 minutes |

**What's Protected:**
- All application data (apps, requests, settings)
- User configurations and branding
- Audit logs and history

### Storage Account (GRS)

| Feature | Configuration |
|---------|---------------|
| **Redundancy** | Geo-Redundant Storage (GRS) |
| **Replication** | 6 copies (3 primary + 3 secondary region) |
| **Failover** | Automatic with RA-GRS or manual |

**What's Protected:**
- Winget packaging queue messages
- Package blobs (temporary)

### Key Vault

| Feature | Configuration |
|---------|---------------|
| **Soft Delete** | Enabled (7-day retention) |
| **Purge Protection** | Optional (prevents permanent deletion) |

**What's Protected:**
- Azure AD client secret
- SQL connection string
- Storage connection string

### Application Code

| Feature | Protection |
|---------|------------|
| **Source Code** | GitHub repository |
| **Deployment Package** | GitHub Releases (immutable) |
| **ARM Template** | GitHub repository |

**Recovery:** Redeploy from ARM template - no code backup needed.

---

## Recovery Procedures

### Scenario 1: Accidental Data Deletion

**Symptoms:** User accidentally deleted requests, apps, or settings.

**Recovery Steps:**

1. **Identify the deletion time** from audit logs or user report

2. **Restore SQL Database to point-in-time:**
   ```bash
   # Azure CLI
   az sql db restore \
     --resource-group <resource-group> \
     --server <sql-server-name> \
     --name AppRequestPortal \
     --dest-name AppRequestPortal-Restored \
     --time "2026-02-14T10:00:00Z"
   ```

3. **Verify restored data** by connecting to the restored database

4. **Option A - Full restore:** Swap the restored database
   ```bash
   # Rename original (backup)
   az sql db rename --resource-group <rg> --server <server> \
     --name AppRequestPortal --new-name AppRequestPortal-Old

   # Rename restored to production
   az sql db rename --resource-group <rg> --server <server> \
     --name AppRequestPortal-Restored --new-name AppRequestPortal
   ```

5. **Option B - Selective restore:** Copy specific records from restored DB to production

6. **Restart App Service** to reconnect:
   ```bash
   az webapp restart --resource-group <rg> --name <app-name>
   ```

**Estimated Recovery Time:** 30-60 minutes

---

### Scenario 2: Key Vault Secret Deleted

**Symptoms:** Application fails to start, "Key Vault reference failed" errors.

**Recovery Steps:**

1. **Check if secret is soft-deleted:**
   ```bash
   az keyvault secret list-deleted --vault-name <vault-name>
   ```

2. **Recover the deleted secret:**
   ```bash
   az keyvault secret recover \
     --vault-name <vault-name> \
     --name AzureAdClientSecret
   ```

3. **If purged (permanently deleted),** recreate the secret:
   ```bash
   # Get new client secret from Azure AD App Registration
   az keyvault secret set \
     --vault-name <vault-name> \
     --name AzureAdClientSecret \
     --value "<new-secret-value>"
   ```

4. **Restart App Service:**
   ```bash
   az webapp restart --resource-group <rg> --name <app-name>
   ```

**Estimated Recovery Time:** 15-30 minutes

---

### Scenario 3: Complete Region Failure

**Symptoms:** All Azure services in primary region unavailable.

**Recovery Steps:**

1. **Create new resource group in secondary region:**
   ```bash
   az group create --name apprequest-dr --location westus2
   ```

2. **Deploy ARM template to secondary region:**
   ```bash
   az deployment group create \
     --resource-group apprequest-dr \
     --template-uri https://raw.githubusercontent.com/PowerStacks-BI/AppRequestPortal/main/azuredeploy.json \
     --parameters \
       environmentName=prod \
       apiClientId=<client-id> \
       apiClientSecret=<client-secret> \
       frontendClientId=<frontend-client-id> \
       sqlAdminPassword=<password>
   ```

3. **Restore SQL Database from geo-backup:**
   ```bash
   # List available geo-backups
   az sql db geo-backup list \
     --resource-group <original-rg> \
     --server <original-server>

   # Restore to new server
   az sql db geo-restore \
     --resource-group apprequest-dr \
     --server <new-server-name> \
     --name AppRequestPortal \
     --geo-backup-id <backup-id>
   ```

4. **Update Azure AD redirect URIs:**
   - Add new App Service URL to Frontend SPA app registration
   - Add new URL to Backend API if needed

5. **Update DNS (if using custom domain):**
   - Point DNS to new App Service
   - Request new SSL certificate or migrate existing

6. **Verify application functionality**

**Estimated Recovery Time:** 2-4 hours

---

### Scenario 4: Storage Account Failure

**Symptoms:** Winget packaging jobs fail, queue errors.

**Recovery Steps:**

1. **For GRS accounts, initiate failover:**
   ```bash
   az storage account failover \
     --name <storage-account-name> \
     --resource-group <rg>
   ```

   > **Note:** Failover may take up to 1 hour and makes secondary region the new primary.

2. **Alternative - Create new storage account:**
   ```bash
   az storage account create \
     --name <new-storage-name> \
     --resource-group <rg> \
     --location <location> \
     --sku Standard_GRS
   ```

3. **Update Key Vault with new connection string:**
   ```bash
   CONNECTION_STRING=$(az storage account show-connection-string \
     --name <new-storage-name> --resource-group <rg> -o tsv)

   az keyvault secret set \
     --vault-name <vault-name> \
     --name StorageConnectionString \
     --value "$CONNECTION_STRING"
   ```

4. **Restart App Service:**
   ```bash
   az webapp restart --resource-group <rg> --name <app-name>
   ```

**Estimated Recovery Time:** 1-2 hours

---

### Scenario 5: Application Corruption / Bad Deployment

**Symptoms:** Application crashes, unexpected behavior after update.

**Recovery Steps:**

1. **Rollback to previous version:**
   ```bash
   # Get previous version URL from GitHub releases
   # Example: v1.5.5
   az webapp config appsettings set \
     --resource-group <rg> \
     --name <app-name> \
     --settings WEBSITE_RUN_FROM_PACKAGE="https://github.com/PowerStacks-BI/App-Request-Portal-Releases/releases/download/v1.5.5/AppRequestPortal.zip"
   ```

2. **Restart App Service:**
   ```bash
   az webapp restart --resource-group <rg> --name <app-name>
   ```

3. **Verify rollback successful**

**Estimated Recovery Time:** 5-15 minutes

---

## High Availability Setup (Tier 3)

For organizations requiring higher availability, implement these additional measures manually.

### SQL Active Geo-Replication

Creates a readable secondary database in another region with continuous replication.

**Setup:**

1. **Create geo-replica:**
   ```bash
   az sql db replica create \
     --resource-group <primary-rg> \
     --server <primary-server> \
     --name AppRequestPortal \
     --partner-server <secondary-server> \
     --partner-resource-group <secondary-rg>
   ```

2. **Configure failover group (recommended):**
   ```bash
   az sql failover-group create \
     --resource-group <primary-rg> \
     --server <primary-server> \
     --name apprequest-fog \
     --partner-server <secondary-server> \
     --partner-resource-group <secondary-rg> \
     --failover-policy Automatic \
     --grace-period 1
   ```

3. **Update connection string** to use failover group endpoint:
   ```
   Server=tcp:apprequest-fog.database.windows.net,1433;...
   ```

**Cost:** ~$5/month additional (Basic tier replica)

**Recovery Time:** Automatic failover in 1-2 minutes

### Traffic Manager / Front Door

Distributes traffic across multiple App Service instances.

**Setup:**

1. **Deploy secondary App Service** in another region using ARM template

2. **Create Traffic Manager profile:**
   ```bash
   az network traffic-manager profile create \
     --resource-group <rg> \
     --name apprequest-tm \
     --routing-method Priority \
     --unique-dns-name apprequest-portal
   ```

3. **Add endpoints:**
   ```bash
   # Primary
   az network traffic-manager endpoint create \
     --resource-group <rg> \
     --profile-name apprequest-tm \
     --name primary \
     --type azureEndpoints \
     --target-resource-id <primary-app-id> \
     --priority 1

   # Secondary
   az network traffic-manager endpoint create \
     --resource-group <rg> \
     --profile-name apprequest-tm \
     --name secondary \
     --type azureEndpoints \
     --target-resource-id <secondary-app-id> \
     --priority 2
   ```

**Cost:** ~$0.75/million queries + secondary App Service (~$55/month)

### Storage RA-GRS with Manual Failover

Enables read access to secondary region for faster failover decisions.

**Setup:**

Change storage redundancy parameter during deployment:
```json
"storageRedundancy": "Standard_RAGRS"
```

**Cost:** ~$0.50/month additional over GRS

---

## Backup Verification

### Monthly Backup Test Checklist

- [ ] **SQL Point-in-Time Restore Test**
  - Restore database to 24 hours ago
  - Verify data integrity
  - Delete test database

- [ ] **Key Vault Recovery Test**
  - List soft-deleted secrets
  - Verify recovery capability
  - Document secret expiration dates

- [ ] **ARM Template Deployment Test**
  - Deploy to test resource group
  - Verify all resources created
  - Delete test deployment

- [ ] **Document Recovery Time**
  - Record actual time for each recovery step
  - Update estimates if needed

### Automated Monitoring

Configure Azure Monitor alerts for:

| Alert | Condition | Action |
|-------|-----------|--------|
| SQL DTU | > 90% for 15 min | Email admins |
| App Service Health | Unhealthy | Email + webhook |
| Key Vault Access Denied | Any | Email security team |
| Storage Availability | < 99% | Email admins |

---

## Emergency Contacts

| Role | Contact | Responsibility |
|------|---------|----------------|
| **Primary Admin** | [Your contact] | First responder for incidents |
| **Azure Support** | https://portal.azure.com/#blade/Microsoft_Azure_Support | Severity A for production down |
| **Security Team** | [Security contact] | Data breach or security incidents |

### Azure Support Plans

| Plan | Response Time (Sev A) | Cost |
|------|----------------------|------|
| Basic | No SLA | Free |
| Developer | 8 hours | ~$29/month |
| Standard | 1 hour | ~$100/month |
| Professional Direct | 15 minutes | ~$1,000/month |

**Recommendation:** Standard support for production deployments.

---

## Recovery Decision Tree

```
Is the application accessible?
├── YES: Can users log in?
│   ├── YES: Is data correct?
│   │   ├── YES: No DR needed
│   │   └── NO: → Scenario 1 (Data Deletion)
│   └── NO: Key Vault issue? → Scenario 2
└── NO: Is it a single resource or entire region?
    ├── Single resource:
    │   ├── SQL → Restore from backup
    │   ├── Storage → Scenario 4
    │   ├── App Service → Redeploy from ARM template
    │   └── Key Vault → Scenario 2
    └── Entire region: → Scenario 3 (Region Failure)
```

---

## Appendix: Resource IDs Quick Reference

After deployment, record these values for emergency use:

| Resource | How to Find |
|----------|-------------|
| Resource Group | Azure Portal → Resource Groups |
| SQL Server Name | Deployment outputs or Portal |
| Storage Account Name | Deployment outputs or Portal |
| Key Vault Name | Deployment outputs or Portal |
| App Service Name | Deployment outputs or Portal |

**Store this information securely outside of Azure** (e.g., password manager, printed runbook).
