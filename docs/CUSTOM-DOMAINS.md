# Custom Domain Configuration

This guide explains how to configure a custom domain (e.g., `apps.yourdomain.com`) for your App Portal for Intune deployment on Azure App Service.

## Overview

By default, your portal is accessible via an Azure-assigned URL like:
```
https://apprequestportal-xxxx.azurewebsites.net
```

You can configure a custom domain to provide a more professional, branded experience:
```
https://apps.yourdomain.com
```

## Prerequisites

- Azure App Service running your portal (Basic tier or higher for custom domains with SSL)
- Access to your domain's DNS management
- Admin access to your Entra ID App Registration

## Step 1: Add Custom Domain in Azure Portal

1. Navigate to **Azure Portal** → **App Services** → Your App Service
2. Click **Custom domains** in the left menu
3. Click **+ Add custom domain**
4. Enter your custom domain (e.g., `apps.yourdomain.com`)
5. Click **Validate**
6. Note the **Custom Domain Verification ID** (you'll need this for DNS)

## Step 2: Configure DNS Records

### For Subdomains (Recommended)

If using a subdomain like `apps.yourdomain.com`:

| Type | Name | Value | TTL |
|------|------|-------|-----|
| CNAME | `apps` | `your-app.azurewebsites.net` | 3600 |
| TXT | `asuid.apps` | `<Custom Domain Verification ID>` | 3600 |

### For Apex/Root Domains

If using your root domain (e.g., `yourdomain.com`):

| Type | Name | Value | TTL |
|------|------|-------|-----|
| A | `@` | `<App Service IP Address>` | 3600 |
| TXT | `asuid` | `<Custom Domain Verification ID>` | 3600 |

**Note:** Get the App Service IP address from **Custom domains** → **IP address** in Azure Portal.

### DNS Propagation

DNS changes can take anywhere from a few minutes to 48 hours to propagate globally. You can verify propagation using:
- [dnschecker.org](https://dnschecker.org)
- `nslookup apps.yourdomain.com`
- `dig apps.yourdomain.com`

## Step 3: Configure SSL/TLS Certificate

### Option A: Free Azure-Managed Certificate (Recommended)

1. In Azure Portal → App Service → **Certificates**
2. Click **+ Add certificate**
3. Select **App Service Managed Certificate**
4. Select your custom domain
5. Click **Create**
6. Once created, go to **Custom domains**
7. Click your domain and select **Add binding**
8. Choose the managed certificate and **SNI SSL**

**Limitations:**
- Only available for App Service Basic tier and above
- Doesn't support wildcard domains
- Doesn't support apex/root domains (use Azure Front Door or a third-party cert)

### Option B: Azure Key Vault Certificate

1. Upload or generate a certificate in Azure Key Vault
2. In App Service → **Certificates** → **+ Add certificate**
3. Select **Import from Key Vault**
4. Choose your Key Vault and certificate
5. Bind to your custom domain

### Option C: Upload Your Own Certificate

1. Obtain a certificate from a Certificate Authority (CA)
2. Export as PFX/PEM with private key
3. In App Service → **Certificates** → **+ Add certificate**
4. Select **Upload certificate**
5. Upload your PFX/PEM file
6. Bind to your custom domain

## Step 4: Update Entra ID App Registration

Your Entra ID App Registration needs updated redirect URIs for authentication to work.

1. Navigate to **Azure Portal** → **Microsoft Entra ID** → **App registrations**
2. Select your App Portal for Intune registration
3. Go to **Authentication** → **Platform configurations** → **Web**
4. Add the following redirect URIs:

```
https://apps.yourdomain.com
https://apps.yourdomain.com/auth/callback
```

5. **Important:** Keep the existing Azure URLs during transition:
```
https://your-app.azurewebsites.net
https://your-app.azurewebsites.net/auth/callback
```

6. Click **Save**

## Step 5: Update Application Configuration

### Update Portal URL Setting

1. Log into your portal as an admin
2. Go to **Admin** → **Settings**
3. In **Email Notifications**, update the **Portal URL** to your custom domain:
   ```
   https://apps.yourdomain.com
   ```
4. Click **Save Settings**

This ensures email notification links point to your custom domain.

### Update Frontend Configuration (if needed)

If you're using environment variables for the API URL, update `REACT_APP_API_URL`:

```env
REACT_APP_API_URL=https://apps.yourdomain.com/api
```

## Step 6: Force HTTPS (Recommended)

Ensure all traffic uses HTTPS:

1. In Azure Portal → App Service → **Configuration**
2. Go to **General settings**
3. Set **HTTPS Only** to **On**
4. Click **Save**

## Step 7: Update Teams Bot Configuration (if enabled)

If you have the Teams Bot enabled for proactive notifications, two things need updating:

### Update Azure Bot Messaging Endpoint

1. Navigate to **Azure Portal** → **Azure Bot** resource → **Configuration**
2. Change **Messaging endpoint** from:
   ```
   https://your-app.azurewebsites.net/api/messages
   ```
   to:
   ```
   https://apps.yourdomain.com/api/messages
   ```
3. Click **Apply**

### Update Teams App Manifest

1. Edit `manifest.json` and add your custom domain to `validDomains`:
   ```json
   "validDomains": [
       "apps.yourdomain.com",
       "your-app.azurewebsites.net"
   ]
   ```
2. Optionally update the `developer` URLs (`websiteUrl`, `privacyUrl`, `termsOfUseUrl`) to use the custom domain
3. Re-zip the manifest files (`manifest.json`, `color.png`, `outline.png`)
4. In **Teams Admin Center** → **Teams apps** → **Manage apps**, find the existing App Portal for Intune bot, click it, and upload the updated package

> **Note:** Keeping both domains in `validDomains` ensures the bot continues to work during the transition. You can remove the `.azurewebsites.net` entry later once the custom domain is fully verified.

## Verification Checklist

After configuration, verify:

- [ ] DNS resolves correctly (`nslookup apps.yourdomain.com`)
- [ ] HTTPS works without certificate warnings
- [ ] Portal loads at custom domain
- [ ] Login/authentication works
- [ ] All navigation links use the custom domain
- [ ] Email notifications contain correct URLs
- [ ] Teams bot notifications still arrive (if enabled)

## Troubleshooting

### "Domain verification failed"

- Verify TXT record is correctly configured
- Wait for DNS propagation (up to 48 hours)
- Ensure the verification ID matches exactly

### "Certificate error" or "Not secure"

- Verify SSL certificate is bound to the custom domain
- Check certificate hasn't expired
- Ensure certificate covers your domain (exact match or wildcard)

### "Authentication failed" after domain change

- Verify redirect URIs are updated in Entra ID
- Clear browser cookies and cache
- Check both old and new URLs are in redirect URIs during transition

### "Mixed content" warnings

- Ensure all API calls use HTTPS
- Update any hardcoded HTTP URLs in configuration

## Multiple Environments

If you have multiple environments (dev, staging, production), configure separate custom domains:

| Environment | Custom Domain |
|-------------|---------------|
| Production | `apps.yourdomain.com` |
| Staging | `apps-staging.yourdomain.com` |
| Development | `apps-dev.yourdomain.com` |

Each requires its own:
- DNS records
- SSL certificate
- Entra ID redirect URIs
- Portal URL setting

## Related Documentation

- [Azure App Service Custom Domains](https://docs.microsoft.com/en-us/azure/app-service/app-service-web-tutorial-custom-domain)
- [Azure App Service SSL Certificates](https://docs.microsoft.com/en-us/azure/app-service/configure-ssl-certificate)
- [Entra ID Redirect URIs](https://docs.microsoft.com/en-us/azure/active-directory/develop/reply-url)
