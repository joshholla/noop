# Caltrack integration development

This branch prepares an iPhone collection and export path for a personal Caltrack journal. Each Caltrack profile uses its own iPhone and WHOOP strap, with the local NOOP database as the collection authority. Reviewed files are the first transport; a private HTTPS export can follow once identity, measurement semantics, and restore are validated.

Status: development plan and configuration example. The Caltrack exporter, receiver, and Google Drive archive job are proposed work.

## Source and license

- Fork: [joshholla/noop](https://github.com/joshholla/noop).
- Upstream: [ryanbr/noop](https://github.com/ryanbr/noop), the continuing community repository named by its current README.
- Base commit: `e58017508d9731c51d0631fe0eb4bb1d7ea0c582`, October 9, 2026.
- Development branch: `codex/caltrack-integration`.
- Receiver and web journal: [joshholla/caltrack_jk](https://github.com/joshholla/caltrack_jk), maintained separately from the collector.
- NOOP original code and documentation remain under [PolyForm Noncommercial 1.0.0](../LICENSE), with the required copyright notice and third-party notices preserved. This integration is for personal noncommercial use.

## Collection expectations

WHOOP 4.0 is the upstream supported path for live heart rate, historical collection, and local analysis. WHOOP 5.0 live heart rate works, while deeper overnight inputs remain experimental and firmware dependent. HRV and sleep availability must be determined from actual collected inputs. NOOP scores are local estimates and must retain their own identity.

Calibrated SpO2 percentage is excluded from both live WHOOP paths. WHOOP 4.0 has no readable on-device step counter; WHOOP 5.0/MG may supply steps. Skin temperature is a calibrated nightly deviation where the firmware layout supports it. See [the support statement](../README.md#strap-support) and [capability implementation](../Packages/WhoopStore/Sources/WhoopStore/WhoopLiveCapabilities.swift).

NOOP registers multiple devices but collects from one active WHOOP at a time. One collecting phone per profile avoids switching a shared collector between two people. Separate installations may both use `my-whoop`; the Caltrack identity must therefore include the installation and profile as well as the device ID. Device generation does not determine its owner.

## Implementation sequence

1. Build the existing `NOOPiOS` app under a custom bundle identity, install on both physical phones, and validate one strap per phone. Confirm foreground collection, history offload, lock-screen gaps, relaunch, and backup export before changing the BLE path.
2. Add a reviewed Caltrack JSON export to the iOS share sheet. Export explicitly selected device rows, preserving identity and metric provenance. Keep database backups and raw capture files as separate archival artifacts.
3. Add a dedicated NOOP importer in Caltrack. The current Apple Health importer is not a compatible transport for this data. Validate and preview profile assignment, retain original batches, and deduplicate before saving.
4. Add source-specific metric display and explicit source selection. Keep RMSSD and SDNN separate. Distinguish calendar-day asleep time from sleep assigned to the wake date, and preserve local timezone and UTC intervals.
5. Add an optional, default-off Experimental one-way export to a user-configured private HTTPS endpoint with a profile-bound credential. Export failures must not block strap collection. Receiver implementation belongs in Caltrack, in accordance with [NOOP scope](../docs/SCOPE.md).
6. Archive consistent database snapshots, photos, and original exports on the Mac. Encrypt dated archives before copying to Google Drive, then verify a restore into a separate directory. Keep an independent second copy.

## Proposed export boundary

The reviewed file should contain a schema version, profile ID, installation ID, device ID, generation, optional firmware version, export timestamp, explicit timezone, and app/parser version. Each measurement needs units, source kind, measurement method, and coverage or quality metadata where available.

Stable record IDs and revisions make retries idempotent. Daily summaries should be keyed by profile, installation, device, date, and day basis. Raw streams and sessions need stable sample or interval identity. Missing values remain absent. An unavailable WHOOP 5.0 metric is never filled with zero or inferred from the WHOOP 4.0 capability set.

Imported official WHOOP values, direct BLE observations, and locally computed NOOP metrics must remain distinguishable. NOOP Charge, Effort, and Rest are separate algorithms from official WHOOP scores; field names alone cannot establish either provenance or scale. Preserve original values and algorithm versions.

The configuration example leaves all assignments unset. Real installation IDs, device identifiers, credentials, and captures belong in private local storage, such as the ignored `caltrack/private/` directory or an ignored `*.local.json` file.

## iPhone build and verification

Use current full Xcode with an iOS SDK and install XcodeGen before generating the project. The current upstream CI uses Xcode with the iOS 26 SDK for app compilation; the deployment target is iOS 17. Xcode Command Line Tools alone cannot build or install this app.

```sh
cp Config/BundleIdSecrets.example.xcconfig Config/BundleIdSecrets.xcconfig
# Set BUNDLE_ID_PREFIX and DEVELOPMENT_TEAM locally in the copied file.
xcodegen generate
open Strand.xcodeproj
```

Select the `NOOPiOS` scheme and the connected phone. If `DEVELOPMENT_TEAM` is not set in the local configuration, select the signing Team for both `NOOPiOS` and `NOOPiOSWidgets` in Xcode. Repeat on the other phone. Configure a unique bundle prefix before installing alongside another NOOP build. The source includes all target and app-group substitutions; use that configuration instead of editing generated project files. Refer to [the iOS guide](../docs/IOS.md) and [build guide](../docs/BUILD.md).

An unsigned simulator build is a compilation check:

```sh
xcodebuild -project Strand.xcodeproj -scheme NOOPiOS \
  -sdk iphonesimulator -destination 'generic/platform=iOS Simulator' \
  CODE_SIGNING_ALLOWED=NO build
```

Bluetooth collection requires physical phones and straps. Keep the official WHOOP app from competing for the strap's bond while testing. A live heart-rate reading alone does not prove the custom historical connection is bonded. Test background and reconnect behavior on each phone; no continuous background guarantee follows from a successful foreground session.

The pure protocol baseline can be checked independently with full Xcode's test frameworks:

```sh
cd Packages/WhoopProtocol
swift test
```

Preparation verification on October 12, 2026: the `whoop-decode` protocol executable built successfully with Swift 6.2.3. The package test suite could not run because the selected Command Line Tools installation has no `XCTest` module. Neither iOS app compilation nor physical Bluetooth behavior has been verified in this workspace.

Future importer tests must cover repeated batches, crossed profile credentials, two devices sharing a date, absent metrics, stale revisions, daylight-saving transitions, and RMSSD/SDNN separation. Full app compile checks and real-device BLE checks remain separate requirements.

## Existing export and backup paths

The iOS share sheet already handles file exports. Backup & Sync produces dated `.noopbak` ZIP files containing the database, selected settings, and a build/schema manifest. These files are unencrypted. Automatic folder backups are a daily catch-up when the app opens, and their default retention is seven snapshots. A folder write does not establish that a cloud provider uploaded the file.

Export raw 5.0 capture sessions separately where they live outside SQLite. Retain original portable exports alongside the native backup. The custom archive implementation should use SQLite's online backup API and verify restore under concurrent collection; checkpointing followed by reading the live file is not sufficient evidence of a consistent snapshot.

The Mac-only [NoopLocalAccess](../Packages/NoopLocalAccess/README.md) package is useful for inspecting an explicitly provided database. It does not reach into an iPhone sandbox. Its source fallback and merged reads need review before serving as a provenance-preserving exporter.

See the Caltrack repository's `docs/noop-device-research.md`, `docs/noop-caltrack-integration.md`, and `docs/noop-backup-research.md` for the detailed research and receiver plan.
