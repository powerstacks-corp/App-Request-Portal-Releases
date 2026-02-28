# Teams Bot Manifest

This directory contains the Microsoft Teams app manifest for the App Portal for Intune bot.

## Setup Instructions

### 1. Update the manifest

Edit `manifest.json` and replace all placeholder values:

| Placeholder | Replace With |
|---|---|
| `{{BOT_APP_ID}}` | Your Azure Bot's Microsoft App ID (GUID) |
| `Your Organization` | Your company name |
| `your-portal-url.azurewebsites.net` | Your App Portal for Intune URL (appears in `websiteUrl`, `privacyUrl`, `termsOfUseUrl`, and `validDomains`) |

### 2. Replace the icons (optional)

- `color.png` — 192x192 full-color icon (displayed in the Teams app catalog)
- `outline.png` — 32x32 transparent outline icon (displayed in the Teams activity bar)

The included icons are placeholders. Replace them with your organization's branding.

### 3. Create the app package

Zip all three files into a single `.zip` file:

```powershell
Compress-Archive -Path manifest.json, color.png, outline.png -DestinationPath AppRequestPortalBot.zip
```

### 4. Upload to Teams Admin Center

1. Go to [Teams Admin Center](https://admin.teams.microsoft.com)
2. Navigate to **Teams apps** > **Manage apps**
3. Click **Upload new app** > **Upload**
4. Select your `AppRequestPortalBot.zip` file

### 5. Deploy to users via Setup Policy

1. In Teams Admin Center, go to **Teams apps** > **Setup policies**
2. Edit the **Global (Org-wide default)** policy or create a custom policy
3. Under **Installed apps**, click **Add apps**
4. Search for "App Portal for Intune" and add it
5. Click **Save**

The bot will be automatically installed for all users in scope. When installed, the bot stores a conversation reference that enables proactive notifications.

> **Note:** It may take up to 24 hours for the setup policy to propagate to all users.

## How It Works

The bot uses **proactive messaging** — it does not respond to user messages. When the bot is installed for a user, Teams sends a `conversationUpdate` event to your API (`/api/messages`). The portal stores the conversation reference for that user. When a notification needs to be sent, the portal uses the stored reference to send an Adaptive Card directly to the user's chat.
