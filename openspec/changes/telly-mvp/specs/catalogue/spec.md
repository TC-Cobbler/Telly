## Purpose

Ingests a user's Trakt, TMDB, and Letterboxd accounts and expands that seed into a much larger, scored catalogue large enough to fill dozens of linear channels around the clock.

## ADDED Requirements

### Requirement: Trakt ingestion
The system SHALL sync a connected Trakt account's watchlist, collection, custom lists, and ratings on a 6-hour cadence.

#### Scenario: Watchlist item becomes eligible content
- **WHEN** a title is present on the connected Trakt watchlist after a sync
- **THEN** it is added to the catalogue as a direct-tier item

#### Scenario: Periodic re-sync picks up changes
- **WHEN** 6 hours have elapsed since the last successful Trakt sync
- **THEN** the system re-syncs watchlist, collection, lists, and ratings and updates the catalogue accordingly

### Requirement: TMDB ingestion and enrichment
The system SHALL sync a connected TMDB account's watchlist and favourites on a 6-hour cadence, and SHALL enrich catalogue items on demand with TMDB metadata, `similar`, `recommendations`, and keywords, caching enrichment data for 30 days.

#### Scenario: TMDB favourite becomes eligible content
- **WHEN** a title is present on the connected TMDB favourites list after a sync
- **THEN** it is added to the catalogue as a direct-tier item

#### Scenario: Enrichment is cached
- **WHEN** enrichment data for an item was fetched fewer than 30 days ago
- **THEN** the cached enrichment is reused instead of issuing a new TMDB request

### Requirement: Letterboxd ingestion
The system SHALL ingest a connected Letterboxd account's watchlist and diary entries rated 4 stars or higher via the user's public RSS feed on a 24-hour cadence, and SHALL support a manual CSV import (the Letterboxd data-export watchlist file) as a fallback when RSS access is unavailable or insufficient.

#### Scenario: High-rated diary entry becomes eligible content
- **WHEN** a Letterboxd RSS sync finds a diary entry rated 4 stars or higher
- **THEN** the corresponding title is added to the catalogue as a direct-tier item

#### Scenario: CSV import works without RSS
- **WHEN** a user imports a Letterboxd watchlist CSV export in Settings
- **THEN** every title in that file is added to the catalogue as a direct-tier item without requiring RSS access

### Requirement: Tiered catalogue expansion
The system SHALL expand the direct catalogue (explicit watchlist/collection/high-rated items, weight 0.45) into an adjacent tier (weight 0.35: TMDB `similar` and `recommendations` plus keyword/cast/crew overlap, scored higher when pointed at by more direct items) and a wildcard tier (weight 0.20: drawn from genres the user has no affinity with, weighted toward decent-rated older material), targeting roughly 15x the direct catalogue size and capped at 4,000 items.

#### Scenario: Adjacent item scored by overlap count
- **WHEN** an item is returned by `similar`/`recommendations` for five different direct-tier items
- **THEN** it is scored higher in the adjacent tier than an item returned for only one direct-tier item

#### Scenario: Wildcard avoids the user's known taste
- **WHEN** the wildcard tier is populated
- **THEN** items are drawn from genres absent from the user's direct-tier affinity, not from genres already well represented there

#### Scenario: Expansion respects the size cap
- **WHEN** expansion would produce more than 4,000 total catalogue items
- **THEN** the catalogue is truncated to 4,000 items, preserving the highest-scored items per tier

### Requirement: Affinity scoring feeds scheduling priority
The system SHALL assign each catalogue item an affinity score reflecting its tier and overlap count, and SHALL expose that score to the scheduler for use in primetime and rotation placement.

#### Scenario: Highest-affinity items are eligible for primetime
- **WHEN** the scheduler selects candidates for a primetime anchor or rotation slot
- **THEN** it can rank candidates by affinity score to prefer higher-affinity items
