# Rumbles Live Standings — Base

A live, gameday-updating standings page for the "Rumbles" fantasy football
scoring system — built for **any public Sleeper league**, not just one
fixed one. Paste a league ID, get live Rumbles standings. No backend, no
server-side build step, no signup.

This is a public spin-off of a private, single-league version of this same
page. That original version bakes standings into a JSON file with a daily
GitHub Actions job and layers in one league's own custom house rules on
top; this version drops all of that and computes everything live, entirely
in the browser, straight from Sleeper's public API, so it works for
anyone's league.

## The Rumbles formula

For each week:

- **+9 Rumbles** for winning your scheduled head-to-head matchup
- **+1 Rumble** for every *other* team in the league you outscore that week

In a 12-team league that's up to **20 Rumbles in a single week** (9 for the
win, 11 for having the top score league-wide — fewer if your league has
fewer teams). Rumbles accumulate over the season and reset every year.

## Using it

Open the page and paste a Sleeper league ID into the box at the top, then
click **Load League**. The league ID is the long number in a Sleeper
league's URL (`sleeper.com/leagues/<this part>/...`), or under League
Settings → "League ID" in the Sleeper app. The last league you loaded is
remembered in your browser for next time (via `localStorage`, on this
page's own origin — nothing is sent anywhere or shared between visitors).

Manager names are shown exactly as they appear on Sleeper (each manager's
`display_name`) — there's no custom name mapping like the private version
has for its own league.

## How it works

Everything happens client-side, on every page load and every 30-second
poll:

1. Fetch the league, its users, and its rosters from Sleeper's public API
   (`api.sleeper.app/v1/league/<id>`, `.../users`, `.../rosters`).
2. Sleeper's own `settings.last_scored_leg` field says how many weeks of
   the current season are already fully scored. For each of those weeks,
   fetch that week's matchups and compute every roster's Rumbles, PF, PA,
   H2H result, and vs.-the-field outscored count — the exact same formula
   described above, applied week by week and summed into a season
   cumulative total.
3. The current, in-progress week (one past whatever's already fully
   scored) is tracked live: actual stats and Sleeper's own projections are
   fetched and re-scored every 30 seconds, so "Points This Week" and the
   standings' Projected-mode totals update in real time without a reload.

There's no daily build job and nothing gets written to disk anywhere —
every visitor's browser does this same computation independently, live,
whenever they load the page or paste a league ID.

### Commissioner score overrides

Sleeper's `custom_points` field is always honored: when a commissioner
manually overrides a roster's score for a week (a house-rule bonus or
penalty, a corrected stat, whatever), that's the OFFICIAL number Sleeper
itself treats as final — so it's what this page uses for that roster's PF,
for their opponent's PA, for who won the H2H matchup, and for the
vs.-the-field outscored count that week, not just PF in isolation. An
override can change how many OTHER teams a roster outscored that week
(and, in turn, how many teams those other rosters got outscored by), which
feeds directly into Rumbles for everyone involved, not only the overridden
roster.

### What this version doesn't have

The private, single-league version this was spun off from also tracks a
custom house rule specific to that league — crediting a manager with a
backup QB's points when their started QB gets hurt mid-game — and shows a
running log of it below the standings. That feature, and all of its
supporting code and documentation, has been completely removed here: it's
a house rule for one particular league, not something that makes sense to
apply to an arbitrary Sleeper league by default. Everything else — live
scoring, the Actual/Projected toggle, matchup coloring, the "Pts This
Week"/"Pts Last Week" tooltips, sorting, and the win/loss streak badge —
works the same as it always has.

### Columns

Rank (`#`), Manager, Rumbles, This Week (Rumbles earned so far this week),
Points This Week (the raw score for this week, to 2 decimal places),
Rumble % (Rumbles earned / max possible so far), PF, PA, H2H, and Vs.
Field. On a narrow screen the table scrolls horizontally (Rank and Manager
stay pinned) — the first five columns are sized to fit on a portrait phone
with no scrolling needed; Rumble % onward requires scrolling right.

Every column, including `#`, is **sortable** — click a header to sort by
it (numbers/records default to biggest-first, Manager defaults to A-Z;
click again to flip direction). The actual VALUE in each team's `#` cell
never changes just because a different column is sorted — it's always
that team's fixed season standing (by cumulative Rumbles, then PF).

**The default sort depends on which mode is selected.** Actual mode
defaults to natural standings order; Projected mode defaults to Points
This Week, highest first, since that's the more useful view while a week
is live. Once you click a column header yourself, your choice sticks
across mode switches.

### Actual vs. Projected

- **Actual** — the solidified view. Season-cumulative Rumbles/PF/PA/H2H/Vs.
  Field are all locked to whatever's already fully scored; the in-progress
  week's actual score is shown live ("Points This Week"/"This Week") but
  not folded into the season totals until the week is fully final.
  Pregame — once matchups haven't kicked off yet — "Pts This Week"/"This
  Week" instead show LAST week's real, final numbers (relabeled "Pts Last
  Week"/"Last Week" accordingly), since a flat 0.00 across the board
  before kickoff isn't useful information.
- **Projected** — folds the in-progress week's projected numbers on top of
  the cumulative totals, once the week has actually kicked off, giving an
  early look at where the standings could land if the week ended right
  now.

During a live week, each manager's name is colored by this week's
scheduled H2H matchup (hover a name to see who they're playing), and
whichever manager is currently ahead has their This Week/Points This Week
values colored to match.

Hovering (or tapping, on mobile) a team's "Points This Week"/"Pts Last
Week" cell shows a per-starter breakdown: actual and projected points for
every player who qualifies under the current mode, ordered by roster slot,
with a live starter's row highlighted. In Projected mode this also shows
each player's kickoff time, colored by timeslot. The "Pts Last Week"
version (pregame Actual mode only) instead shows each player's real score
vs. their real pregame projection, colored green/red by whether they beat
it.

A manager's current H2H winning/losing streak, as of the last completed
week, shows as a small "W"/"3W"/"L"/"2L" badge next to their name —
pregame Actual mode only, alongside the "Last Week"/"Pts Last Week"
relabeled columns it gives context for.

### Live scoring

While a week is in progress, this page fetches every starter's actual
stats and Sleeper's own projection for them directly in the browser, on
every 30-second poll, and scores them by dot-producting each stat line
against the league's real `scoring_settings` — never Sleeper's generic
`pts_ppr`/`pts_std` fields, which won't match a league with any custom
scoring rules. An in-progress player's shown "proj" blends toward their
real actual total as their game progresses, weighted by how much of the
game remains, rather than sitting frozen on a stale pregame number.

### Status indicator: pregame vs. live

The status dot/text tracks two different things separately: whether a
week's matchups exist to be scored, and whether a game has actually
kicked off. Before kickoff, the dot stays grey and the text reads "Week N
begins &lt;Day&gt; &lt;time&gt;"; once the week has genuinely started, the
dot turns green and reads "Live: Week N". Offseason and preseason get
their own messages.

## Deploying your own copy

This is a single self-contained HTML file with no build step and no
server — `rumbles.html` is the whole app. Any static host works (this
repo is set up for GitHub Pages): push it to a repo, turn on Pages for
that repo, and the page is live. Nothing needs configuring per-league —
that's the whole point of the league-ID input.

## Relationship to the private version

This project started as a fork/spin-off of a private, single-league
Rumbles page. The two are independent: this repo has no server-side build
step, no bundled league data, and none of that league's custom house
rules — just the generic Rumbles formula, computed live for whichever
league ID you give it.
