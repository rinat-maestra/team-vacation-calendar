# Team Vacation Calendar — Daily Regeneration Spec

Give this whole file to Claude Code as context/instructions. It describes exactly
what the existing `calendar.html` does, so a fresh session can regenerate it
correctly every day without re-deriving all the decisions from scratch.

## Source of truth

Notion database: "Team Vacation Tracker"
URL: https://app.notion.com/p/118e0880507880b09f1bc68bb9dcc451

Pull every row (all years, no date filter) via the Notion connector. Each row has:
- `Name` (title) — free-text description, e.g. "Vacation", "Vacation / Sick Leave / Personal",
  "Personal (No Coverage)", "NO VACATIONS (eu)", etc.
- `Date` — a date range (start, optional end)
- `Created by` — the Notion user who owns the entry (resolve to their display name)
- `Status` — Approved / Pending (not currently used in the output, but keep it available)

## Classifying each row

Lowercase the `Name` field and check for substrings:
- contains `"no coverage"` → tag = `noCoverage`
- else contains `"no vacations"` → tag = `noVacations`
- else → tag = `coverage`

## People, teams, and colors

Team assignments (fixed, not derived from Notion):

| Team | People |
|---|---|
| EU (default view) | Alexandra Ryazantseva, Anna Stepanova, Sergio Filatov, Rinat Lukhmanov, Eugenia Smirnova, Konstantin Serov, Roman Fedorov |
| US | Philipp Volnov, Julia Lo, Sasha Haishun, Bruno Matumoto, Efraim Hermes |
| Integrator | Aleksandr Glazkov, Paul Yukin |
| Accounting | Sonya Chesnokova |
| Other | anyone else that shows up (e.g. a removed/unresolvable Notion account) — label as "Unknown" |

If a new person appears in Notion data who isn't in this table, put them in "Other"
rather than guessing, and flag it in the commit message / PR description so a human
can assign them properly.

Color palette (hex), one per person — reuse existing assignments, only add new ones
for genuinely new people, picking any visually distinct color not already used:

```
Alexandra Ryazantseva  #3b82f6
Anna Stepanova          #22c55e
Sergio Filatov          #ef4444
Rinat Lukhmanov         #a855f7
Eugenia Smirnova        #f59e0b
Konstantin Serov        #14b8a6
Roman Fedorov           #64748b
Philipp Volnov          #ec4899
Julia Lo                #f97316
Sasha Haishun           #8b5cf6
Bruno Matumoto          #eab308
Efraim Hermes           #d946ef
Aleksandr Glazkov       #06b6d4
Paul Yukin              #84cc16
Sonya Chesnokova        #10b981
Unknown                 #9ca3af
```

`noVacations`-tagged entries always render as **white dots** regardless of person color.

## Counting rule

Only Monday–Friday days count ("weekdays"). Weekend days within a range are excluded
from every count and total, but weekend cells are still shown (shaded) on the calendar
grid for context.

For a given selected year, an entry that crosses a year boundary (e.g. Dec 28 – Jan 2)
should only count the days that fall inside that year — clip the range to
[Jan 1, Jan 31] before counting, don't just include/exclude the whole entry.

## Output: single self-contained `calendar.html`

Reuse the existing file's structure and CSS (dark/light theme via `prefers-color-scheme`,
mobile-safe-area padding, responsive month grid) — don't redesign it, just refresh the
data inside it. The page has:

1. **Header row**: legend (one dot + name per person in the currently selected team,
   plus a white "No Vacations" swatch) + three dropdowns: Team (EU default / US /
   Integrator / Accounting / Other / All), Year (2024–current+1), View mode
   (All / All vacations / Coverage / No Coverage / No Vacations — controls what's
   plotted as dots on the calendar, nothing else).
2. **Calendar grid**: 12 month cards for the selected year, Monday-start weeks,
   weekend cells shaded, colored dots per day per matching entry.
3. **Vacation Status** cards, one per person in the selected team: total weekdays
   for the selected year (Coverage only), days since their last completed
   vacation, days until their next planned one (or "On vacation" if today falls
   inside a range) — computed against **today's real date**, independent of
   which year is selected in the dropdown.
4. **Vacation List** table (Coverage entries for selected team+year): Person, Dates, Weekdays.
5. **No Coverage** table: same columns, `noCoverage`-tagged entries only.
6. **No Vacations** table: same columns, `noVacations`-tagged entries only.

All four of #3–#6 filter by the currently selected Team and Year (not by the
calendar's View mode dropdown, which only affects the calendar's dots).

## Regeneration steps (what the daily task should actually do)

1. Query the Notion data source for all rows (see URL above).
2. Resolve each `Created by` person ID to a display name via the Notion user list
   (don't guess from dates — call the users endpoint).
3. Re-run the classification + team/color lookup above.
4. Regenerate `calendar.html` in place, preserving its existing structure/styling.
5. Run `node --check` (or equivalent) on the extracted `<script>` block to catch
   syntax errors before committing.
6. Commit to the repo with a message like `chore: refresh vacation calendar data (YYYY-MM-DD)`.
   If GitHub Pages is enabled on this repo, the committed file is what serves live.
7. If any row's `Created by` doesn't resolve to a name in the People/Colors table
   above, don't silently drop it — add them under "Other" and note it in the
   commit message so a human can fix the mapping.
