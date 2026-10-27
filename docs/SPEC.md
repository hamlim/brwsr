# brwsr: product and architecture spec (v1)

Name: **brwsr** (may change later). Bundle id prefix `app.brwsr`. Goal: an Arc-style browser,
shipped as native apps, with our own end-to-end encrypted sync service.

v1 folds in the review in `docs/REVIEW-for-fable.md` and `docs/REVIEW-crest-vs-spec.md`.
Details that the review asked for live in two ADRs:

- `docs/adr/0001-e2ee-sync.md`: keys, enrollment, op format, conflict rule, recovery.
- `docs/adr/0002-engine-pipeline.md`: CEF, Chromium pin, patches, licenses, build, update.

Fixed decisions: CEF (Chrome runtime) on macOS, WKWebView on iOS, E2EE from day one, name brwsr.

---

## 1. Platforms and engines

| Target | Phase | Shell | Engine |
|---|---|---|---|
| macOS | 1 | Swift (AppKit + SwiftUI), Objective-C++ bridge to CEF | Chromium via CEF, Chrome runtime |
| iOS | 2 | Swift (SwiftUI) | WKWebView. Apple allows other engines only in some regions; we do not use that |
| Windows, Linux, Android | later | TBD | same Chromium tree |

**Engine embedding.** Native Swift shell owns every piece of UI. CEF in Chrome runtime mode
supplies page rendering, navigation, downloads, prefs, profiles, devtools, printing, and
extension loading. Direct `content` embedding is a fallback for single features only.

**Chromium source.** We pin a CEF branch, and through it a Chromium tag. On top we apply the
ungoogled-chromium patch set and our own patches. Helium patches are **not** taken (Helium's
own code is GPL-3.0). What that costs and how we replace it is in ADR 0002. Ad blocking is a
bundled uBlock Origin extension, not a patch, and depends on the M1 extension result.

**No schedule claims before M1.** "Weeks to a usable browser" is removed. The first estimate
for the macOS alpha is made after M1 reports.

---

## 2. Feature set

**P0** = needed to ship the public beta. **P1** = first follow-up. **P2** = later.

### 2.1 Sidebar and tabs (desktop)

- **P0** Vertical sidebar, left, collapsible (`⌘S`). Sections: Favorites, Pinned, Today.
- **P0** **Favorites**: icon grid at the top. Shared across all Spaces that use the same
  Profile. Each Profile has its own live page for a Favorite. A Favorite never shows in a
  Space whose Profile does not own it.
- **P0** **Pinned tabs**: persistent per Space. A pinned tab stores a `homeURL` and a
  `currentURL`. "Reset" returns to `homeURL`.
- **P0** **Today tabs**: unpinned. Auto-archive after N hours (default 12h, user set, or off).
  The clock counts from the last activation. Tabs playing audio or video, with an active
  call (camera or mic in use), with an in-progress download, or with a `beforeunload` handler
  are not auto-archived.
- **P0** **Folders** in Pinned, one level deep.
- **P0** Drag and drop between Favorites, Pinned, Folders, Today, and across Spaces. Moving a
  tab to a Space with a different Profile opens the URL in the destination Profile. It does
  not move the live page or its session.
- **P0** Tab search / switcher in the command bar.
- **P0** Archive view with restore, searchable, 30-day retention. A restored tab keeps its
  item id and `currentURL`; the page reloads.
- **P1** Rename tab. Media control card. Downloads in the sidebar (downloads themselves are P0,
  see 2.7).
- **P2** Live Folders.

### 2.2 Spaces and Profiles

- **P0** **Spaces**: named tab sets with icon and color theme. Switch with `⌃Tab`, `⌃1..9`,
  two-finger swipe on the sidebar.
- **P0** Each Space binds to one **Profile**: cookie jar, local storage, history, extensions,
  permissions, credentials. Many Spaces can share one Profile. The command bar, history,
  permission prompts, and autofill are filtered to the Profile of the active Space.
- **P0** Space theme: color or gradient on the window chrome.
- **P1** Per-Space default search engine.

### 2.3 Command bar

- **P0** `⌘T` floating command bar: URL, search, open tabs, history, bookmarks. `⌘L` edits in
  place. `>` prefix runs commands.
- **P1** Inline calculator and unit conversion.

### 2.4 Window and view modes

- **P0** **Split view**: 2 to 4 panes in one sidebar item, resizable. A split is an item with
  an ordered list of child tab ids and a layout string. Children are ordinary tab items.
- **P0** **Little window**: links from other apps open in a small floating window with
  "open in Space" and "close".
- **P0** **Peek**: a link from a Favorite opens in an overlay; escape dismisses it.
- **P0** Fullscreen and hidden sidebar. Hover the left edge to reveal.
- **P1** Auto picture-in-picture. Always-on-top mini window.

### 2.5 Boosts

- **P1** Per-site CSS and JS with a Zap element remover. Per-site colors and fonts.
- **P1** CSS Boosts sync. JS Boosts do not sync by default.
- **P2** Import and export by file.

### 2.6 Routing

- **P0** Default-browser registration and protocol handlers on macOS.
- **P1** Link routing rules: URL pattern to Space.
- **P2** Scriptable actions.

### 2.7 Standard browser features

- **P0** History, bookmarks, import from Chrome, Arc, and Safari, find in page, print, PDF
  viewer, devtools, downloads (working downloads, not the sidebar UI), site permissions UI,
  incognito window per Profile, media keys, tab mute, crash recovery of sessions.
- **P0** Passwords: autofill from the macOS Keychain and from password manager extensions.
- **P0** Extensions from the Chrome Web Store, to the support level that M1 records. The
  release criterion is the M1 report, not "if CEF allows it". The M1 report lists what is
  supported, what is not, and which CEF patches we carry.
- **P1** Our own encrypted password vault with autofill and sync (see 2.9). Reader mode.
  uBlock Origin bundled.

### 2.8 Mobile (iOS, phase 2)

- **P0** Sign in and enroll the device from a desktop device or a recovery key, on a real
  device, before M5 counts as done.
- **P0** Same Spaces, Favorites, Pinned tabs, Folders as desktop. Bottom sheet sidebar. Swipe
  between Spaces. Today tabs local to the device and synced read-only to other devices.
- **P0** Bottom URL and search bar. Reader mode. Share extension that writes to the local store
  and outbox, so it works offline. Default browser.
- **P0** Catch-up sync on foreground. Durable offline writes. Correct behavior when iOS
  suspends the app mid-sync.
- **P0** One `WKWebsiteDataStore` per Profile. Verify on the chosen minimum iOS version.
- **P1** Send tab to device. WebKit content blocker rules from the same lists as desktop.
- **P1** iPhone Duo: sidebar and page side by side when unfolded, single column when folded,
  state preserved across the fold.
- **P2** CSS Boosts.
- Engine features are exposed through a capability interface in the shared core. Desktop
  extensions, cookie attributes, credential access, and media behavior are not assumed to
  exist on iOS.

### 2.9 Account and sync

- **P0** Login for the internal alpha: email magic link only. Sign in with Apple, Google, and
  passkeys before public beta.
- **P0** **End-to-end encryption for all synced user data.** The service stores ciphertext
  plus the metadata listed in ADR 0001 section "What the service sees".
- **P0** Synced: Spaces, Profiles (name, icon, which Spaces use it; never cookies or storage),
  Favorites, Pinned tabs, Folders, Splits, Archived tabs, account settings, routing rules,
  CSS Boosts, keyboard shortcuts, extension list (ids and enabled state, per Profile).
- **P0** Per device: Today tabs, visible read-only on other devices. Device settings
  (sidebar width, window positions) are not synced.
- **P0** Devices list. Remote sign-out: new ops from that device are rejected within 60 s,
  its WebSocket is closed, and its refresh token is deleted. A removed device already holds
  the account key, so rotation protects future data only. Rotation on removal is P1.
- **P1** History sync (opt in).
- **P1** **Password vault sync** (opt in).
- **P1** **Session transfer for cookies** (opt in, default off). This is not "sync all cookies".
  Rules:
  - The user approves a host and a receiving device. Precedence: deny list beats allow list;
    a host rule covers its subdomains unless a more specific rule exists. Changing a rule
    deletes already transferred copies on all devices.
  - Transferred: persistent cookies only, with Profile, partition key, domain, host-only flag,
    path, name, value, expiry, Secure, HttpOnly, SameSite. Session cookies, incognito, and
    local storage are not transferred. Jars never merge across Profiles. A cookie the receiving
    engine cannot represent is rejected, not approximated.
  - Logout and token rotation propagate as their own ops: a cookie deletion or replacement on
    the source device wins over any older copy, and an older copy can never resurrect a
    cookie after a newer deletion. This uses the per-item version in ADR 0001.
  - We tell users this transfers some logins and that sites may still ask them to sign in.
- **P2** Share a Space by link. Needs per-Space keys (ADR 0001).
- **Never synced**: raw cookie jars, local storage, Keychain items.

### 2.10 Not in v1

Arc Max AI features, Easels, Notes, mail or calendar, "Browse for me".

### 2.11 Quality bars (P0)

- Cold start to first paint under a target set after M1 (measured, not guessed).
- Full VoiceOver coverage of the sidebar, command bar, and prompts. Keyboard-only operation.
- No user browsing content in logs, telemetry, or crash reports. Crash reports upload only
  with user consent and are scrubbed of URLs and titles.

---

## 3. Architecture

### 3.1 Repo layout

```
brwsr/
  docs/                 spec, ADRs, reviews
  engine/               pin.json (CEF branch, Chromium tag, patch revisions, toolchain,
                        SDK, flags), patches/, build and verify scripts
  packages/
    core-swift/         model, SQLite store, outbox, crypto, sync client, capability
                        interface. Shared by macOS and iOS. No UI.
    protocol/           sync protocol schema, generated TypeScript and Swift types
  apps/macos/           AppKit/SwiftUI shell, Objective-C++ CEF host
  apps/ios/             SwiftUI, WKWebView
  service/worker/       Cloudflare Worker (Hono): auth, HTTP API, WebSocket upgrade
  service/do-user/      UserSync Durable Object
  service/migrations/   D1 migrations
```

### 3.2 Client data model

A synced thing is an **item**: `id` (UUIDv7), `kind`, `parentId`, `order` (fractional key,
ties broken by `id`), `version` (integer, per item, server assigned), and an encrypted
`fields` record. Kinds: `space`, `profile`, `favorite`, `pinnedTab`, `folder`, `split`,
`todayTab`, `archivedTab`, `boost`, `routeRule`, `setting`, `cookie`, `credential`, `device`.

Local store: SQLite with an `items` table and an `outbox` table. A UI edit writes the item
and its outbox row in one transaction. The outbox row is deleted only after the server ack.

### 3.3 Sync protocol (summary, full contract in ADR 0001)

- One **UserSync Durable Object** per user, SQLite backed, hibernatable WebSockets.
- An op is a **whole-item write**: `{ opId, deviceId, itemId, kind, parentId, order,
  baseVersion, del, nonce, ciphertext }`. `ciphertext` is the complete encrypted field record.
  Metadata is bound into the AEAD associated data, so it cannot be changed by the service.
- Server rule: accept if `baseVersion` equals the item's current version, assign `seq` and
  `version = baseVersion + 1`, commit, then ack. Otherwise reject as **stale** and return the
  current item. The client rebases: it re-applies its own field changes on top of the current
  record and resubmits. The rebased op is a new whole-item write, so what the server stores is
  always canonical. No client does silent field merging on receive.
- A rejected op never leaves the outbox until it is rebased or the user discards it.
- Duplicate `opId` returns the original ack. Acks are sent only after durable commit.
- Snapshot: all live items plus the `seq` they are consistent with, served from one SQLite
  read transaction. Snapshot and replay yield the same state.
- Op log retained 30 days. A client with an older cursor must take a snapshot, drop outbox
  entries for items absent from the snapshot, and rebase the rest. Deleted items cannot
  return after tombstone expiry because the stale client's write fails the version check.
- Device Today tabs expire if the device is silent 7 days (DO alarm).

### 3.4 Service components

| Piece | Use | Needed by |
|---|---|---|
| Worker (Hono) | auth, API, WebSocket upgrade, routing to the DO | M2 |
| UserSync DO | items, op log, devices, live fan-out, alarms | M2 |
| D1 | account index: user id, auth identities, one-time login tokens (atomic consume) | M2 |
| Email Service | magic link mail | M2 |
| Turnstile | sign-up abuse control | public beta |
| R2 | blobs, encrypted client side with the account key. Deferred until a blob type exists | later |

Magic link tokens are consumed with a single atomic D1 `UPDATE ... WHERE used = 0`. KV is
not used for one-time tokens because it is eventually consistent.

Ownership: D1 owns accounts and auth identities. The DO owns the device registry, device
public keys, and revocation. A hibernated socket is re-checked against the revocation list
on every message.

Limits: op payload 64 KB, items per account 100k, op log 30 days, service logs 7 days with
no user content. Devices per account: no user-facing limit. An abuse cap of 50 exists in the
DO and is raised on request.

### 3.4a Business model

The browser is free. The **account** is the paid product: one subscription, monthly or
annual with a discount, unlimited devices. Without an account the browser works fully but
does not sync. Billing through Stripe on the web and through StoreKit on iOS, because Apple
requires in-app purchase for a subscription sold inside an iOS app. D1 holds plan state;
the DO reads it on connect and refuses sync for a lapsed account after a 30-day grace period.
Data is kept 90 days after lapse, then deleted. Trial length is open.

### 3.5 E2EE summary

Full design in ADR 0001. Each device has a keypair in the Secure Enclave or Keychain. The
account key is generated on the first device and wrapped to each enrolled device's public key.
Enrollment uses a short code as the secret in a PAKE between the two devices, relayed as
opaque bytes by the service, with a QR path that also carries a key fingerprint. A 24-word
recovery phrase (BIP39, with checksum) wraps the account key. Cookies and credentials use
sub-keys derived with HKDF from the account key.

What the service sees: user id, device ids and public keys, item ids, kinds, parent ids, order
keys, versions, sizes, timestamps, and traffic timing. Nothing else.

Rollback and tampering: AEAD over metadata rejects any modified op. Clients detect missing
ops by `seq` gaps and per-item `version` gaps. Full protection against a malicious service
that serves a consistent old view to one device is not provided in v1 and is documented.

### 3.6 Engine pipeline (summary, full in ADR 0002)

`engine/pin.json` pins CEF branch, Chromium tag, ungoogled-chromium revision, our patch list
and order, Xcode and macOS SDK versions, GN flags. A clean runner reproduces the build and
publishes a signed tarball with a hash manifest. The app verifies the hash and signature of
any engine it loads. Build time and disk use are measured in M1, not assumed.

Security updates: the 7-day clock starts at the upstream Chromium security release. The
engine ships independently of UI work. One named owner handles the case where CEF or
ungoogled-chromium lags; the fallback is to ship the Chromium tag without the lagging patches.

---

## 4. Milestones

| # | Milestone | Done when |
|---|---|---|
| M0 | Spec and ADRs | this doc, ADR 0001, ADR 0002 |
| M1 | Engine trial | The pinned, patched CEF build runs inside a Swift host with native views for everything but the page. Verified: keyboard focus, IME, VoiceOver, menus, permission prompts, popups, two Profiles, four panes in one window, clean shutdown, Objective-C++ boundary and thread ownership documented. Extensions: a password manager, uBlock Origin, and one work extension installed, restarted, updated, removed; popups, tabs/windows APIs, service workers, native messaging under brwsr's signing identity, private-mode isolation. Report records supported, unsupported, patches carried, clean build time, disk, memory, incremental build time, both CPU architectures. Estimate for M3 produced |
| M2 | Sync service | Worker and DO deployed. Three CLI clients converge after conflicting offline edits, duplicate sends, a lost ack, and a kill mid-write. One client starts from a snapshot. One returns after tombstone expiry without restoring deleted data. Enrollment with a substituted public key fails. Reused magic link fails. Revoked device cannot write. Recovery phrase restores on a clean device. Modified ciphertext, metadata, and deletions are rejected |
| M3 | macOS alpha, daily-drivable | Sidebar, Spaces, Profiles, command bar, Favorites/Pinned/Today/Folders, archive, split view, Peek, Little window, downloads, permissions, session recovery, default browser, Keychain autofill, import, sync, signed and notarized builds, signed engine download and failed-update recovery |
| M4 | macOS beta | Extensions at the M1-recorded level, media card, link routing, Boosts CSS, Sparkle-style auto update, crash reporting with consent |
| M5 | iOS alpha | Real-device enrollment and recovery, Spaces and pinned tabs, bottom bar, reader, share extension offline, foreground catch-up, suspension-safe sync |
| M6 | Public beta | Apple, Google, passkeys login; Turnstile; TestFlight; one engine version bump demonstrated end to end |

---

## 5. Decisions made

- CEF (Chrome runtime) is the engine embedding. `content` embedding only as a per-feature fallback.
- E2EE from day one. Whole-item writes with version check. No silent client-side field merge.
- Cookie feature is "session transfer", opt in, per host and per device. Password vault is P1.
- Favorites belong to a Profile. Tabs that cross Profiles reopen by URL.
- Name brwsr. Alpha login is magic link only.
- Engine patches: ungoogled-chromium only. Helium patches are not taken (GPL-3.0). Features
  Helium adds that we want are reimplemented in the Swift shell or as extensions.
- Minimum OS: the current major version and the one before it. At launch planning time that
  is macOS 26 and 27, iOS 26 and 27. The iPhone Duo ships on iOS 27.1, so it is covered.
  Fold-aware layouts for the Duo are P1, tracked in 2.8.
- Business model: paid account, unlimited devices, monthly or annual. See 3.4a.

## 6. Open decisions

1. PAKE choice (CPace or OPAQUE) and AEAD library for all three platforms. ADR 0001 lists the
   candidates and the test that settles it.
2. Subscription price and trial length.
