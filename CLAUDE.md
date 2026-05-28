# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the App

This is a zero-dependency, single-file web app. Open it directly in any browser:

```powershell
Start-Process "index.html"
```

No build step, no server, no package manager. Changes to `index.html` take effect on the next browser refresh.

## Architecture

Everything lives in `index.html` as three co-located sections:

1. **`<style>`** — All CSS using custom properties defined on `:root`. The color palette follows a Tailwind-style naming convention (`--blue-500`, `--gray-200`, etc.). Category accent colors are defined in the `CAT_COLORS` JS object, not in CSS, so they can be applied dynamically via inline styles.

2. **`<body>` (HTML)** — Static shell only. The bookmark cards are never in the HTML source; they are always rendered by JS into `#grid`. The modal (`#modal-backdrop`) and toast (`#toast`) are always present in the DOM, shown/hidden via CSS classes (`.open`, `.show`, `.visible`).

3. **`<script>`** — Vanilla JS, no framework. Key sections in order:
   - **DATA**: `STORAGE_KEY`, `SAMPLE_BOOKMARKS`, `CAT_COLORS`, `uid()`, `loadBookmarks()` / `saveBookmarks()`
   - **STATE**: Three mutable variables — `bookmarks` (array), `activeCategory` (string), `searchQuery` (string)
   - **RENDER**: `render()` is the single source of truth for the UI. It calls `renderFilters()` then re-generates all cards from scratch on every state change. Cards are built as HTML strings via `cardHTML()` and injected with `innerHTML` into a wrapper div, then the root `.card` element is appended to `#grid`.
   - **EVENT HANDLERS**: Delete uses event delegation on `#grid`. Search is `input` on `#search-input`. Modal open/close is wired to several triggers including `Escape` key and backdrop click.

## Data Model

Each bookmark is a plain object stored as JSON in `localStorage` under the key `bm_dashboard_v1`:

```js
{
  id: string,        // uid() — base-36 timestamp + random suffix
  title: string,
  url: string,       // always includes protocol after validation
  category: string,  // free-form; preset options + custom text field
  notes: string,     // optional, may be empty string
  createdAt: number  // Date.now() timestamp
}
```

Bookmarks are stored newest-first (`unshift` on add). The array order in localStorage is the display order.

## Key Behaviours to Preserve

- **`render()` is always a full re-render** — there is no partial/diffed update. This keeps state consistent but means adding staggered `animationDelay` is index-based.
- **Category filter buttons are rebuilt on every `render()`** — `renderFilters()` removes all `.category-btn` elements before recreating them, deriving categories from the live `bookmarks` array.
- **Custom category wins over dropdown** — in the save handler, `cusObj || selCat` means a non-empty custom input always takes precedence. Clearing the custom field reverts to the dropdown value.
- **`escHtml()` must be called on all user-supplied strings** before interpolating into `cardHTML()` to prevent XSS.
- **Favicon images** are fetched from Google's favicon service (`https://www.google.com/s2/favicons?sz=64&domain=…`). The `onerror` handler hides broken images silently.
- **Sample data seeding** happens only when `localStorage.getItem(STORAGE_KEY)` returns `null` (first visit). Clearing `localStorage` resets to samples.
- **Edit mode** is tracked by `editingId` (string | null). `openModal(bm)` pre-fills the form and sets `editingId = bm.id`; `openModal()` (no arg) is add mode. `closeModal()` always resets `editingId` to `null`. The save handler branches on `editingId`: edit uses `bookmarks.map()` to update in-place (preserving `id` and `createdAt`); add uses `bookmarks.unshift()`. `PRESET_CATEGORIES` is a constant array used to decide whether a bookmark's category goes into the `<select>` or the custom text input when the edit modal opens.
