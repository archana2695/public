
name: google-sheets
description: Read and modify the user's Google Sheets with cell-level precision — read/write ranges, append rows, create spreadsheets, add/delete/rename tabs, insert/delete rows & columns, apply formatting (bold, color, background, alignment, number formats), write formulas, and share. Uses scoped access via a Google service account: only sheets explicitly shared with the robot email are reachable. Use whenever the user wants to read from, write to, or manipulate a Google Sheet / spreadsheet.
user-invocable: true
allowed-tools:
  - Read
  - Bash

# Google Sheets

Drive the bundled CLI at `scripts/sheets.py` to read and modify Google Sheets.
Access is **scoped**: a service account (robot email) can only touch sheets the
user has explicitly shared with it.

## Running the tool

Always use the skill's venv Python. The skill root is this file's directory.

```
SKILL=~/.claude/skills/google-sheets
$SKILL/.venv/bin/python $SKILL/scripts/sheets.py <command> [options]
```

Every command prints JSON. Success → `{"ok": true, ...}`. Failure →
`{"ok": false, "error": "..."}` with a non-zero exit code — read the error and act on it.

## First-run checks

1. If `$SKILL/.venv` does not exist, run `bash $SKILL/scripts/install.sh` first.
2. If `$SKILL/credentials/service_account.json` is missing, the user hasn't done
   setup — point them to `setup.md` and offer to walk them through it. Do not guess.
3. To find the robot email to share sheets with: run `sheets.py whoami`.
4. If a command returns a permission/404 error, the sheet almost certainly isn't
   shared with the robot email yet — tell the user to Share → add that email as Editor.

## Identifying a sheet

`--sheet` accepts either a full URL (`https://docs.google.com/spreadsheets/d/.../edit`)
or the bare key/ID. URLs are safest — just paste what the user gives you.

`--tab` accepts a tab title (e.g. `Q3`) or a 0-based index. Omit it to use the first tab.

## Commands (quick reference)

Read / inspect:
- `info --sheet <s>` — title + every tab with dimensions
- `list-tabs --sheet <s>`
- `read --sheet <s> [--tab T] [--range A1:C10]` — omit range to read the whole tab.
  A range with a `!` (e.g. `Sheet1!A1:B2`) is honored as-is.
- `find [--query name]` — list spreadsheets the robot can see

Write data (`--values` is a JSON 2D array; values are USER_ENTERED by default so
`=SUM(A1:A3)`, numbers, and dates parse naturally — add `--raw` to store literally):
- `write --sheet <s> [--tab T] --range A1 --values '[["Name","Score"],["Al",90]]'`
- `append --sheet <s> [--tab T] --values '[["new","row"]]'`
- `clear --sheet <s> [--tab T] [--range A1:C10]` — omit range to clear the tab

Structure:
- `create --title "My Sheet" [--share you@example.com]`
  ⚠️ In scoped mode the robot owns what it creates; pass `--share <your email>` so
  YOU can see/edit it, or the new sheet is only visible to the robot.
- `add-tab --sheet <s> --title "Q3" [--rows 1000 --cols 26]`
- `delete-tab --sheet <s> --tab "Q3"`
- `rename-tab --sheet <s> --tab "Sheet1" --new-title "Summary"`
- `insert-rows --sheet <s> [--tab T] --index 2 --count 3 [--inherit]` (1-based)
- `delete-rows --sheet <s> [--tab T] --start 2 --end 5` (1-based, inclusive)
- `insert-cols` / `delete-cols` — same shape as rows

Formatting (`--range` required; combine flags freely):
- `format --sheet <s> [--tab T] --range A1:C1 --bold --bg "#fff2cc" --halign center`
- text: `--bold/--no-bold`, `--italic/--no-italic`, `--font-size 12`,
  `--font-family "Century Gothic"`, `--color "#1a1a1a"`
- cell: `--bg "#fff2cc"`, `--halign left|center|right`, `--valign top|middle|bottom`, `--wrap`
- numbers: `--number-format "#,##0.00"` (or `"0.0%"`, `"yyyy-mm-dd"`, `"$#,##0"`),
  optionally `--number-format-type CURRENCY|PERCENT|DATE|...`

Sharing / identity:
- `share --sheet <s> --email someone@x.com [--role reader|commenter|writer|owner] [--notify]`
- `whoami` — prints the active mode and the service-account email

## Working style

- Before a destructive op (clear, delete-rows/cols, delete-tab, overwriting a
  populated range), confirm with the user unless they were explicit. Consider a
  `read` first to show what's there.
- After a write, optionally `read` the affected range back to confirm.
- Quote shell args carefully — the JSON in `--values` contains quotes; wrap the
  whole value in single quotes.
- Relay the tool's JSON results to the user in plain language, not raw JSON dumps.
- **Match the sheet's existing formatting.** `write` sets values only — it never
  changes font, size, or color, so a value dropped into an already-styled cell
  inherits that cell's look. But a cell that's genuinely empty, or a row added
  with `insert-rows` *without* `--inherit`, falls back to the Google default
  (Arial 10, black) and will visibly clash. Before styling new cells, sample a
  neighbor in the same block and copy its font family, size, and number format.
- **There is no command to *read* formatting.** To inspect it, call the REST API
  with an authorized session (this venv has `google-auth` + `gspread`, **not**
  `googleapiclient`) and pass `includeGridData=true`:

  ```python
  from google.oauth2 import service_account
  from google.auth.transport.requests import AuthorizedSession
  creds = service_account.Credentials.from_service_account_file(
      SA_FILE, scopes=["https://www.googleapis.com/auth/spreadsheets"])
  s = AuthorizedSession(creds)
  r = s.get(f"https://sheets.googleapis.com/v4/spreadsheets/{SID}",
            params={"ranges": "Tab!A1:H60", "includeGridData": "true"}).json()
  ```

  Then read `effectiveFormat.textFormat` (`fontFamily`, `fontSize`, `bold`,
  `foregroundColorStyle.rgbColor`) and `effectiveFormat.numberFormat.pattern`.
  A `userEnteredValue.formulaValue` key means the cell is a formula, not a
  hardcoded value. Note `effectiveFormat` is what renders; `userEnteredFormat`
  omits inherited styling.
