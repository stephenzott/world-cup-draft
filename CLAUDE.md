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
| `opta-predictions` | Parked, no PR | Adds public Odds tab with Opta pre-tournament predictions. Simulator idea shelved pending group interest — would need team strength ratings or per-match odds (stage probabilities we have are simulation outputs, not inputs). |

Main tab nav is now always visible; Manual Entry is admin-only (`?admin`).

## TODO

- **Extend bracket to full R16/QF/SF/Final**: The Projected tab currently only populates the R32 column from group standings. The R16, QF, SF, Final columns show static "TBD" placeholders. Once the knockout stage begins (≈2026-07-04), wire actual match results into those columns so the bracket tracks real progress. The `knockout-bracket` branch has `bracketMatches` + `renderBracketCols()` that pulls live KO data from the ESPN scoreboard — merging that branch (scheduled 2026-06-28) will be the foundation; the Projected tab should then consume those results for the later rounds.

## UI

### Tabs
- **Draft Order** — ranked table + read-only groups grid + Best 3rd Place Teams table. Public.
- **Projected** — projected R32 bracket + Teams Still Alive grid. Public. On `main`.
- **Odds** — sortable table of all 48 teams × 7 stage probabilities from Opta. Public. On `opta-predictions` branch (parked).
- **Manual Entry** — editable groups grid (round + goals for/against) + coin flip seeds. Admin-only (`?admin`).

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
- Read-only view shows a FIFA-style standings table per group: **Team | Pts | GF | GA | chip**, sorted by Pts → GD → GF. Group-stage stats (W/D/L/GF/GA) are stored in `state.results[team].groupStats` — computed from API group-stage matches only (`stage === 'group-stage'`), separate from the total-tournament `gf/ga` used for fantasy scoring. If no API data, cells show `—`.
- Manual entry view adds round dropdown and goals for/against inputs
- `renderGroups()` (manual entry form) is only rebuilt when switching to the Manual Entry tab or when an API sync fires while that tab is already active — avoids unnecessary DOM reconstruction

### Projected Tab
- Shows a column-per-round bracket layout (R32 → R16 → QF → SF → Final). R32 is populated from projected group standings; later rounds show TBD placeholders until knockout stage data arrives.
- **`rankGroupTeams(letter)`** — full FIFA tiebreaker. Step 1: H2H pts → H2H GD → H2H GF (recursive: re-runs `sortSub()` on any still-tied sub-group). Step 2 (overall): GD → GF → fair-play (`yellow + 3*red` from `teamCards`, lower = better) → FIFA ranking. Uses `allNormalized` for individual H2H match lookups.
- **`FIFA_RANKINGS`** — constant object mapping team name → ranking number for all 48 qualified nations. Used as final tiebreaker (lower rank number = better).
- **`THIRD_COMBOS`** — 495-entry constant object. Key: sorted 8-char string of the group letters whose 3rd-place teams advanced (e.g. `'ABCDEFGH'`). Value: 8-char slot string in match order `[M74,M77,M79,M80,M81,M82,M85,M87]` (e.g. `'CFHEBAGD'`). Covers every valid combination of 8-from-12 groups.
- **`R32_MATCHES`** — 16-entry array. Each entry has `id`, `home` slot (e.g. `'1E'`), and either `away` slot or `mslot` (0–7 index into the 3rd-place slot assignment from `THIRD_COMBOS`).
- **`getBest8ThirdPlace()`** — sorts all 12 3rd-place teams by pts → GD → GF → FIFA ranking, returns top 8 with their group letter.
- **`getProjectedR32()`** — calls `getBest8ThirdPlace()`, builds the sorted 8-group key, looks up `THIRD_COMBOS`, returns a slot→team map for all 32 R32 participants.
- **`buildProjMatchCard(homeTeam, awayTeam)`** — reuses `.match-card` CSS; shows flag + team name + fantasy owner. No scores shown.
- **`renderProjBracketCols()`** — renders R32 column from projected standings + TBD columns for later rounds.
- **`renderAlive()`** — renders the Teams Still Alive grid (12 owner cards). Each card shows owner name, `X/4` alive badge (purple when >0, gray when 0), and all 4 teams with eliminated ones dimmed + round chip. Called from `renderOrder()` and on tab switch to Projected.
- **Best 3rd Place Teams table** (bottom of Draft Order tab): shows all 12 3rd-place teams ranked, top 8 highlighted in gold with "ADV" badge, dividing line before 9th place. Rendered by `renderThirdPlaceTable()`, called from `renderOrder()`.
- **`fetchGroupCards(normalized)`** — async, called in `syncNow()`. Fetches ESPN summary endpoint (`/summary?event={id}`) for all completed group-stage matches not already in `cardCache`. Parses `yellowCards` + `redCards` from `boxscore.teams[].statistics` and populates `teamCards` (module-level). Results persisted to `wcCards2026` localStorage key.
- **Temporary banner**: shown above Projected tab content until 2026-06-28T09:00:00-04:00 (9am ET). Notes that bracket will update as group stage completes. Uses explicit timezone offset to avoid UTC-midnight parse bug.

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
- `normalizeEvent()` converts ESPN event shape to internal `{ id, stage, completed, home, away }` format. `id` is used by `fetchGroupCards()` to fetch summaries.
- **Summary endpoint**: `https://site.api.espn.com/apis/site/v2/sports/soccer/fifa.world/summary?event={eventId}` — fetched by `fetchGroupCards()` for card data. Results cached in `cardCache` (persisted as `wcCards2026`).
- ESPN uses `season.slug` for stage (e.g. `group-stage`); KO stage slugs unconfirmed until July 4 — matched with permissive regex in `KO_STAGES`
- Shows sync status pill: Syncing / Live (with timestamp) / Error
- Manual "Sync Now" button + "Pause Auto-sync" toggle
- Team name normalization handles API variants (e.g. "United States" → "USA", "South Korea" → "Korea Republic", "Türkiye" → "Turkey")
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

## localStorage Keys

`wcDraft2026` — stores `{ results, coinFlip }` objects keyed by team name / fantasy team ID. Names are no longer persisted (always read from `defaultName`).

Each `results[team]` entry: `{ round, gf, ga, groupStats }` where `gf/ga` are total-tournament goals (used for fantasy scoring) and `groupStats` is `{ p, w, d, l, gf, ga }` for group-stage matches only (used for the standings display). `groupStats` is `null` until API data arrives.

`wcCards2026` — card cache for fair-play tiebreaker. Object keyed by ESPN event ID; values are `{ teamName: { yellow, red } }`. Populated by `fetchGroupCards()` and read into `teamCards` on each sync.
