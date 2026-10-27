# ADR 0002: Engine embedding, build, and distribution

Status: proposed. Owner: Matt Hamlin. Date: 2026-10-27.

## Decision

macOS embeds Chromium through CEF in Chrome runtime mode, hosted by a native Swift app with
an Objective-C++ bridge. We build CEF ourselves from a pinned tree with the ungoogled-chromium
patch set and our own patches. Helium patches are held until the license decision below.

## Pin

`engine/pin.json`:

```
{
  "cefBranch":        "<CEF branch, e.g. 7xxx>",
  "cefCommit":        "<sha>",
  "chromiumTag":      "<from CEF's CHROMIUM_BUILD_COMPATIBILITY.txt, not chosen by us>",
  "ungoogledRev":     "<sha>",
  "patches":          ["ungoogled/*", "brwsr/0001-...", "..."],   // applied in this order
  "xcode":            "<version>", "macosSDK": "<version>", "minMacOS": "<version>",
  "gnArgs":           { ... },
  "arches":           ["arm64", "x86_64"]
}
```

The Chromium tag comes from CEF's compatibility file. Patches are applied on that tag. A patch
that fails to apply blocks the bump until fixed or dropped with a note.

## Licenses

- Chromium and CEF: BSD-3. ungoogled-chromium: BSD-3.
- Helium: its own code is GPL-3.0. If we apply Helium patches, the engine build is a GPL
  derivative and we must publish its source. The Swift app links CEF as a separate framework;
  whether that app is also covered is a question for counsel, and we do not want the question.
- Decision (2026-10-27): ungoogled-chromium only. No Helium patches.

## What we give up without Helium patches, and the replacement

| Helium feature (its patch layer) | Source of the feature | Our replacement |
|---|---|---|
| Fingerprint noise for canvas, audio, hardware | Helium patch | Not in v1. ungoogled-chromium has some fingerprinting flags; a brwsr patch later if wanted |
| Reduced Accept-Language, Privacy Sandbox and Topics off | Helium patch | ungoogled-chromium already disables Privacy Sandbox; Accept-Language is a small brwsr patch |
| DNS over HTTPS on by default, ClientHello splitting | Helium patch | DoH is a Chromium pref we set in the shell. ClientHello splitting is not taken |
| Custom theme, toolbar, tab redesign | Helium patch | Irrelevant. Our UI is the Swift shell |
| Parallel downloads, tab freezing, MRU tab cycling, split view | Helium patch | Chromium flags for parallel downloads and tab freezing. MRU cycling and split view are shell features anyway |
| uBlock Origin built in | Helium downloads a custom build | Bundled uBlock Origin extension, depends on M1 |
| Helium Services (proxy for extension updates, safe browsing replacement, no Google calls) | Helium server + patches | ungoogled-chromium removes the Google calls. Extension updates go to the Web Store directly, or through our own Worker proxy as P1 |
| 13k offline bangs, AI bangs | Helium patch | Command bar feature in the shell, P1 |
| Custom onboarding, new tab page | Helium patch | Shell |

Most of Helium's patch layer is UI, and we replace the UI anyway. The real losses are the
fingerprinting noise and the services proxy, both deferrable.
- We ship uBlock Origin (GPL-3.0) as a separate extension package, which is the normal case
  and does not affect the app's license.

## Build

- Runner: self-hosted Mac or large cloud Mac. Clean build, measured in M1, both architectures.
- Output: `cef-<branch>-<arch>.tar.zst`, a `manifest.json` with SHA-256 for every file, signed
  with our Ed25519 release key. Stored in R2 with public read.
- Developers download the artifact. Nobody builds Chromium locally unless they change a patch.
- The app verifies manifest signature and file hashes before loading the framework. A failed
  verification keeps the previous engine and shows an error.
- Framework is signed with the app's identity and notarized with the app. Hardened runtime and
  the CEF helper app entitlements are checked in M1.

## Update and security

- Engine updates ship through the app updater as a separate component from the UI build.
- Security clock: 7 days from the upstream Chromium stable security release to our release.
- If CEF lags the Chromium release: ship the previous CEF branch with Chromium's backported
  security commits applied as patches, or skip to CEF's next branch within the window.
- If ungoogled-chromium lags: ship without the patches that fail, note them in the release.
- Owner for lag handling: Matt Hamlin until a second maintainer exists.
- Failed update recovery: the previous engine stays on disk until the new one has started once.

## Unknowns and the test that settles each (all in M1)

| Unknown | Test |
|---|---|
| Native non-Views host works with Chrome runtime (CEF issue 3294) | Swift window hosts the browser view; focus, IME, menus, VoiceOver, popups, permission prompts |
| Programmatic extension install and management (CEF issue 3450) | install, restart, update, remove a password manager, uBlock Origin, one work extension; popups, tabs/windows APIs, service workers, native messaging under our signing identity |
| Multiple Profiles and four browser views in one window | two Profiles, split 4 panes, destroy and recreate views, clean shutdown, no leaks |
| Build cost | clean build time, peak disk, memory, incremental time, both arches, recorded |
| Patch set applies on the CEF-chosen Chromium tag | apply ungoogled-chromium at the pinned revision; list failures |
| Hardened runtime and notarization with CEF helpers | notarize a demo app and run it on a clean Mac |

The M1 report lists supported, unsupported, patches carried, and the measured numbers. It
produces the first estimate for M3.
