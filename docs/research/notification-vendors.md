# Hosted in-app notification vendors: Knock vs Novu Cloud vs MagicBell vs Courier

Research for issue #5 (map #4, "In-app notifications for Cadence"). Researched 2026-11-17 from each vendor's official docs and pricing pages. Knock, Novu and Courier publish their full docs as `llms-full.txt`; quotes from those files cite the canonical doc page they came from.

Constraints from the map: hosted SaaS only, in-app bell and feed only (email is a tie-breaker), must work on the free tier, all four roles (Student, Instructor, Platform Admin, Team Admin) can be recipients.

## Recommendation: Knock

1. **The most usable free tier.** Free includes 10,000 messages a month, no MAU cap, unlimited feeds, and unlimited isolated environments, so we get separate development and production ([pricing](https://knock.app/pricing)). Novu's free plan caps us at 5 workflows and 30-day inbox retention. MagicBell's free plan has 1,000 deliveries and 1 project.
2. **Auth is local and fits the DevUI user switcher.** The server signs an RS256 JWT with `signUserToken` from `@knocklabs/node` and needs no network call ([security](https://docs.knock.app/in-app-ui/security-and-authentication)). That makes it cheap to mint a token on every loader for whichever user DevUI has selected. Courier has to call its API to issue each token.
3. **No separate user sync.** Recipients are upserted inline in the trigger call, and `KnockProvider` upserts the user client-side ([identifying recipients](https://docs.knock.app/managing-recipients/identifying-recipients)). A trigger takes up to 1,000 recipients, which covers "notify every Platform Admin" in one call ([triggering](https://docs.knock.app/send-notifications/triggering-workflows/api)).
4. **The fullest state model.** Knock tracks `seen`, `read`, `interacted` and `archived` separately, and the feed returns both `unseen_count` and `unread_count`. The bell's badge is configurable as `unseen`, `unread` or `all` ([message statuses](https://docs.knock.app/send-notifications/message-statuses), [NotificationIconButton](https://docs.knock.app/in-app-ui/react/sdk/components/notification-icon-button)).
5. **Both a prebuilt UI and headless hooks.** You can drop in `NotificationFeedPopover` and `NotificationIconButton`, or build your own UI with `useKnockFeed` / `useNotifications` / `useNotificationStore` ([React overview](https://docs.knock.app/in-app-ui/react/overview), [headless](https://docs.knock.app/in-app-ui/react/headless/feed)). The websocket connection is managed by the SDK, and no plan gating is documented for it.

**Runner-up: Courier.** Its free tier is a match at 10,000 sends a month, and it has good React components and real-time. It loses on token issuance, which takes an API round-trip per user. It also has no separate "seen" state for an unseen-versus-unread badge, and sends from the Test environment count against the plan cap.

**Viable but constrained: Novu Cloud.** It has good components and hooks and the simplest auth (HMAC-SHA256 of the subscriber ID). But the free plan allows only **5 workflows**, which is roughly one per notification type, and Cadence's v1 list may exceed that. Inbox items are kept for 30 days, a trigger takes at most 100 recipients, and Novu branding can't be removed on the free plan.

**Ruled out: MagicBell.** The free "Builder" plan has only **1,000 deliveries a month and 1 project**, so there is no separate dev and prod. Its React headless hooks package is deprecated, so a custom UI means using the plain JS client. And the docs we read say nothing about plan gating for real-time.

## Comparison at a glance

| | Knock | Novu Cloud | Courier | MagicBell |
|---|---|---|---|---|
| Free volume | 10,000 messages/mo, no hard cap (Knock contacts you) | 10,000 workflow runs/mo | 10,000 sends/mo, hard cap (`THROTTLED`) | 1,000 deliveries/mo, per-use overage |
| Other free limits | Knock branding; 30-day log retention | 5 workflows, 2 environments, 30-day inbox retention, 1-day activity log, 3 team members, branding | All features on Developer; Test sends count toward cap | 1 project |
| Users / MAU cap | None listed ("unlimited" stored profiles) | Subscribers unlimited | None listed | "Unlimited Users" |
| React prebuilt | `NotificationFeedPopover`, `NotificationFeed`, `NotificationIconButton` | `<Inbox />` (with `Bell`, `Notifications`, `InboxContent`, `Preferences`) | `<CourierInbox />`, `<CourierInboxPopupMenu />`, `<CourierToast />` | `FloatingInbox`, `Inbox`, `InboxButton` |
| Headless | `useKnockFeed`, `useNotifications`, `useNotificationStore`, `FeedClient` | `useNotifications`, `useCounts`, `useNovu` | `useCourier()` → `inbox.*` | No React hooks (`@magicbell/react-headless` deprecated); `magicbell-js/user-client` + `magicbell-js/realtime` |
| Browser auth | Public key + RS256 user JWT signed locally (`signUserToken`) | `applicationIdentifier` + HMAC-SHA256(secret, subscriberId) | JWT issued by Courier API (`auth.issueToken`) with scopes | HS256 user JWT signed locally with project secret |
| Recipients per trigger | 1,000 (+ objects/subscriptions for larger) | 100 (+ topics) | Array in `to`, or lists/audiences | 1–1,000 per broadcast |
| States | seen, read, interacted, archived (+ un-) | seen, read, archived, snoozed (snooze paid) | read, opened, clicked, archived | read/unread, archived; "seen" not documented for inbox |
| Badge counts | `unseen_count`, `unread_count` | `useCounts` with `{read:false}` / `{seen:false}` filters | `totalUnreadCount`, per-feed `unreadCount` | `fetchUnreadNotificationsCount` |
| Real-time | Managed websocket; no plan gating stated | WebSocket on by default; no plan gating stated | Realtime connection on sign-in; no plan gating stated | WebSocket via `Realtime.listen`; plan gating not stated |
| Environments on free | Unlimited (dev, prod, custom) | Dev + Prod | Production + Test + up to 18 custom | 1 project |
| Email later | Yes | Yes | Yes | Yes |

## Knock

**React feed and headless.** `@knocklabs/react` ships `NotificationFeedPopover` ("a managed popover"), `NotificationFeed` (a full-page list) and `NotificationIconButton` ("a bell icon that shows the current count of unread or unseen notifications") ([React overview](https://docs.knock.app/in-app-ui/react/overview)). For a custom UI, `KnockFeedProvider` exposes `useKnockFeed()`, which returns `{ feedClient, items, metadata }`. The lower-level hooks are `useNotifications(knockClient, channelId)` and `useNotificationStore(feedClient)` ([headless feed](https://docs.knock.app/in-app-ui/react/headless/feed)). The `FeedClient` has `markAsSeen`/`markAllAsSeen`, `markAsRead`/`markAllAsRead`, `markAsUnread`, `markAsArchived`/`markAllAsArchived`, `markAsUnarchived`, `markAsInteracted`, `fetch` and `fetchNextPage` ([FeedClient reference](https://docs.knock.app/in-app-ui/javascript/sdk/reference)). The prebuilt popover calls `markAllAsSeen()` for you. With the hooks you call it yourself.

**Free tier.** $0: up to 10,000 messages a month, where a message is "a single message delivered to a single user on a single channel". Stored profiles and feeds per recipient are unlimited, and so are isolated environments. Free does not include branding removal or email client previews, and log retention is 30 days. There is no hard cap on messages: Knock contacts you about billing ([pricing](https://knock.app/pricing)).

**Node SDK and triggering.** Install `@knocklabs/node`, then:

```ts
const knock = new Knock({ apiKey: process.env.KNOCK_API_KEY });
await knock.workflows.trigger("refund-request-opened", {
  recipients: [{ id: "user_1", name: "Ada" }, "user_2"],
  actor: "user_9",
  data: { refundRequestId: 42 },
});
```

That is adapted from the Next.js quickstart in the docs. The trigger body takes `recipients`, `actor`, `data`, `tenant`, `cancellation_key` and `settings` ([trigger API](https://docs.knock.app/api-reference/workflows/trigger)). The docs say: "You can trigger a workflow for up to 1000 recipients at a time." For larger groups they point to [subscriptions](https://docs.knock.app/concepts/subscriptions), where an object such as `platform-admins` holds the subscribers ([triggering via API](https://docs.knock.app/send-notifications/triggering-workflows/api)).

**Auth and identity.** The client starts with the public API key. In **enhanced security mode** it must also send a signed user token. Knock calls that mode "required" before going to production. Your backend signs the token as an RS256 JWT with the environment's private signing key. `sub` is required, and `iat`/`exp` are recommended. `signUserToken` (from `@knocklabs/node`) reads `KNOCK_SIGNING_KEY`, and its expiry defaults to 3600 s. The browser passes the token as `userToken` to `KnockProvider`, which has an `onUserTokenExpiring` callback for refreshing it ([security and authentication](https://docs.knock.app/in-app-ui/security-and-authentication)). Users are identified by "upsert" in one of three ways: `PUT /v1/users/{id}`, inline in a trigger's `recipients`, or client-side when `KnockProvider` mounts with user data ([identifying recipients](https://docs.knock.app/managing-recipients/identifying-recipients)). The Cadence user ID can be the Knock user ID.

**State and badges.** "Engagement statuses are mutually inclusive": `seen` (`seen_at`), `read` (`read_at`), `interacted` (`interacted_at`), `link_clicked` and `archived` (`archived_at`). On the in-app channel, Knock and the React SDK manage these for you ([message statuses](https://docs.knock.app/send-notifications/message-statuses)). The feed API returns badge counts as `unread_count` and `unseen_count`, and "`unseen_count` usually drives badge counts". Archived messages are left out of the feed by default ([in-app APIs and real-time](https://docs.knock.app/in-app-ui/overview)). Each recipient gets their own message, so fan-out to several admins gives each admin their own read state.

**Real-time.** "Knock provides a real-time web socket connection" and sends a `new-message` event with updated badge counts. The SDKs handle reconnection, and you can customize auto-disconnect for inactive tabs ([real-time socket behavior](https://docs.knock.app/in-app-ui/feeds/socket-behavior)). The pricing page doesn't mention real-time, and no doc gates it by plan.

**Email later.** Email is a channel step in the same workflow, so adding it later means no code change at the trigger.

**Dev, test and automated tests.** You get a Development environment alongside Production, each with its own public and secret keys, and "data is never copied between environments" ([environments](https://docs.knock.app/concepts/environments)). The test runner and the API settings `settings.sandbox_mode` and `settings.skip_delay` let you run a workflow without delivering to providers. The call still reaches Knock, though ([testing workflows](https://docs.knock.app/send-notifications/testing-workflows)). Knock documents no offline mock. For automated tests, wrap `knock.workflows.trigger` behind a Cadence notifier module and inject a fake. Token signing is local and needs no network, so it can run in tests as-is with a throwaway key pair.

## Novu Cloud

**React feed and headless.** `@novu/react` provides `<Inbox />`, which is built from `Bell`, `Notifications`, `InboxContent` and `Preferences` ([Inbox](https://docs.novu.co/platform/inbox)). Headless mode uses `NovuProvider` with `useNotifications` and `useCounts`. "You handle the layout ... Novu provides the notification state, real-time updates, and actions" ([headless mode](https://docs.novu.co/platform/inbox/headless-mode)). `useNotifications` takes `read`, `seen`, `archived` and `snoozed` filters and a `realtime` flag. It returns `readAll`, `seenAll`, `archiveAll`, `archiveAllRead`, `fetchMore` and `refetch`. Each `Notification` has `isRead`, `isSeen`, `isArchived` and `isSnoozed`, plus methods such as `notification.read()` ([useNotifications](https://docs.novu.co/platform/sdks/react/hooks/use-notifications)). `useCounts({ filters: [{ read: false }, { seen: false }] })` drives badges ([useCounts](https://docs.novu.co/platform/sdks/react/hooks/use-counts)).

**Free tier.** $0 with no card: 10,000 workflow runs a month, **up to 5 workflows**, 2 environments (Dev and Prod), 3 team members and all channels ([pricing](https://novu.co/pricing)). The limits page adds 1 layout, 20 steps per workflow, a 1-day delay and digest window, **inbox feed retention of 30 days**, and activity-feed retention of 1 day ([limits](https://docs.novu.co/platform/developer/limits)). Removing Novu branding and snooze are not available on the free plan ([pricing](https://novu.co/pricing), feature matrix in [prepare for production](https://docs.novu.co/platform/inbox/prepare-for-production)). One workflow run is counted per subscriber triggered ([trigger concepts](https://docs.novu.co/platform/concepts/trigger)).

**Node SDK and triggering.** Install `@novu/api`, then call `new Novu({ secretKey }).trigger({ workflowId, to, payload })`. `to` can be one subscriber, an array, or a topic. "Maximum number of recipients can be 100" per trigger. Use topics for larger fan-out ([trigger event API](https://docs.novu.co/api-reference/events/trigger-event)). `triggerBulk` accepts up to 100 events. The worker upserts the subscriber from the `to` object on each trigger ([trigger concepts](https://docs.novu.co/platform/concepts/trigger)).

**Auth and identity.** The Inbox takes a public `applicationIdentifier` and a `subscriberId`. Without more security, "a malicious actor could potentially guess another subscriber's `subscriberId`". To prevent that, enable HMAC on the Novu In-App integration and pass `subscriberHash = HMAC-SHA256(secretKey, subscriberId)` as a hex string, computed on the server. This also authorizes the `subscriber` prop to update profile fields ([prepare for production](https://docs.novu.co/platform/inbox/prepare-for-production)). The hash never expires and is computed locally.

**Real-time.** Inbox and hooks open a WebSocket by default (`realtime` defaults to `true`) ([useNotifications](https://docs.novu.co/platform/sdks/react/hooks/use-notifications)). The docs we read don't gate it by plan.

**Dev, test and automated tests.** Development and Production are separate, with their own `applicationIdentifier`, secret key, subscribers and activity. Workflows are published from Dev to Prod. Custom environments need the Team plan or higher ([environments](https://docs.novu.co/platform/developer/environments)). "Keyless mode" lets you try the Inbox with no account ([setup inbox](https://docs.novu.co/platform/inbox/setup-inbox)). Novu documents no mock for its server SDK, so stub the trigger behind our own module.

**Why not.** Five workflows is the binding constraint: Cadence's v1 notification types each map to a workflow. The 30-day retention also means old notifications disappear from users' feeds on the free plan.

## Courier

**React feed and headless.** `@trycourier/courier-react` (React 18+) provides `<CourierInbox />`, `<CourierInboxPopupMenu />`, `<CourierToast />` and `<CourierPreferences />`, plus `useCourier()` ([React SDK](https://www.courier.com/docs/sdk-libraries/courier-react-web)). From `useCourier()`, a custom UI reads `inbox.feeds` and `inbox.totalUnreadCount`. It calls `readMessage`/`unreadMessage`, `openMessage`, `clickMessage`, `archiveMessage`/`unarchiveMessage`, `readAllMessages`, `fetchNextPageOfMessages` and `listenForUpdates` ([custom inbox](https://www.courier.com/docs/in-app/build-a-custom-inbox)).

**Free tier.** Developer plan, free: up to 10,000 sends a month, with Journeys, SDKs and CLI included. One template to one recipient is one send, however many channels it goes to. "Test and custom environments count the same as Production" ([pricing](https://www.courier.com/pricing)). Hitting the cap marks messages `THROTTLED` after the API has already returned `202` ([statuses](https://www.courier.com/docs/send/statuses)).

**Node SDK and triggering.** Call `client.send.message({ message: { to: { user_id }, template, data, routing: { method: "single", channels: ["inbox"] } } })` ([send to the inbox](https://www.courier.com/docs/in-app/send-to-the-inbox)). "Pass an array in `to` to reach several recipients", or send to a list or audience for large groups ([send overview](https://www.courier.com/docs/send/overview)). The docs we read state no per-call recipient cap.

**Auth and identity.** The backend calls `client.auth.issueToken({ scope: "user_id:user_123 inbox:read:messages inbox:write:events", expires_in: "2 days" })`, which is `POST /auth/issue-token`. The browser then calls `signIn({ userId, jwt })` ([authenticate users](https://www.courier.com/docs/in-app/authenticate-users)). **Each token takes a network call to Courier.** Recipients are addressed by `user_id`, and a stored profile is optional.

**State and real-time.** Read, opened, clicked and archived states sync across devices. "Once a user signs in, the inbox opens a realtime connection," and the socket uses the same JWT. A message stays until it is archived or reaches its `expiresAt` ([how the inbox behaves](https://www.courier.com/docs/in-app/how-the-inbox-behaves)). There is no "seen" state separate from "opened", and the badge is an unread count.

**Dev, test and automated tests.** Production and Test environments come by default, with up to 18 custom ones. Each key belongs to one environment ("Courier has no separate test mode"). Draft keys (`dk_…`) render unpublished templates ([environments](https://www.courier.com/docs/workspaces/environments)). There is no documented mock, so automated tests must also stub token issuance, not just sends.

## MagicBell

**React feed and headless.** `@magicbell/react` (v1.14.0) exports `ContextProvider` (which needs a `token`, the user JWT), `FloatingInbox` (toggled by `InboxButton`, a bell), `Inbox`, `InlineNotification`, `ToastContainer`, `UserPreferences` and push and Slack buttons ([React components](https://www.magicbell.com/docs/libraries/magicbell-react); package `exports` checked with `npm view`). The package exports no hooks. `@magicbell/react-headless` (v5.5.1) says: "This package is deprecated. Please use `@magicbell/react` or `magicbell-js` instead" (npm readme). A custom UI would use `magicbell-js/user-client`: `listNotifications`, `markNotificationRead`/`Unread`, `markAllNotificationsRead`, `archiveNotification`, `fetchUnreadNotificationsCount`. Real-time would use `magicbell-js/realtime` with `realtime.listen(cb)` over WebSocket ([magicbell-js](https://www.magicbell.com/docs/libraries/magicbell-js)).

**Free tier.** The "Builder" plan, $0: **1,000 deliveries a month** and **1 project**, with unlimited users and team members. Each channel a user receives a notification on counts as a delivery. Beyond the limit, deliveries are "charged per use". The next plan, Startup, costs $249 a month ([pricing](https://www.magicbell.com/pricing)).

**Node SDK and triggering.** Call `client.broadcasts.createBroadcast(broadcast)` from `magicbell-js/project-client`, which is `POST /broadcasts` authenticated with a Project JWT. `recipients` takes 1–1,000 entries, keyed by `external_id` or `email` ([magicbell-js](https://www.magicbell.com/docs/libraries/magicbell-js), [API reference](https://www.magicbell.com/docs/api/reference)). Users are upserted by external ID or email, and an external ID "cannot be changed or unset" once set ([user](https://www.magicbell.com/docs/primitive/user)).

**Auth.** The user JWT is signed with HS256 using the project secret key. It carries `api_key` plus `user_external_id` and/or `user_email`. The docs' example uses `expiresIn: '1y'` ([user auth](https://www.magicbell.com/docs/api/authentication/user)). Signing is local.

**State and real-time.** The in-app docs list read/unread, archive and delete. "MagicBell maintains a persistent connection to deliver notifications in real-time" ([in-app channel](https://www.magicbell.com/docs/channels/in_app)). The docs and pricing page we read don't say whether real-time is gated by plan, and they don't document a separate "seen" state.

**Why not.** At 1,000 deliveries, every fan-out to all Platform Admins and every test send eats into a small budget. With only one project, development and production share data. And a custom UI loses React hooks support.

## Testing approach (applies to whichever vendor we pick)

None of the four documents an offline or mock mode for its server SDK. The workable pattern is:

- Put a narrow Cadence module (for example, `notify(type, recipients, data)` and `feedTokenFor(userId)`) in front of the vendor SDK. Unit and integration tests inject a fake that records calls, so tests never hit the network.
- Run local dev and manual QA against the vendor's development environment with that environment's keys.
- With Knock and Novu the browser credential is computed locally (RS256 JWT, HMAC), so the token code can be tested for real. With Courier, minting a token is an API call and must be stubbed too.

## Open points for the map

- **Free-tier realtime:** no vendor gates websockets by plan in the pages we read, so a live badge looks free with Knock. This could be confirmed with a sign-up spike (a `task` ticket) if the "Real-time vs on-load" decision depends on it.
- **Knock branding** stays on the free tier. Accept it or plan for it.
- **Group fan-out:** Knock offers either a recipient list per trigger (up to 1,000) or an object such as `platform-admins` with subscriptions. Either way, each admin gets their own read state.
