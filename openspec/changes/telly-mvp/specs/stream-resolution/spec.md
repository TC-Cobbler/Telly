## Purpose

Turns a scheduled programme into a playable stream ahead of airtime, using the Stremio addon protocol with debrid-backed resolution as the primary path, and degrades honestly when that isn't available.

## ADDED Requirements

### Requirement: Addon-agnostic Stremio client
The system SHALL act as a client of the Stremio addon protocol against manifest URLs the user supplies, SHALL ship with no bundled addon manifests, and SHALL NOT default to pointing at any particular index.

#### Scenario: No manifest configured, no resolution attempted
- **WHEN** no addon manifest URL has been configured in Settings
- **THEN** the system does not attempt stream resolution and reports the missing configuration rather than falling back to a built-in addon

#### Scenario: User-supplied manifest is used as-is
- **WHEN** a user pastes a Torrentio manifest URL that includes their own debrid token
- **THEN** the system calls that manifest's stream endpoint directly for resolution

### Requirement: Debrid-first stream selection
The system SHALL prefer a debrid-backed direct HTTP stream over a cached local torrent, which SHALL be preferred over a direct HTTP addon stream, which SHALL be preferred over cold torrent streaming, when multiple resolver types are available for the same programme.

#### Scenario: Debrid stream is chosen over a cold torrent
- **WHEN** an addon response includes both a debrid direct `url` and a raw `infoHash` for the same item
- **THEN** the debrid direct stream is selected

### Requirement: Stream selection policy
The system SHALL apply a configurable stream selection policy (max resolution, codec preference, max bitrate, audio preference, seekability requirement, cam/telesync rejection patterns, and per-type size caps) when choosing among candidate streams, biased toward reliability over quality.

#### Scenario: Non-seekable stream is rejected
- **WHEN** `require_seekable` is enabled and a candidate stream does not support range requests
- **THEN** that candidate is excluded from selection

#### Scenario: Cam-quality release is rejected
- **WHEN** a candidate stream's release tag matches a configured reject pattern such as CAM or TELESYNC
- **THEN** that candidate is excluded from selection

### Requirement: Pre-resolution pipeline
The system SHALL resolve a stream for every programme starting within the next hour (T-60), SHALL validate that resolved URL and capture the actual media duration at T-10, and SHALL pre-buffer the next programme in a second player instance at T-2.

#### Scenario: Actual duration overrides metadata
- **WHEN** the T-10 validation pass measures a duration that differs from the programme's TMDB-sourced runtime
- **THEN** the programme's stored duration is updated to the measured value

#### Scenario: Next programme is warm before the boundary
- **WHEN** a programme is within 2 minutes of ending
- **THEN** the next programme's stream is pre-buffered in a separate player instance ahead of the transition

### Requirement: Resolution failure escalation
On resolution or validation failure the system SHALL escalate through, in order: the next-best stream candidate, a different configured addon, and a substitute programme from the channel's standby pool; a failed slot SHALL never be shown to the user as an error screen and SHALL instead show a testcard interstitial.

#### Scenario: Standby pool covers total failure
- **WHEN** no stream candidate resolves across every configured addon for a scheduled programme
- **THEN** a substitute programme from the channel's standby pool airs in that slot instead of an error screen

### Requirement: Cold-torrent fallback
When no debrid-backed or direct HTTP stream is available, the system SHALL fall back to sequential torrent download via an embedded torrent client and local HTTP server, accepting a longer time-to-first-frame and degraded seek behaviour.

#### Scenario: Torrent fallback engages without a debrid account
- **WHEN** stream resolution finds only a raw `infoHash` and no debrid provider is configured
- **THEN** the system begins sequential torrent download and serves the result through its embedded local HTTP server

### Requirement: Degraded-mode signalling
The system SHALL treat debrid configuration as required for guaranteed join-in-progress playback; when it is absent or a specific programme could not be resolved with wall-clock accuracy, the system SHALL start that programme from the beginning at tune-in instead of failing, and SHALL surface a persistent degraded-mode indicator.

#### Scenario: No debrid configured runs in degraded mode
- **WHEN** no debrid provider is configured in Settings
- **THEN** channels still produce a schedule and EPG, tuning in starts programmes from the beginning rather than at the wall-clock offset, and a persistent warning is shown in Settings

### Requirement: Per-channel subtitle resolution
The system SHALL resolve a subtitle track for each programme at pre-resolution time according to a per-channel subtitle preference, since linear playback offers no mid-programme track switch.

#### Scenario: Subtitle preference is applied before airtime
- **WHEN** a channel has a configured subtitle preference and a programme is pre-resolved for that channel
- **THEN** the matching subtitle track, if available, is attached to the resolved stream before the programme airs

### Requirement: Stream link refresh
The system SHALL re-resolve a programme's stream link at T-10 even if previously resolved, and SHALL be able to swap in a refreshed URL mid-playback without an interruption visible to the viewer when a link expires during airing.

#### Scenario: Expiring link is refreshed before it airs
- **WHEN** a previously resolved stream's link would expire before its programme finishes airing
- **THEN** the system refreshes the link and swaps it in without a visible playback interruption
