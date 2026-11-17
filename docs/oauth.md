---
title: OAuth setup
description: Google Drive, Google IMAP/SMTP and Microsoft OAuth applications for WhelkMail.
---

# OAuth setup

OAuth is optional for generic IMAP and for app-password accounts. It is required for Microsoft 365 / Outlook accounts, which no longer accept basic authentication for IMAP/SMTP.

## Google OAuth for IMAP/SMTP

Use a dedicated OAuth client, separate from the Drive client, so the Drive-only configuration remains non-sensitive.

1. Google Cloud Console â†’ create or select a project.
2. OAuth consent screen â†’ **External** (or **Internal** for Workspace-only use).
3. Add scopes:
   - `openid`
   - `email`
   - `https://mail.google.com/`
4. Create an OAuth client ID of type **Web application**.
5. Authorized redirect URIs:
   - `https://<your-domain>/api/google/oauth/callback`
   - `http://localhost:8788/api/google/oauth/callback` for local development
6. Set `GOOGLE_MAIL_CLIENT_ID` and `GOOGLE_MAIL_CLIENT_SECRET`.

Important: `https://mail.google.com/` is a **restricted** Google scope.

- A public production app requires Google verification (CASA).
- A personal deployment can stay in **Testing** by adding the Google account as a test user. OAuth works, but refresh tokens expire after 7 days.
- A Google Workspace **Internal** app avoids verification but only works inside that organization.
- If OAuth is not configured, use an app password with 2-Step Verification enabled.

## Google Drive OAuth (relay and IMAP archive)

Used by relay accounts that archive on Drive and by IMAP accounts that enable **Archive bodies and attachments** in their sync settings.

1. Google Cloud Console â†’ enable the Google Drive API.
2. OAuth consent screen â†’ **External**.
3. Scopes: `openid`, `email`, `https://www.googleapis.com/auth/drive.file`.
4. Create an OAuth client ID of type **Web application**.
5. Redirect URIs:
   - `https://<your-domain>/api/drive/oauth/callback`
   - `http://localhost:8788/api/drive/oauth/callback`
6. Set `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET`.
7. Set the consent screen to **In production** to avoid 7-day refresh-token expiry. `drive.file` is non-sensitive, so no CASA assessment is required.

The same credentials serve IMAP accounts that enable **Archive bodies and attachments** in Settings. They are separate from the Gmail mail client (`GOOGLE_MAIL_CLIENT_ID` / `GOOGLE_MAIL_CLIENT_SECRET`); without them the Connect Drive button returns `oauth_not_configured`.

Files live in a `MailClient` folder. Google Drive offers 15 GB free for personal accounts. Service accounts have no storage quota and cannot be used as a free bucket.

## Microsoft OAuth (Outlook / Microsoft 365)

1. Microsoft Entra ID â†’ **App registrations** â†’ **New registration**.
2. Supported account types: personal and/or organizational, depending on your users.
3. Redirect URI (Web):
   - `https://<your-domain>/api/microsoft/oauth/callback`
   - `http://localhost:8788/api/microsoft/oauth/callback`
4. **Certificates & secrets** â†’ create a client secret.
5. API permissions (delegated):
   - Office 365 Exchange Online: `IMAP.AccessAsUser.All`, `SMTP.Send`
   - OpenID: `offline_access`, `openid`, `email`, `profile`
6. Grant admin consent if the tenant requires it.
7. Set `MICROSOFT_CLIENT_ID`, `MICROSOFT_CLIENT_SECRET`, and optionally `MICROSOFT_TENANT` (`common` for personal + work).

## Relay provider setup

For send-only accounts:

1. Create an account with the relay provider.
2. Verify the sending domain with the provider (SPF and DKIM).
3. Copy the provider API key.
4. Paste it in the account creation form or in Settings.
5. Choose where the sent copies and drafts go: **Google Drive**, **R2** or **another account's mailbox**.

> A relay account always needs a **working** destination. If the sent copy cannot be saved (Drive not connected, no `BUCKET` binding, or the chosen IMAP mailbox disconnected) the send is aborted **before** the relay is called and comes back as `storage_failed`.

Supported providers: SMTP2GO, Resend, Mailgun, Brevo, SendGrid, Mailjet, Postmark, Cloudflare Email Service. See [Relay accounts](relay-accounts) for provider details.

### Cloudflare Email Service

> **Experimental.** The maintainer has not tested this provider against a real Cloudflare account. The steps follow the documented REST API, but actual delivery, suppression handling and rate limits are unverified: treat it as a preview and keep a tested relay for mail you care about.

- API key: an API token with the **Email Sending: Edit** permission. An account API token is simplest: it reaches exactly one account.
- Account ID: optional. Left blank, WhelkMail looks it up from the API token when the account is saved, which works when the token reaches exactly one account. A user API token that reaches several accounts needs the ID entered. A token that reaches **no** account at all is refused with `relay_account_unavailable`.
- The sending domain must be onboarded under **Email Service â†’ Email Sending** in the same account. Each subdomain is onboarded separately. If it isn't, sending fails with a message naming the domain (Cloudflare error `10203`).
- By default, a message with any recipient on Cloudflare's suppression list is rejected outright (error `10205`). Turn on **Drop suppressed recipients** for the sending domain to send to the other recipients instead.
- The custom SMTP option can't be used with Cloudflare's SMTP endpoint (`smtp.mx.cloudflare.net`): Workers block TCP sockets to Cloudflare's own IP ranges.
- Limits enforced by WhelkMail before the API call: at most **50 recipients** per message (to, cc and bcc combined) and **15 MiB of attachments**, to stay under Cloudflare's 25 MiB encoded-message limit.

## OAuth security

- One-time state stored in D1 with a 10-minute TTL.
- PKCE (`S256`) for Google, Microsoft and Google Drive.
- Refresh tokens are encrypted with AES-256-GCM using `TOKEN_ENC_KEY`.

## Related

- [Configuration](configuration)
- [Deployment](deployment)
- [Troubleshooting](troubleshooting)
