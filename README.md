# QA Studio

Call and email quality program for every client of **Expenova** and **PeopleOCity**. Data is organized as **Company › Client › Customer › LOB › Product**: for example Expenova › CUSTCAP › IDFCFIRST › Tripstacc › Flights, and PeopleOCity › UPPERCASE. Each client's stakeholders see only their own client's data, and staff see only their own company's clients.

The app runs audits, scoring, feedback, disputes, allocations, volume-based audit targets and auditor plan adherence. It also covers learning (JIT trainings, briefing notes and quizzes), calibration, dashboards, a reports catalogue and an agent workspace. The stack is Next.js 14 (App Router), TypeScript, PostgreSQL (Supabase) through Drizzle ORM, hosted on Vercel, with an Apple-style interface (system fonts, translucent sidebar, grouped surfaces, light and dark), and NextAuth with Microsoft 365 (Entra ID) and password sign-in.

## Deploy

See **[DEPLOY.md](DEPLOY.md)** for the step-by-step GitHub + Supabase + Vercel + Microsoft 365 set-up. These are the deployment files:

| File | What it does |
|---|---|
| `.env.example` | Every environment variable, with where to get it |
| `vercel.json` | Mumbai region (bom1), the two scheduled jobs, function timeout |
| `.github/workflows/ci.yml` | On every pull request: typecheck, tests, migrations on a clean Postgres, build |
| `.github/workflows/database.yml` | Applies migrations to Supabase when `drizzle/` changes on `main`; the first-run setup on demand |
| `drizzle/` | SQL migrations. `0001_supabase_lockdown.sql` switches on row level security and closes Supabase's Data API roles; it re-runs after every migration |
| `scripts/check-rls.sql` | Quick check in the Supabase SQL Editor that every table is locked down |

The database starts empty. The first-run setup (`npm run db:setup`, or the GitHub Database workflow) creates:
- the companies, clients, CUSTCAP's customers, LOBs and products;
- the call and email forms with the full evaluation legend;
- your admin login.

No sample data or demo logins are created.

## Run it locally

```bash
npm install
cp .env.example .env.local      # set NEXTAUTH_SECRET and NEXTAUTH_URL=http://localhost:3000; remove DATABASE_URL and DIRECT_URL
ADMIN_EMAIL=you@zipstacc.com ADMIN_PASSWORD='a-12+-character-password' npm run db:setup   # embedded Postgres in ./.pglite
npm run dev                     # http://localhost:3000, sign in with ADMIN_EMAIL's username and ADMIN_PASSWORD
```

To start again, delete `.pglite` and run the setup again.

### Microsoft 365 sign-in (staff and client stakeholders)

Staff of both companies sign in with their **zipstacc.com** accounts. Each client's stakeholders sign in with their own work accounts, for example **customer-capital.com** for CUSTCAP. The domain decides who someone is: a zipstacc account is staff, and a client domain belongs to exactly one client and sees only that client.

1. **Register the app.** In the zipstacc Entra ID › App registrations, create a **New registration** with redirect URI `https://<your-domain>/api/auth/callback/azure-ad`. Copy the client ID, create a client secret, and set `AZURE_AD_CLIENT_ID` and `AZURE_AD_CLIENT_SECRET`.
2. **Choose how client stakeholders sign in.** Pick one of these:
   - **Guests (simplest, recommended to start).** Keep the app *single tenant* and set `AZURE_AD_TENANT_ID` to the zipstacc tenant ID. Invite each client stakeholder as a B2B guest (Entra ID › Users › Invite external user). They sign in with their own customer-capital.com password and MFA, and IT stays in control of who is invited.
   - **Their own tenant.** Set *Supported account types* to *Accounts in any organizational directory (multitenant)* and `AZURE_AD_TENANT_ID=organizations`. The client's admin consents to the app once. Then add the client's **tenant ID** on the client under Admin › Master data, and the zipstacc tenant ID under Admin › Login & SSO (or `STAFF_TENANT_IDS`). In this mode a sign-in from any tenant not listed is refused, because an email domain alone is not proof of identity across tenants.
3. **Set the domains.** Staff domains are under Admin › Login & SSO (default `zipstacc.com`). Each client's domains are under Admin › Master data › the client › *Microsoft 365 sign-in* (seeded from `CUSTCAP_SSO_DOMAINS`, default `customer-capital.com`). A domain can belong to only one client and never to staff.
4. **Decide who gets in.**
   - **Staff:** an existing user with that email is linked on first sign-in. Otherwise the user's Entra ID **group** decides the role and company (Admin › Login & SSO maps group object ID → role and company; add a *groups claim* under Token configuration).
   - **Client stakeholders:** add each person under Admin › Users with the role *Client stakeholder*, their client and their work email. Alternatively, tick *Create a read-only stakeholder login automatically* on the client to let anyone on that client's domain in as a read-only viewer.
   - After the first sign-in the login is bound to that Microsoft account (tenant + object ID), so a different account with the same email is refused.
5. **Allow email sending.** Add the **Mail.Send** *application* permission (Microsoft Graph) and grant admin consent. Then set `GRAPH_SENDER` to the sending mailbox. You can limit the app to that one mailbox with an Exchange application access policy.
6. **Set up Teams.** Create a Workflows "Post to a channel when a webhook request is received" flow in the QA channel and put its URL in `TEAMS_WEBHOOK_URL`.

The rules are in `src/lib/sso.ts` (unit-tested in `tests/sso.test.ts`) and the sign-in callback in `src/lib/auth.ts`.

### Google sign-in

When `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` are set, a **Sign in with Google** button appears.
- The same domain rules apply: a staff domain, or a client's domains.
- Only verified **Google Workspace** accounts are accepted: Google's hosted-domain claim must match the email's domain.
- Google sends no groups, so staff who use Google are added under Users first.
- The login is bound to the Google account (`users.google_sub`) on first sign-in.

The set-up steps are in DEPLOY.md, step 5b.

### Data scope

`src/lib/scope.ts` decides what each person sees. Every audit carries its `clientId`, so one check covers dashboards, trends, reports, audit lists and audit pages.
- A **client stakeholder** sees only their client.
- **Staff with a company** (Expenova or PeopleOCity) see only that company's clients, agents, customers, plan lines and learning audiences, and company admins manage only their own company's users and master data.
- **Group-wide** staff (no company; for example the group admin) see everything and get a *Client* slicer.

## Roles

All rules are enforced on the server, both in the server actions and in the page loaders. The Admin › Roles & permissions page shows the full matrix.

| Role | Summary |
|---|---|
| Admin | Everything, including users, forms, master data, sign-in, notifications, best practices and the activity log |
| Manager | All dashboards, reports, audits and the feedback tracker; learning and quiz grading; calibration results. Cannot audit or change settings |
| QA Lead | Like Admin for quality work: all audits, allocations, targets, forms, imports, calibration sessions and disputes |
| QA Auditor | Audits **any** agent; own queue and targets; disputes on own audits; learning; calibration participant |
| Team Leader | Audits **only agents mapped to them** (the server rejects any other agent); team views; feedback logging |
| Agent | Agent workspace; acknowledge or dispute; learning and quizzes; own trends. Belongs to a client program and one of its LOBs |
| Client stakeholder | Read-only dashboards, client-safe reports and audits of **their own client only** (phone numbers masked). Signs in with Microsoft 365 on the client's domain, or a password |

## Plan adherence

**Add client / products** on `/targets` adds a new or existing client with several products at once, across LOBs. Tick each product the client takes, or add a new one inline. Each product gets its own projected monthly calls and emails, and one plan line is created per product and channel. New clients, LOBs and products go into Master data. The server action is `addClientProducts` in `src/app/actions/ops.ts`. The single-line **Add line** editor stays for one-off lines.

**Import past audits** has a **Download sample file** button. It returns an .xlsx built for the chosen form, with headings the importer recognises, two example rows and an Instructions sheet. The route is `GET /api/import/sample?formId=` and the builder is `src/lib/importSample.ts`.

**JIT training and briefing attachments.** Authors can upload up to 10 files per item, each up to 4 MB: PDF, PowerPoint, Word, Excel, CSV, text, images, audio and video. Files are stored in the `learning_files` table (bytea) and served by `GET /api/learning/files/[id]`. Staff who create learning can open them. An agent can open them only for items assigned to that agent. The 4 MB cap keeps uploads under Vercel's 4.5 MB request limit. For larger media, link to SharePoint or OneDrive, or move storage to Vercel Blob or S3.

`/plan` ("My plan" for QA Auditors; "Plan adherence" with an auditor picker for QA Leads, Managers and Admins) compares the auditor's slice of the volume plan with the audits they have done this week or month. It breaks this down by channel, client, LOB · product, plan line and agent, with a mix-match score. **Audit next** lists the lines furthest behind pace, each with the agent who needs that audit most, and opens New audit pre-filled. The New audit pane also shows a "Plan fit" card. The logic is in `src/lib/adherence.ts` and is covered by `tests/adherence.test.ts`.

## Scoring rules

- **Overall** = weight of parameters rated Y or NA ÷ total weight × 100. NA counts as met.
- **Actual** = 0 when any fatal parameter is rated N; otherwise it equals Overall.
- Every parameter counts. The original sheet left out the CRM tagging column, which capped the score at 95; that is fixed here.
- Tests cover the two real audits from the sheet: CC66120 scores 88% / 0%, and CC68546 scores 100% / 100%.

## Project layout

```
src/db/schema.ts          data model (Drizzle)
src/db/seed.ts            first-run setup: companies, clients, master data, forms, admin login (no sample data)
src/db/master-data.json   call and email forms with the full rating legend
drizzle/                  SQL migrations (npm run db:generate after schema changes)
src/lib/                  scoring, rbac, quiz, calibration, targets, adherence, feedback, reports, notifications
src/app/actions/          server actions (all writes; validated with zod and permission-checked)
src/app/(app)/            pages: dashboard, trends, reports, audits, queue, allocations, targets, plan,
                          tracker, disputes, learning (+ quiz builder and grading), calibration,
                          me (agent workspace), admin/*, legend, help
tests/                    unit tests (npm test)
```

## Commands

| Command | What it does |
|---|---|
| `npm run dev` / `build` / `start` | Next.js |
| `npm test` | Unit tests: scoring, roles, quiz, calibration, targets, plan adherence, feedback |
| `npm run typecheck` | TypeScript check |
| `npm run db:generate` | New migration after editing `schema.ts` |
| `npm run db:migrate` / `db:seed` / `db:setup` | Apply migrations / seed / both |

## Known limits and next steps

- **Best practices.** Each one needs Admin approval before it takes effect. "Dispute window" and "Area of improvement for every No" are enforced. Coaching tasks, minimum sampling, lock-after-acknowledgement and the monthly calibration reminder are recorded but not yet enforced.
- **Sign-in throttling.** It is held in memory per server instance. For multi-region production, move it to a shared store such as Upstash Redis.
- **Report scale.** Reports and dashboards aggregate in the app server. That is fine for tens of thousands of audits; beyond that, move the heavy reports to SQL views.
- **Same LOB name under two clients.** LOB names are unique across the group, so give each client its own LOB names (for example "UPPERCASE Support").
- **Learning, calibration and best-practice settings** are shared across the group; learning audiences and calibration audits are limited to the author's company.
- **Custom reports.** They are placeholders until the formats are shared. Each one is then added as a definition in `src/lib/reports.ts`.

## Push to GitHub

```bash
git remote add origin https://github.com/<org>/qa-studio.git
git branch -M main
git push -u origin main
```
