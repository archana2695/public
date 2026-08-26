---
name: google-slides
description: Read and modify the user's Google Slides with the same scoped service-account robot used by the google-sheets and google-docs skills — inspect a deck (slides, object ids, per-element text, speaker notes), add/delete/duplicate/move slides, find-and-replace text across the deck (template filling), set or style a shape's text (bold/italic/underline/color/font/size/link, alignment/spacing), add text boxes, images, shapes, and tables (fill from a 2D grid), set slide backgrounds and speaker notes, render a slide to PNG for visual review, list and share decks. Only presentations explicitly shared with the robot email are reachable. Use whenever the user wants to read from, edit, build, or update a Google Slides presentation / slide deck.
user-invocable: true
allowed-tools:
  - Read
  - Bash
---

# Google Slides

Drive the bundled CLI at `scripts/slides.py` to read and modify Google Slides. It
reuses the SAME robot (service account) as the google-sheets and google-docs
skills, so access is scoped: the robot can only touch presentations the user
explicitly shared with its email.

## House style (build slides the way Archana likes them)

When building or editing decks **for Archana**, follow her house style — read
`references/house-style.md` first. In short: banker/consultant idiom, one message
per slide, **action-title headlines** (a quantified full-sentence takeaway, never a
label), a `PART X · SECTION` kicker + rule on every content slide, dark-navy covers /
white content slides, teal-CAPS section labels with **navy-italic "so what" sub-notes**,
banker number formatting (`$mm`, 1-dp, parens-negatives, `→` for ramps), heavy
footnoting/sourcing with detail pushed to an appendix, and **no em-dashes** in prose.
Palette: navy `#1B4D6B`, teal `#2E7B8F`, amber `#C77D2E`; Tier-1 blue / Tier-2 brown in
charts. Signature exhibits: KPI stat tiles, banker P&L tables, heat-map scorecards,
navy→teal process chevrons, navy-side-block comparison grids (bold labels, regular
values). The file has the full spec, hexes, and per-exhibit recipes.

## Running the tool

```
SKILL=~/.claude/skills/google-slides
$SKILL/.venv/bin/python $SKILL/scripts/slides.py <command> [options]
```

Every command prints JSON. Success → `{"ok": true, ...}`. Failure →
`{"ok": false, "error": "..."}` with a nonzero exit code — read it and act.

## First-run checks

1. If `$SKILL/.venv` doesn't exist, run `bash $SKILL/scripts/install.sh` first.
2. The robot key is shared with the sheets/docs skills (auto-copied; the tool
   also falls back to those keys). To see the robot email: `slides.py whoami`.
3. **If a command fails with "Google Slides API has not been used / is disabled"**,
   the user hasn't enabled the Slides API yet — point them to `setup.md` Step 1
   (enable the Slides API in the Cloud project). This is separate from Sheets/Docs.
4. A permission/404 error → the deck isn't shared with the robot email yet. Tell
   the user to Share → add that email as Editor.

## Identifying a presentation

`--pres` accepts a full URL
(`https://docs.google.com/presentation/d/<ID>/edit`) or the bare presentation id.
URLs are safest — paste what the user gives you.

## The model (read this first)

A presentation is a list of **slides**; each slide has **page elements** (shapes,
text boxes, images, tables) identified by an **objectId**. Text edits target a
shape by its objectId, or use deck-wide find-and-replace. **Always run `read`
first** — it gives you the objectIds and current text you need for everything else.
Slides are addressed by **0-based index OR objectId** anywhere you see `--slide`.
Positions/sizes are in **points (PT)**; a standard 16:9 slide is 720×405 pt.

## Commands

Read / inspect:
- `info --pres <p>` — title, slide count, page size
- `read --pres <p> [--slide <i|id>]` — per-slide dump: index, objectId, layout,
  every element's objectId + type + text, and speaker notes. Your primary tool.
- `list-slides --pres <p>` — compact index / objectId / first-text summary
- `find [--query name]` — list presentations the robot can see
- `thumbnail --pres <p> --slide <i|id> [--out file.png] [--size LARGE|MEDIUM|SMALL]`
  — render a slide to PNG so you can visually verify your edits

Create:
- `create --title "Deck" [--share you@example.com]`
  ⚠️ **This usually FAILS for Archana** — the robot service account has no Drive
  storage, so `create` returns 403 / "quota exceeded". Instead create the deck with
  the **Google Drive connector** (`create_file`, `mimeType:
  application/vnd.google-apps.presentation`) into her **Claude** folder
  (`1cI0cCpD21svvuuKy-d09rodNBuARav4l`) — files there are owned by her and
  auto-shared with the robot — then edit it by id with this skill. See the global
  memory `google-drive-docs-sheets-protocol`.

Slides (structure):
- `add-slide --pres <p> [--layout BLANK|TITLE|TITLE_AND_BODY|TITLE_ONLY|SECTION_HEADER|ONE_COLUMN_TEXT|MAIN_POINT|BIG_NUMBER|CAPTION_ONLY|SECTION_TITLE_AND_DESCRIPTION] [--index N]`
  — returns the new slide's objectId. Default layout BLANK.
- `delete-slide --pres <p> --slide <i|id>`
- `duplicate-slide --pres <p> --slide <i|id>` — returns the new slide's objectId
- `move-slide --pres <p> --slide <i|id> --to N` — reorder to index N

Text:
- `replace --pres <p> --find "{{name}}" --replace "Acme" [--match-case] [--slide <i|id>]`
  — find & replace across the deck (or one slide). **The workhorse for filling
  templated decks** — author placeholders like `{{client}}`, then replace.
- `set-text --pres <p> --object <shapeId> --text "..."` — replace ALL text in one
  shape (empty `--text` clears it). Get the objectId from `read`.
- `add-text-box --pres <p> --slide <i|id> --text "..." [--x 50 --y 50 --w 400 --h 100]`
  — add a free text box (coords in PT).
- `style-text --pres <p> (--object <shapeId> | --match "text") [--all] [--match-case] \`
  `[--bold|--no-bold] [--italic|--no-italic] [--underline] [--strikethrough] \`
  `[--link https://...] [--font-size 18] [--color "#1a73e8"] [--bg "#fff2cc"] [--font "Arial"]`
  — with just `--object`, styles the whole shape's text; with `--match`, styles the
  matched substring (inside `--object` if given, else the first matching shape;
  `--all` = every occurrence everywhere).
- `format-text --pres <p> --object <shapeId> [--align left|center|right|justify] \`
  `[--line-spacing 1.5] [--space-above 6] [--space-below 6]` — paragraph formatting
  for a shape's text.

Speaker notes:
- `set-notes --pres <p> --slide <i|id> --text "..."` — set the slide's speaker notes
  (empty clears). `read` reports current notes per slide.

Media, shapes, tables:
- `add-image --pres <p> --slide <i|id> --url <publicURL> [--x --y --w --h]`
  — Slides fetches the image server-side, so the URL must be publicly reachable.
- `add-shape --pres <p> --slide <i|id> [--shape RECTANGLE|ELLIPSE|ROUND_RECTANGLE|RIGHT_ARROW|CLOUD|STAR_5|...] [--text "..."] [--fill "#fce8b2"] [--x --y --w --h]`
- `add-table --pres <p> --slide <i|id> --rows R --cols C [--json '[["a","b"],...]']`
  — create a table, optionally filled on creation.
- `fill-table --pres <p> --object <tableId> --json '[["Name","Qty"],["Pens","12"]]'`
  — clear and rewrite an existing table's cells (get the tableId from `read`).

Slide background & sharing:
- `set-background --pres <p> --slide <i|id> --color "#0b1f3a"`
- `share --pres <p> --email someone@x.com [--role reader|commenter|writer|owner] [--notify]`
- `whoami` — robot email + which key file is in use

## Working style

- **`read` first**, always — you need objectIds and current text before editing,
  and it's how you confirm what's actually on each slide.
- Prefer `replace` for templated decks (placeholders → values) and `set-text` /
  `set-notes` for targeted per-shape edits. Reach for raw coordinates
  (`add-text-box` / `add-shape`) only when adding genuinely new elements.
- After meaningful edits, optionally `thumbnail` a slide to verify it looks right,
  or `read` it back, and relay results to the user in plain language.
- Before destructive ops (delete-slide, clearing/overwriting text, large replace),
  confirm with the user unless they were explicit; consider `read` first to show
  current content.
- Quote shell args carefully; wrap `--text` / `--match` / `--json` values in single
  quotes.
- If a match or objectId is absent/ambiguous, the tool tells you — relay that
  rather than guessing.
