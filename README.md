# Product Bot

Product Bot uses GitHub Pages for its dashboard, a Cloudflare Worker with D1
for OAuth, configuration, interactions, and billing, plus a persistent Node.js
Discord Gateway process for server events. GitHub Pages and the Worker do not
replace the always-on Gateway process.

## Implemented Discord features

- Discord OAuth sign-in, server selection, and a bot invite link.
- A GitHub Pages landing page that links to the dashboard. OAuth and billing
  return links at the site root are forwarded to the dashboard without losing
  their query parameters or session hash.
- Persisted verification and ticket-panel configuration.
- Publish real Discord panels with buttons. Verification assigns the selected
  role. Tickets create private channels and can be closed by the opener, support
  role, or a server moderator.
- Configurable application panels with Discord modal forms (1-5 questions),
  review-channel notifications, and persistent submissions. Each server
  can publish three panels for free; additional publications require Product Bot
  Premium. Panel images use public HTTPS URLs; direct file uploads are not
  supported.
- Live member, verification, ticket, and activity counts.
- Custom Design modules: **Verifizierung**, **Ticket-Panel**, and
  **Bewerbungs-Panel**. Premium design supports public HTTPS image URLs.
  Panel title, description, button-label, and application-question previews
  reflect the text that is saved and published to Discord. Welcome and goodbye
  templates also have live examples and support `{user}`, `{username}`, and
  `{server}` placeholders.
- Premium modules: **Server-Statistiken** (`/productstats`), native Discord
  polls (`/communitypoll`, with 2-5 answers), formatted announcements
  (`/announce`), server details (`/serverinfo`), member details (`/userinfo`),
  and avatars (`/avatar`). Buy Premium, enable the module switches, then click
  **Discord-Befehle aktualisieren** to register the guild commands. This also
  registers the free moderation commands; premium commands still require the
  corresponding active subscription and enabled module.
- Administrators can use `/dashboard` to open the GitHub Pages dashboard from
  Discord. The command is hidden from other members and checks the caller's
  Administrator permission again when invoked. Click **Discord-Befehle
  aktualisieren** after deploying to register the new command.
- Moderation: **Warnsystem** with `/warn` and `/warnings`; warnings are stored
  in D1 and permission-checked. **AutoMod** removes configured Discord invites
  and messages exceeding the mention limit. **Anti-Raid** reports join bursts.
  **Raid-Modus** additionally blocks Discord invite links for five minutes after
  a configured join burst; it never kicks or bans members.
- Gateway modules: **AutoMod**, **Anti-Raid**, **Anti-Spam**, **Anti-Link**, **Wortfilter**,
  **Account-Alter-Check**, **Willkommensnachrichten**,
  **Abschiedsnachrichten**, **Auto-Rollen**, **Nachrichten-Logs**,
  **Voice-Logs**, **Rollen-Logs**, **Join-/Leave-Logs**, and **Moderations-Logs**. Their settings
  live on the dashboard's server settings page. The Anti-Spam role exclusion is
  a Premium-only option; custom panel images and custom panel buttons are
  Premium design options.
- Tickets support private channels, one open ticket per member, categories,
  optional support-role pings, priority marking, closing, and optional transcript
  storage in D1 plus output in the closed ticket channel. Enable **Ticket-Transkripte**
  or **Ticket-Transkript-Export** for transcripts.
- The module page can enable all implemented features for the selected server
  in one confirmed action and sync slash commands. Anti-Link can remove every
  link unless allowed domains are configured.
- Every implemented module card has a **Konfigurieren** action: it opens the
  corresponding panel or highlights that module's existing settings. Slash
  command modules show the command and any options that are entered when the
  command is run. Planned modules remain unavailable.
- The protected admin page shows server usage, confirmed Stripe events, and
  premium gifts. Admins can find an exact username/display-name match within a
  configured server and grant that Discord account 30 days of Premium. A
  recipient can activate account Premium on servers they manage by signing in
  and saving that server's configuration; the linked account entitlement then
  enables Premium features for that server while it remains active.
- 54 module entries are displayed. Thirty are implemented; the remaining
  24 stay disabled and marked as planned.

The Worker uses Discord's signed HTTP interaction endpoint, while event-driven
modules use the separately hosted Gateway process in `bot/`. The gateway
requires the privileged **Server Members Intent** and **Message Content Intent**
in the Discord Developer Portal.

## Local checks

Run the backend interaction and dashboard consistency tests with Node.js 20 or
newer:

```powershell
node --test worker/src/index.test.mjs
node --check worker/src/index.js
```

For the Gateway process, from `bot/`:

```powershell
npm ci
npm test
npm run check
```

The tests use mocked Discord responses. They do not replace testing with a real
Discord application, Stripe account, and deployed Cloudflare Worker.

## GitHub Pages deployment

1. Create a GitHub repository and push these project files to its `main` branch.
2. In **Settings → Pages**, select **GitHub Actions** as the source.
3. The workflow publishes the dashboard as the site homepage and as
   `dashboard.html`. The first deployment uses a placeholder
   Worker URL; the website will explain that the API needs configuration.
4. After deploying the Worker, create the repository variable
   `PRODUCT_BOT_API_URL` with its base URL (for example
   `https://product-bot-api.example.workers.dev`) and push a commit or run the
   workflow again.
   The workflow includes the supplied public Discord application ID for the
   invite link by default. To use a different application, set the repository
   variable `PRODUCT_BOT_CLIENT_ID` to its 17-20 digit application ID.

For a project site, set `PAGES_URL` to
`https://YOUR-GITHUB-USERNAME.github.io/YOUR-REPOSITORY` and `PAGES_ORIGIN` to
`https://YOUR-GITHUB-USERNAME.github.io`. For a user site, the two URLs are the
same.

## Cloudflare Worker and D1

Install and authenticate Wrangler:

```powershell
npm install --global wrangler
wrangler login
```

Create the database and initialize the schema:

```powershell
wrangler d1 create product-bot
wrangler d1 execute product-bot --remote --file worker/schema.sql
```

Put the returned database ID in `wrangler.toml`, set `PAGES_URL` and
`PAGES_ORIGIN`, and deploy once:

```powershell
wrangler deploy
```

Copy the `workers.dev` URL Wrangler prints into `API_PUBLIC_URL` in
`wrangler.toml`, then deploy again. This URL is also the value for the GitHub
repository variable `PRODUCT_BOT_API_URL`.

Add these values as Worker secrets. Never commit or share them:

```powershell
wrangler secret put DISCORD_CLIENT_ID
wrangler secret put DISCORD_CLIENT_SECRET
wrangler secret put DISCORD_PUBLIC_KEY
wrangler secret put DISCORD_BOT_TOKEN
wrangler secret put SESSION_SECRET
wrangler secret put INTERNAL_SERVICE_TOKEN
wrangler secret put STRIPE_SECRET_KEY
wrangler secret put STRIPE_WEBHOOK_SECRET
wrangler secret put ADMIN_DISCORD_USER_IDS
```

`SESSION_SECRET` must be a long, random value. Stripe secrets can be added after
the Discord features are deployed; premium checkout remains unavailable until
both Stripe secrets are configured.

Set `ADMIN_DISCORD_USER_IDS` to a comma-separated allowlist of Discord user
IDs (Snowflakes), not usernames. Only those accounts can access admin overview,
purchase, and gift APIs. The status endpoint reveals only whether the signed-in
user is allowlisted. Add or remove IDs by updating this Worker secret.

Generate a separate long random value for `INTERNAL_SERVICE_TOKEN`. This secret
must be identical in the Worker and the private Gateway process environment; do
not use the Discord bot token for it.

## Discord Developer Portal

1. Create an application and bot, then copy its client ID, client secret, bot
   token, and public key to the Worker secrets above.
2. Add this OAuth redirect URL to the application:
   `https://YOUR-WORKER.workers.dev/oauth/callback`
3. Set the **Interactions Endpoint URL** to:
   `https://YOUR-WORKER.workers.dev/discord/interactions`
4. In **Bot → Privileged Gateway Intents**, enable **Server Members Intent**
   and **Message Content Intent**. Invite the bot with the dashboard's **Bot
   einladen** link. It needs Manage Roles, Manage Channels, View Channels, Send
   Messages, Read Message History, Embed Links, and Manage Messages for the
   moderation modules. Move the bot's role above the roles it assigns.
5. In the dashboard, sign in with Discord, choose a server, set channels/roles
   (including the application and review channels), save, then publish panels.

## Persistent Gateway process

Deploy `bot/` as a continuously running Node.js 22+ service or use its
`Dockerfile`. Copy `bot/.env.example` to the host's environment settings and set
`DISCORD_BOT_TOKEN`, `WORKER_API_URL`, and `INTERNAL_SERVICE_TOKEN`. Set
`PORT` if required by the host. The `/health` endpoint returns HTTP 200 while Discord
and the Worker's config/heartbeat API are reachable; it reports the number of
loaded server settings.
The service polls the Worker's private configuration endpoint every 30 seconds.
The Worker must also have the same `INTERNAL_SERVICE_TOKEN` secret set before
the bot process can load server configurations.

For local development, create `bot/.env` from `bot/.env.example` and run
`npm run start:local` from `bot/`. Keep `.env` private; it is ignored by Git.

The Gateway process uses the bot token only on the server. Never deploy it in
the Pages site or expose the internal service token to browser code.

The application requests only `identify` and `guilds` OAuth scopes. Do not put
your bot token in the dashboard or browser.

### Optional free Gateway hosting

The Pages landing page loads the bot's Discord invite URL from the deployed
Worker's public configuration endpoint once `PRODUCT_BOT_API_URL` and
`DISCORD_CLIENT_ID` are configured. The dashboard also provides the invite
action after setup.

For persistent Gateway hosting, deploy `bot/` on a continuously running Node.js
host or VM and configure its private environment as described above. The
dashboard is the only public Pages entry point; hosting setup is not published
as a separate site page. The Gateway token and `INTERNAL_SERVICE_TOKEN` belong
only in the host's private environment; the Worker must use the same internal
token.

## Premium and payments

Custom Design costs **$2.99 USD/month**. Product Bot Premium costs **$9.99
USD/month** and includes Custom Design plus the six Premium modules above.
Stripe Checkout offers card and PayPal for monthly subscriptions only when
those methods are enabled for the Stripe account. A separate $2.99 one-time
Custom Design checkout grants 30 days and is not a subscription.

Configure Stripe's customer portal in the Stripe Dashboard. Add a webhook
endpoint at
`https://YOUR-WORKER.workers.dev/api/billing/webhook` and subscribe to:

- `checkout.session.completed`
- `checkout.session.async_payment_succeeded`
- `customer.subscription.updated`
- `customer.subscription.deleted`
- `invoice.payment_failed`
- `invoice.paid`

The one-time Custom Design checkout accepts cards only. Girocard acceptance is
**not guaranteed** by the online card checkout; confirm it with Stripe before
advertising it. The app never stores card details.

## Important limitations

- The remaining 24 module cards are not implemented bot features. They are
  intentionally marked as planned and cannot be enabled.
- D1 stores application submissions, moderation warnings, account Premium
  gifts, confirmed billing events, and admin audit records. Apply
  `worker/schema.sql` to create the new tables before deploying this version.
- Application answers are also delivered to the configured Discord review
  channel. D1 submissions currently have no automatic retention or dashboard
  deletion control; manage their retention in Cloudflare D1.
- The Worker must be publicly reachable over HTTPS for Discord OAuth and signed
  interactions. Use Discord and Stripe test modes before accepting real users
  or payments.
- Creating/publishing a GitHub repository requires your own GitHub account and
  authentication. No credentials are included in this project.
