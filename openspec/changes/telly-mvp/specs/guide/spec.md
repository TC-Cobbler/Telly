## Purpose

Presents the materialised schedule as a conventional two-dimensional electronic programme guide the user can browse and tune from, without becoming a library or search surface.

## ADDED Requirements

### Requirement: Grid layout
The Guide SHALL present channels on the vertical axis and time on the horizontal axis in 30-minute columns, with 2.5 hours visible at once, and a vertical line indicating the current time.

#### Scenario: Current time is always visible on open
- **WHEN** the Guide is opened
- **THEN** the visible viewport includes the current-time indicator line

### Requirement: D-pad navigation
The Guide SHALL support left/right to move through time, up/down to move through channels, select on a currently-airing programme to tune in, select on a future programme to open its detail panel, and back to return to the player.

#### Scenario: Selecting a live programme tunes in
- **WHEN** the user presses select on a programme that is currently airing
- **THEN** the player starts playing that channel at the current wall-clock offset

#### Scenario: Selecting a future programme opens detail
- **WHEN** the user presses select on a programme that has not started yet
- **THEN** a detail panel opens showing synopsis, artwork, cast, a "remind me" action, and a "never on this channel again" action

### Requirement: Guide entry points
The Guide SHALL be reachable from the player via a dedicated guide/info key press or by entering a channel number.

#### Scenario: Opening the guide from the player
- **WHEN** the user presses the guide/info key while in the player
- **THEN** the Guide opens showing the currently playing channel

### Requirement: Render window and performance
The Guide SHALL render 14 days forward and 24 hours back from the current time, and SHALL complete a cold render of the visible viewport in under 300 milliseconds.

#### Scenario: Looking back at what aired
- **WHEN** the user navigates to a time earlier than now but within the last 24 hours
- **THEN** the Guide shows the programmes that aired in that window

### Requirement: Programme badges
The Guide SHALL mark each programme cell with its type where applicable: repeat `(R)`, new `(N)`, wildcard `(W)`, or omnibus `(O)`.

#### Scenario: Repeat is visibly marked
- **WHEN** a daytime slot is filled with a repeat of a recent primetime programme
- **THEN** its Guide cell is marked `(R)`

### Requirement: Jump to now
The Guide SHALL provide a dedicated key that returns the viewport to the current time and the currently-playing channel's row.

#### Scenario: Returning to now after browsing forward
- **WHEN** the user has navigated several days ahead and presses the jump-to-now key
- **THEN** the viewport returns to the current time

### Requirement: Channel long-press actions
Long-pressing a channel row in the Guide SHALL offer enable/disable, reorder, and viewing that channel's weekly template.

#### Scenario: Disabling a channel from the guide
- **WHEN** the user long-presses a channel row and chooses disable
- **THEN** that channel no longer appears in the active Guide or is tunable from the player

### Requirement: Now/next overlay from the player
A collapsed now/next bar overlay SHALL be reachable in a single key press from the player, without leaving the currently playing programme.

#### Scenario: Checking what's next without leaving the programme
- **WHEN** the user presses the now/next key while watching
- **THEN** a collapsed overlay shows the current and next programme without stopping playback
