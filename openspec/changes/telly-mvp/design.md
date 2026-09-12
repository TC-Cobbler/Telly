## Context

Greenfield Android TV / Google TV app; nothing exists in this repo yet. See `proposal.md` for motivation and `specs/*/spec.md` for behavioral requirements. This document covers the architecture that makes those requirements buildable, scoped to PRD Phases 0-2 combined (this change) with Phase 3 `TvInputService` integration and Phase 4 companion-server/household-sync explicitly deferred to later changes.

The hardest constraint shaping everything below: a linear channel is defined by joining it already in progress. That rules out cold torrent streaming as anything but a fallback, and it means the schedule for "tonight" must already be fully pre-resolved before anyone turns the TV on.

## Goals / Non-Goals

**Goals:**
- Prove tune-in latency on real hardware before committing to the full build (Phase 0 spike, first task in `tasks.md`).
- A pure, deterministic, unit-testable scheduler module that has no Android dependency.
- An addon-agnostic content pipeline: Stremio resolution and debrid caching sit behind interfaces so a future non-Stremio source (e.g. a Jellyfin/Plex adapter) is a drop-in, not a rewrite.
- Materialisation of the full channel set completes within budget on low-end Google TV hardware (Chromecast-class, ~2GB RAM), or is deferred to a future companion-server change without restructuring the scheduler.

**Non-Goals:**
- `TvInputService`/`TvContract` publishing and Google TV Live-tab presence (Phase 3).
- Companion server, XMLTV export, and multi-device synchronized channels (Phase 4).
- Play Store distribution — sideload only, for the legal reasons in the PRD (§13); not re-litigated here.

## Decisions

### Module boundaries
```
:app                  Compose TV shell, navigation, three screens
:core:model           Entities, value types
:core:database        Room, DAOs, migrations
:core:datastore       Settings (DataStore/Proto)
:feature:guide        EPG grid
:feature:player       Media3 wrapper, tune-in, drift correction, interstitials
:feature:settings     Account linking, addon config, channel management
:data:trakt
:data:tmdb
:data:letterboxd
:data:stremio         Addon protocol client, manifest handling, stream selection
:data:resolver        Debrid adapters, torrent fallback, stream validation
:domain:scheduler     Deterministic generator. Pure Kotlin, no Android deps.
:domain:catalogue     Ingestion, expansion, affinity scoring
```
Rationale: `:domain:scheduler` and `:domain:catalogue` carry the requirements in `specs/scheduler` and `specs/catalogue` and must be testable without an emulator (property-based + golden-file snapshot tests asserting a given catalogue+seed reproduces an identical week). `:data:stremio` and `:data:resolver` are kept behind interfaces so resolution is source-agnostic, per the PRD's own architecture stance (§13) and this repo's non-goal of ever bundling a specific addon.

### Stack
Kotlin; Compose for TV (`androidx.tv.material3`) over Leanback, since Leanback is legacy and Compose TV is Google's supported path; MVVM + unidirectional state with Hilt DI; Media3/ExoPlayer for HTTP range/HLS/adaptive playback and eventual TIF compatibility; Room+SQLite for the relational, time-ranged schedule queries; DataStore (Proto) for settings; Ktor client + kotlinx.serialization for networking; WorkManager for sync, materialisation, and pre-resolution background jobs; libtorrent4j + embedded NanoHTTPD for the torrent-fallback path only.

Alternative considered: a thin WebView/web-stack app to move faster across platforms. Rejected — Media3's TIF compatibility and precise seek/range-request control over debrid HTTP streams (needed for accurate wall-clock offset tune-in) are a native-player concern, and Phase 3 TvInputService integration requires a native Android component regardless.

### Determinism
`seed = xxhash64(channel_id ‖ iso_year ‖ iso_week ‖ user_salt)`, injected into the scheduler as a pure function argument rather than read from a global clock/RNG. This is what makes `specs/scheduler`'s "same inputs produce the same schedule" requirement testable and what lets the schedule regenerate identically on a new device (no server-side schedule storage needed for this change).

### Compute placement for materialisation
Materialising ~28 channels x 14 days on a 2GB-RAM Amlogic-class device is the primary performance risk called out in the PRD (§9.3). Mitigation order for this change: (1) incremental materialisation, one channel at a time via WorkManager, spread across the nightly window; (2) precomputed per-channel eligibility sets so the generator isn't re-filtering the full catalogue per slot. Full companion-server offload (§9.3 mitigation 3) is Phase 4 and out of scope here, but `:domain:scheduler` and `:domain:catalogue` must not take an Android or on-device-only dependency, so that offload is a deployment change later, not a rewrite.

### Failure posture
Every failure mode in `specs/stream-resolution` (resolution failure, link expiry, no debrid) resolves to a testcard/interstitial or a degraded-mode indicator, never an error screen — this is a product decision (§5.4, §7 of the PRD) as much as a technical one, and it's why the standby pool and degraded-mode signalling are first-class parts of the scheduler/resolver contract rather than error handling bolted on afterward.

## Risks / Trade-offs

- **[Risk] Cold-torrent fallback complexity.** Embedding libtorrent4j plus a local HTTP server is a meaningfully different code path from the debrid-HTTP primary path and was originally a later phase. → Mitigation: build it behind the same `:data:resolver` interface as debrid, and treat degraded-mode signalling (already required by spec) as the honest UI answer when the fallback still can't guarantee join-in-progress — don't try to make cold torrent feel like debrid.
- **[Risk] Low-end device materialisation budget.** 2GB-RAM hardware materialising the full channel set nightly is the PRD's own flagged main performance risk. → Mitigation: incremental per-channel materialisation via WorkManager; treat the Phase 0 spike's tune-in latency result as the gate for whether the full channel count is viable before Phase 2 work proceeds.
- **[Risk] Three ingestion sources with three different reliability profiles** (Trakt API, TMDB API, Letterboxd RSS/CSV) feeding one scheduler. → Mitigation: `:domain:catalogue` treats a source outage as "stale, keep last-known catalogue" rather than blocking materialisation; each source syncs independently on its own cadence.
- **[Trade-off] Combining Phase 1 and Phase 2 into one change** means a larger single review/apply cycle than the PRD's own phase split. Accepted per explicit scope decision: repeats, drift correction, watershed, blocklist, and degraded mode are load-bearing enough to the "linear channel" premise that shipping without them would not be a coherent end state.

## Migration Plan

Not applicable — greenfield project, no existing users or data to migrate. First release is a signed sideload APK per the PRD's distribution posture (§13).
