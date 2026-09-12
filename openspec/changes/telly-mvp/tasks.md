## 1. Phase 0 — Latency Spike (gate)

- [ ] 1.1 Build a throwaway single-channel harness: one hardcoded channel, a hardcoded schedule of five programmes, and a Media3 player, and verify it builds and runs on a real Google TV device (not just emulator)
- [ ] 1.2 Wire the harness to a real debrid account and resolve a real stream for each of the five programmes, and verify each resolves to a direct HTTP URL
- [ ] 1.3 Implement mid-programme tune-in (seek to wall-clock offset on channel select) in the harness, and verify by tuning in at several different offsets and confirming the playhead matches wall-clock position within 1s
- [ ] 1.4 Measure time-to-first-frame across at least 20 tune-in attempts on real hardware, and verify p50 < 1.5s and p95 < 3.0s per the success metrics in proposal.md's linked PRD context — if this gate fails, stop and revisit the design before continuing to section 2

## 2. Project Setup

- [ ] 2.1 Scaffold the module structure from design.md (`:app`, `:core:*`, `:feature:*`, `:data:*`, `:domain:*`) and verify each module builds independently with `./gradlew build`
- [ ] 2.2 Add Compose for TV, Media3, Room, DataStore, Ktor, WorkManager, Hilt, and libtorrent4j dependencies to the relevant modules and verify a clean build
- [ ] 2.3 Define the core Room entities from the domain model (Account, CatalogueItem, Series, Episode, Channel, Slot, Programme, Stream, ScheduleWeek) in `:core:database` and verify migrations apply cleanly against an empty DB

## 3. Catalogue Ingestion & Expansion

- [ ] 3.1 Implement `:data:trakt` client (watchlist, collection, lists, ratings) with device-code OAuth, and verify against a real Trakt account that all four resources sync into `CatalogueItem`/`Account` rows
- [ ] 3.2 Implement `:data:tmdb` client (watchlist/favourites sync, metadata/similar/recommendations/keywords enrichment with 30-day cache), and verify enrichment cache hits avoid a second network call within the cache window
- [ ] 3.3 Implement `:data:letterboxd` client (RSS watchlist + 4-star diary, with CSV import fallback), and verify both the RSS path and the CSV import path populate `CatalogueItem` rows
- [ ] 3.4 Implement tiered catalogue expansion in `:domain:catalogue` (direct/adjacent/wildcard weighting, overlap scoring, 15x target capped at 4,000 items) as a pure function, and verify with unit tests covering the overlap-scoring and size-cap scenarios from `specs/catalogue/spec.md`
- [ ] 3.5 Wire WorkManager jobs for the 6h (Trakt/TMDB) and 24h (Letterboxd) sync cadences, and verify via WorkManager test harness that jobs are scheduled at the correct intervals and survive a device reboot

## 4. Scheduler

- [ ] 4.1 Implement the seeded PRNG (`xxhash64(channel_id, iso_year, iso_week, user_salt)`) and the deterministic weekly generator in `:domain:scheduler` as pure Kotlin with no Android dependency, and verify with a golden-file snapshot test that identical inputs reproduce a byte-identical schedule
- [ ] 4.2 Implement the default channel definitions (~28 channels, theme rules) and auto-disable logic for under-filled channels, and verify with a unit test that a channel below the eligible-hours threshold is marked disabled and its number is not reused
- [ ] 4.3 Implement anchoring: binding, permanence across re-sync, the fallback order (unwatched next / repeat / sub-let), and the one-new-anchor-per-channel-per-week limit, and verify each behavior with a dedicated unit test matching the scenarios in `specs/scheduler/spec.md`
- [ ] 4.4 Implement composition rules (anchor %, rotation with 72h no-repeat, wildcard minimums, omnibus blocks, daytime repeats marked `(R)`) and watershed enforcement (certification gating by daypart, Kids/Family hard cap), and verify with unit tests per rule
- [ ] 4.5 Implement the 14-day rolling materialisation job (nightly at 04:00, days 0-1 frozen, days 2-14 provisional) on WorkManager, and verify a re-run does not alter already-materialised frozen days
- [ ] 4.6 Implement 5-minute slot grid placement with film-overflow elastic interstitial alignment, and verify with a unit test that a film's overrun does not shift the next anchored slot's start time
- [ ] 4.7 Verify materialisation of the full channel set x 14 days completes within budget on a low-end reference device (Chromecast-class / ~2GB RAM), using incremental per-channel WorkManager scheduling if the single-pass budget is exceeded

## 5. Stream Resolution

- [ ] 5.1 Implement `:data:stremio` addon protocol client (manifest parsing, stream endpoint calls) taking a user-supplied manifest URL with no bundled manifest, and verify against a real Torrentio manifest URL that stream objects are parsed correctly
- [ ] 5.2 Implement `:data:resolver` debrid adapters (Real-Debrid, AllDebrid, Premiumize, TorBox) behind a common interface, and verify each adapter resolves a magnet/infoHash to a direct HTTP URL against a real account
- [ ] 5.3 Implement the stream selection policy (resolution/codec/bitrate/audio/seekability/reject-patterns/size-caps) as configurable and defaulted per `specs/stream-resolution/spec.md`, and verify with unit tests covering the seekability-rejection and cam-pattern-rejection scenarios
- [ ] 5.4 Implement the pre-resolution pipeline (T-60 resolve, T-10 validate + duration probe, T-2 pre-buffer) on WorkManager, and verify with an integration test that a scheduled programme has a validated `Stream` row with measured duration before its slot starts
- [ ] 5.5 Implement the three-strike failure escalation (next-best stream → different addon → standby-pool substitute) with a testcard interstitial as the terminal fallback, and verify by forcing resolution failures and confirming no error screen is ever shown
- [ ] 5.6 Implement the cold-torrent fallback (libtorrent4j + embedded NanoHTTPD local server) behind the `:data:resolver` interface, and verify it serves a playable stream when only a raw infoHash is available
- [ ] 5.7 Implement degraded-mode detection and signalling (start-from-beginning behavior, persistent Settings warning) when join-in-progress cannot be guaranteed, and verify the warning appears when no debrid provider is configured
- [ ] 5.8 Implement per-channel subtitle preference resolution at pre-resolution time, and verify the resolved stream carries the configured subtitle track before airtime
- [ ] 5.9 Implement stream link refresh (re-resolve at T-10, seamless mid-playback URL swap on expiry), and verify with an integration test that a link forced to expire mid-programme is swapped without a visible playback gap

## 6. Player

- [ ] 6.1 Implement the tune-in algorithm (locate current programme, compute offset, synchronous resolve with 4s timeout, seek, play, preload next) in `:feature:player`, and verify with an instrumentation test across multiple tune-in offsets
- [ ] 6.2 Implement the two-warm-player-instance channel-change path with speculative pre-resolution of adjacent channels while the Guide is open, and verify channel-up/down feels instantaneous in a manual pass and via a latency instrumentation test
- [ ] 6.3 Implement drift correction (absorb on early end, trim on <3min overrun, hard-cut on ≥3min overrun, anchored slots never slip), and verify each branch with a unit test against synthetic drift scenarios
- [ ] 6.4 Implement the interstitial asset set (ident, up-next, coming-this-week, countdown, testcard+tone, clock) as procedurally composed bundled assets, and verify each interstitial triggers under its documented gap-length condition
- [ ] 6.5 Implement the banner-only player UI (5s reveal on keypress, channel/programme/progress/now-next), and verify no other persistent UI renders during normal playback
- [ ] 6.6 Implement the `allow_pause` toggle (off by default) with live-rejoin-on-resume behavior, and verify pause is a no-op when the toggle is off and rejoins live position when it's on

## 7. Guide

- [ ] 7.1 Implement the EPG grid (channels x 30-min time columns, 2.5h viewport, current-time indicator) in `:feature:guide`, and verify cold render of the visible viewport completes in under 300ms via a Compose UI benchmark
- [ ] 7.2 Implement d-pad navigation (time/channel movement, select-to-tune, select-future-for-detail, back-to-player) and the guide/info-key and channel-number entry points, and verify each navigation path with a UI test
- [ ] 7.3 Implement the 14-day-forward/24-hour-back render window and programme badges (`(R)(N)(W)(O)`), and verify badges render correctly for representative programmes of each type
- [ ] 7.4 Implement jump-to-now and channel long-press actions (enable/disable, reorder, view weekly template), and verify each action against a test channel
- [ ] 7.5 Implement the collapsed now/next overlay reachable in one press from the player, and verify it appears without interrupting playback

## 8. Settings

- [ ] 8.1 Implement the Accounts section (Trakt device-code OAuth, TMDB token/approval, Letterboxd username/CSV import, sync-now, last-synced timestamp, item counts), and verify each linking flow against a real account
- [ ] 8.2 Implement the Sources section (addon manifest entry, debrid key + connection test + mode indicator, quality policy controls, subtitle preference, test-resolution tool), and verify the test-resolution tool displays a real decision trace for a known title
- [ ] 8.3 Implement the Channels section (enable/disable/reorder/rename, theme summary + wildcard%/omnibus-day toggles, anchors list with unpin, watershed toggle + certification body, destructive regenerate-with-confirmation, blocklist), and verify blocklisted titles are excluded from the next materialisation
- [ ] 8.4 Implement global settings (timezone, clock format, `allow_pause`, degraded-mode banner, diagnostics/log export) and encrypted credential storage for provider API keys, and verify a saved API key never appears in plaintext in logs or exported diagnostics

## 9. Integration & Verification

- [ ] 9.1 Run an end-to-end pass across all three screens with real Trakt/TMDB/Letterboxd/Torrentio/debrid accounts, and verify a full week's schedule materialises, tunes in correctly, and survives channel surfing without errors
- [ ] 9.2 Measure the success metrics from the PRD against the integrated build (tune-in p50/p95, resolution success rate at T-10, testcard-fallback rate, anchor schedule adherence, nightly materialisation duration on low-end hardware), and verify each meets its stated target or file a follow-up if not
- [ ] 9.3 Produce the signed sideload APK build and release process (GitHub Releases), and verify a clean-install device can link accounts and reach a playing channel with no Play Store involvement
