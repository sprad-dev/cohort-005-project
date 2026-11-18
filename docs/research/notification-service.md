# Which notification service fits Cadence?

Research for issue #2. Researched 2026-11-17 against vendor docs, pricing pages, npm registry metadata and the published package tarballs. Prices and package versions change, so re-check them before signing up.

**Cadence's needs:** a saved, in-app **Inbox** of **Notifications**. The service stores Notifications and their read state. Cadence only sends events from server actions. The app uses React 19.2, React Router 8.3 (SSR), Tailwind 4 and shadcn. Email later is a plus.

## Recommendation

**Use Novu (Novu Cloud, free tier).** **Knock is the runner-up.**

Why Novu:

- **Pricing grows gently.** The free tier has 10K workflow runs a month. The next tier, Pro, starts at $30 for 30K, then costs $1.20 per 1K. Knock, Courier and MagicBell all go from free to about $250 a month, or charge per send from the first message.
- **It is the best fit for Tailwind and shadcn.** `appearance.elements` accepts Tailwind class strings for each part of the Inbox. Render props (`renderNotification`, `renderBell`, …) replace whole parts. Headless hooks (`useNotifications`, `useCounts`, `useNovu`) let us build the Inbox entirely from shadcn components.
- **It is safe under SSR by design.** `@novu/react` has a `server` export condition that renders a stub on the server, so React Router's SSR will not crash. The Inbox itself mounts only on the client.
- **There is an exit path.** The core is MIT-licensed and self-hostable, so we are not locked into the vendor.
- **Identity checks are easy.** Verification is a single HMAC-SHA256 of the user id (`subscriberHash`) using `node:crypto`, with no extra dependency.

Pick Knock instead if we would rather pay for maturity. Knock has the cleaner provider and hook API, a ready-made token signer, and a bigger email provider list. Its catch is the jump from Free (10K messages) to Starter at $250 a month, and it cannot be self-hosted.

## Comparison

| Criterion | **Novu** | **Knock** | **Courier** | **MagicBell** | **SuprSend** |
|---|---|---|---|---|---|
| Free tier | 10K workflow runs/mo, unlimited subscribers, 24h activity-feed (log) retention [N1] | 10K messages/mo, Knock branding, no hard cap (sales contacts you) [K1] | 10K sends/mo [C1] | **1K deliveries/mo**, 1 project [M1] | 10K notifications/mo, SuprSend branding on feed [S1] |
| Next tier | Pro from **$30** (30K, then $1.20/1K). Team from $250 (250K) [N1] | Starter **$250/mo** (50K, then $0.005/msg) [K1] | Business: **$0.005 per send from the first one**, so 50K = $250 [C1] | Startup **$249/mo** (50K, then $0.0025) [M1] | Essentials **$100/mo** (50K, then $0.002). Business $250 [S1] |
| React package | `@novu/react` 3.20.0 | `@knocklabs/react` 0.15.1 | `@trycourier/courier-react` 9.3.5 | `@magicbell/react` 1.14.0 | `@suprsend/react` 1.6.0 |
| React 19 peer dep | `^18 \|\| ^19` [npm] | `^17 \|\| ^18 \|\| ^19` (next / tanstack peers are optional) [npm] | `>=18` [npm] | none declared [npm] | `>=16.8` [npm] |
| SSR behavior | `server` export condition renders a stub (hidden `<div>`) on the server. The Inbox mounts on the client [pkg] | ESM entry starts with `"use client"`. No SSR guidance in the docs [pkg][K3] | "render client-side only" (docs) [C3]. Built from custom elements in Shadow DOM [pkg] | Source files carry `"use client"`. No SSR guidance [pkg][M2] | No SSR guidance found. Uses Emotion for styles [npm] |
| Restyling | `appearance.variables`, `elements` (**Tailwind classes accepted**), `icons`, `baseTheme`. Styles are injected into `<head>` (regular DOM) [N3] | `--rnf-*` CSS variables, `rnf-` classes, `renderItem`, `renderHeader`. Default CSS is optional [K3] | Theme objects only (`CourierInboxTheme`). Components live in **open Shadow DOM**, so app Tailwind classes cannot reach inside [C3][pkg] | CSS variables (`--magicbell-*`), predefined class names, `unstyled` on buttons [M2] | Theme object plus Emotion. Headless option documented [S2] |
| Headless | `NovuProvider` with `useNotifications`, `useCounts`, `useNovu` (mark read, archive, `novu.on()` events) [N4] | `useKnockClient`, `useNotifications`, `useNotificationStore`. You must call `markAsSeen` yourself [K4] | `useCourier()` returns `inbox.totalUnreadCount` etc. You must call `inbox.listenForUpdates()` [C3] | Not documented on the React page [M2] | "Headless" SDK listed as an option [S2] |
| Trigger from server action | `@novu/api`: `novu.trigger({ workflowId, to: subscriberId, payload })` [pkg README] | `@knocklabs/node`: `knock.workflows.trigger(key, { recipients, data })`. Supports `Idempotency-Key` [pkg][K5] | `@trycourier/courier` 10.0.1 (send API) [npm] | REST API, `magicbell-js` [M3] | `@suprsend/node-sdk` 1.15.1 [npm] |
| Viewer identity | **HMAC-SHA256** of `subscriberId` with `NOVU_SECRET_KEY`, passed as `subscriberHash`. Optional. When on, the Inbox will not load without it [N2] | **RS256 JWT** user token ("enhanced security mode"). `signUserToken(userId, { expiresInSeconds })` defaults to 3600s. Without it, "a request with your public key can act as any user" [K2][pkg] | JWT minted by calling `POST /auth/issue-token` with scopes `user_id:<id> inbox:read:messages inbox:write:events`. **No `expires_in` means it never expires**. No auto-refresh [C2] | **HS256 JWT** signed with the project Secret Key, claims `user_external_id` / `user_email` [M3] | **ES256 JWT** (`entity_type: subscriber`, `entity_id`), `refreshUserToken` callback. Not verified in Sandbox [S3] |
| Live updates | WebSocket (socket.io / partysocket in `@novu/js`). Hooks update in realtime [N4][npm] | WebSocket (Phoenix in `@knocklabs/client`). Disconnects inactive tabs. Optional cross-tab BroadcastChannel [K3][K6][npm] | WebSocket, but only after `listenForUpdates()` [C3] | "persistent connection". Transport not documented (`reconnecting-websocket` dependency) [M4][npm] | Realtime (not detailed) |
| Email later | 19 providers incl. Resend, Postmark, SES, SMTP. Demo provider for tests [N5] | 13 providers incl. Resend, Postmark, SES, SMTP. "Knock (testing only)" [K7] | Core multichannel product | SES, Mailgun, SendGrid, SMTP, … [M4] | Yes |
| Self-host | **Yes**: MIT core, `enterprise/` folders under a commercial licence. Needs MongoDB, 2× Redis, S3-compatible storage [N6][N7] | No | No | No | Enterprise / BYOC only [S1] |

## Facts that matter for implementation

### Identity (all vendors)

The browser gets a **public** key plus a server-minted proof of who the user is. Mint the proof in the root `loader` from the session user, never in the client.

- **Novu:** `createHmac('sha256', process.env.NOVU_SECRET_KEY).update(subscriberId).digest('hex')`. This hash never expires, so one value per user is enough. Turn on "HMAC encryption" in the In-App provider settings in production [N2].
- **Knock:** `KNOCK_SIGNING_KEY` (RS256 private key, shown once). Match the token's expiry to the session, or refresh it via `onUserTokenExpiring` [K2].
- **Courier:** always pass `expires_in` [C2].

### React 19 and React Router 8 SSR

None of the vendors document React Router or Remix.

- **Novu** is the only package built to be safe under SSR at the package level (the `exports["."].import.default` server entry is a stub).
- **Knock / MagicBell** mark themselves `"use client"`. That directive means nothing to React Router's non-RSC SSR, so the components will execute on the server. Whether they touch `window` during render is **unverified**.
- **Courier** says it renders client-only.

For any vendor, wrap the Inbox in a client-only boundary: render it after a `useEffect` mount flag, or with `React.lazy` plus `<Suspense>`. Our own **Inbox** page and bell badge would use headless hooks inside that boundary.

### Styling against Tailwind and shadcn

- **Novu:** `elements` accepts Tailwind classes, and styles are injected into the regular DOM `<head>` [N3]. Tailwind 4 only generates classes it finds in scanned sources, so the classes must appear in our `.tsx` files (they will). The Inbox UI itself is rendered by Solid.js inside `@novu/js` (`solid-js`, `@kobalte/core`, `tailwind-merge` dependencies; about 2.5 MB unpacked). To get fully native shadcn markup, use the headless hooks.
- **Courier:** this is the main styling limit. Its UI is a set of custom elements in an **open Shadow DOM** (`attachShadow({ mode: "open" })` in `@trycourier/courier-ui-inbox`), so we can only restyle through its theme objects, not with our CSS or Tailwind.
- **Knock:** pulls in its own design system (`@telegraph/*`, Radix dialog and hover card, `lucide-react`). The headless hooks avoid all of it.

### Local dev with switched users (DevUI panel)

- **Map Cadence `users.id` to the service's user id.** That is Novu `subscriberId`, Knock `user.id`, Courier `user_id`, MagicBell `user_external_id` and SuprSend `distinct_id`.
- **Re-mount the provider when the DevUI switches users** (`key={userId}`). Every provider opens a socket and caches its feed for one user.
- **The service learns about each user from the first trigger.** Novu creates the subscriber from `to`. Knock identifies the recipient inline. So seeded users need no separate sync step until we want names and emails stored there.
- **Use the vendor's dev environment and keys locally.** Novu and Knock both have per-environment keys.
- **Identity checks may be off in dev.** Novu HMAC and Knock enhanced security are per-environment switches. SuprSend explicitly does not verify tokens in Sandbox [S3]. Keep verification **on** in dev, so that a wrong-user token fails locally and not first in production.
- **The Inbox does not refresh itself in a background tab.** Two browser tabs as two different users both receive live updates. Knock's socket disconnects inactive tabs after 2 s by default, so a backgrounded tab updates when refocused [K3].

## Sources

npm metadata was read with `npm view <pkg> version peerDependencies dependencies exports`. Tarballs were inspected with `npm pack` on 2026-11-17 and are marked [pkg] / [npm] above.

- [N1] https://novu.co/pricing
- [N2] https://docs.novu.co/platform/inbox/prepare-for-production
- [N3] https://docs.novu.co/platform/inbox/react/styling and https://docs.novu.co/platform/sdks/react
- [N4] https://docs.novu.co/platform/inbox/headless-mode
- [N5] https://docs.novu.co/platform/integrations/email
- [N6] https://docs.novu.co/community/self-hosting-novu/overview
- [N7] https://github.com/novuhq/novu (README licence section)
- [K1] https://knock.app/pricing
- [K2] https://docs.knock.app/in-app-ui/security-and-authentication
- [K3] https://docs.knock.app/in-app-ui/react/feed
- [K4] https://docs.knock.app/in-app-ui/react/custom-notifications-ui
- [K5] https://docs.knock.app/reference
- [K6] https://docs.knock.app/in-app-ui/feeds/overview
- [K7] https://docs.knock.app/integrations/email/overview
- [C1] https://www.courier.com/pricing
- [C2] https://www.courier.com/docs/platform/inbox/authentication
- [C3] https://www.courier.com/docs/sdk-libraries/courier-react-web
- [M1] https://www.magicbell.com/pricing
- [M2] https://www.magicbell.com/docs/libraries/magicbell-react
- [M3] https://www.magicbell.com/docs/api/authentication/user
- [M4] https://www.magicbell.com/docs/channels/in_app
- [S1] https://www.suprsend.com/pricing
- [S2] https://docs.suprsend.com/docs/inbox-react
- [S3] https://docs.suprsend.com/docs/client-authentication

### Not verified

- Whether Knock's and MagicBell's React components render without errors on the server. Spike it, or wrap them client-only.
- Exactly what Novu's "24h activity feed retention" covers on Free. It reads as delivery logs, not stored Inbox Notifications, but the pricing page does not say.
- SuprSend's styling API and realtime transport. The detailed pages returned 404 or were not reached.
