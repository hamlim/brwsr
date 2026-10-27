# brwsr: Crest compared with the spec

Review date: 2026-10-26. Keep the fixed decisions: **CEF, E2EE at launch, and the name brwsr**.

**Recommendation: keep the spec approach, but complete the engine and sync trials before building the full UI.**
Crest is a useful source of working designs and test cases. A full fork would require major changes to meet the fixed decisions.
Changing its appearance is the small part. Its Chromium host, profile model, core language, and sync system differ from the spec.

Evidence: read `docs/SPEC.md`; cloned Crest outside this repo at `/tmp/crest-review-20261026`.
All Crest paths below refer to [commit a6f0f0c35427c5cf9029a1f01bd47d20ea3b42a6][crest], dated 2026-10-25.
Read the live GitHub release and issue APIs and the YAB website and manual.
No app was built or run. Tests were read, not executed. Runtime compatibility, build duration, and daily use remain unverified.

## 1. Spec review

### E2EE and conflict rules: resolve before M2

1. **The merge rules do not define one result (§§3.2, 3.3, 3.6).**
   Suppose two offline devices change different fields in the same encrypted item.
   The server keeps the last whole item. Existing clients may merge both changes; a new client gets only the winning snapshot.
   The new client can therefore show different data. The format does not say whether `set` contains a full record or a field delta.
   Choose one rule: whole-item replacement with explicit lost-update behavior, or encrypted field operations with deterministic client replay.
   For field operations, define causal versions, merge output, and encrypted checkpoints that produce the same result as replay.
   Do not leave field merging as an optional client action. E2EE does not itself require whole-item LWW.
   M2 should test three devices, including one that starts from a snapshot after conflicting offline edits.
   Repeat with duplicate operations, interrupted writes, revoked devices, and an expired replay cursor.
2. **Transport order is not user intent (§3.3).**
   A stale offline edit can arrive last and replace newer work. State this behavior, or add base versions and conflict handling.
   Persist `opId` deduplication, the item change, and `seq` in one transaction before acknowledgment.
   Define the snapshot sequence boundary, replay pagination, outbox rebasing, and recovery after a lost acknowledgment.
   A device offline beyond the 30-day tombstone period must not restore deleted records. Require a full resync or retain deletion knowledge.
   Fractional keys can collide. Add a stable tie-breaker, a compaction rule, and rules for concurrent moves and deleted parents.
3. **Encryption lacks an authenticated protocol (§3.6).**
   Define the threat model, including an active service that substitutes keys, replays records, or changes metadata.
   Bind account, item, kind, parent, order, operation identity, schema, and key version to authenticated data.
   Specify nonce rules for the selected cipher. Retries must resend the same encrypted operation safely.
   Authenticate deletions as well as edits. Define replay and rollback detection; a server-assigned `seq` alone is not cryptographic proof.
   The stated leak is larger than counts and timing: kind, parent, order, and device links reveal structure.
   Clarify “only ciphertext” in §2.9. Put content-bearing metadata inside encryption unless the service needs it.
4. **Enrollment, recovery, and revocation need separate rules (§§2.9, 3.4, 3.6).**
   Account sign-in must not grant access to the account key.
   Use an established authenticated exchange. A short code needs a PAKE or equivalent protection, expiry, and attempt limits.
   Define QR key verification, device approval, and protection against service key substitution.
   Specify what the 24 words encode, their checksum, and how they recover or unwrap the account key.
   Store keys in the Keychain. Define key versions, rotation, recovery tests, and removal of a compromised device.
   Remote sign-out cannot erase keys or plaintext already copied. Protect future data through key rotation where required.
   Per-kind keys do not provide per-Space sharing. Design separate Space keys before adding the sharing feature.
5. **The service must preserve these rules (§§3.3, 3.4, 6).**
   KV is eventually consistent. Do not use a KV read/delete pair as the sole control for consuming a magic link once.
   Use an atomic database transition for one-time auth state. [Cloudflare KV behavior][kv]
   Deleting a refresh token does not end an active JWT or WebSocket. Check device revocation and close existing connections.
   Hibernation clears memory. Restore connection identity and authorization, not just the replay cursor. [WebSocket lifecycle][do-ws]
   Define limits, backpressure, log retention, schema upgrades, and encrypted R2 object handling.
   Plan limits affect quotas and protocol behavior, not only the D1 schema as §6 states.

### Cookie and password sync: preserve the P1 goal, add safety criteria

**Cookie prefixes do not identify device binding (§2.9).** `__Secure-` requires secure delivery.
`__Host-` also constrains domain and path. Neither prefix proves that a cookie is bound to a device.
Persistent cookies without these prefixes can still be bearer credentials or depend on device state.
The exclusion rule therefore cannot work as written. [HTTP cookie specification, work in progress][cookies]

Start with an explicit host allow list and selected receiving devices. Keep the global switch off by default.
Define host-only versus domain matching, subdomains, public suffixes, and when policy changes remove existing remote copies.
Identify a cookie by Profile, partition key, domain, path, and name. Preserve expiry, Secure, HttpOnly, and SameSite attributes.
Do not silently discard partition information when moving between Chromium and WebKit. Reject unsupported cases.

Define logout deletion, expiry, concurrent token rotation, and prevention of stale cookie restoration.
Do not apply generic whole-item LWW to an authentication cookie set without testing the site's behavior.
Use privileged cookie APIs. Exclude private and session data at collection, storage, export, and replay boundaries.
Without local storage or device keys, a copied cookie may not restore a login. Present this as limited session transfer.
Test cookie sync separately from ordinary tab sync. Encryption protects transfer and storage; it does not prevent session theft on an enrolled device.

Password scope conflicts: §2.7 puts the own vault at P2, but §2.9 requires it at P1.
Define password records, deletion, origin matching, autofill approval, and independent security review before enabling P1 password sync.
“Use macOS Keychain” does not define an autofill implementation or access to another password manager's records.

### CEF and the engine pipeline: enlarge M1

**Chrome runtime does not prove that all UI can belong to Swift (§§1, 2.7, 4).**
CEF still tracks embedded non-Views windows and programmatic extension management as open work.
The extension issue describes management through `chrome://extensions` and existing administrative mechanisms.
These are specific integration risks, not proof that extensions cannot work. [CEF #3294][cef-host], [CEF #3450][cef-ext]

M1 must use the intended runtime, browser style, native host, and patched engine together.
Test focus, keyboard input, IME, accessibility, child windows, menus, split panes, and shutdown.
For extensions, test installation, restart, update, removal, permissions, popups, service workers, tab/window APIs, and native messaging.
Include the owner's password manager, a content blocker, and a work extension.
Test normal and private Profiles. Native companion apps may also check the new browser's signing identity.
A loaded test extension is not evidence of Chrome Web Store compatibility. Keep CEF; budget specific bridges or patches where tests fail.

**The pin file is incomplete (§§1, 3.5).** Pin CEF, Chromium, every patch source, toolchain, SDK, build flags, and artifact hashes.
CEF publishes a specific Chromium checkout in its [compatibility file][cef-pin]. A Helium tag alone does not select compatible CEF code.
Verify patch dependencies and application order; do not assume that two patch sets compose without changes.
Helium also has platform packaging and separate service dependencies. Its `deps.ini` downloads a customized uBlock build.
Thus, “ad blocking from patches” in §2.7 needs a concrete component and filter-update plan. [Helium sources][helium], [dependencies][helium-deps]

Helium's original code is GPL-3.0; imported code retains its original terms. Taking only its patches does not remove license obligations.
Identify each selected patch's license and the resulting distribution duties before choosing the shell license. [Helium license][helium-license]

The “weeks” claim and ~100 GB estimate are unmeasured. Measure clean builds, incremental builds, peak disk use, and memory on both architectures.
Budget patch repair, toolchain changes, symbols, sandboxed helpers, signing, notarization, and rollback-safe updates.
Use authenticated artifacts and verify them before Xcode consumes them.
The seven-day security target needs a clock tied to upstream fixes and an owner who can act if Helium lags.
Repeat the selected browsing and extension tests after every engine update. CEF does not remove this custom-build burden.

### iOS and product boundaries

**Keep WKWebView, but define its contract (§§1, 2.8, 3.1, 4).**
“Required by App Store” is too broad: Apple permits alternative engines under restricted regional entitlements.
This does not require a change to the selected engine. [Apple's EU engine rules][apple-engines]
Specify the minimum OS and separate website data stores per Profile. Test private storage and cross-Profile tab moves.
Share the model, sync, and commands through `core-swift`; define an engine interface and platform capability flags.
Do not assume CEF extensions, password integration, cookie attributes, or DRM behavior carry across to WebKit.
An iOS app cannot depend on a permanent background WebSocket. Define foreground catch-up, outbox persistence, and share-extension handoff.
Add device tests for suspension, recovery-key entry, enrollment, and offline edits. Default-browser and credential capabilities need their own release checks.

The spec also needs these product corrections:

- **Global Favorites and shared Profiles (§§2.1, 2.2):** define which Profile opens a Favorite and what happens when it moves across Profiles.
- **Incomplete model (§3.2):** add split membership/layout, pinned home versus current URL, archive state, and device-local versus synced settings.
- **Archive rules (§2.1):** define inactivity, sleep time, active calls, downloads, unsaved forms, and restore behavior.
- **Conflicting priorities (§§2.4, 2.6, 2.7, 4):** P0 splits, Peek, and Little-window appear only in M4; default-browser handling is P1.
  Define an M3 daily-use subset. Put working downloads, permissions, recovery, and a secure update path before optional visual features.
- **Unbalanced detail (§§2, 3.4, 4):** gestures and multiple login methods are specified before lifecycle and failure behavior.
  Move minor interaction choices to UI design work. Add measurable startup, memory, tab-resume, crash-recovery, and accessibility criteria.

## 2. Crest: verified facts

| Area | Evidence and consequence |
| --- | --- |
| Languages and UI | Native Swift/SwiftUI with AppKit on Mac; C#/.NET NativeAOT core; C++/Objective-C++ Chromium host. See `project.yml`, `CrestCore/global.json`, `CrestCore/src/`, and `CrestEngines/Chromium/Overlay/`. The pinned .NET SDK is 10.0.201. This is not a Swift-only base. |
| Engines | Mac defaults to Chromium, with WebKit available. Mobile uses WebKit. `source.lock.json` pins Chromium **155.0.8059.84** and ungoogled-chromium-macos revision `28552f14731fe8f60422e19bbc2361468d420b25`. This recipe is not the spec's Helium recipe. |
| Chromium embedding | **Not CEF.** `Patches/native-host.patch` and `Overlay/chrome/browser/ui/crest/crest_browser_window.mm` replace Chrome window integration. `crest_chrome_host.mm` connects native views and extensions to Swift. Chromium retains process and run-loop ownership. `CrestEngines/Chromium/README.md` explains the host. |
| WebKit embedding | `CrestMac/Infrastructure/WebKit/BrowserWebKitPageAdapter.swift` owns the page adapter, delegates, content controller bridges, and WebKit view. `CrestMobile/` supplies the mobile presentation. |
| Sync and accounts | `CrestShared/Infrastructure/Sync/CloudKit/CloudSyncEngine.swift` uses `CKSyncEngine`. `CloudRecordCodec.swift` writes payloads and tombstones through `CKRecord.encryptedValues`; IDs and clocks remain outside them. The sync guide requires the same Apple Account. Password values use Keychain separately. |
| E2EE fit | Crest does encrypt CloudKit fields; it must not be described as plaintext sync. However, this is not the specified Cloudflare account-key protocol, enrollment, or 24-word recovery. Apple controls the CloudKit key mechanism. No equivalent brwsr account service was found in the inspected implementation. [Apple encryption model][apple-crypto] |
| Profiles | `CrestCore/.../Intents/Spaces/CreateSpace.cs` creates a separate profile for each Space. `Documentation/ARCHITECTURE.md` makes this a privacy boundary. Multiple Spaces per Profile would change model, storage, permissions, deletion, and sync rules. The request is also open as [#720][issue-profile]. |
| License | `LICENSE` is MPL-2.0. Distributed covered files and modifications must remain available under MPL; separate files can use other terms. Preserve notices. `TRADEMARKS.md` requires new branding, bundle IDs, signing, update feeds, and CloudKit identity. brwsr fits that naming requirement. [Mozilla FAQ][mpl] |

### Existing Arc-like behavior

The following features have implementation code, not only README claims:

- **Sidebar, Spaces, pins, saved tabs, folders, and current tabs:** `CrestShared/Features/Sidebar/` and `CrestCore/src/CrestCore/Application/Session/Intents/`.
  `Lifecycle/TogglePin.cs` changes placement. `Tabs/ReturnToSavedAddress.cs` supports the saved home URL.
  Pins belong to a Space; they are not the spec's global Favorites.
- **Split groups:** `Intents/Batches/SplitTabs.cs` and `Contracts/Session/SplitGroupState.cs` manage membership and group metadata.
  The documented layout supports two to four pages. Mobile has separate split presentation under `CrestMobile/Features/SplitView/`.
- **Peek and Little-window equivalent:** `CrestMac/Features/TransientBrowsing/Models/BrowserPeekModel.swift` and `CrestMac/Features/QuickWindow/`.
  The Peek model holds a page lease and coordinates its promotion and release.
- **Command bar, routing, and archive:** `CrestShared/Features/Chrome/Components/BrowserCommandPalette/`, `Application/Device/Intents/Links/AddLinkRoute.cs`, and `Application/Session/Intents/Records/RestoreArchivedTab.cs`.
  Crest documents automatic cleanup and archive retention. Exact equivalence to the spec's 12-hour and 30-day defaults was not established.
- **Chrome extensions:** `crest_chrome_host.mm` downloads packages and uses Chromium's `CrxInstaller`, with profile and private-mode checks.
  `crest_engine_extensions.cc` observes installed extensions. `HelpCenter/docs/extensions/compatibility.md` states limits: Chromium pages only, separate installation per Space, and no installed extensions in private windows.
  Native companions can reject a fork's signature. Code inspection does not establish support for every extension.

No matching per-site CSS/JS Boost authoring system was established in this review. Do not count it as supplied by the fork.

### Maturity and builds

Git history contains **633 commits**, from 2026-08-15 through 2026-10-25: 195 in August, 266 in September, 172 in October.
The [contributors API][contributors] returned 18 identities, including two bots. Paul Davis accounts for 564 commits, about 89%.
`GOVERNANCE.md` names him as lead maintainer and release owner. The project is active, young, and dependent on one person.

The [release API][releases] shows stable v0.7.0 on October 3, v0.8.0 on October 12, and v0.8.50 on October 23.
The latest stable has an arm64 DMG, checksum, and symbols. Recent nightly releases occur daily; development releases are more frequent.
These channels are distinct. Frequent development tags are not evidence of a stable-release guarantee.

A paginated API read returned **177 open issues and 60 open PRs**; counts can change during the review.
Reported problems include [extension popup paste #727][issue-paste], [Peek links #718][issue-peek], and [closed tabs reopening #714][issue-tabs].
These are user reports, not reproduced findings. They identify useful acceptance tests for daily use.

Tests are substantial: 85 Swift source files under `CrestTests`, 13 under `CrestMobileTests`, and 98 C# source files under `CrestCore/tests`.
These totals include helpers and are not test-case counts or a coverage percentage.
Read examples include `Sync/SyncConflictTests.cs`, `CrestTests/BrowserPageEngineMoveTests.swift`, and the native ABI checks.
`.github/workflows/ci.yml` runs core and script tests, checks ABI contracts, and builds Mac, iOS, and Chromium UI targets.
The inspected workflow builds Apple apps but does not run their XCTest suites. No measured coverage percentage or passing local run was verified.

Build setup: Apple silicon, Xcode supporting 26.1 targets, XcodeGen, and the pinned .NET SDK.
`Scripts/bootstrap.sh` generates the Xcode project and runs repository checks. `project.yml` is the configuration source.
The `Crest` scheme is WebKit-only. It does not reproduce the released dual-engine app.
Chromium releases embed prebuilt engine artifacts, the Swift UI framework, and the NativeAOT library; no installed .NET runtime is needed.
`CrestEngines/Chromium/README.md`, `.github/workflows/chromium-engine.yml`, and `.github/workflows/release.yml` define separate engine and app publication.
Changed engine inputs require a new artifact. A fork needs its own reproducible artifact and signing path.
The inspected Mac configuration is arm64-only; Intel support in §3.5 remains extra work.

## 3. Restyling Crest toward YAB

The [YAB sidebar manual][yab-sidebar] confirms left/right/off placement, Comfortable/Compact density, pin grids, and edge reveal.
**Off means horizontal tabs**, not just a hidden sidebar. [Spaces][yab-spaces] have colors and can share or separate sign-ins.
The [appearance manual][yab-appearance] describes color, grain, transparency, and Liquid Glass on macOS 26, with accessibility fallbacks.
The [website][yab] states that YAB uses the Mac's WebKit. Its appearance does not establish Chromium integration behavior.

Crest already exposes sidebar side, border width, material, tab scale, and Space color.
See `CrestShared/Features/Settings/Components/BrowserLookAndFeelGroups/{BrowserWindowAppearanceGroup,BrowserTabAppearanceGroup}.swift`.
Its `BrowserCommandPaletteShellMaterial.swift` already uses `glassEffect` with a Reduce Transparency fallback.

A focused restyle can simplify the crest artwork, use emoji Space controls, add two density presets, and adjust row and frame dimensions.
Reuse the existing appearance controls and sidebar presentation. Add and test horizontal tabs if YAB's Off mode is required.
Apply glass only where text remains readable; test reduced transparency, contrast, focus, dragging, and engine-view layering.
No runtime visual match was verified. Copy the interaction goals, not YAB's brand assets.
Shared Profiles and global Favorites are model changes. They cannot be delivered through appearance settings.

## 4. Side-by-side decision

Time ranges below are planning estimates, not measurements. Assume one experienced full-time Swift/C++ engineer and working build hardware.
They cover an internal Mac alpha, not a public release or the later cookie/password sync work.

| Criterion | Build from spec: Swift + CEF | Fork Crest and restyle |
| --- | --- | --- |
| Daily-use Mac alpha | Plan in months; roughly 3–6+ months pending M1/M2. The spec's “weeks” claim excludes major unproven work. | Roughly 2–6 weeks for a restyled Crest with its existing behavior, subject to runtime checks. With required CEF, Cloudflare E2EE, and shared Profiles, plan in months; no reliable saving is established. |
| Long-term control | Own the Swift model and engine boundary. CEF APIs and the patch stack constrain implementation. | Full source access, but inherit C# core, native ABI, custom Chrome host, and Space rules. Large changes reduce the value of upstream merges. |
| Chromium updates | Maintain CEF/Chromium/Helium compatibility and builds for two CPU architectures. | Existing arm64 pipeline helps while upstream stays compatible. The custom BrowserWindow host still needs Chromium maintenance. Adding CEF replaces that integration. |
| Extensions | Must prove support on the exact patched engine and native host. | Existing install and UI code gives a stronger starting point. Documented limits and open bugs remain; companion trust does not transfer automatically. |
| iOS | Separate WKWebView UI; shared Swift core can reduce duplication. No implementation yet in the spec. | Existing native mobile app and shared core are substantial assets. CloudKit removal and profile changes affect both platforms. |
| License fit | Helium's original code is GPL-3.0. Review the selected patches and combined distribution before choosing the shell license. CEF/Chromium notices also apply. | MPL permits commercial distribution with source obligations for covered files. Rebrand and replace official service identities. brwsr's intended distribution license is not specified. |
| Upstream project stops | Dependencies can stall; own build capacity and an emergency patch process are required. | Young project; 89% of commits and release ownership depend on one person. Source access permits continuation but does not provide replacement maintainers. |

## 5. Recommendation and proposed spec changes

Keep CEF and E2EE as fixed requirements. Use Crest as a reference with limited source reuse, not as the application base.
The fast full-fork route retains Crest's engine and sync choices. It does not meet the stated brwsr architecture.
Replacing those parts, plus profile ownership, removes much of that route's time advantage.

Use these specific Crest pieces:

1. **Port ordering cases** from `CrestCore/tests/CrestCore.Tests/Sync/SyncConflictTests.cs` into Swift protocol tests.
   Study `CrestCore/src/CrestCore/Domain/Sync/SyncOrderTokens.cs` for gap exhaustion. Add concurrent equal-key and replay tests; do not copy its merge model unchanged.
2. **Use lifecycle cases** from `CrestTests/BrowserPageEngineMoveTests.swift` and `CrestCore/tests/CrestCore.Tests/Session/NativeTransientTests.cs` as CEF adapter test requirements.
   They cover page identity, view replacement, and temporary page ownership. They are not a drop-in CEF adapter.
3. **Adapt the small glass modifier** `BrowserCommandPaletteShellMaterial.swift` if its OS target fits.
   Use the two appearance-group files as UI references. Their settings dependencies make wholesale copying less useful.
4. **Use the engine artifact process as a design reference:** `source.lock.json`, `host-inputs.json`, and the separate engine workflow.
   Write a CEF-specific recipe. Do not transplant Crest's BrowserWindow overlay or its CloudKit transport.

Retain MPL notices and source obligations for copied or adapted code and tests. No Crest code was copied by this review.

If the owner keeps the spec approach, make these five changes to `docs/SPEC.md`:

1. **§§3.2, 3.3, 3.6:** require an E2EE protocol ADR before M2. Specify convergence, authenticated operations, enrollment, recovery, revocation, and snapshot recovery.
2. **§§2.9, 3.4:** add the cookie transfer contract, atomic auth-state consumption, and active-session revocation. Resolve the password-vault priority conflict.
3. **§§1, 3.5, 4:** replace the assumed engine schedule with expanded M1 checks, complete pins, a patch-license review, measured resource needs, and security-release ownership.
4. **§§2, 3.2, 4:** define the daily-use alpha, Profile/Favorite ownership, split records, archive behavior, and lifecycle acceptance tests. Align P0/P1 labels with milestones.
5. **§§1, 2.8, 3.1:** add the mobile engine contract, minimum OS, persistent Profile isolation, background-sync limits, and device test plan.

[crest]: https://github.com/pauljoda/Crest/tree/a6f0f0c35427c5cf9029a1f01bd47d20ea3b42a6
[contributors]: https://api.github.com/repos/pauljoda/Crest/contributors?per_page=100
[releases]: https://api.github.com/repos/pauljoda/Crest/releases?per_page=100
[issue-profile]: https://github.com/pauljoda/Crest/issues/720
[issue-paste]: https://github.com/pauljoda/Crest/issues/727
[issue-peek]: https://github.com/pauljoda/Crest/issues/718
[issue-tabs]: https://github.com/pauljoda/Crest/issues/714
[kv]: https://developers.cloudflare.com/kv/concepts/how-kv-works/
[do-ws]: https://developers.cloudflare.com/durable-objects/best-practices/websockets/
[cookies]: https://httpwg.org/http-extensions/draft-ietf-httpbis-rfc6265bis.html
[cef-host]: https://github.com/chromiumembedded/cef/issues/3294
[cef-ext]: https://github.com/chromiumembedded/cef/issues/3450
[cef-pin]: https://github.com/chromiumembedded/cef/blob/master/CHROMIUM_BUILD_COMPATIBILITY.txt
[helium]: https://github.com/imputnet/helium
[helium-deps]: https://github.com/imputnet/helium/blob/main/deps.ini
[helium-license]: https://github.com/imputnet/helium/blob/main/LICENSE
[apple-engines]: https://developer.apple.com/support/alternative-browser-engines/
[apple-crypto]: https://developer.apple.com/documentation/cloudkit/encrypting-user-data
[mpl]: https://www.mozilla.org/en-US/MPL/2.0/FAQ/
[yab-sidebar]: https://yetanotherbrowser.com/manual/sidebar
[yab-spaces]: https://yetanotherbrowser.com/manual/spaces
[yab-appearance]: https://yetanotherbrowser.com/manual/appearance
[yab]: https://yetanotherbrowser.com
