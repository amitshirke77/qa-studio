# Deploying QA Studio: GitHub + Supabase + Vercel

This takes about 45 minutes the first time. You need admin access to the zipstacc GitHub organization, a Supabase account, a Vercel account and the zipstacc Microsoft Entra ID (Azure AD).

**Where everything runs**
- **GitHub** holds the code. Its CI workflow checks every change, and its Database workflow applies database changes.
- **Supabase** runs the PostgreSQL database in Mumbai (ap-south-1).
- **Vercel** builds and hosts the app in Mumbai (bom1). It deploys automatically on every push to `main` and runs the scheduled jobs.
- **Microsoft Entra ID**, and optionally **Google Workspace**, handle sign-in for staff (zipstacc.com) and for client stakeholders (for example customer-capital.com).

The database starts empty: no sample audits, agents or demo logins. The first-run setup creates only real master data:
- the companies Expenova and PeopleOCity, and the clients CUSTCAP and UPPERCASE;
- CUSTCAP's customers, LOBs and products, and the call and email forms with the full evaluation legend;
- your admin login.

---

## 1. GitHub

1. Create a **private** repository, for example `zipstacc/qa-studio`.
2. Push the code:
   ```bash
   cd qa-studio
   git init -b main && git add -A && git commit -m "QA Studio"
   git remote add origin https://github.com/zipstacc/qa-studio.git
   git push -u origin main
   ```
3. Go to **Settings › Branches** and protect `main`: require a pull request and the **CI / build** check.

## 2. Supabase

1. Go to **New project**.
   - Organization: zipstacc. Name: `qa-studio`. Region: **South Asia (Mumbai)**.
   - Set a strong database password and save it in your password manager.
2. Click **Connect** at the top of the project, then **ORMs** / **Connection string**. Copy two strings:
   - **Transaction pooler** (port **6543**): this is `DATABASE_URL`, used by the app.
   - **Session pooler** (port **5432**): this is `DIRECT_URL`, used for migrations and first-run setup.

   Replace `[YOUR-PASSWORD]` in both.
3. Recommended security settings:
   - **Project Settings › Data API:** turn the Data API **off**. QA Studio talks to Postgres directly and never uses it. The migrations also switch on row level security and revoke the `anon` and `authenticated` roles, so the tables stay closed even if the Data API is on.
   - **Project Settings › Database › Network restrictions:** allow only the IP ranges you need, if your plan has this.
   - **Backups** are daily on Pro. Turn on **Point-in-time recovery** for production.

## 3. Create the tables and the first login

Pick one of these:

**Option A: from GitHub (no local setup).**
1. In the repository, go to **Settings › Secrets and variables › Actions** and add these secrets:
   - `DIRECT_URL` (the session pooler string)
   - `ADMIN_EMAIL` (for example `amit.shirke@zipstacc.com`)
   - `ADMIN_NAME`
   - optionally `ADMIN_PASSWORD` (12+ characters), `CUSTCAP_PASSWORD` and `UPPERCASE_PASSWORD` (12+ characters each, for shared client logins)
2. Optionally add these as *variables*: `CUSTCAP_SSO_DOMAINS` (default `customer-capital.com`), `UPPERCASE_SSO_DOMAINS` and `ALLOWED_EMAIL_DOMAINS` (default `zipstacc.com`).
3. Go to **Actions › Database › Run workflow**, tick **first_run**, and run it. From then on, the workflow applies new migrations by itself whenever one is merged to `main`.

**Option B: from your machine.**
```bash
npm ci
DIRECT_URL='postgresql://...:5432/postgres' npm run db:migrate
DATABASE_URL='postgresql://...:5432/postgres' ADMIN_EMAIL=amit.shirke@zipstacc.com ADMIN_NAME='Amit Shirke' ADMIN_PASSWORD='…' npm run db:seed
```

Either way, check the result in Supabase › SQL Editor by running `scripts/check-rls.sql`. Every table should show `true`.

## 4. Vercel

1. Go to **Add New › Project**, import the GitHub repository, and keep the framework as **Next.js**.
2. Go to **Settings › Environment Variables**. For **Production** (and Preview, with a separate Supabase project or branch if you use previews), add:

   | Variable | Value |
   |---|---|
   | `DATABASE_URL` | Supabase transaction pooler string (port 6543) |
   | `NEXTAUTH_URL` | `https://qa.zipstacc.com` (your final domain) |
   | `NEXTAUTH_SECRET` | output of `openssl rand -base64 32` |
   | `CRON_SECRET` | output of `openssl rand -hex 32` |
   | `AZURE_AD_TENANT_ID`, `AZURE_AD_CLIENT_ID`, `AZURE_AD_CLIENT_SECRET` | from step 5 |
   | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | optional, from step 5b |
   | `ALLOWED_EMAIL_DOMAINS` | `zipstacc.com` |
   | `GRAPH_SENDER` | the mailbox that sends notifications |
   | `TEAMS_WEBHOOK_URL` | optional, from step 6 |
   | `APP_TIMEZONE` | `Asia/Kolkata` |

3. **Deploy.** `vercel.json` already sets the Mumbai region and the two scheduled jobs:
   - `/api/cron/outbox` sends emails and Teams messages every 5 minutes;
   - `/api/cron/auto-ack` applies the dispute window hourly.

   Vercel sends `CRON_SECRET` with each call. 5-minute crons need the **Pro** plan; Hobby runs them once a day.
4. **Domain.** Go to **Settings › Domains**, add `qa.zipstacc.com`, and create the CNAME it shows in your DNS. Then update `NEXTAUTH_URL` and redeploy.
5. **Check.** Opening `https://qa.zipstacc.com/api/health` should return `{"ok":true}`.

## 5. Microsoft 365 sign-in (Entra ID)

1. In the zipstacc tenant, go to **Entra ID › App registrations › New registration**.
   - Name: `QA Studio`.
   - Redirect URI (Web): `https://qa.zipstacc.com/api/auth/callback/azure-ad`.
   - Supported account types: **Single tenant** (recommended), or **Multitenant** if clients will sign in from their own tenant.
2. Copy the **Application (client) ID** and the **Directory (tenant) ID**.
3. Under **Certificates & secrets**, create a client secret.
4. Put them in Vercel as `AZURE_AD_CLIENT_ID`, `AZURE_AD_TENANT_ID` and `AZURE_AD_CLIENT_SECRET`. For multitenant, set `AZURE_AD_TENANT_ID=organizations`.
5. Optional: under **Token configuration › Add groups claim**, add security groups. Then map group object IDs to role and company under Admin › Login & SSO.
6. Set up client stakeholders, for example CUSTCAP. Pick one approach:
   - **Single tenant:** invite each person as a guest (Entra ID › Users › Invite external user).
   - **Multitenant:** add CUSTCAP's tenant ID under Admin › Master data › CUSTCAP › Microsoft 365 sign-in. CUSTCAP's IT must consent to the app once.
7. Email notifications: add the Microsoft Graph **Mail.Send** *application* permission and grant admin consent. You can limit it to the `GRAPH_SENDER` mailbox with an Exchange application access policy.

## 5b. Google sign-in (optional)

Google sign-in accepts only **Google Workspace** accounts on a configured domain: a staff domain, or a client's domain under Master data. Personal Google accounts are refused, even ones registered with a work email.

1. In the [Google Cloud console](https://console.cloud.google.com/), create or pick a project.
2. Go to **APIs & Services › OAuth consent screen**. Choose **Internal** if only zipstacc Workspace users will use Google; choose **External** if client stakeholders on other Workspace domains will too. Add the app name, the support email and the `qa.zipstacc.com` domain.
3. Go to **Credentials › Create credentials › OAuth client ID**.
   - Type: **Web application**.
   - Authorized redirect URI: `https://qa.zipstacc.com/api/auth/callback/google`.
4. Put the client ID and secret in Vercel as `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET`. Optionally set `GOOGLE_HD=zipstacc.com` to show only zipstacc accounts in Google's chooser. Leave it empty if client stakeholders use Google too.
5. Redeploy. A **Sign in with Google** button appears on the sign-in page.

Google sends no group membership, so add staff who use Google under Admin › Users first. Each login is linked to its Google account on first sign-in, and a different Google account with the same email is refused.

## 6. Teams (optional)

In the QA channel, go to **Workflows › "Post to a channel when a webhook request is received"**. Copy the URL into `TEAMS_WEBHOOK_URL`.

## 7. First sign-in and set-up order

1. Sign in as the admin with Microsoft 365, or with `ADMIN_PASSWORD` if you set one.
2. **Admin › Master data:**
   - check CUSTCAP's customers, LOBs and products;
   - confirm the CUSTCAP Microsoft 365 domain;
   - add UPPERCASE's customers, LOBs and products.
3. **Admin › Forms:** map or create the forms for UPPERCASE's LOBs.
4. **Admin › Users:** add Team Leaders, QA Auditors and QA Leads (with their company), then agents (with their client, LOB and TL), then client stakeholders.
5. **Audit targets:** enter the last 3 months of volume, and set auditor shares.
6. **Import past audits:** download the sample file, fill it, and import.

## Day-to-day

- **Code change:** open a pull request; CI must pass; merge. Vercel deploys `main` automatically.
- **Database change:** run `npm run db:generate` locally, commit the new file in `drizzle/`, and merge. The Database workflow applies it, and the lockdown re-runs so new tables get row level security.
- **Rollback:** in Vercel, open Deployments and choose **Promote** on the previous deployment. Migrations are forward-only, so write a new migration to undo a schema change.
- **Secrets:** rotate `NEXTAUTH_SECRET`, `CRON_SECRET`, the Entra client secret and the database password at least yearly. Rotating `NEXTAUTH_SECRET` signs everyone out.

## Limits to know

- **Attachments** on JIT trainings are stored in Postgres, at most 4 MB each because of Vercel's 4.5 MB request limit. Put videos on SharePoint and link them.
- **Sign-in throttling** is held in memory per server instance. For heavy use, move it to a shared store such as Upstash Redis.
