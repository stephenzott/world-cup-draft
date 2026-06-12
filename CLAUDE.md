# 2026 FIFA World Cup — Fantasy Draft Tracker

Single-file app (`index.html`) for tracking a 12-team fantasy draft during the 2026 FIFA World Cup. All state lives in `localStorage`. No build step, no dependencies.

## Structure

- `index.html` — the entire app: HTML, CSS (`<style>`), and JS (`<script>`)
- `trophy.png` — FIFA World Cup trophy image used in the header (links to FIFA standings)
- `fantasy_league_assignments.md` — the draft assignments reference doc (12 teams × 4 WC teams each)
- `draft_preview.html` — styled pre-tournament draft preview page (one card per pick, matches app theme). Linked from a gold button in the header that auto-hides on and after 2026-06-11.

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

## Open Branches

| Branch | Status | Notes |
|--------|--------|-------|
| `knockout-bracket` | PR #1 open, auto-merges 2026-06-28 9am ET via scheduled remote agent | Adds public Bracket tab + Teams Still Alive |
| `opta-predictions` | Parked, no PR | Adds public Odds tab with Opta pre-tournament predictions. Simulator idea shelved pending group interest — would need team strength ratings or per-match odds (stage probabilities we have are simulation outputs, not inputs). |

Both branches also make the tab nav always visible and move Manual Entry to admin-only. Main currently keeps tab nav admin-gated.

## UI

### Tabs
- **Draft Order** — ranked table + read-only groups grid
- **Bracket** — knockout bracket (column-per-round) + Teams Still Alive grid. Public tab. On `knockout-bracket` branch (merges 2026-06-28).
- **Odds** — sortable table of all 48 teams × 7 stage probabilities from Opta. Public tab. On `opta-predictions` branch (parked).
- **Manual Entry** — editable groups grid (round + goals for/against) + coin flip seeds. Admin-only.

On `main`: tab nav is admin-gated (shown only with `?admin`). On feature branches: tab nav always visible, Manual Entry button hidden unless `?admin`.

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
- Read-only grid: 3 columns default → 2 (≤1100px) → 2 (≤768px) → 1 (≤480px)
- Manual entry grid: 4 columns default → 3 (≤1100px) → 2 (≤768px) → 1 (≤480px)
- Read-only view shows a FIFA-style standings table per group: **Team | Pts | GF | GA | chip**, sorted by Pts → GD → GF. Group-stage stats (W/D/L/GF/GA) are stored in `state.results[team].groupStats` — computed from API group-stage matches only (`stage_name === 'First stage'`), separate from the total-tournament `gf/ga` used for fantasy scoring. If no API data, cells show `—`.
- Manual entry view adds round dropdown and goals for/against inputs
- `renderGroups()` (manual entry form) is only rebuilt when switching to the Manual Entry tab or when an API sync fires while that tab is already active — avoids unnecessary DOM reconstruction

### Bracket Tab (knockout-bracket branch)
- Column-per-round layout: R32 → R16 → QF → SF → Final. 3rd place match appears below the Final column. Horizontally scrollable.
- Each match card shows flag emoji, WC team name, and fantasy owner. Completed matches highlight the winner and dim the loser. Live matches get a green border.
- TBD slots show placeholder cards until API data arrives. Empty state shows a "waiting for 2026 data" message.
- **Teams Still Alive** section above the bracket: 12 owner cards in a responsive grid. Each shows the owner's 4 teams with status chips for eliminated teams and an `X/4` alive badge (purple when > 0, gray when 0). A team is "alive" when `round === -1`.
- `bracketMatches` — module-level array (not persisted to localStorage) populated by `syncNow()` from all KO stage matches. `renderBracketCols()` is called when this updates. `renderAlive()` is called from `renderOrder()` so the alive grid stays current with any result change.
- KO stage names captured: `Round of 32`, `Round of 16`, `Quarter-final`, `Quarter-finals`, `Semi-final`, `Semi-finals`, `Final`, `Play-off for third place`, `Third place play-off`.

### Odds Tab (opta-predictions branch)
- Sortable table: all 48 WC teams × 7 columns — Top Group, R32, R16, QF, SF, Final, 🏆 (champion). Data from Opta via The Analyst, hardcoded pre-tournament (June 2026).
- Each row shows flag, team name, and fantasy owner. Cells heat-map tinted: green ≥70%, light green ≥40%, light blue ≥15%.
- `OPTA_ODDS` — constant object keyed by app team name (normalized from Analyst names e.g. "Türkiye"→"Turkey", "Cabo Verde"→"Cape Verde").
- `ODDS_COLS` — array of `{ key, label }` for the 7 data columns.
- `oddsSortKey` / `oddsSortDir` — module-level sort state. `sortOddsBy(key)` toggles direction on re-click. Default: champion % descending.
- `renderOdds()` — lazy, called on tab switch. `fmtPct(v)` formats values: `—` for 0%, `<0.1%` for tiny values.

## Live API Sync

Pulls from the ESPN scoreboard API (`https://site.api.espn.com/apis/site/v2/sports/soccer/fifa.world/scoreboard`) every 90 seconds. No API key required; CORS is open (`*`). Previously used `worldcupjson.net` but that site only has 2022 data.

- Fetches one request per date from June 11 through tomorrow in parallel (Promise.all), typically 2–40 requests depending on how far into the tournament we are
- `normalizeEvent()` converts ESPN event shape to internal `{ stage, completed, home, away }` format
- ESPN uses `season.slug` for stage (e.g. `group-stage`); KO stage slugs unconfirmed until July 4 — matched with permissive regex in `KO_STAGES`
- Shows sync status pill: Syncing / Live (with timestamp) / Error
- Manual "Sync Now" button + "Pause Auto-sync" toggle
- Team name normalization handles API variants (e.g. "United States" → "USA", "South Korea" → "Korea Republic", "Türkiye" → "Turkey")
- KO stage logic correctly resolves 3rd/4th place: 3rd-place game winner = round 5, loser = round 4
- Manual Entry data is preserved; API only writes when it has completed match data
- On each sync, KO stage matches are also captured into `bracketMatches` and `renderBracketCols()` is called (bracket tab feature, on `knockout-bracket` branch)

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

Each `results[team]` entry: `{ round, gf, ga, groupStats }` where `gf/ga` are total-tournament goals (used for fantasy scoring) and `groupStats` is `{ p, w, d, l, gf, ga }` for group-stage matches only (used for the standings display). `groupStats` is `null` until API data arrives.
