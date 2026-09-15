# Little Log

A single-page baby tracker — feeds, diapers, and sleep — that stores everything
in one Google Sheet. No server, no build step, no database. The whole app is
`index.html`.

**Live:** https://jasonrunge87.github.io/Little-Log/

## How it works

Sign in with Google, and the page reads and writes a single spreadsheet pinned
by the `SHARED_SHEET_ID` constant near the top of the script. Everyone who signs
in uses that same sheet, so two parents see one shared log instead of each
getting a private copy.

The sheet has three tabs — `Feeds`, `Poops`, `Sleeps` — one row per entry, each
carrying an `id` and a `deleted` flag. Deletes are soft, so row numbers stay
stable and nothing is ever actually removed.

Each tab shows a form, a chart over a selectable range (past day / week /
2 weeks), and the 25 most recent entries.

## Setup

Everything is configured in the `Google OAuth config` block at the top of the
`<script>` in `index.html`:

| Constant | What it is |
| --- | --- |
| `GOOGLE_CLIENT_ID` | OAuth client id. Public by design — it identifies the app, it is not a secret. |
| `SHARED_SHEET_ID` | The sheet id from `docs.google.com/spreadsheets/d/THIS_PART/edit`. Leave it empty to fall back to per-account search-or-create. |
| `FILE_NAME` | Name used when searching Drive or creating a new sheet. |

Scopes requested: `drive.file`, `spreadsheets`, `userinfo.email`,
`userinfo.profile`. The `spreadsheets` scope is what lets a non-owner account
open the pinned sheet by id — `drive.file` alone cannot see a file another
account created.

To give someone access, share the sheet with their Google address as an
**Editor** — named people only, not "anyone with the link".

To host it elsewhere, add that origin to **Authorized JavaScript origins** on
the OAuth client (origin only, no repo path).

## Deploying

The page is static. Push `index.html` to the repo and GitHub Pages serves it.
GitHub's CDN caches for a minute or two, so hard-refresh (`Ctrl`/`Cmd` +
`Shift` + `R`) and give it a moment before deciding a change didn't land.

## Two traps worth knowing

**Never call `google.accounts.oauth2.revoke()` on sign out.** Revoking wipes the
`drive.file` grant, which makes every file the app created invisible on the next
sign-in. The app then creates a fresh empty spreadsheet and the old entries look
lost. Sign-out only drops the in-page token. If data does go missing this way,
**Use another file…** can reconnect the old sheet by URL — the `spreadsheets`
scope can still read it by id.

**Never let Google Sheets decide what a date or time cell means.** Rows written
with `valueInputOption=USER_ENTERED` get coerced: `"2026-09-14"` becomes a real
date cell and `"14:30"` a real time cell, both stored as serial numbers. Read
back with the default `FORMATTED_VALUE`, those return whatever the sheet
*displays* (`9/14/2026`, `2:30:00 PM`), which the chart code cannot parse — so
every entry silently falls out of its bucket and the charts draw flat zeros
while the log tables still list the rows. The app now writes with `RAW`, reads
with `UNFORMATTED_VALUE`, and runs every date/time column through
`normalizeDate` / `normalizeTime`, which convert serials (days since
1899-12-30, fraction = time) back to `YYYY-MM-DD` and `HH:MM`.
