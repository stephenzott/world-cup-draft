# 2026 FIFA World Cup — Fantasy Draft Tracker

Single-file app (`index.html`) for tracking a 12-team fantasy draft during the 2026 FIFA World Cup. All state lives in `localStorage`. No build step, no dependencies.

## Structure

- `index.html` — the entire app: HTML, CSS (`<style>`), and JS (`<script>`)
- `trophy.png` — FIFA World Cup trophy image used in the header (links to FIFA standings)
- `fantasy_league_assignments.md` — the draft assignments reference doc (12 teams × 4 WC teams each)

## Fantasy League Setup

- **12 fantasy teams**, each assigned 4 World Cup teams (one from each pot)
- **48 total teams** across **12 groups** (A–L)
- No fantasy team has two teams in the same group
- Draft order is determined post-tournament: worst performers pick first

## Scoring / Draft Order Logic

Ranking (worst → best, i.e. earliest draft pick = worst result):

1. **Best Result** (`maxRound`) — highest round any team on the fantasy roster reached
2. **Total Rounds** — sum of all round values across the roster
3. **Total Goal Difference** — net GF–GA across all played matches
4. **Coin Flip seed** — tiebreaker of last resort (lower number drafts earlier)

Round values: Group exit = 0, R32 = 1, R16 = 2, QF = 3, Semi-Final (in progress) = 3.5, 4th place = 4, 3rd place = 5, Runner-Up = 6, Champion = 7.

Round 3.5 is a manual-entry-only intermediate state for teams currently in the semi-finals. The API will overwrite it with the correct resolved value (4/5/6/7) once the match completes. Withdrawn/forfeited matches award a 3-0 win to the non-withdrawing side — the API should reflect this in goals and no special handling is needed.

## Fantasy Team Owners

Team names are hardcoded as `defaultName` in `FANTASY_TEAMS` — not editable by users:

| ID | Owner |
|----|-------|
| 1 | Steve |
| 2 | Max |
| 3 | Scuba |
| 4 | Justin |
| 5 | Andrew/Tyler |
| 6 | Patrick |
| 7 | Luke |
| 8 | Reggie |
| 9 | Keanan |
| 10 | Matt |
| 11 | Chambers |
| 12 | Shane |

## UI

### Tabs
- **Draft Order** — ranked table + read-only groups grid (tab nav hidden on public page)
- **Manual Entry** — editable groups grid (round + goals for/against) + coin flip seeds

Tab navigation bar is only shown when `?admin` is in the URL.

### Header
A **tournament stage pill** in the sync-bar reflects the current stage derived from the highest round in state: Group Stage → Round of 32 → Round of 16 → Quarter-Finals → Semi-Finals → Final → Complete. Updates every time `renderOrder()` runs.

### Draft Order Table
Columns: Rank · Fantasy Team (with per-team chips) · Best Result · Total Rounds · Total GD

Responsive column hiding:
- ≤768px: hides Total Rounds and Total GD columns
- ≤480px: also hides Best Result column

Team names are plain text (not editable).

Ties are not flagged visually on the public page; coin flip asterisks appear only in the admin coin flip settings.

### Groups Grid
- 4 columns default → 3 (≤1100px) → 2 (≤768px) → 1 (≤480px)
- Read-only view shows team name, owner, and status chip
- Manual entry view adds round dropdown and goals for/against inputs
- `renderGroups()` (manual entry form) is only rebuilt when switching to the Manual Entry tab or when an API sync fires while that tab is already active — avoids unnecessary DOM reconstruction

## Live API Sync

Pulls from `https://worldcupjson.net/matches` every 90 seconds.

- Detects whether 2026 data is live (checks match datetimes ≥ 2026-06-11)
- Shows sync status pill: Waiting / Syncing / Live (with timestamp) / Error
- Manual "Sync Now" button + "Pause Auto-sync" toggle
- Team name normalization handles API variants (e.g. "United States" → "USA", "South Korea" → "Korea Republic")
- KO stage logic correctly resolves 3rd/4th place: 3rd-place game winner = round 5, loser = round 4
- Manual Entry data is preserved; API only writes when it has completed match data

## Visual Design

Color palette (CSS custom properties):
- `--bg: #f5f5f5` — page background
- `--surface: #ffffff` — card/table background
- `--border: #304fff` — blue borders and group card headers
- `--accent: #6300e9` — purple, used for rank #1 and active tab
- `--gold: #ecff43` — yellow, used for 3rd place chips and champion chip
- `--red: #ce0f11` — red, used for rank #3 and group-exit chips
- Dark header bar (`#111111`) with gold title text

Team chips in the draft table include flag emojis (full lookup table for all 48 qualified nations).

Chip styles by round: `chip-tbd`, `chip-out`, `chip-r32`, `chip-r16`, `chip-qf`, `chip-sf` (orange, semi-final in progress), `chip-4th`, `chip-3rd`, `chip-final` (silver with outline, runner-up), `chip-champ` (gold with dark outline + bold, champion).

## localStorage Key

`wcDraft2026` — stores `{ results, coinFlip }` objects keyed by team name / fantasy team ID. Names are no longer persisted (always read from `defaultName`).
