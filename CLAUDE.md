# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single self-contained HTML file (`index.html`) — a client-side React app that converts supplier "Argento" jewelry Excel files (`.xls`/`.xlsx`), plus an optional folder of product images, into one merged, translated Hebrew Excel workbook with dropdown-validated columns and embedded images. No build step, no server, no npm — it's opened directly in a browser and loads React, ReactDOM, Babel standalone, SheetJS (`xlsx@0.18.5`), and ExcelJS (`exceljs@4.4.0`) from CDNs via `<script>` tags. JSX is transpiled in-browser by Babel standalone.

There is no package.json, test suite, or lint config in this repo. There is nothing to "build" or "run" beyond opening `index.html` in a browser (or deploying it as-is, e.g. GitHub Pages).

## Development

- Edit `index.html` directly — all app code (parsing logic, Excel-writing logic, React UI) lives in the single `<script type="text/babel">` block. Code comments and all UI/output strings are in Hebrew; keep new ones consistent.
- To test changes: open `index.html` in a browser, upload real supplier `.xls`/`.xlsx` files, and verify the output workbook. All `*.xls`/`*.xlsx` files are gitignored, so samples must be sourced locally (e.g. a local `2607/` folder), never committed.
- When changing ExcelJS or SheetJS usage, verify the API against the pinned CDN versions above, not the latest docs.
- `PLAN.md` is the authoritative design/requirements doc — read it before making non-trivial changes to the parsing rules, category tree, or output columns. It documents the reasoning behind decisions (why certain things are flagged rather than guessed, why subcategory isn't auto-derived, etc.) that isn't obvious from the code alone.

## Architecture

The whole pipeline runs client-side in one pass triggered by the "הרץ המרה" (Run conversion) button (`handleRun` in `App`):

1. **Parse** (`parseWorkbookFile`): each uploaded workbook has two sheets (`Regular Tag`, `New Tag`). Row 0 = title, row 1 = header, data from row 2. Columns are located **by header name** (`colIndex("ARG code")`, `"Size"`, `"Stone Shape"`, `"Stone Size"`, `"Plating color"`, `"Remark"`, …), not fixed position — supplier column order varies. The CI number comes from the filename (`CI_RE`). Each row's `ARG code` (e.g. `R1000509-1-1-7`; a leading `*` means "נטו" net price, *not* the sheet's new/regular status) is decoded via `CODE_RE` into: tag, category letter (E/N/B/R), base SKU, plating digit, stone digit, size/suffix. Malformed codes go to `errors` instead of the main output.
2. **Translate**:
   - Category letter / plating digit / stone digit map through fixed tables (`LETTER_TO_KEY`, `PLATING_DIGIT_TO_KEY`, `STONE_DIGIT_TO_KEY`) derived from the supplier's code scheme — not user-editable.
   - Everything else goes through `translations`: user-editable dictionary seeded from `DEFAULT_TRANSLATIONS`, edited in the UI via per-group tables (`DICT_GROUPS`: `fixed-map` = edit values only, `editable-map` = keys are English match terms, `list` = Hebrew-only dropdown options) or a raw JSON editor. Persisted to `localStorage` (`TRANSLATIONS_STORAGE_KEY`); `mergeTranslations` merges stored data over defaults so new default keys/groups appear without clobbering user edits — when adding a dictionary group, add it to both `DEFAULT_TRANSLATIONS` and `DICT_GROUPS`.
   - Field precedence: size = `Size` column, else code suffix (unless `A`); stone size = `Stone Size` column, else an `mm` value in the description's 2nd+ lines; stone shape = `Stone Shape` code column via `stone_shape_codes`, else substring match of description against `stone_shapes`/`shapes`.
   - `Remark` is normalized by plating code (`REMARK_RULES`: rhd/14k/rose), not material name (supplier misspells materials).
   - Anything unrecognized appends to `flag_reason` and sets `needs_review = "כן"` rather than being guessed.
3. **Images**: rows match images by `image_key` = `{letter}{baseSku}-{plating}-{stone}` (size excluded — size variants share a photo). Source order: uploaded image folder (`buildFolderImageMap`/`extractImageKey`) → fallback to images embedded in the source workbook at the same row (`extractSourceImages`, via ExcelJS). Images are downscaled to ≤500px JPEG (`prepareImageForEmbedding`) and cached per key. Rows with no image are flagged `needs_review`.
4. **Build workbook** (`buildWorkbook`, via ExcelJS — needed because SheetJS's free tier can't write data validation or images): rows sorted by category (`CATEGORY_SORT_ORDER`: N, B, R, E) then base SKU. Produces `Variants` (RTL; the sheet the user works in; dropdown columns backed by hidden `Lists`; `product_name`, `description_free`, `subcategory_final`, `subsubcategory_final` intentionally left empty for manual fill), `מקור` (source traceability), `Errors` (malformed rows), `Flags` (rows with `needs_review`), and hidden `Lists`. Column set/order is defined by the `variantCols` array.
5. **Download**: serialized to a Blob, offered as `argento-variants-merged.xlsx`.

Key invariant to preserve when editing: the ARG code is *decomposed*, never reconstructed — each parsed field maps 1:1 to a source code segment, source column, or dictionary lookup, and anything unrecognized is surfaced via `needs_review`/`flag_reason` rather than silently defaulted or guessed.
