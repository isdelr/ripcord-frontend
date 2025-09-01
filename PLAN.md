# Ripcord (Discord‑like) Frontend Plan — SvelteKit

This document is a step‑by‑step implementation plan to build a Discord‑style client using modern SvelteKit. It maps the provided backend Swagger API (auth, users, clubs/roles/channels, messages, presence, streams, voice) to concrete frontend routes, components, utilities, stores, and tasks.

The plan favors incremental delivery, strong typing, composable components, and real‑time UX, while remaining pragmatic (start with plaintext messages; add advanced features later).


## 1) Goals and Scope

- Core goals
  - Auth (register, login, refresh, logout) and account management
  - Server (club) list and membership
  - Text channels: history, pagination, send/edit/delete, attachments, embeds
  - Presence indicators and members list
  - Roles & permissions UI (basic management and per‑channel overwrites)
  - Voice channels: join, voice state controls; stream start (basic flow)
- Out of scope (phase 1)
  - E2E encryption flows (fields exist in API; we’ll design for future support)
  - Full WebRTC media pipeline details (we prepare hooks/stores and UI)
  - Push notifications; advanced moderation tooling


## 2) Project Structure (SvelteKit)

Target structure aligned with SvelteKit today:

```
src/
  lib/
    components/
      common/            # Button, Input, Modal, Dropdown, Tooltip, Avatar, Icon
      layout/            # AppShell, Sidebar, TopBar, Panels, Splitters
      auth/              # LoginForm, RegisterForm, ResetForms
      chat/              # ChannelList, ChannelItem, MessageList, MessageItem, Composer
      presence/          # PresenceDot, MemberList, RoleBadge
      voice/             # VoiceControls, DeviceSelector, StreamView
      clubs/             # ClubList, ClubItem, RoleManager, OverwriteEditor
    stores/              # auth.ts, clubs.ts, channels.ts, messages.ts, presence.ts, voice.ts, ui.ts
    api/                 # http.ts(fetch wrapper), services/* (Auth, Users, Clubs,...)
    utils/               # formatters.ts, pagination.ts, time.ts, file.ts, retry.ts
    server/              # server-only helpers (cookies, SSR session)
    assets/              # favicon.svg, logos, illustrations
  params/
    uuid.ts              # optional: validate IDs (or string matcher)
  routes/
    +layout.svelte       # global styles, toasts, auth gate wiring
    +layout.server.ts    # SSR: get session from cookies
    +page.svelte         # redirect or landing (if unauthenticated)

    auth/
      login/+page.svelte
      register/+page.svelte
      verify/+page.svelte            # token handling
      reset/request/+page.svelte
      reset/confirm/+page.svelte

    app/
      +layout.svelte                 # app shell (sidebar + content)
      +layout.server.ts              # protect routes, load initial user data
      +page.svelte                   # default dashboard or last club

      settings/
        profile/+page.svelte         # update user

      clubs/[clubId]/
        +layout.svelte               # loads club, channels, roles
        +page.svelte                 # summary/member list
        text/[channelId]/+page.svelte    # chat view
        voice/[channelId]/+page.svelte   # voice lobby/controls/stream

      relationships/+page.svelte     # friends/blocked list

  app.html
  error.html
  hooks.client.ts
  hooks.server.ts
  service-worker.ts (optional)
static/
  favicon.png etc.
```

Notes
- $lib alias is available per SvelteKit conventions.
- Colocate route‑only components under routes if they’re not shared.


## 3) Environment & Config

Define env variables (SvelteKit public/private):
- PUBLIC_API_BASE_URL — backend API gateway base URL
- PUBLIC_BUILD_VERSION — for diagnostics
- Cookie names (httpOnly recommended for SSR usage): access, refresh (if you choose cookie‑based tokens on SSR)

Configuration tasks
- Decide token storage strategy:
  - Easiest: store access_token in memory/localStorage; refresh_token in httpOnly cookie set via server endpoints (preferable) or localStorage (simpler dev). We’ll support both via a unified auth store and SSR hooks.
- Add SvelteKit hooks.server.ts to read cookies into event.locals, expose session to load functions.
- Add hooks.client.ts for navigation guard (redirect to /auth/login when no session on app pages).


## 4) Data Models (TypeScript)

Map Swagger types to TS interfaces in $lib/api/types.ts. Use strings for int64 bitfields. Examples:

- User: userUser
- Club (Server): clubClub
- Channel: clubChannel with type: "TEXT" | "VOICE"
- Role: clubRole { permissions: string } // bitfield as string
- Member: clubMember { role_ids: string[] }
- Message: messageMessage with attachments, embeds
- Presence: presencePresence; status: "ONLINE" | "IDLE" | "DO_NOT_DISTURB" | "OFFLINE"
- Voice: voiceVoiceState, voiceJoinVoiceResponse

Keep payload bodies (e.g., MessageServiceSendMessageBody) as separate request types for clarity.


## 5) API Layer ($lib/api)

Create a small, resilient HTTP client on top of fetch with:
- Base URL from PUBLIC_API_BASE_URL
- JSON parsing and error normalization
- Authorization header injection when access_token present
- 401 handler: attempt refresh via /auth/refresh
- Concurrency lock for refresh; queue failed requests until refresh completes
- Retry/backoff utility for transient failures (429/5xx)

Services (map to Swagger):
- AuthService: login, register, logout, refresh, request/confirm verification, request/confirm reset
- UserService: getUser(user_id), updateMe
- ClubService: createClub, getClub, updateClub, deleteClub, getClubs(user_id), getChannels(club_id), createChannel, updateChannel, deleteChannel
- RoleService: getRoles(club_id), createRole, updateRole, deleteRole, assignRole, unassignRole
- Overwrites: getChannelOverwrites, updateChannelOverwrite, deleteChannelOverwrite
- MessageService: getMessages(channel_id, {before_id, after_id, limit}), sendMessage, editMessage, deleteMessage, finalizeAttachments
- PresenceService: batchGetPresence(user_ids)
- StreamService: startStream(channel_id, { type })
- VoiceService: joinVoice({ channel_id }), serverSetVoiceFlags

Consider separate files under $lib/api/services/*.ts with clear request/response typing.


## 6) State Management (Svelte stores)

- auth.ts
  - user: User | null
  - accessToken: string | null (kept in memory; optionally persisted)
  - refreshToken: string | null (prefer cookie; fallback localStorage for dev)
  - actions: login, logout, loadFromCookies, refresh, setUser

- clubs.ts
  - map of id -> Club; derived arrays for sorting; currentClubId
  - methods to load user’s clubs and create/update/delete

- channels.ts
  - per clubId list; order by position; helpers for move/sort

- messages.ts
  - per channelId: items (ordered by created_at), paging cursors (before_id/after_id)
  - loadInitial, loadOlder(before_id), loadNewer(after_id), addOptimistic, reconcile

- presence.ts
  - map userId -> Presence; batch loader for visible members

- voice.ts
  - current voice state, device permissions, selected input/output, join/leave flows

- ui.ts
  - theme, toasts, modals, sidebar collapsed, loading flags


## 7) Routing & Pages

Public routes
- /auth/login — email/password
- /auth/register — username/email/password
- /auth/verify — token input or query param handling
- /auth/reset/request — request email
- /auth/reset/confirm — submit token + new password

Protected app routes (wrap in app layout with guard)
- /app (default dashboard: recent activity or first/last club)
- /app/settings/profile — update username, email, avatar
- /app/relationships — list friends/blocked, actions create/delete
- /app/clubs/[clubId]
  - summary page (member list, basic info)
  - text/[channelId] — chat
  - voice/[channelId] — voice lobby (join), stream panel

Navigation
- Sidebar (left): club list (icons) + friend hub; nested panel for channels of selected club; bottom user bar (avatar, status, mic/headphone)

Load strategies
- +layout.server.ts: resolve session; prefetch user and clubs
- Club layout: load channels, roles, maybe member sample
- Channel pages: lazy‑load messages with pagination cursors


## 8) Components (high‑level)

Common
- Button, IconButton, Input, Select, TextArea, Switch, Checkbox
- Modal, Drawer, Popover, DropdownMenu, ContextMenu
- Avatar, PresenceDot, RoleBadge
- Toasts (use a simple store with a ToastContainer in root layout)

Layout
- AppShell (sidebar + topbar + content)
- Sidebar: ClubList, UserBar
- ChannelList: grouped by TEXT/VOICE; drag‑reorder (phase 2)
- MemberList: presence indicators and role badges

Chat
- MessageList (virtualized, infinite scroll backward via before_id)
- MessageItem (author, timestamp, content, attachments, embeds)
- MessageComposer (text input, file upload -> finalize attachments, emoji)
- TypingIndicator (phase 2)

Clubs/Roles/Permissions
- RoleManager (CRUD), AssignRoleDialog
- OverwriteEditor for channel (target user/role, allow/deny bitfields), with readable presets

Voice/Stream
- VoiceControls (mute/deaf, join/leave)
- DeviceSelector (mic/speaker)
- StreamView (basic: show stream info after start)

Auth
- LoginForm, RegisterForm, RequestResetForm, ConfirmResetForm, VerifyEmailForm


## 9) Messaging UX Details

- History API: GET /channels/{channel_id}/messages with before_id/after_id/limit
  - Initial load: fetch latest N (no before_id/after_id)
  - Infinite scroll up: use before_id = currentOldest.id
  - Live updates: poll, SSE, or WS (start with polling every 2–5s; upgrade later)
- Send: POST /channels/{channel_id}/messages
  - Optimistic insert with temporary ID; reconcile with server response
- Edit/Delete: PATCH/DELETE /channels/{channel_id}/messages/{message_id}
- Attachments: upload to storage (out of scope here) then POST /attachments/finalize
- Embeds: render simple rich cards (title, desc, thumbnail) when provided
- Encryption: fields exist (ciphertext_b64, enc_header, enc_algo). Phase 1 uses plaintext only; UI remains compatible.


## 10) Presence

- Batch load visible members’ presence via POST /presence/batchGet
- Cache in presence store, refresh periodically
- Show status dots on avatars and member list; activity line ("Playing …") when present


## 11) Roles & Permissions

- Roles (club‑level):
  - List: GET /clubs/{club_id}/roles
  - Create: POST /clubs/{club_id}/roles
  - Update: PATCH /roles/{role_id}; Delete: DELETE /roles/{role_id}
  - Assign/Unassign to users: POST /clubs/{club_id}/roles/(un)assign
  - UI: role name, color, permission bitfield (string). Provide presets and readable toggles.

- Channel overwrites (role/user target):
  - GET /channels/{channel_id}/overwrites
  - PUT update overwrite; DELETE remove
  - UI presents allow/deny bitfields with masks; validate mutually exclusive bits


## 12) Voice & Stream (Phase 1 basics)

- Join voice: POST /voice/join with channel_id — returns room metadata and ICE servers
- Maintain voice state: voice store; display server_mute/deaf flags
- Server moderation flags: POST /voice/moderate
- Start stream (screen/camera): POST /channels/{channel_id}/streams/start
- Media/WebRTC plumbing: plan interfaces and UI, leave SFU integration for phase 2


## 13) Error Handling & UX

- Centralize APIError type; normalize googlerpcStatus { code, message }
- Toasts for errors/success; inline form errors
- 401 auto‑refresh; if refresh fails, redirect to /auth/login
- Retry with exponential backoff for transient errors
- Gracefully handle empty states and skeleton loading


## 14) Security & Privacy

- Prefer httpOnly cookies for refresh tokens (via server endpoints). Avoid storing refresh in localStorage in production.
- Escape/sanitize message content when rendering markdown/links (if added). Avoid dangerouslySetInnerHTML equivalents.
- Validate IDs via param matchers where possible; guard actions by permissions in UI.


## 15) Performance

- Virtualized message list (only visible DOM nodes)
- Batched presence requests
- Cache clubs/channels in stores; only refetch when invalidated
- Image lazy‑loading for avatars/attachments


## 16) Testing Strategy

- Unit (Vitest):
  - stores (auth/messages/presence) reducers and pagination logic
  - api http client (refresh flow, retries)
- Component tests: MessageItem, Composer, RoleManager
- E2E (Playwright): auth flow, join club, open channel, send/edit/delete message


## 17) Incremental Milestones (Suggested Order)

1. Bootstrap & config
   - Add hooks.server.ts, hooks.client.ts, base http client; env
2. Auth flows
   - Login/Register/Logout/Refresh; guard app routes
3. Shell & clubs
   - AppShell, Sidebar, load user clubs; navigate
4. Channels & messages
   - Channel list per club; text channel page with history + send/edit/delete
5. Attachments & embeds (basic)
6. Presence indicators & member list
7. Roles CRUD and assignment
8. Channel overwrites editor
9. Voice join and voice controls (UI + store)
10. Stream start flow (UI + API call)
11. Relationships page (friends/blocked)
12. Settings: profile update
13. Polish, tests, performance

Each milestone should be merged behind feature flags if needed, with route‑level guards.


## 18) Endpoint Mapping Cheatsheet

Auth
- POST /auth/register  -> register({ username, email, password }) => { user, access_token, refresh_token }
- POST /auth/login     -> login({ email, password })
- POST /auth/logout    -> logout({ refresh_token })
- POST /auth/refresh   -> refresh({ refresh_token }) => { access_token }
- POST /auth/verify/request -> requestEmailVerification({ email })
- POST /auth/verify/confirm -> verifyEmail({ token })
- POST /auth/reset/request  -> requestPasswordReset({ email })
- POST /auth/reset/confirm  -> resetPassword({ token, new_password })

Users
- GET  /users/{user_id}     -> getUser(user_id)
- PATCH /users/me           -> updateMe(partial)

Clubs & Channels
- POST /clubs                         -> createClub({ name, icon_url? })
- GET  /users/{user_id}/clubs         -> getClubs(user_id) [streaming; treat as list]
- GET  /clubs/{club_id}               -> getClub(club_id)
- PATCH/DELETE /clubs/{club_id}       -> updateClub/deleteClub
- GET  /clubs/{club_id}/channels      -> getChannels(club_id)
- POST /channels                      -> createChannel({ club_id, name, type })
- PATCH/DELETE /channels/{channel_id} -> updateChannel/deleteChannel

Roles & Overwrites
- GET/POST /clubs/{club_id}/roles     -> getRoles/createRole
- PATCH/DELETE /roles/{role_id}       -> updateRole/deleteRole
- POST /clubs/{club_id}/roles/assign  -> assignRole({ user_id, role_id })
- POST /clubs/{club_id}/roles/unassign-> unassignRole({ user_id, role_id })
- GET /channels/{channel_id}/overwrites -> getChannelOverwrites
- PUT /channels/{channel_id}/overwrites -> updateChannelOverwrite(body)
- DELETE /channels/{channel_id}/overwrites -> deleteChannelOverwrite(body)

Messages
- GET  /channels/{channel_id}/messages -> getMessages({ before_id?, limit?, after_id? })
- POST /channels/{channel_id}/messages -> sendMessage(body)
- PATCH/DELETE /channels/{channel_id}/messages/{message_id} -> editMessage/deleteMessage
- POST /attachments/finalize -> finalizeAttachments({ channel_id, message_id, attachments })

Presence
- POST /presence/batchGet -> batchGetPresence({ user_ids }) -> map

Voice/Stream
- POST /voice/join   -> joinVoice({ channel_id })
- POST /voice/moderate -> serverSetVoiceFlags({ channel_id, target_user_id, ... })
- POST /channels/{channel_id}/streams/start -> startStream({ type })


## 19) Implementation Notes & Snippets

HTTP client (pseudo‑TS):
```ts
// $lib/api/http.ts
export async function apiFetch<T>(path: string, init: RequestInit = {}): Promise<T> {
  const base = import.meta.env.PUBLIC_API_BASE_URL;
  const res = await fetch(`${base}${path}`, {
    headers: { 'content-type': 'application/json', ...(init.headers || {}) },
    ...init
  });
  if (!res.ok) {
    const err = await safeJson(res);
    throw normalizeError(res.status, err);
  }
  return safeJson(res);
}
```

Messages pagination pattern:
```ts
// $lib/stores/messages.ts
loadOlder = async (channelId: string) => {
  const oldest = getOldestMessageId(channelId);
  const res = await MessageService.getMessages(channelId, { before_id: oldest, limit: 50 });
  upsertAtTop(channelId, res);
};
```

Auth refresh flow:
```ts
// on 401 from apiFetch, attempt refresh then retry original request
```


## 20) Future Enhancements

- Real‑time via WebSocket/SSE for messages, presence, voice state
- E2E encryption (ciphertext_b64 path) using X3DH + Double Ratchet
- Drag‑and‑drop channel reordering; categories; threads
- Rich text/markdown rendering with sanitization
- Media gallery, link unfurling, inline playback


---
This plan aligns with modern SvelteKit project structure and maps every available backend endpoint to clear frontend concerns. Follow the milestones, and you’ll iterate from a simple authenticated shell to a fully‑featured Discord‑like client.


You are an expert in JavaScript, TypeScript, and SvelteKit framework for scalable web development.

Key Principles
- Write concise, technical responses with accurate SvelteKit examples.
- Leverage SvelteKit's server-side rendering (SSR) and static site generation (SSG) capabilities.
- Prioritize performance optimization and minimal JavaScript for optimal user experience.
- Use descriptive variable names and follow SvelteKit's naming conventions.
- Organize files using SvelteKit's file-based routing system.

SvelteKit Project Structure
- Use the recommended SvelteKit project structure:
  ```
  - src/
    - lib/
    - routes/
    - app.html
  - static/
  - svelte.config.js
  - vite.config.js
  ```

Component Development
- Create .svelte files for Svelte components.
- Implement proper component composition and reusability.
- Use Svelte's props for data passing.
- Leverage Svelte's reactive declarations and stores for state management.

Routing and Pages
- Utilize SvelteKit's file-based routing system in the src/routes/ directory.
- Implement dynamic routes using [slug] syntax.
- Use load functions for server-side data fetching and pre-rendering.
- Implement proper error handling with +error.svelte pages.

Server-Side Rendering (SSR) and Static Site Generation (SSG)
- Leverage SvelteKit's SSR capabilities for dynamic content.
- Implement SSG for static pages using prerender option.
- Use the adapter-auto for automatic deployment configuration.

Performance Optimization
- Minimize use of client-side JavaScript; leverage SvelteKit's SSR and SSG.
- Implement code splitting using SvelteKit's dynamic imports.
- Use Svelte's transition and animation features for smooth UI interactions.
- Implement proper lazy loading for images and other assets.

Data Fetching
- Use load functions for server-side data fetching.
- Implement proper error handling for data fetching operations.
- Utilize SvelteKit's $app/stores for accessing page data and other stores.

SEO and Meta Tags
- Use Svelte:head component for adding meta information.
- Implement canonical URLs for proper SEO.
- Create reusable SEO components for consistent meta tag management.

State Management
- Use Svelte stores for global state management.
- Leverage context API for sharing data between components.
- Implement proper store subscriptions and unsubscriptions.

Forms and Actions
- Utilize SvelteKit's form actions for server-side form handling.
- Implement proper client-side form validation using Svelte's reactive declarations.
- Use progressive enhancement for JavaScript-optional form submissions.

Authentication
- Implement authentication using SvelteKit's hooks and server-side sessions.
- Use secure HTTP-only cookies for session management.
- Implement proper CSRF protection for forms and API routes.

Styling with Tailwind CSS
- Use Tailwind utility classes extensively in your Svelte components.
- Leverage Tailwind's responsive design utilities (sm:, md:, lg:, etc.).
- Utilize Tailwind's spacing scale for consistency.
- Avoid using the @apply directive; prefer direct utility classes in HTML.

Accessibility
- Ensure proper semantic HTML structure in Svelte components.
- Implement ARIA attributes where necessary.
- Ensure keyboard navigation support for interactive elements.
- Use Svelte's bind:this for managing focus programmatically.

Key Conventions
1. Follow the official SvelteKit documentation for best practices and conventions.
2. Use TypeScript for enhanced type safety and developer experience.
3. Implement proper error handling and logging.
4. Leverage SvelteKit's built-in features for internationalization (i18n) if needed.
5. Use SvelteKit's asset handling for optimized static asset delivery.

Performance Metrics
- Prioritize Core Web Vitals (LCP, FID, CLS) in development.

Refer to SvelteKit's official documentation for detailed information on components, routing, and server-side rendering for best practices.


Svelte 5 migration guide
On this page
Svelte 5 migration guide
Reactivity syntax changes
Event changes
Snippets instead of slots
Migration script
Components are no longer classes
<svelte:component> is no longer necessary
Whitespace handling changed
Modern browser required
Changes to compiler options
The children prop is reserved
Breaking changes in runes mode
Other breaking changes
Version 5 comes with an overhauled syntax and reactivity system. While it may look different at first, you’ll soon notice many similarities. This guide goes over the changes in detail and shows you how to upgrade. Along with it, we also provide information on why we did these changes.

You don’t have to migrate to the new syntax right away - Svelte 5 still supports the old Svelte 4 syntax, and you can mix and match components using the new syntax with components using the old and vice versa. We expect many people to be able to upgrade with only a few lines of code changed initially. There’s also a migration script that helps you with many of these steps automatically.

Reactivity syntax changes
At the heart of Svelte 5 is the new runes API. Runes are basically compiler instructions that inform Svelte about reactivity. Syntactically, runes are functions starting with a dollar-sign.

let → $state
In Svelte 4, a let declaration at the top level of a component was implicitly reactive. In Svelte 5, things are more explicit: a variable is reactive when created using the $state rune. Let’s migrate the counter to runes mode by wrapping the counter in $state:


<script>
	let count = $state(0);
</script>
Nothing else changes. count is still the number itself, and you read and write directly to it, without a wrapper like .value or getCount().

$: → $derived/$effect
In Svelte 4, a $: statement at the top level of a component could be used to declare a derivation, i.e. state that is entirely defined through a computation of other state. In Svelte 5, this is achieved using the $derived rune:


<script>
	let count = $state(0);
	$: const double = $derived(count * 2);
</script>
As with $state, nothing else changes. double is still the number itself, and you read it directly, without a wrapper like .value or getDouble().

A $: statement could also be used to create side effects. In Svelte 5, this is achieved using the $effect rune:


<script>
	let count = $state(0);

	$:$effect(() => {
		if (count > 5) {
			alert('Count is too high!');
		}
	});
</script>
Note that when $effect runs is different than when $: runs.

export let → $props
In Svelte 4, properties of a component were declared using export let. Each property was one declaration. In Svelte 5, all properties are declared through the $props rune, through destructuring:


<script>
	export let optional = 'unset';
	export let required;
	let { optional = 'unset', required } = $props();
</script>
There are multiple cases where declaring properties becomes less straightforward than having a few export let declarations:

you want to rename the property, for example because the name is a reserved identifier (e.g. class)
you don’t know which other properties to expect in advance
you want to forward every property to another component
All these cases need special syntax in Svelte 4:

renaming: export { klass as class}
other properties: $$restProps
all properties $$props
In Svelte 5, the $props rune makes this straightforward without any additional Svelte-specific syntax:

renaming: use property renaming let { class: klass } = $props();
other properties: use spreading let { foo, bar, ...rest } = $props();
all properties: don’t destructure let props = $props();

<script>
	let klass = '';
	export { klass as class};
	let { class: klass, ...rest } = $props();
</script>
<button class={klass} {...$$restPropsrest}>click me</button>
Event changes
Event handlers have been given a facelift in Svelte 5. Whereas in Svelte 4 we use the on: directive to attach an event listener to an element, in Svelte 5 they are properties like any other (in other words - remove the colon):


<script>
	let count = $state(0);
</script>

<button on:click={() => count++}>
clicks: {count}
</button>
Since they’re just properties, you can use the normal shorthand syntax...


<script>
	let count = $state(0);

	function onclick() {
		count++;
	}
</script>

<button {onclick}>
clicks: {count}
</button>
...though when using a named event handler function it’s usually better to use a more descriptive name.

Component events
In Svelte 4, components could emit events by creating a dispatcher with createEventDispatcher.

This function is deprecated in Svelte 5. Instead, components should accept callback props - which means you then pass functions as properties to these components:

App

<script lang="ts">
	import Pump from './Pump.svelte';

	let size = $state(15);
	let burst = $state(false);

	function reset() {
		size = 15;
		burst = false;
	}
</script>

<Pump
on:inflate={(power) => {
size += power.detail;
if (size > 75) burst = true;
}}
on:deflate={(power) => {
if (size > 0) size -= power.detail;
}}
/>

{#if burst}
<button onclick={reset}>new balloon</button>
<span class="boom">💥</span>
{:else}
<span class="balloon" style="scale: {0.01 * size}">
🎈
</span>
{/if}
Pump

<script lang="ts">
	import { createEventDispatcher } from 'svelte';
	const dispatch = createEventDispatcher();

	let { inflate, deflate } = $props();
	let power = $state(5);
</script>

<button onclick={() => dispatch('inflate', power)inflate(power)}>
inflate
</button>
<button onclick={() => dispatch('deflate', power)deflate(power)}>
deflate
</button>
<button onclick={() => power--}>-</button>
Pump power: {power}
<button onclick={() => power++}>+</button>
Bubbling events
Instead of doing <button on:click> to ‘forward’ the event from the element to the component, the component should accept an onclick callback prop:


<script>
	let { onclick } = $props();
</script>

<button on:click {onclick}>
click me
</button>
Note that this also means you can ‘spread’ event handlers onto the element along with other props instead of tediously forwarding each event separately:


<script>
	let props = $props();
</script>

<button {...$$props} on:click on:keydown on:all_the_other_stuff {...props}>
click me
</button>
Event modifiers
In Svelte 4, you can add event modifiers to handlers:


<button on:click|once|preventDefault={handler}>...</button>
Modifiers are specific to on: and so do not work with modern event handlers. Adding things like event.preventDefault() inside the handler itself is preferable, since all the logic lives in one place rather than being split between handler and modifiers.

Since event handlers are just functions, you can create your own wrappers as necessary:


<script>
	function once(fn) {
		return function (event) {
			if (fn) fn.call(this, event);
			fn = null;
		};
	}

	function preventDefault(fn) {
		return function (event) {
			event.preventDefault();
			fn.call(this, event);
		};
	}
</script>

<button onclick={once(preventDefault(handler))}>...</button>
There are three modifiers — capture, passive and nonpassive — that can’t be expressed as wrapper functions, since they need to be applied when the event handler is bound rather than when it runs.

For capture, we add the modifier to the event name:


<button onclickcapture={...}>...</button>
Changing the passive option of an event handler, meanwhile, is not something to be done lightly. If you have a use case for it — and you probably don’t! — then you will need to use an action to apply the event handler yourself.

Multiple event handlers
In Svelte 4, this is possible:


<button on:click={one} on:click={two}>...</button>
Duplicate attributes/properties on elements — which now includes event handlers — are not allowed. Instead, do this:


<button
onclick={(e) => {
one(e);
two(e);
}}
>
	...
</button>
When spreading props, local event handlers must go after the spread, or they risk being overwritten:


<button
{...props}
onclick={(e) => {
doStuff(e);
props.onclick?.(e);
}}
>
	...
</button>
Snippets instead of slots
In Svelte 4, content can be passed to components using slots. Svelte 5 replaces them with snippets, which are more powerful and flexible, and so slots are deprecated in Svelte 5.

They continue to work, however, and you can pass snippets to a component that uses slots:

Child

<slot />
<hr />
<slot name="foo" message="hello" />
Parent

<script lang="ts">
	import Child from './Child.svelte';
</script>

<Child>
	default child content

	{#snippet foo({ message })}
		message from child: {message}
	{/snippet}
</Child>
(The reverse is not true — you cannot pass slotted content to a component that uses {@render ...} tags.)

When using custom elements, you should still use <slot /> like before. In a future version, when Svelte removes its internal version of slots, it will leave those slots as-is, i.e. output a regular DOM tag instead of transforming it.

Default content
In Svelte 4, the easiest way to pass a piece of UI to the child was using a <slot />. In Svelte 5, this is done using the children prop instead, which is then shown with {@render children()}:


<script>
	let { children } = $props();
</script>

<slot />
{@render children?.()}
Multiple content placeholders
If you wanted multiple UI placeholders, you had to use named slots. In Svelte 5, use props instead, name them however you like and {@render ...} them:


<script>
	let { header, main, footer } = $props();
</script>

<header>
	<slot name="header" />
	{@render header()}
</header>

<main>
	<slot name="main" />
	{@render main()}
</main>

<footer>
	<slot name="footer" />
	{@render footer()}
</footer>
Passing data back up
In Svelte 4, you would pass data to a <slot /> and then retrieve it with let: in the parent component. In Svelte 5, snippets take on that responsibility:

App

<script lang="ts">
	import List from './List.svelte';
</script>

<List items={['one', 'two', 'three']} let:item>
{#snippet item(text)}
<span>{text}</span>
{/snippet}
<span slot="empty">No items yet</span>
{#snippet empty()}
<span>No items yet</span>
{/snippet}
</List>
List

<script lang="ts">
	let { items, item, empty } = $props();
</script>

{#if items.length}
<ul>
{#each items as entry}
<li>
<slot item={entry} />
{@render item(entry)}
</li>
{/each}
</ul>
{:else}
<slot name="empty" />
{@render empty?.()}
{/if}
Migration script
By now you should have a pretty good understanding of the before/after and how the old syntax relates to the new syntax. It probably also became clear that a lot of these migrations are rather technical and repetitive - something you don’t want to do by hand.

We thought the same, which is why we provide a migration script to do most of the migration automatically. You can upgrade your project by using npx sv migrate svelte-5. This will do the following things:

bump core dependencies in your package.json
migrate to runes (let → $state etc)
migrate to event attributes for DOM elements (on:click → onclick)
migrate slot creations to render tags (<slot /> → {@render children()})
migrate slot usages to snippets (<div slot="x">...</div> → {#snippet x()}<div>...</div>{/snippet})
migrate obvious component creations (new Component(...) → mount(Component, ...))
You can also migrate a single component in VS Code through the Migrate Component to Svelte 5 Syntax command, or in our Playground through the Migrate button.

Not everything can be migrated automatically, and some migrations need manual cleanup afterwards. The following sections describe these in more detail.

run
You may see that the migration script converts some of your $: statements to a run function which is imported from svelte/legacy. This happens if the migration script couldn’t reliably migrate the statement to a $derived and concluded this is a side effect instead. In some cases this may be wrong and it’s best to change this to use a $derived instead. In other cases it may be right, but since $: statements also ran on the server but $effect does not, it isn’t safe to transform it as such. Instead, run is used as a stopgap solution. run mimics most of the characteristics of $:, in that it runs on the server once, and runs as $effect.pre on the client ($effect.pre runs before changes are applied to the DOM; most likely you want to use $effect instead).


<script>
	import { run } from 'svelte/legacy';
	run(() => {
	$effect(() => {
		// some side effect code
	})
</script>
Event modifiers
Event modifiers are not applicable to event attributes (e.g. you can’t do onclick|preventDefault={...}). Therefore, when migrating event directives to event attributes, we need a function-replacement for these modifiers. These are imported from svelte/legacy, and should be migrated away from in favor of e.g. just using event.preventDefault().


<script>
	import { preventDefault } from 'svelte/legacy';
</script>

<button
onclick={preventDefault((event) => {
event.preventDefault();
// ...
})}
>
	click me
</button>
Things that are not automigrated
The migration script does not convert createEventDispatcher. You need to adjust those parts manually. It doesn’t do it because it’s too risky because it could result in breakage for users of the component, which the migration script cannot find out.

The migration script does not convert beforeUpdate/afterUpdate. It doesn’t do it because it’s impossible to determine the actual intent of the code. As a rule of thumb you can often go with a combination of $effect.pre (runs at the same time as beforeUpdate did) and tick (imported from svelte, allows you to wait until changes are applied to the DOM and then do some work).

Components are no longer classes
In Svelte 3 and 4, components are classes. In Svelte 5 they are functions and should be instantiated differently. If you need to manually instantiate components, you should use mount or hydrate (imported from svelte) instead. If you see this error using SvelteKit, try updating to the latest version of SvelteKit first, which adds support for Svelte 5. If you’re using Svelte without SvelteKit, you’ll likely have a main.js file (or similar) which you need to adjust:


import { mount } from 'svelte';
import App from './App.svelte'

const app = new App({ target: document.getElementById("app") });
const app = mount(App, { target: document.getElementById("app") });

export default app;
mount and hydrate have the exact same API. The difference is that hydrate will pick up the Svelte’s server-rendered HTML inside its target and hydrate it. Both return an object with the exports of the component and potentially property accessors (if compiled with accessors: true). They do not come with the $on, $set and $destroy methods you may know from the class component API. These are its replacements:

For $on, instead of listening to events, pass them via the events property on the options argument.


import { mount } from 'svelte';
import App from './App.svelte'

const app = new App({ target: document.getElementById("app") });
app.$on('event', callback);
const app = mount(App, { target: document.getElementById("app"), events: { event: callback } });
Note that using events is discouraged — instead, use callbacks

For $set, use $state instead to create a reactive property object and manipulate it. If you’re doing this inside a .js or .ts file, adjust the ending to include .svelte, i.e. .svelte.js or .svelte.ts.


import { mount } from 'svelte';
import App from './App.svelte'

const app = new App({ target: document.getElementById("app"), props: { foo: 'bar' } });
app.$set({ foo: 'baz' });
const props = $state({ foo: 'bar' });
const app = mount(App, { target: document.getElementById("app"), props });
props.foo = 'baz';
For $destroy, use unmount instead.


import { mount, unmount } from 'svelte';
import App from './App.svelte'

const app = new App({ target: document.getElementById("app"), props: { foo: 'bar' } });
app.$destroy();
const app = mount(App, { target: document.getElementById("app") });
unmount(app);
As a stop-gap-solution, you can also use createClassComponent or asClassComponent (imported from svelte/legacy) instead to keep the same API known from Svelte 4 after instantiating.


import { createClassComponent } from 'svelte/legacy';
import App from './App.svelte'

const app = new App({ target: document.getElementById("app") });
const app = createClassComponent({ component: App, target: document.getElementById("app") });

export default app;
If this component is not under your control, you can use the compatibility.componentApi compiler option for auto-applied backwards compatibility, which means code using new Component(...) keeps working without adjustments (note that this adds a bit of overhead to each component). This will also add $set and $on methods for all component instances you get through bind:this.


/// svelte.config.js
export default {
compilerOptions: {
compatibility: {
componentApi: 4
}
}
};
Note that mount and hydrate are not synchronous, so things like onMount won’t have been called by the time the function returns and the pending block of promises will not have been rendered yet (because #await waits a microtask to wait for a potentially immediately-resolved promise). If you need that guarantee, call flushSync (import from 'svelte') after calling mount/hydrate.

Server API changes
Similarly, components no longer have a render method when compiled for server-side rendering. Instead, pass the function to render from svelte/server:


import { render } from 'svelte/server';
import App from './App.svelte';

const { html, head } = App.render({ props: { message: 'hello' }});
const { html, head } = render(App, { props: { message: 'hello' }});
In Svelte 4, rendering a component to a string also returned the CSS of all components. In Svelte 5, this is no longer the case by default because most of the time you’re using a tooling chain that takes care of it in other ways (like SvelteKit). If you need CSS to be returned from render, you can set the css compiler option to 'injected' and it will add <style> elements to the head.

Component typing changes
The change from classes towards functions is also reflected in the typings: SvelteComponent, the base class from Svelte 4, is deprecated in favour of the new Component type which defines the function shape of a Svelte component. To manually define a component shape in a d.ts file:


import type { Component } from 'svelte';
export declare const MyComponent: Component<{
foo: string;
}>;
To declare that a component of a certain type is required:


import { ComponentA, ComponentB } from 'component-library';
import type { SvelteComponent } from 'svelte';
import type { Component } from 'svelte';

let C: typeof SvelteComponent<{ foo: string }> = $state(
let C: Component<{ foo: string }> = $state(
Math.random() ? ComponentA : ComponentB
);
The two utility types ComponentEvents and ComponentType are also deprecated. ComponentEvents is obsolete because events are defined as callback props now, and ComponentType is obsolete because the new Component type is the component type already (i.e. ComponentType<SvelteComponent<{ prop: string }>> is equivalent to Component<{ prop: string }>).

bind:this changes
Because components are no longer classes, using bind:this no longer returns a class instance with $set, $on and $destroy methods on it. It only returns the instance exports (export function/const) and, if you’re using the accessors option, a getter/setter-pair for each property.

<svelte:component> is no longer necessary
In Svelte 4, components are static — if you render <Thing>, and the value of Thing changes, nothing happens. To make it dynamic you had to use <svelte:component>.

This is no longer true in Svelte 5:


<script>
	import A from './A.svelte';
	import B from './B.svelte';

	let Thing = $state();
</script>

<select bind:value={Thing}>
	<option value={A}>A</option>
	<option value={B}>B</option>
</select>

<!-- these are equivalent -->
<Thing />
<svelte:component this={Thing} />
While migrating, keep in mind that your component’s name should be capitalized (Thing) to distinguish it from elements, unless using dot notation.

Dot notation indicates a component
In Svelte 4, <foo.bar> would create an element with a tag name of "foo.bar". In Svelte 5, foo.bar is treated as a component instead. This is particularly useful inside each blocks:


{#each items as item}
<item.component {...item.props} />
{/each}
Whitespace handling changed
Previously, Svelte employed a very complicated algorithm to determine if whitespace should be kept or not. Svelte 5 simplifies this which makes it easier to reason about as a developer. The rules are:

Whitespace between nodes is collapsed to one whitespace
Whitespace at the start and end of a tag is removed completely
Certain exceptions apply such as keeping whitespace inside pre tags
As before, you can disable whitespace trimming by setting the preserveWhitespace option in your compiler settings or on a per-component basis in <svelte:options>.

Modern browser required
Svelte 5 requires a modern browser (in other words, not Internet Explorer) for various reasons:

it uses Proxies
elements with clientWidth / clientHeight/offsetWidth/offsetHeight bindings use a ResizeObserver rather than a convoluted <iframe> hack
<input type="range" bind:value={...} /> only uses an input event listener, rather than also listening for change events as a fallback
The legacy compiler option, which generated bulkier but IE-friendly code, no longer exists.

Changes to compiler options
The false / true (already deprecated previously) and the "none" values were removed as valid values from the css option
The legacy option was repurposed
The hydratable option has been removed. Svelte components are always hydratable now
The enableSourcemap option has been removed. Source maps are always generated now, tooling can choose to ignore it
The tag option was removed. Use <svelte:options customElement="tag-name" /> inside the component instead
The loopGuardTimeout, format, sveltePath, errorMode and varsReport options were removed
The children prop is reserved
Content inside component tags becomes a snippet prop called children. You cannot have a separate prop by that name.

Breaking changes in runes mode
Some breaking changes only apply once your component is in runes mode.

Bindings to component exports are not allowed
Exports from runes mode components cannot be bound to directly. For example, having export const foo = ... in component A and then doing <A bind:foo /> causes an error. Use bind:this instead — <A bind:this={a} /> — and access the export as a.foo. This change makes things easier to reason about, as it enforces a clear separation between props and exports.

Bindings need to be explicitly defined using $bindable()
In Svelte 4 syntax, every property (declared via export let) is bindable, meaning you can bind: to it. In runes mode, properties are not bindable by default: you need to denote bindable props with the $bindable rune.

If a bindable property has a default value (e.g. let { foo = $bindable('bar') } = $props();), you need to pass a non-undefined value to that property if you’re binding to it. This prevents ambiguous behavior — the parent and child must have the same value — and results in better performance (in Svelte 4, the default value was reflected back to the parent, resulting in wasteful additional render cycles).

accessors option is ignored
Setting the accessors option to true makes properties of a component directly accessible on the component instance.


<svelte:options accessors={true} />

<script>
	// available via componentInstance.name
	export let name;
</script>
In runes mode, properties are never accessible on the component instance. You can use component exports instead if you need to expose them.


<script>
	let { name } = $props();
	// available via componentInstance.getName()
	export const getName = () => name;
</script>
Alternatively, if the place where they are instantiated is under your control, you can also make use of runes inside .js/.ts files by adjusting their ending to include .svelte, i.e. .svelte.js or .svelte.ts, and then use $state:


import { mount } from 'svelte';
import App from './App.svelte'

const app = new App({ target: document.getElementById("app"), props: { foo: 'bar' } });
app.foo = 'baz'
const props = $state({ foo: 'bar' });
const app = mount(App, { target: document.getElementById("app"), props });
props.foo = 'baz';
immutable option is ignored
Setting the immutable option has no effect in runes mode. This concept is replaced by how $state and its variations work.

Classes are no longer “auto-reactive”
In Svelte 4, doing the following triggered reactivity:


<script>
	let foo = new Foo();
</script>

<button on:click={() => (foo.value = 1)}>{foo.value}</button
>
This is because the Svelte compiler treated the assignment to foo.value as an instruction to update anything that referenced foo. In Svelte 5, reactivity is determined at runtime rather than compile time, so you should define value as a reactive $state field on the Foo class. Wrapping new Foo() with $state(...) will have no effect — only vanilla objects and arrays are made deeply reactive.

Touch and wheel events are passive
When using onwheel, onmousewheel, ontouchstart and ontouchmove event attributes, the handlers are passive to align with browser defaults. This greatly improves responsiveness by allowing the browser to scroll the document immediately, rather than waiting to see if the event handler calls event.preventDefault().

In the very rare cases that you need to prevent these event defaults, you should use on instead (for example inside an action).

Attribute/prop syntax is stricter
In Svelte 4, complex attribute values needn’t be quoted:


<Component prop=this{is}valid />
This is a footgun. In runes mode, if you want to concatenate stuff you must wrap the value in quotes:


<Component prop="this{is}valid" />
Note that Svelte 5 will also warn if you have a single expression wrapped in quotes, like answer="{42}" — in Svelte 6, that will cause the value to be converted to a string, rather than passed as a number.

HTML structure is stricter
In Svelte 4, you were allowed to write HTML code that would be repaired by the browser when server-side rendering it. For example you could write this...


<table>
	<tr>
		<td>hi</td>
	</tr>
</table>
... and the browser would auto-insert a <tbody> element:


<table>
	<tbody>
		<tr>
			<td>hi</td>
		</tr>
	</tbody>
</table>
Svelte 5 is more strict about the HTML structure and will throw a compiler error in cases where the browser would repair the DOM.

Other breaking changes
Stricter @const assignment validation
Assignments to destructured parts of a @const declaration are no longer allowed. It was an oversight that this was ever allowed.

:is(...), :has(...), and :where(...) are scoped
Previously, Svelte did not analyse selectors inside :is(...), :has(...), and :where(...), effectively treating them as global. Svelte 5 analyses them in the context of the current component. Some selectors may now therefore be treated as unused if they were relying on this treatment. To fix this, use :global(...) inside the :is(...)/:has(...)/:where(...) selectors.

When using Tailwind’s @apply directive, add a :global selector to preserve rules that use Tailwind-generated :is(...) selectors:


main :global {
@apply bg-blue-100 dark:bg-blue-900;
}
CSS hash position no longer deterministic
Previously Svelte would always insert the CSS hash last. This is no longer guaranteed in Svelte 5. This is only breaking if you have very weird css selectors.

Scoped CSS uses :where(...)
To avoid issues caused by unpredictable specificity changes, scoped CSS selectors now use :where(.svelte-xyz123) selector modifiers alongside .svelte-xyz123 (where xyz123 is, as previously, a hash of the <style> contents). You can read more detail here.

In the event that you need to support ancient browsers that don’t implement :where, you can manually alter the emitted CSS, at the cost of unpredictable specificity changes:


css = css.replace(/:where\((.+?)\)/, '$1');

Error/warning codes have been renamed
Error and warning codes have been renamed. Previously they used dashes to separate the words, they now use underscores (e.g. foo-bar becomes foo_bar). Additionally, a handful of codes have been reworded slightly.

Reduced number of namespaces
The number of valid namespaces you can pass to the compiler option namespace has been reduced to html (the default), mathml and svg.

The foreign namespace was only useful for Svelte Native, which we’re planning to support differently in a 5.x minor.

beforeUpdate/afterUpdate changes
beforeUpdate no longer runs twice on initial render if it modifies a variable referenced in the template.

afterUpdate callbacks in a parent component will now run after afterUpdate callbacks in any child components.

beforeUpdate/afterUpdate no longer run when the component contains a <slot> and its content is updated.

Both functions are disallowed in runes mode — use $effect.pre(...) and $effect(...) instead.

contenteditable behavior change
If you have a contenteditable node with a corresponding binding and a reactive value inside it (example: <div contenteditable=true bind:textContent>count is {count}</div>), then the value inside the contenteditable will not be updated by updates to count because the binding takes full control over the content immediately and it should only be updated through it.

oneventname attributes no longer accept string values
In Svelte 4, it was possible to specify event attributes on HTML elements as a string:


<button onclick="alert('hello')">...</button>
This is not recommended, and is no longer possible in Svelte 5, where properties like onclick replace on:click as the mechanism for adding event handlers.

null and undefined become the empty string
In Svelte 4, null and undefined were printed as the corresponding string. In 99 out of 100 cases you want this to become the empty string instead, which is also what most other frameworks out there do. Therefore, in Svelte 5, null and undefined become the empty string.

bind:files values can only be null, undefined or FileList
bind:files is now a two-way binding. As such, when setting a value, it needs to be either falsy (null or undefined) or of type FileList.

Bindings now react to form resets
Previously, bindings did not take into account reset event of forms, and therefore values could get out of sync with the DOM. Svelte 5 fixes this by placing a reset listener on the document and invoking bindings where necessary.

walk no longer exported
svelte/compiler reexported walk from estree-walker for convenience. This is no longer true in Svelte 5, import it directly from that package instead in case you need it.

Content inside svelte:options is forbidden
In Svelte 4 you could have content inside a <svelte:options /> tag. It was ignored, but you could write something in there. In Svelte 5, content inside that tag is a compiler error.

<slot> elements in declarative shadow roots are preserved
Svelte 4 replaced the <slot /> tag in all places with its own version of slots. Svelte 5 preserves them in the case they are a child of a <template shadowrootmode="..."> element.

<svelte:element> tag must be an expression
In Svelte 4, <svelte:element this="div"> is valid code. This makes little sense — you should just do <div>. In the vanishingly rare case that you do need to use a literal value for some reason, you can do this:


<svelte:element this={"div"}>
Note that whereas Svelte 4 would treat <svelte:element this="input"> (for example) identically to <input> for the purposes of determining which bind: directives could be applied, Svelte 5 does not.

mount plays transitions by default
The mount function used to render a component tree plays transitions by default unless the intro option is set to false. This is different from legacy class components which, when manually instantiated, didn’t play transitions by default.

<img src={...}> and {@html ...} hydration mismatches are not repaired
In Svelte 4, if the value of a src attribute or {@html ...} tag differ between server and client (a.k.a. a hydration mismatch), the mismatch is repaired. This is very costly: setting a src attribute (even if it evaluates to the same thing) causes images and iframes to be reloaded, and reinserting a large blob of HTML is slow.

Since these mismatches are extremely rare, Svelte 5 assumes that the values are unchanged, but in development will warn you if they are not. To force an update you can do something like this:


<script>
	let { markup, src } = $props();

	if (typeof window !== 'undefined') {
		// stash the values...
		const initial = { markup, src };

		// unset them...
		markup = src = undefined;

		$effect(() => {
			// ...and reset after we've mounted
			markup = initial.markup;
			src = initial.src;
		});
	}
</script>

{@html markup}
<img {src} />
Hydration works differently
Svelte 5 makes use of comments during server-side rendering which are used for more robust and efficient hydration on the client. You therefore should not remove comments from your HTML output if you intend to hydrate it, and if you manually authored HTML to be hydrated by a Svelte component, you need to adjust that HTML to include said comments at the correct positions.

onevent attributes are delegated
Event attributes replace event directives: Instead of on:click={handler} you write onclick={handler}. For backwards compatibility the on:event syntax is still supported and behaves the same as in Svelte 4. Some of the onevent attributes however are delegated, which means you need to take care to not stop event propagation on those manually, as they then might never reach the listener for this event type at the root.

--style-props uses a different element
Svelte 5 uses an extra <svelte-css-wrapper> element instead of a <div> to wrap the component when using CSS custom properties.