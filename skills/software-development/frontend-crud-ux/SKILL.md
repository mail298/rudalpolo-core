---
name: frontend-crud-ux
description: "Use when building client CRUD apps to enforce safe UX."
version: 1.0.0
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [frontend, react, ui-ux, crud, state-management, filter-safety]
---

# Frontend CRUD & Client-Side Collection UX

## Overview

Client-side collections (notes, tasks, cards, tickets, bookmarks) frequently suffer from UX antipatterns that confuse users: over-engineered hierarchies, creation-filter collisions where new items immediately disappear, search leaks on locked items, and rigid view layouts.

This skill provides the mandatory engineering patterns and pitfalls for building fast, robust, and intuitive client-side CRUD applications.

---

## Core Engineering Workflows

### 1. Mental Model Alignment (Zero-Friction First)

When designing or implementing personal productivity tools (scratchpads, notes, to-dos):
- **Detect Intent:** Differentiate between lightweight scratchpad tools (e.g., Sticky Notes, Quick Capture) and structured relational tools (e.g., Knowledge Bases, ERPs).
- **Zero-Friction Rule:** For quick-capture tools, eliminate title/body modal barriers. Allow immediate typing on open, auto-save on every keystroke (debounced 300-500ms), and eliminate explicit "Save" button requirements.
- **Never Over-Engineer Navigation:** Avoid wrapping simple single-screen collection tools into nested sub-screen tabs, multi-step wizards, or corporate PRD terminology unless explicitly demanded.

---

### 2. Filter-Safe Creation (The Creation-Filter Collision Pattern)

#### The Pitfall
Creating a new entity while a category, color, or tag filter is active causes the new item to be immediately hidden if its default attributes do not match the active filter. The user observes an empty screen or an unchanged count and concludes the creation operation failed.

#### Mandatory Rules
1. **Empty-State Inheritance:**
   - Any "Create Item" call-to-action rendered inside an empty filtered state MUST inherit the currently filtered attribute, never a hardcoded primary default.
   - Example: On an empty "Purple Notes" filter, the button must create a purple note, not fallback to yellow.
2. **Direct Attribute Creation:**
   - Provide multi-attribute split buttons or inline palettes on primary "New" buttons so users can pick the exact category/color on creation.
3. **Auto-Reset Filter on Attribute Mismatch:**
   - If an item is created with an attribute that does not match the current filter, automatically reset the active filter to `all` (or switch to the new item's attribute) and scroll/focus onto the newly created entity. Never leave a user looking at a screen where their newly created item is invisible.

---

### 3. Stealth Privacy & Search Suppression

#### The Pitfall
Masking text on the card canvas while leaving search indexing active leaks confidential information through search match counters and highlighted snippet cards.

#### Mandatory Rules
1. **Strict Content Masking:**
   - When locked/stealth mode is active without verified credentials, both title and body content must be completely masked behind opaque/frosted overlays. Do not render plaintext into the DOM with CSS opacity alone.
2. **Search Suppression:**
   - When collection is locked, suppress search filtering across protected fields. Searching for confidential terms must return zero matches rather than indicating that locked cards contain the query.
3. **Tamper-Resistant Storage:**
   - When PIN or master password protection is enabled, encrypt sensitive payloads in storage (e.g., WebCrypto AES-GCM + PBKDF2) so inspecting local storage or dev tools does not expose raw text.
4. **Immediate Lock on PIN Setup (Never Auto-Unlock):**
   - When a user sets up, enables, or changes their PIN/password, do NOT leave the vault in an unlocked state (`isUnlocked: true`). Immediately lock the application (`isUnlocked: false`) so content masking takes effect instantly. Leaving records exposed right after setup makes the user believe the lock mechanism failed to activate.
5. **Cross-Device Security/Credential Sync:**
   - Security configurations (hashed PIN, salt, auto-lock timeouts) MUST be synchronized with the remote vault payload alongside data records. Never isolate PIN state solely to browser `localStorage`; otherwise, opening the app on a fresh device/browser after login leaves private items completely unprotected.
6. **1-Click Instant Lock Toggle:**
   - When vault protection is active and currently unlocked, clicking the header/toolbar lock control must trigger an instant 1-click lock (`setIsUnlocked(false)`). Never force users to navigate through configuration modals just to lock their screen.
7. **Mandatory Security Hydration in Auth/Sync Modals:**
   - When authenticating or connecting an account via a modal or OAuth flow, the modal component MUST accept security state handlers (`onSaveSecurity`, `onLockNow`) and inspect remote security payloads. If the remote account has an active PIN/credential, the modal MUST persist that configuration locally and trigger `onLockNow()` immediately before closing, guaranteeing that freshly downloaded records never flash or render unmasked on a new device.

---

### 4. Dual-View Scaling & Responsive Maximize UX

#### The Pitfall
Fixed-height card boards (e.g. 280px sticky note cards) become unusable once users write long paragraphs, code blocks, or extensive checklists. Furthermore, when implementing a "Maximize" or "Focus" view, locking the container into an arbitrarily narrow desktop box (e.g. `max-w-2xl` / 672px) cramps the user and wastes wide screens, while formatting toolbars without horizontal overflow clipping defenses break or overflow on small mobile viewports.

#### Mandatory Rules
1. **Ultra-Wide Adaptive Maximize (Avoid Narrow Box Antipattern):**
   - When maximizing an item into focus view, do NOT clamp the modal into a narrow container. Scale dynamically across viewports:
     - **Desktop/Large Monitors:** Use an ultra-wide responsive canvas (`w-full h-[92vh] max-w-[98vw] 2xl:max-w-[1750px]`) with proportional typography (`text-2xl` to `text-3xl` titles, `text-lg` body) so long memos, checklists, and code have generous horizontal and vertical room.
     - **Tablets:** Fill `96%` width and height without wasted side margins.
     - **Mobile Viewports (<640px):** Go true edge-to-edge (`w-full h-full`) with compact padding (`p-2`) to maximize typing area when the on-screen keyboard is active.
2. **Mobile Toolbar Overflow Safety (Anti-Clipped Controls):**
   - WYSIWYG formatting bars with multiple buttons (Bold, Italic, Underline, Strikethrough, Bullet, Checklist) must never clip on narrow phones (320px–380px). Style toolbar button groups with `overflow-x-auto no-scrollbar` and collapse text labels (`hidden sm:inline`), keeping all formatting controls horizontally swipeable and fully touch-accessible.
3. **Effortless Modal Dismissal (Esc & Backdrop):**
   - Always bind global `Escape` key listeners (`window.addEventListener('keydown')`) and background click dismiss (`if (e.target === e.currentTarget) onClose()`) for intuitive keyboard and touch exit.
4. **Full List Mode (Auto-Height Feed):**
   - Provide a global view toggle between **Board/Grid** (compact multi-column view) and **Full List** (single-column auto-height feed).
   - In Full List mode, cards expand vertically to fit their full content without truncation, enabling effortless vertical scanning.
5. **Dedicated Title-Body Visual Hierarchy:**
   - Even in informal card designs, provide a distinct bold Title field separated by a subtle colored divider line from the body to keep cards organized at a glance.

---

### 5. Clean-Slate Initial Session (The Seed Data Privacy Pitfall)

#### The Pitfall
Injecting hardcoded mock data or demo accounts (e.g., `user@example.com`, sample personal to-dos) into uninitialized local storage creates a severe privacy panic. Users opening a new tab, incognito window, or sharing a URL perceive that another user's private data is visible or that the system is public and vulnerable to snooping.

#### Mandatory Rules
1. **Zero Hardcoded Seed Data in Production:**
   - Fresh sessions and uninitialized storage MUST initialize to a clean slate (0 items, disconnected/guest state), never pre-populated with mock identities or sample tasks.
2. **Clean Guest State:**
   - Account badges must explicitly display a neutral disconnected state (e.g., `[ ☁️ Hubungkan Akun ]` with an idle indicator) rather than a mock email or name.
3. **Auto-Lock on Fresh Tab/Page Load:**
   - If vault/PIN protection is configured, any fresh tab, new window, or browser refresh must immediately initialize to a locked state (`isUnlocked: false`). Never leave private data exposed on page refresh.

---

### 6. Real Cloud Sync vs Mock UI (The Cross-Device Sync Pitfall)

#### The Pitfalls
1. **Mock UI Illusion:** Displaying cloud sync or email connection controls that merely simulate synchronization with client-side timers (`setTimeout`) and local state. When users test cross-device workflows (opening a new tab, incognito window, or mobile browser) and enter their email, their notes fail to appear, destroying trust in the product.
2. **Blind Hydration / Overwrite Data Loss:** When an account connects and remote records are found, bluntly replacing the local state (`setItems(remoteItems)`) wipes out any items created locally before authentication.
3. **Identifier Normalization Mismatch:** Treating free-form or OAuth-returned user identifiers without strict normalization (`trim().toLowerCase()`) or failing to verify exact string parity across devices. A subtle user alias or prefix difference (e.g. `mail262@gmail.com` vs `wakmail262@gmail.com`) creates isolated storage partitions where data appears not to transfer.
4. **Logout State Closure Wipeout (The Empty Array Overwrite Pitfall):** When implementing "Log Out and Clear Screen", calling account disconnection alongside a state reset (`setNotes([])`) causes React closure / batching races. The debounced background sync timer sees the previous render's `account.isLoggedIn === true` and dispatches `[]` to the remote store, silently wiping out the user's cloud vault.
5. **Auto-Seed Overwrite Hazard:** Automatically seeding a welcome note when a user connects and immediately pushing it to the cloud without checking if existing cloud notes were temporarily shadowed or delayed.

#### Mandatory Rules
1. **No Simulated Cloud Sync:**
   - Never present an account connection as "Cloud Sync" without a real remote persistence layer (e.g., an SQLite API endpoint or database-backed sync service).
2. **Bidirectional Merging on Connect (Never Overwrite Blindly):**
   - When an account identifier is connected and remote records exist, merge local and remote records:
     - Deduplicate by item unique `id`.
     - Resolve conflicts by comparing `updatedAt` timestamps (latest modification wins).
     - Preserve newly created local items that do not exist remotely.
     - Persist the merged collection locally and push the unified set back to the remote store immediately.
3. **Strict Identifier Normalization:**
   - Normalize user keys (`email.trim().toLowerCase()`) consistently on both client and server before storage lookup or mutation.
4. **Synchronous Ref Guard & Dedicated Local Clear on Logout:**
   - Always mirror authentication state in a synchronous ref (`accountRef.current = account`) and clear all pending debounced sync timers (`clearTimeout(syncTimeoutRef.current)`) immediately when disconnecting.
   - Separate clearing local screen state from cloud synchronization: provide a distinct `onClearLocalNotes()` that never dispatches a remote push.
5. **Server-Side Empty Payload Guard:**
   - The backend sync endpoint (e.g., `sync.php` or REST handler) MUST reject or ignore empty payloads (`count === 0`) if the account already has existing records, unless an explicit `allow_empty: true` parameter is supplied.
6. **Debounced Background Sync on Mutation:**
   - Debounce local mutations (create, edit, delete, pin) by 500–1000ms to automatically push updates to the remote sync endpoint without requiring manual clicks.
7. **Context-Isolated E2E Verification:**
   - Validate cloud sync across two distinct browser contexts (Context A creates and connects; Context B connects the same account in an isolated session and verifies immediate retrieval).
8. **Diagnostic Verification via Server Logs & Storage Keys:**
   - When troubleshooting cross-device transfer failures, inspect the raw server access logs and database records for the exact identifier keys rather than guessing. User credential variations between manual entry and OAuth providers are the most common source of "missing" data.
9. **Unified Security & Entity Payload Contracts:**
   - Remote sync endpoints and client services must treat access credentials (PIN hashes, encryption salts) as first-class citizens in the sync payload alongside record collections. Decoupling security state from entity sync causes fresh clients to restore private records while leaving them in an unauthenticated, fully exposed state.

---

### 7. Access & Session Audit Logging (Edge Proxy IP & Stealth Monitoring)

#### The Pitfall
Exposing an application with multi-device cloud synchronization without access observability leaves administrators blind to account activity, unauthorized access attempts, or synchronization anomalies. Furthermore, naively reading `REMOTE_ADDR` behind CDN/WAF proxies (e.g. Cloudflare, Nginx reverse proxy) logs only internal proxy IPs instead of the user's real public IP, while exposing public monitoring dashboards creates an attack reconnaissance surface.

#### Mandatory Rules
1. **Real Client IP Resolution Behind Proxies:**
   - Always inspect edge proxy headers in priority order:
     `CF-Connecting-IP` (Cloudflare) -> first IP in `X-Forwarded-For` -> fallback to `REMOTE_ADDR`.
   - Capture edge metadata when provided by the reverse proxy (e.g., `HTTP_CF_IPCOUNTRY` for geolocation).
2. **Lightweight Device Parsing Without Heavy Dependencies:**
   - Parse basic device categories (`Desktop`, `Mobile`, `Tablet`) via regex matching on `User-Agent` directly in the backend logger, avoiding bloat from heavy third-party user-agent libraries.
3. **Semantic Action Tagging:**
   - Distinguish operational intent in log records using consistent action verbs:
     `login` (initial account connection), `sync_push` (content mutation), `sync_pull` (initial or periodic sync), `pin_setup` (security change).
4. **Stealth & Private Delivery by Default (Telegram > Public Web):**
   - Do NOT expose sensitive session or login audit trails via web endpoints/dashboards by default, even if protected by stealth keys, as web URLs can be discovered or cause security concern.
   - When users request monitoring for logins/sessions, route alerts directly to private communication channels (e.g. administrator's Telegram bot with debounce rate-limiting) unless they explicitly request a web dashboard. Always clarify the desired delivery mechanism before building.
5. **Debounce Rate-Limiting on Alert Delivery:**
   - When firing external alerts (such as Telegram messages) on login/session actions, apply a 10–15s debounce check against recent log entries to prevent duplicate notifications from paired GET/POST synchronization requests.

---

### 8. ContentEditable & Conditional Render Hydration (The PIN/Tab Unlock Content-Wipe Pitfall)

#### The Pitfall
In applications with conditional rendering (e.g. stealth PIN lock masking, full-screen focus modals, or tab switching), the editor DOM node is unmounted when locked and remounted when unlocked. If content hydration relies on `useEffect(() => { ... }, [note.content])`, React will NOT re-run the effect upon unlock because `note.content` did not change while locked. Consequently, the newly mounted `contentEditable` element renders completely blank (`innerHTML = ""`). When the user clicks or types into the apparently blank card, `onInput` reads the empty DOM node and saves `""` back to state and cloud database, permanently wiping the note's text.

#### Mandatory Rules
1. **Callback Ref for Guaranteed Instant Hydration:**
   - Attach a callback ref (`setEditorRef`) instead of a raw `useRef` to `contentEditable` elements:
     ```tsx
     const setEditorRef = useCallback((node: HTMLDivElement | null) => {
       editorRef.current = node;
       if (node) {
         const formatted = markdownToHtml(note.content || '');
         if (node.innerHTML !== formatted && document.activeElement !== node) {
           node.innerHTML = formatted;
           setIsEmpty(!node.textContent?.trim());
         }
       }
     }, [note.content]);
     ```
   - Callback refs execute synchronously the exact instant the element is created and mounted into the DOM upon unlock, guaranteeing that `innerHTML` is populated before the browser paints and before any user input or focus event can fire.
2. **Mount/Lock State in Effect Dependencies:**
   - If maintaining a secondary synchronization effect, always include the conditional visibility flag in the dependency array (e.g. `[note.content, isLocked]`), and guard against overwriting active user typing (`document.activeElement !== editorRef.current`).
3. **Empty-State Initialization Parity:**
   - Initialize placeholder and empty indicators from the actual entity content (`useState(!note.content?.trim())`), never defaulting to an uninitialized boolean (`false`) that flashes placeholder text over real content.
