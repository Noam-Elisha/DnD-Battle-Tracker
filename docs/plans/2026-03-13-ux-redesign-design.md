# DnD Battle Tracker UX Redesign

## Problem
The app requires too many taps for common combat actions, making paper faster. Primary user is on phone/tablet.

## Design: "Paper, but better"

Every common combat action should be **one tap**.

### 1. JS Data Model
Characters stored as `[{id, name, hp, maxHp, initiative, notes}]` array instead of parsed from DOM. All rendering driven from this array. Enables persistence and clean state management.

### 2. localStorage Persistence
Auto-save character array on every mutation. Restore on page load. Survives refresh/tab close. "Clear All" button for new encounters.

### 3. Inline HP Adjustment
Each character card gets `-5`, `-1`, `+1`, `+5` buttons directly on the card. No modal needed for the most common action (damage/heal). HP clamped to 0..maxHp. Modal retained only for editing name/initiative/notes (rare actions).

### 4. Turn Tracker
- "Next Turn" button pinned at top of character list
- Current character highlighted with a glowing accent border
- Round counter (increments when cycling past last character)
- "Reset" button to restart initiative order

### 5. Health Bars
Thin color bar on each card showing HP proportion. Green (>50%) -> yellow (25-50%) -> red (<25%). Instant visual scan of party/enemy status.

### 6. Mobile-First Layout
- **Phone**: Single column. Character list full screen. "Add Character" form behind a floating `+` button that opens a bottom sheet.
- **Desktop**: Keep two-column layout but with the same improved card design.

### Constraints
- Keep existing dark parchment theme (colors, gradients, typography)
- Remain a single HTML file, no build tools
- No external dependencies
