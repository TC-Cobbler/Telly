## Purpose

Provides the only configuration surface in the app: connecting accounts, configuring content sources and playback policy, and managing channels — nothing else.

## ADDED Requirements

### Requirement: Account linking
Settings SHALL support linking Trakt via a TV-appropriate device-code OAuth flow, TMDB via a request-token-and-approval flow or a pasted v4 access token, and Letterboxd via a username for RSS or a CSV import; for each connected account it SHALL show a "sync now" action, the last-synced timestamp, and item counts.

#### Scenario: Linking Trakt via device code
- **WHEN** the user starts the Trakt linking flow
- **THEN** a device code is displayed for the user to enter on trakt.tv from another device, and the account is linked once authorised

#### Scenario: Sync status is visible
- **WHEN** an account is connected
- **THEN** Settings shows its last-synced timestamp and current item counts

### Requirement: Source configuration
Settings SHALL let the user configure a Stremio addon manifest URL (with Torrentio pre-filled as an example, not a default), a debrid provider and API key with a connection test and a clear full-mode/degraded-mode indicator, the stream quality policy, and per-channel subtitle preference; it SHALL provide a test-resolution tool that resolves a known title and shows the resolver's decision tree.

#### Scenario: Debrid connection test reports mode
- **WHEN** the user enters a debrid API key and runs the connection test
- **THEN** Settings reports success or failure and updates the full-mode/degraded-mode indicator accordingly

#### Scenario: Test resolution shows the decision path
- **WHEN** the user runs test resolution against a known title
- **THEN** Settings displays which addon, stream, and selection-policy decisions were made to reach the result

### Requirement: Channel management
Settings SHALL let the user enable/disable, reorder, and rename channels; view a read-only summary of each channel's theme rules with a few adjustable toggles (wildcard percentage, omnibus day); list and unpin anchors; toggle watershed enforcement and set a regional certification body; regenerate the schedule with a destructive-action confirmation; and maintain a blocklist of titles, genres, and keywords never to schedule.

#### Scenario: Unpinning an anchor
- **WHEN** the user unpins an anchor from the anchors list
- **THEN** that slot becomes available for reassignment on the next schedule generation

#### Scenario: Regenerating the schedule requires confirmation
- **WHEN** the user selects "regenerate schedule"
- **THEN** the system requires explicit confirmation before discarding and rebuilding the materialised schedule

#### Scenario: Blocklisted title is never scheduled
- **WHEN** a title matches an entry on the blocklist
- **THEN** it is excluded from catalogue expansion and from every channel's schedule

### Requirement: Global playback and diagnostics settings
Settings SHALL expose timezone, clock format, the `allow_pause` toggle, a visible degraded-mode banner state, and a diagnostics/log export action.

#### Scenario: Degraded mode is visible from Settings
- **WHEN** the app is running without a configured debrid provider
- **THEN** Settings shows a persistent degraded-mode banner

### Requirement: Credential storage
The system SHALL store debrid and other provider API keys in encrypted on-device storage, SHALL NOT write them to logs, and SHALL NOT transmit them anywhere other than the corresponding provider.

#### Scenario: API key is never logged
- **WHEN** a debrid API key is saved in Settings
- **THEN** it does not appear in application logs or diagnostics exports in plaintext
