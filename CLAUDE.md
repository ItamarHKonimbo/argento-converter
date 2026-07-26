# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single self-contained HTML file (`index.html`) — a client-side React app that converts supplier "Argento" jewelry Excel files (`.xls`/`.xlsx`) into one merged, translated Hebrew Excel workbook with dropdown-validated columns. No build step, no server, no npm — it's opened directly in a browser and loads React, ReactDOM, Babel standalone, SheetJS (`xlsx`), and ExcelJS from CDNs via `<script>` tags. JSX is transpiled in-browser by Babel standalone.

There is no package.json, test suite, or lint config in this repo. There is nothing to "build" or "run" beyond opening `index.html` in a browser (or deploying it as-is, e.g. GitHub Pages).

## Development

- Edit `index.html` directly — all app code (parsing logic, Excel-writing logic, React UI) lives in the single `<script type="text/babel">` block.
- To test changes: open `index.html` in a browser, upload real supplier `.xls`/`.xlsx` files, and verify the output workbook. Sample supplier files are gitignored (see `.gitignore` patterns like `System documents-CI-*.xls`) so they must be sourced locally, not from the repo.
- `PLAN.md` is the authoritative design/requirements doc — read it before making non-trivial changes to the parsing rules, category tree, or output columns. It documents the reasoning behind decisions (why certain things are flagged rather than guessed, why subcategory isn't auto-derived, etc.) that isn't obvious from the code alone.

## Architecture

The whole pipeline runs client-side in one pass triggered by the "הרץ המרה" (Run conversion) button:

1. **Parse** (`parseWorkbookFile`): each uploaded workbook has two sheets (`Regular Tag`, `New Tag`). Data rows start at index 2 (row 0 = title, row 1 = header). Each row's `ARG code` (e.g. `R1000509-1-1-7`, optionally prefixed with `*`) is decoded via `CODE_RE` into: new/regular tag, category letter (E/N/B/R), base SKU number, plating digit, stone digit, and size/suffix. Rows with malformed codes are collected into an `errors` list instead of the main output.
2. **Translate**: category letter, plating digit, and stone digit map through fixed internal tables (`LETTER_TO_KEY`, `PLATING_DIGIT_TO_KEY`, `STONE_DIGIT_TO_KEY` — these are derived directly from the supplier's code scheme and are *not* meant to be user-edited). Free-text `Description` and shape/subcategory matching go through `translations`, a user-editable JSON dictionary (`DEFAULT_TRANSLATIONS` is the seed value, editable live in a textarea in the UI) covering `base_types`, `categories`, `materials`, `stone_colors`, `subcategories`, `subsubcategories`, `stone_shapes`, `shapes`. Anything not found in the dictionary is flagged for manual review (`needs_review`) rather than guessed.
3. **Build workbook** (`buildWorkbook`, via ExcelJS — needed because SheetJS's free tier can't write dropdown/data-validation cells): produces 5 sheets — `Variants` (the one sheet the user actually works in, with dropdown-validated columns backed by a hidden `Lists` sheet), `מקור` (source/traceability columns), `Errors` (skipped malformed rows), `Flags` (valid rows flagged `needs_review`), and hidden `Lists` (dropdown option sources).
4. **Download**: workbook is serialized to a Blob and offered as a download link (`argento-variants-merged.xlsx`).

Key invariant to preserve when editing: the ARG code is *decomposed*, never reconstructed — each parsed field maps 1:1 to a source code segment or dictionary lookup, and anything unrecognized is surfaced via the `needs_review`/`flag_reason` mechanism rather than silently defaulted or guessed.
