# How Content Filler works

This document explains what **Content Filler** does, how it works inside Canva,
what permissions it uses, and how to test it end to end.

---

## 1. Overview

Content Filler removes the copy-paste work of producing many text versions of
the same design — translations, A/B copy, personalized messaging, or per-market
campaigns.

The workflow has two steps:

1. **Collect** — pull every text element from the design into a numbered table
   and export it (clipboard TSV or downloaded CSV).
2. **Populate** — paste back a table with one column per variant and either
   auto-generate a page per variant, or apply a variant to a page you
   duplicated yourself.

---

## 2. How it runs (architecture)

- Content Filler is a Design Editor intent app. It renders as a React
  single-page app inside a sandboxed iframe in the Canva editor's side panel.
- It runs entirely client-side, in the user's browser. It has no backend,
  no database, no analytics, and makes no external network requests**.
- All design access happens through the official Canva Apps SDK (`@canva/design`).
- The only browser API it uses beyond the SDK is `navigator.clipboard.writeText`
  (for the "Copy" button) and a standard in-browser file download (for "Download .csv").

### Permissions (scopes)

| Scope | Why it's needed |
|---|---|
| `canva:design:content:read` | Read the text on the current page during **Collect**, and read the page's elements when auto-creating pages. |
| `canva:design:content:write` | Write translated/variant text back into the design during **Populate**, and add new pages. |

No other scopes are requested. The app never reads or writes anything outside
the design the user is actively working on.

---

## 3. The Collect step

1. The user opens the app and stays on the **Collect** tab.
2. Clicking **Collect current page** reads every text element on the current
   page (via `editContent` with `target: "current_page"`) and lists them in a
   numbered table (`#`, `original text`).
3. **Multi-page designs:** the Canva SDK reads one page at a time, so the user
   navigates to the next page in Canva and clicks **Add another page**. Each
   page's text is appended into one continuous, seamlessly numbered table.

3. The user exports with **Copy (TSV)** (paste straight into Google Sheets /
   Excel) or **Download .csv** (named after the design).

### Exported table layout

```
My Design                      ← design name, single top-left cell (row 1)
#       original text          ← header (row 2)
1       Hello                  ← data rows
2       World
```

---

## 4. The Populate step

The user fills in the spreadsheet by adding one column per variant (language
codes like `de`, `fr`; A/B labels; audience names — anything), then copies the
whole table and pastes it into the app's **Populate** tab.

```
#   original text   de      fr
1   Hello           Hallo   Bonjour
2   World           Welt    Monde
```

Clicking **Load table** parses it. The parser tolerates the optional leading
design-name row and the optional leading `#` column; every column after
`original text` is treated as a variant. There are then two ways to apply it:

### 4a. Auto-create pages

Clicking **Auto-create N page(s)** clones the current page once per variant:

- It reads the current page's elements, rebuilds them on a fresh page via
  `addPage`, fills in that column's text, and names each page after the column.
- **Text and images** reconstruct cleanly. Other element types (shapes, video,
  groups) can't be rebuilt via the SDK and are reported as skipped, with a
  prompt to use the manual workflow for full fidelity.

### 4b. Apply manually

For perfect fidelity, the user duplicates the page themselves (Ctrl/Cmd + D),
then clicks **Apply `<code>` to current page**. The app replaces text in place
on the current page using the variant's values (`editContent` →
`replaceText` → `sync`).

### Unpopulated-text notice

In both modes, if any text on the page has **no value** for the chosen variant,
the app shows a **prominent bold banner** stating how many text elements were
left unpopulated, plus a detailed list of exactly which strings were missed.
This makes it obvious when a column is incomplete before the user publishes.

---

## 5. Known platform limitations (and how the app handles them)

- **No "duplicate page" API:** the app rebuilds pages with `addPage` instead
  (text + images), and offers the manual duplicate-then-apply path for full
  fidelity.
- **No "rename existing page" API:** page titles can only be set when a page is
  created, so auto-created pages are named at creation; for the manual path the
  app tells the user to rename the page themselves.
- **Single-page reads:** text is read one page at a time, which is why
  multi-page collection is done page-by-page with **Add another page**.

---

## 6. Privacy & data handling

- The app **collects, stores, and transmits nothing**. There is no server.
- All design content stays between the user's browser and Canva.
- Clipboard contents and downloaded CSVs are entirely under the user's control
  on their own device.


## 7. Step-by-step test (for reviewers)

1. Open any multi-element design and launch **Content Filler** from the side panel.
2. On the **Collect** tab, click **Collect current page** — the page's text
   appears in a numbered table with the design name in the top-left.
3. (Optional) Go to another page in the design and click **Add another page** —
   its text is appended; duplicates are skipped.
4. Click **Copy (TSV)** and paste into a spreadsheet. Add a column with a header
   (e.g. `de`) and fill in values for some — but not all — rows.
5. Copy the spreadsheet table, switch to the **Populate** tab, paste it, and
   click **Load table**.
6. Click **Auto-create 1 page(s)** — a new page named after your column is
   created with the filled-in text. Because you left some rows blank, a bold
   banner reports the unpopulated text.
7. To test in-place apply: duplicate a page (Ctrl/Cmd + D), then click
   **Apply `de` to current page** and confirm the text is replaced in place.
