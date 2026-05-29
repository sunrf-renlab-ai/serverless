# Transactional email / magic links

Auth flows (magic-link sign-in, email OTP, invites, password reset) need to
actually deliver email to **arbitrary strangers** — your users. This is the
single most underestimated part of a $0 launch, because every path has a
gotcha. Here's what actually works.

## The core constraint nobody tells you

**Every reputable email service blocks you from sending to arbitrary recipients
until you verify a sending domain.** This is anti-spam, not a paywall. Resend,
SendGrid, Postmark, Mailgun, AWS SES — all the same. Before domain verification
you can only send to *your own account email* (a sandbox), which is useless for
a real signup flow.

So the decision is really:

1. **Verify a domain** (add DKIM/SPF/MX DNS records) → professional sender
   (`noreply@yourdomain`), high volume, good deliverability. Needs DNS access.
2. **Use a real mailbox's SMTP** (Gmail / Outlook / QQ / 163) → sends from a
   personal address to anyone, **no DNS**, ~hundreds/day. Instant, slightly less
   professional, can land in spam at volume.

For a closed beta, **(2) is the fastest unblock**. For public launch, do (1).

## Supabase Auth built-in email — fine for ~nothing

Supabase ships a built-in mailer so signups "work" out of the box, but it's
rate-limited to roughly **2–4 emails/hour** and explicitly "for testing only".
It will silently throttle your beta. You almost always need custom SMTP.

## Wiring custom SMTP into Supabase (Management API)

No dashboard needed — PATCH the auth config with a Supabase PAT (the same
`sbp_...` token used for migrations; on macOS the Supabase CLI stores it in the
keychain under service `"Supabase CLI"`, go-keyring-base64-encoded):

```bash
PAT=$(security find-generic-password -s "Supabase CLI" -w \
      | sed 's/^go-keyring-base64://' | base64 -d)
REF="<project-ref>"
curl -s -X PATCH "https://api.supabase.com/v1/projects/$REF/config/auth" \
  -H "Authorization: Bearer $PAT" -H "Content-Type: application/json" \
  -d '{
    "smtp_host": "smtp.gmail.com",
    "smtp_port": "465",
    "smtp_user": "you@gmail.com",
    "smtp_pass": "<app password>",
    "smtp_admin_email": "you@gmail.com",
    "smtp_sender_name": "Your App",
    "smtp_max_frequency": 5
  }'
```

**GOTCHA: `smtp_port` must be a STRING (`"465"`), not an integer.** Passing
`465` silently no-ops — the PATCH returns 200 but every smtp field stays null
and you'll wonder why nothing changed. (To revert to built-in email, PATCH the
smtp fields to `null`, not `""` — empty string fails the email-format validator
on `smtp_admin_email`.)

After this, signups that previously 500'd on the built-in mailer go through your
SMTP. A `200` from `POST /auth/v1/otp` means the SMTP handshake succeeded (the
message was accepted for delivery).

## Gmail SMTP — the no-DNS path

1. The Gmail account needs **2-Step Verification ON** (App Passwords require it).
2. https://myaccount.google.com/apppasswords → create → copy the **16-char**
   password, **strip the spaces**.
3. SMTP: `smtp.gmail.com` port `465` (SSL), user = full address, pass = the App
   Password (NOT the login password).
4. Limits: ~500 recipients/day. Sender shows as the Gmail address. Fine for beta.
5. Workspace (company) accounts may have App Passwords disabled by an admin — use
   a personal Gmail.

A throwaway freshly-registered Gmail/Outlook works great as a dedicated "virtual"
sender. **Disposable inbox sites (temp-mail, 10-minute-mail) do NOT** — they're
receive-only, no SMTP.

QQ: `smtp.qq.com:465` (enable SMTP in settings, get an auth code). 163:
`smtp.163.com`. Outlook: `smtp-mail.outlook.com:587`.

## Resend — and why "full-access key" doesn't bypass verification

A teammate will insist "a full-permission Resend key just sends." It doesn't.
Resend itself returns, verbatim:

> You can only send testing emails to your own email address (you@acct.com). To
> send emails to other recipients, please verify a domain at resend.com/domains,
> and change the `from` address to an email using this domain.

Two independent things people conflate:
- **Key permission** (send-only vs full) → controls what the *API* can do
  (create domains, read logs). A *send-only* key can't `POST /domains` and can't
  `GET /emails`. It is NOT about who you can email.
- **Domain verification** → controls whether you can send to *anyone*. Required
  regardless of key permission.

To verify a domain via API (needs a **full-access** key):

```bash
RESEND=$(security find-generic-password -s <name> -w)
# create
curl -s -X POST https://api.resend.com/domains -H "Authorization: Bearer $RESEND" \
  -H "Content-Type: application/json" -d '{"name":"mail.yourdomain.com"}'
# fetch the DKIM/SPF/MX records to hand to your DNS admin
curl -s "https://api.resend.com/domains/$DOMAIN_ID" -H "Authorization: Bearer $RESEND" \
  | jq -r '.records[] | "\(.type) \(.name) → \(.value)"'
```

Resend's records land on sub-names (`resend._domainkey.<domain>`,
`send.<domain>`), so they don't collide with an existing website CNAME at the
domain apex. After the DNS admin adds them, poll the domain until
`status == "verified"`, then set Supabase `smtp_admin_email` to
`noreply@<verified-domain>` and `smtp_host=smtp.resend.com`, `smtp_user=resend`,
`smtp_pass=<resend key>`.

Free tier: 3,000/month, 100/day.

## Verifying delivery end-to-end without your own inbox

You can drive the whole auth flow programmatically with the **service_role key**
(fetch it via Management API — the dashboard isn't required):

```bash
curl -s "https://api.supabase.com/v1/projects/$REF/api-keys?reveal=true" \
  -H "Authorization: Bearer $PAT" | jq -r '.[]|select(.name=="service_role")|.api_key'
```

With it you can `admin/generate_link`, set a password and use the password grant,
delete test users, etc. — useful to prove "login → token → protected API" works
before any real email is clicked. But to prove email *delivery to a stranger*,
send to a second real address you control (sender ≠ recipient) and eyeball the
inbox; the magic-link OTP API tokens are finicky to verify headlessly
(`otp_expired` even when fresh), so password-grant is the reliable headless path.
