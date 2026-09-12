## Purpose

Turns a per-channel weekly template plus the ingested catalogue into a deterministic, materialised grid of dated programmes, and keeps that grid stable week over week through anchoring.

## ADDED Requirements

### Requirement: Deterministic weekly generation
The system SHALL generate each channel's weekly schedule from a seeded PRNG derived from the channel id, ISO year, ISO week, and a per-user salt, such that identical catalogue state, template, and seed always produce a byte-identical schedule.

#### Scenario: Same inputs produce the same schedule
- **WHEN** the scheduler regenerates a channel's schedule for a given ISO week using an unchanged catalogue, template, and seed
- **THEN** the resulting sequence of programmes is identical to the previous generation for that week

#### Scenario: Regeneration on a new device is unnoticed
- **WHEN** a user reinstalls the app or moves to a new device with the same connected accounts
- **THEN** the regenerated schedule matches the one they had before, slot for slot

### Requirement: Full default channel set with auto-disable
The system SHALL define a default set of themed channels (up to ~28) and SHALL auto-disable, at generation time, any channel whose eligible unique catalogue is below the channel's minimum threshold, rather than shipping it half-empty.

#### Scenario: Thin catalogue disables a channel
- **WHEN** a channel's theme rule matches fewer than 20 hours of eligible unique content in the current catalogue
- **THEN** that channel is marked disabled at generation time and does not appear in the active channel list

#### Scenario: Disabled channel leaves a numbering gap
- **WHEN** a previously enabled channel becomes auto-disabled
- **THEN** its channel number is not reassigned to another channel

### Requirement: Anchoring binds a series to a permanent slot
The system SHALL bind a series to a specific `(channel_id, weekday, start_min)` anchor the first time it is scheduled, and SHALL treat that binding as permanent until the series completes plus a grace period, or the user explicitly unpins it.

#### Scenario: New episode airs in its anchor
- **WHEN** a new episode of an anchored series becomes available
- **THEN** it is placed in that series' existing anchor slot rather than being scheduled elsewhere

#### Scenario: Anchor survives a re-sync
- **WHEN** the catalogue is re-synced or the schedule is regenerated
- **THEN** existing anchors are preserved unchanged

#### Scenario: Anchor is not released by a missed week
- **WHEN** no new episode is available for an anchored slot in a given week
- **THEN** the anchor remains assigned to that series and the slot is filled by the fallback order below

#### Scenario: Explicit unpin clears an anchor
- **WHEN** the user removes an anchor from Settings
- **THEN** the slot becomes available for reassignment in a future schedule generation

### Requirement: Anchor fallback order
When an anchored slot has no new episode for the current week, the system SHALL fill it, in order of preference: (1) the next unwatched episode in sequence, (2) a repeat of the most recent episode, (3) a same-channel substitute of similar length as a temporary sub-let that does not release the anchor.

#### Scenario: Back-catalogue fill
- **WHEN** an anchored series has unwatched earlier episodes and no new episode this week
- **THEN** the next unwatched episode in sequence airs in the anchor slot

#### Scenario: Sub-let does not release the anchor
- **WHEN** no unwatched episode and no repeat candidate exists
- **THEN** a same-channel substitute programme airs in the slot and the anchor remains bound to the original series

### Requirement: Anchor change rate limit
The system SHALL introduce no more than one new anchor per channel per week; newly eligible series queue for the next free primetime slot.

#### Scenario: Second new anchor in the same week is deferred
- **WHEN** two series become eligible for a new anchor on the same channel in the same week
- **THEN** only one is anchored that week and the other queues for the next available primetime slot

### Requirement: Composition rules per channel per week
The system SHALL enforce, per channel per week: 15-35% of primetime as anchor slots; ~40% rotation from direct/adjacent tiers with no title repeated within 72 hours on the same channel; ~15% wildcard with a minimum of one per day and at least one in primetime per week; at least one omnibus block per week (defaulting to weekend afternoons); and daytime slots may repeat a primetime programme aired within the last 7 days, marked as a repeat.

#### Scenario: Rotation avoids near-term repeats
- **WHEN** the generator selects a rotation programme for a channel
- **THEN** it excludes any title that already aired on that channel within the previous 72 hours

#### Scenario: Wildcard minimums are met
- **WHEN** a channel's week is generated
- **THEN** every day includes at least one wildcard slot and at least one wildcard slot falls in primetime that week

#### Scenario: Daytime repeat is marked
- **WHEN** a daytime slot is filled with a programme that aired in primetime within the last 7 days
- **THEN** the resulting programme is flagged as a repeat for EPG display

### Requirement: Watershed enforcement
The system SHALL prevent content rated above the configured regional daytime threshold from airing before 21:00 on any channel, SHALL prevent content rated above the configured evening threshold from airing before 21:00 anywhere, and SHALL hard-cap Kids and Family channels to their certification ceiling at all times regardless of daypart.

#### Scenario: High-certification film cannot land before the watershed
- **WHEN** the generator considers a candidate programme rated above the configured evening threshold for a slot starting before 21:00
- **THEN** that candidate is excluded from the slot on every channel

#### Scenario: Kids channel ignores time of day
- **WHEN** the generator fills any slot on a Kids or Family channel
- **THEN** only content at or below that channel's certification ceiling is eligible, including primetime and late slots

### Requirement: Materialisation horizon
The system SHALL maintain a rolling 14-day materialised schedule, regenerated nightly, with days 0 and 1 frozen against further changes and days 2-14 provisional and subject to revision as new episodes are detected.

#### Scenario: Tonight's schedule never changes mid-day
- **WHEN** the nightly regeneration job runs
- **THEN** programmes already materialised for day 0 and day 1 are left unchanged

#### Scenario: A newly detected episode updates a future day
- **WHEN** a new episode is detected for an anchored series airing on day 5
- **THEN** the day 5 materialisation is updated to include it on the next nightly run

### Requirement: Slot grid resolution and film overflow
The system SHALL place programmes on 5-minute grid boundaries and fill gaps with interstitials; a film SHALL be permitted to break the grid, followed by an elastic interstitial block that restores alignment before the next anchored slot.

#### Scenario: Film overrun is absorbed before the next anchor
- **WHEN** a scheduled film's runtime does not divide evenly into 5-minute slots
- **THEN** an elastic interstitial block follows it and the next anchored slot still starts at its assigned time
