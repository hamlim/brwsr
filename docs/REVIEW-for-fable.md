# brwsr spec review for Fable

Reviewed 2026-10-27 against `docs/SPEC.md` v0.

**Keep the product direction. Resolve the sync and engine contracts before full implementation.**
CEF, E2EE at launch, and the name brwsr remain fixed. Cookie and password sync remain opt-in P1 features.
The findings below identify missing behavior and tests. They do not propose a different browser architecture.

## Required before M1/M2 are complete

### 1. The conflict model can produce different results on different devices

**Sections: 3.2, 3.3, 3.6. Severity: data correctness blocker.**

The server keeps the last encrypted item. Clients can also merge fields “when they can.”
Example: device A renames a pinned tab while offline. Device B changes its home URL.
Existing clients could preserve both edits. A new device receives only the server's last whole-item snapshot and loses one edit.
The spec does not define how a merged result becomes the canonical server record.
It also does not say whether `set` contains a complete record or only changed fields.

**Recommendation:** select one complete merge contract in the E2EE ADR.
For the smallest v0, use whole-item replacement with a base-version check and explicit stale-write handling.
Remove optional local field merging. A rejected stale edit must remain recoverable, not disappear from the outbox.
If automatic field merging is required, specify encrypted field versions, deterministic merge rules, and publication of the merged result.
In either design, snapshot recovery must produce the same state as operation replay.

Also define atomic local store/outbox updates, server-side `opId` deduplication, and acknowledgment after durable commit.
Make snapshot contents and their `seq` consistent. Define expired cursors, replay limits, and schema upgrades.
A device offline for more than 30 days must not restore an item whose tombstone was removed.
Fractional indexing needs a stable tie-breaker for equal keys, plus compaction and concurrent-move rules.
E2EE prevents content-based server deduplication; it does not prevent duplicate-operation detection by `opId`.

**Acceptance:** three clients converge after conflicting offline edits, duplicate sends, a lost acknowledgment, and process termination.
One client must start from a snapshot. Another must return after tombstone expiry without restoring deleted data.

### 2. E2EE specifies a cipher family but not a complete trust model

**Sections: 2.9, 3.4, 3.6. Severity: security blocker.**

The service could substitute a new device's public key unless enrollment authenticates that key.
The operation format does not state how metadata or deletions are authenticated.
The recovery words have no defined encoding or key-wrapping function.
Removing a refresh-token row does not invalidate issued JWTs, close WebSockets, or erase a copied account key.

**Recommendation:** separate account authentication, device authorization, and access to encryption keys.
Choose a reviewed enrollment protocol. Specify QR verification and protection for short codes, including expiry and attempt limits.
Use an established PAKE or equivalent authenticated method where a short code is the shared secret.
Specify cipher/library versions, nonce rules, and authenticated metadata: account, item, operation, kind, parent, order, schema, and key version.
Authenticate deletions. Define replay and rollback protection against an active service, not only passive database access.

Define recovery-word entropy and checksum, key wrapping, recovery tests, Keychain storage, and key rotation.
Remote sign-out must stop new operations and close active connections. State the maximum delay before revocation takes effect.
Key rotation can protect future data; it cannot remove plaintext or keys already copied by a device.
Per-kind keys do not provide per-Space sharing. Add distinct Space keys before implementing shared Spaces.

Replace “only ciphertext” with a precise disclosure statement. The proposed service also sees kinds, parent links, order, device links, and traffic timing.
Specify encrypted handling for R2 objects. Exclude browsing content from service logs, telemetry, and crash-report uploads unless separately authorized.

Do not implement one-time magic links through a KV read followed by a delete alone.
KV is eventually consistent; use an atomic consume operation in a strongly consistent store. [Cloudflare KV documentation][kv]
Keep account and device registry ownership clear between D1 and the user DO. Hibernation must preserve authorization state correctly.

**Acceptance:** a substituted enrollment key fails verification; a reused login token fails; a revoked connection cannot submit operations.
Recovery works on a clean device. Modified ciphertext, metadata, and deletion records are rejected.
Document which rollback attacks are detectable and the limits when every trusted device is lost.

### 3. Cookie sync uses an invalid test for device-bound sessions

**Section: 2.9. Severity: security and data-loss risk; resolve before P1 implementation.**

`__Host-` and `__Secure-` constrain cookie scope and secure delivery. They do not identify device binding.
Cookies without these prefixes can still grant account access or depend on device state. [HTTP cookie draft][cookies]
Excluding session cookies and local storage does not make persistent cookie transfer safe or complete.
Generic LWW can restore a logged-out session or replace a rotated token with a stale value.

**Recommendation:** keep sync off by default and begin with explicit host and receiving-device approval.
Define allow/deny precedence, subdomain rules, and removal of existing synced copies after policy changes.
Preserve Profile, partition, domain, host-only flag, path, name, expiry, Secure, HttpOnly, and SameSite information.
Reject transfers that the receiving engine cannot represent. Never merge cookie jars across Profiles.
Specify logout propagation, expiry, token rotation, and stale-update rejection separately from ordinary tab sync.
Describe the feature as limited session transfer; do not promise that it restores all website logins.

Resolve the password-vault conflict: §2.7 says P2; §2.9 requires the vault at P1.
Define origin matching, approval for autofill, and password deletion. Keychain storage alone does not supply an autofill system.

**Acceptance:** logout stays effective after an offline device reconnects; expired tokens do not return.
Private data never enters the outbox. Partitioned cookies remain isolated or are rejected on transfer.

### 4. M1 does not test the difficult part of the selected CEF integration

**Sections: 1, 2.7, 4. Severity: feasibility blocker.**

Loading a page and one extension does not prove that Swift can own all browser UI.
CEF still has open issues for embedded non-Views windows and programmatic extension management.
These issues identify integration risks; they do not prove that the intended design is impossible. [CEF #3294][cef-host], [CEF #3450][cef-ext]

**Recommendation:** keep CEF. Expand M1 to test the exact Chrome runtime, browser style, native host, and patched build together.
Specify the Swift/Objective-C++ boundary, event-loop integration, thread ownership, helper processes, and page destruction.
Test keyboard focus, IME, accessibility, menus, permission prompts, popups, multiple Profiles, split panes, and shutdown.
Test a password manager, a content blocker, and a work extension through install, restart, update, and removal.
Include extension popups, tab/window APIs, service workers, native messaging, and private-mode isolation.
Use brwsr's signing identity when testing native companion apps; installation alone does not prove that the companion trusts the browser.

Replace “usable browser in weeks” with an estimate made after M1.
Record supported features, missing features, required CEF patches, and an explicit extension release policy.
“P0 if CEF allows it” is not a release criterion. List the tested support required for M3 and M4.

**Acceptance:** the selected host passes these checks on the pinned engine, with known limits recorded.
Repeat the critical checks after an engine update. Do not infer compatibility from a different CEF sample configuration.

### 5. The custom engine pipeline lacks compatibility, licensing, and release rules

**Sections: 1, 2.7, 3.5. Severity: maintenance and distribution blocker.**

The proposed pin includes Chromium and Helium but omits CEF and other build inputs.
CEF itself names a specific Chromium checkout. Patch application is not proof of compatibility. [CEF compatibility file][cef-pin]
Helium's original code is GPL-3.0; imported code retains its original terms. Taking patches does not remove their license duties. [Helium license][helium-license]
Helium also downloads a customized uBlock build. Its privacy patches alone do not establish the promised ad blocker. [Helium dependencies][helium-deps]

**Recommendation:** pin CEF, Chromium, patch revisions, patch order, toolchain, SDK, build flags, and artifact hashes.
Review the selected patch licenses and the combined distribution before selecting brwsr's license.
Specify the blocker component and filter-update path.
Measure clean-build duration, peak disk use, memory, and incremental rebuild cost on arm64 and x86_64.
Treat the current “hours” and “~100 GB” statements as estimates, not capacity requirements that have been verified.

Define authenticated engine downloads, sandbox checks, signing, notarization, update verification, and recovery from a failed update.
Start the seven-day security clock from the upstream security release. Assign an owner and a process for when Helium or CEF lags.
Allow an urgent security update without waiting for unrelated UI work.

**Acceptance:** a clean runner reproduces the selected build; the app rejects a modified artifact.
Demonstrate one engine-version update and a failed-update recovery before calling the release pipeline complete.

## Product and platform changes for the next spec revision

1. **Define Profile ownership (§§2.1, 2.2, 3.2).**
   Global Favorites have no stated login context when Spaces use different Profiles.
   Recommended default: show Favorites across Spaces that share a Profile; use a separate live page per Profile.
   Moving a tab between Profiles should open its URL in the destination session, not transfer its live authenticated page.
   Apply Profile filtering to command-bar history, permissions, credentials, and extensions.

2. **Complete the model and lifecycle rules (§§2.1, 2.4, 3.2).**
   Specify split membership and layout, pinned home versus current URL, window ownership, and tab identity during archive/restore.
   Separate account settings, Profile settings, and device settings. Define whether extension metadata syncs.
   Define the auto-archive clock and exclusions for calls, playback, downloads, and unsaved work.
   Define crash recovery, memory-pressure eviction, and private-data cleanup.

3. **Make M3 genuinely suitable for daily use (§§2.4, 2.6, 2.7, 4).**
   Split view, Peek, and Little-window are P0 but appear in M4. Default-browser registration is P1.
   Password-vault priority conflicts. Import is P0 but also appears in M4. Resolve these differences explicitly.
   Separate working downloads from the optional sidebar download UI.
   Put reliable downloads, permissions, session recovery, default-browser handling, and a secure update path in M3.
   Make signing available before tests that depend on the browser identity; do not leave every signing task until M6.

4. **Define the iOS contract (§§1, 2.8, 3.1, 4).**
   Keep WKWebView. Replace the absolute App Store claim: regional alternative-engine entitlements exist. [Apple engine rules][apple-engines]
   Select the minimum OS and test persistent website data stores per Profile.
   Share model and sync behavior, but expose engine capabilities through an interface.
   Do not assume desktop extensions, cookie attributes, credential access, or media behavior carry across.
   Specify foreground catch-up, durable offline writes, share-extension handoff, and behavior during background suspension.
   Add real-device enrollment and recovery tests before M5 is complete.

5. **Reduce early service scope without weakening E2EE (§§2.9, 3.4, 6).**
   Consider one login method for the internal alpha, then add the others before the intended public release.
   Specify which services M2 actually needs. Defer R2 features until the first required blob type exists.
   Define message sizes, storage quotas, log retention, retry behavior, and large-account recovery.
   Plan limits affect the protocol and user experience, not only D1 as §6 claims.
   Add performance and accessibility acceptance criteria instead of fixing more minor gestures in v0.

## Requested handback from Fable

Return a revised spec plus two ADRs: **E2EE/sync** and **CEF/engine distribution**.
Each ADR should state the chosen behavior, remaining unknowns, and a test that resolves each unknown.
Keep CEF, E2EE at launch, and brwsr fixed. Align priorities with M1–M6 and remove unsupported schedule claims.
Do not claim implementation feasibility, build cost, or extension compatibility until the relevant trial has run.

Review method: read the current spec and primary platform sources. No engine build or protocol implementation was tested.
The source spec was not changed.

[kv]: https://developers.cloudflare.com/kv/concepts/how-kv-works/
[cookies]: https://httpwg.org/http-extensions/draft-ietf-httpbis-rfc6265bis.html
[cef-host]: https://github.com/chromiumembedded/cef/issues/3294
[cef-ext]: https://github.com/chromiumembedded/cef/issues/3450
[cef-pin]: https://github.com/chromiumembedded/cef/blob/master/CHROMIUM_BUILD_COMPATIBILITY.txt
[helium-license]: https://github.com/imputnet/helium/blob/main/LICENSE
[helium-deps]: https://github.com/imputnet/helium/blob/main/deps.ini
[apple-engines]: https://developer.apple.com/support/alternative-browser-engines/
