## Purpose

Plays a channel as a live broadcast: joining already in progress at the correct wall-clock offset, correcting drift against the fixed schedule, and keeping the on-screen UI to a banner rather than a media-player chrome.

## ADDED Requirements

### Requirement: Tune-in at the correct offset
On channel selection, the system SHALL locate the programme whose scheduled window contains the current time, compute the elapsed offset into that programme, resolve the stream synchronously with a 4-second timeout if it is stale or missing, seek to the computed offset, and begin playback.

#### Scenario: Tuning in mid-programme
- **WHEN** a user selects a channel 20 minutes into a 60-minute programme's scheduled slot
- **THEN** playback starts at approximately the 20-minute mark of that programme, not from the beginning

#### Scenario: Stale stream is resolved synchronously within the timeout
- **WHEN** the stored stream for the current programme is stale at tune-in time
- **THEN** the system resolves a fresh stream synchronously, aborting after 4 seconds if resolution has not completed

### Requirement: Tune-in latency target
The system SHALL target first frame within 2,000 ms of channel selection with a hard ceiling of 4,000 ms, after which the channel ident plays over audio-only until video is ready.

#### Scenario: Slow resolution falls back to audio-only ident
- **WHEN** video is not ready 4,000 ms after channel selection
- **THEN** the channel ident plays with audio while video continues loading in the background

### Requirement: Instantaneous channel changing
The system SHALL keep two player instances warm and SHALL speculatively pre-resolve (without pre-buffering) adjacent channels while the Guide is open, so that changing channels feels instantaneous.

#### Scenario: Changing channels while browsing the guide
- **WHEN** the Guide is open and the user moves the selection to an adjacent channel row
- **THEN** that channel's current programme is pre-resolved in the background before the user tunes to it

### Requirement: Drift correction
The system SHALL compare playback position against wall clock continuously; when a programme ends early, the following interstitial block SHALL absorb the gap; when it overruns by less than 3 minutes, it SHALL be allowed to finish with the next interstitial trimmed; when it overruns by 3 minutes or more, it SHALL be hard-cut at the slot boundary. The next anchored slot SHALL never be allowed to slip.

#### Scenario: Small overrun is absorbed by trimming
- **WHEN** a programme overruns its slot by 90 seconds
- **THEN** it is allowed to finish and the following interstitial is shortened to compensate

#### Scenario: Large overrun is hard-cut
- **WHEN** a programme overruns its slot by 4 minutes
- **THEN** playback is cut at the slot boundary regardless of whether the programme has finished

#### Scenario: Anchored slot holds its start time
- **WHEN** drift accumulates ahead of an anchored slot
- **THEN** the anchored slot still starts at its scheduled time

### Requirement: Interstitials as first-class filler
The system SHALL play interstitials to fill non-programme time: a channel ident (5-15s), an "Up Next" card (10s), a "Coming This Week" card (30s) for gaps warranting it, a countdown clock (30-60s) before a primetime anchor, a testcard with tone for gaps over 3 minutes and for hard failures, and a clock display for gaps over 10 minutes.

#### Scenario: Gap before a primetime anchor gets a countdown
- **WHEN** there is a scheduled gap immediately before a primetime anchor slot
- **THEN** a countdown clock interstitial plays leading into it

#### Scenario: Long gap shows the clock
- **WHEN** a gap in the schedule exceeds 10 minutes
- **THEN** a clock interstitial is shown for the duration of the gap

### Requirement: Banner-only player UI
During normal playback the system SHALL show no persistent on-screen UI; on any keypress a banner SHALL appear for 5 seconds showing channel number, channel name, programme title, episode designation, a progress bar for position within the programme, and now/next times. No scrubbing or play/pause overlay SHALL be shown by default.

#### Scenario: Keypress reveals the banner
- **WHEN** the user presses any key during playback
- **THEN** the banner appears with channel, programme, progress, and now/next information, then disappears after 5 seconds of inactivity

### Requirement: Trick-play is off by default
Pause SHALL be unavailable by default. A single Settings toggle `allow_pause`, off by default, SHALL enable pausing for accessibility; when enabled, pausing SHALL display "PAUSED — channel continues" and resuming SHALL rejoin the live wall-clock position rather than continuing from the pause point.

#### Scenario: Pause is unavailable by default
- **WHEN** `allow_pause` is off and the user presses a pause-equivalent key
- **THEN** playback continues unaffected

#### Scenario: Enabled pause rejoins live on resume
- **WHEN** `allow_pause` is on, the user pauses, waits, and resumes
- **THEN** playback resumes at the channel's current live wall-clock position, not at the moment it was paused
