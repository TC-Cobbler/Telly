## Why

Streaming turned television into an unbounded browsing task: a large personal watchlist accumulates on Trakt and never gets watched, because opening an app at night means minutes of choosing before anything plays. Telly removes the choice by turning a connected Trakt/TMDB catalogue into a fixed weekly grid of linear channels — you press a number, something is already playing, and there is no library screen to fall back into. This change builds the first working version of that idea: a real scheduler, real debrid-backed playback, and the three screens (Guide, Player, Settings) needed to use it end to end.

## What Changes

- New Android TV / Google TV app (Kotlin, Compose for TV, Media3) with exactly three screens: Guide, Player, Settings.
- New deterministic scheduling engine that materialises a weekly grid of channels from a seeded PRNG, so a given catalogue + seed always produces the same schedule, across the full default channel set (~28 channels, auto-disabling any channel the catalogue cannot fill).
- New anchoring system: once a series is placed in a weekly slot, it stays there permanently until the series ends or the user unpins it, including the standby/sub-let fallback order when no new episode is available.
- New composition rules per channel per week: anchor slots, rotation (no repeat within 72h), wildcard (min. one per day, one in primetime), omnibus blocks, daytime repeats of recent primetime programmes, and watershed content-certification gating by daypart.
- New catalogue ingestion from Trakt (watchlist, collection, lists, ratings), TMDB (watchlist/favourites, metadata, `similar`/`recommendations`, keywords), and Letterboxd (watchlist + 4-star-and-up diary via RSS, with CSV-import fallback), expanded via a tiered direct/adjacent/wildcard model.
- New stream resolution against the Stremio addon protocol (addon-agnostic client, no bundled manifests), with debrid-backed direct HTTP as the primary path, a cold-torrent fallback engine (libtorrent4j + local HTTP server) for users without debrid, and an honest "degraded mode" indicator whenever join-in-progress cannot be guaranteed.
- New pre-resolution pipeline that resolves, validates, and buffers programmes ahead of airtime (T-60/T-10/T-2 minute passes) so tune-in lands at the correct wall-clock offset instead of starting from the beginning, plus per-channel subtitle preference resolved at pre-resolution time.
- New EPG grid screen (channels x time, 14 days forward), tune-in-in-progress playback with a banner-only player UI, and channel drift correction against the fixed schedule (absorb, trim, or hard-cut at the slot boundary so anchors never slip).
- New Settings screen covering account linking (Trakt device-code OAuth, TMDB, Letterboxd), source configuration (Stremio addon manifest, debrid key, stream quality policy, subtitle preference), and channel management (enable/disable, anchors, watershed toggle, blocklist).
- Scoped out of this change (left for later changes): `TvInputService`/Android TV system-guide integration (PRD Phase 3), and companion-server/household sync (PRD Phase 4). This change targets the PRD's Phase 0 (latency spike), Phase 1 (MVP), and Phase 2 (depth) combined.

## Capabilities

### New Capabilities
- `scheduler`: Deterministic weekly schedule generation from a seeded PRNG across the full default channel set, anchoring (permanent series-to-slot binding) and its fallback order, composition rules (rotation, wildcard, omnibus, repeats, watershed), auto-disabling under-filled channels, and the materialisation horizon that turns the template into dated programmes.
- `catalogue`: Trakt, TMDB, and Letterboxd ingestion, tiered catalogue expansion (direct/adjacent/wildcard) and affinity scoring that feeds the scheduler.
- `stream-resolution`: Stremio addon client, debrid-first stream resolution and selection policy, cold-torrent fallback, per-channel subtitle resolution, pre-resolution timing (T-60/T-10/T-2), and degraded-mode signalling when join-in-progress cannot be guaranteed.
- `guide`: The EPG grid screen — channel x time layout, d-pad navigation, now/next, programme detail panel.
- `player`: Tune-in at the correct wall-clock offset, drift correction against the fixed schedule, interstitials, and the banner-only playback UI.
- `settings`: Account linking (Trakt, TMDB, Letterboxd), source configuration (addon manifest, debrid key, quality policy, subtitles), and channel management (enable/disable, anchors list, watershed toggle, blocklist).

### Modified Capabilities
- None — this is the first change in the project; there are no existing specs to modify.

## Impact

- New Android TV application from scratch: no existing code, specs, or infrastructure in this repo today.
- External dependencies: Trakt API, TMDB API, Letterboxd (RSS + optional CSV import), a user-supplied Stremio addon manifest (e.g. Torrentio), and a user-supplied debrid provider (Real-Debrid, AllDebrid, Premiumize, or TorBox). libtorrent4j is embedded for the torrent-fallback path.
- Distribution is sideload-only (signed APK / GitHub Releases); no Play Store submission, since the app is a Stremio-addon client and cannot control what addons resolve to.
- Establishes the module boundaries (`:domain:scheduler`, `:domain:catalogue`, `:data:trakt`, `:data:tmdb`, `:data:letterboxd`, `:data:stremio`, `:data:resolver`, `:feature:guide`, `:feature:player`, `:feature:settings`) that later changes (TvInputService, companion server) will extend rather than restructure.
