# Categories — Product Spec

A desktop app for cataloguing personal belongings: organise items into nested categories, track them in editable tables, and keep a dated log of what was worn, bought, stored and got rid of.

- **Stack:** Python 3.11+, tkinter / ttk, SQLite (stdlib `sqlite3`), Pillow (images)
- **Platform:** desktop (Windows first, should run on macOS/Linux)
- **Working name:** Categories (package name `categories`)

---

## 1. Core concepts

| Concept | Meaning |
|---|---|
| **Category** | A named node in a tree. Can have any number of subcategories, any depth. |
| **Item** | A thing the user owns (or owned). Belongs to exactly **one** category, at any level of the tree. |
| **Field** | A user-defined table column. Defined on a category and **inherited** by all its descendants. |
| **Event type** | A kind of event. Built-in: `bought`, `discarded`, `stored`, `worn`; the user can create more. A type can be **scoped** to a category (e.g. `worn` → Clothing) so it's only offered for items in that subtree. |
| **Event** | One dated occurrence of an event type involving **one or more** items. Wearing three things on one day is **one** `worn` event with three items. The calendar is a view over events. |
| **Status** | An item's current state: `owned`, `stored`, `discarded`. |

### Nesting rule
An item is stored only in the category it was added to. Viewing a category shows the items of that category **plus all its descendants**. Example: a T-shirt added to `Clothing › T-shirts` appears when viewing `T-shirts` and when viewing `Clothing`. Items are never copied up the tree; membership is computed (recursive query).

### Column inheritance rule
A category's visible columns = its own fields + all fields of its ancestors. When viewing a parent, the table shows the **union** of columns across the parent and all descendants; cells that don't apply to an item are left blank.

Example:
- `Clothing` defines `Brand`, `Colour`
- `Clothing › T-shirts` defines `Size`
- Viewing `T-shirts` → Name | Brand | Colour | Size
- Viewing `Clothing` → Name | Brand | Colour | Size (Size blank for non-T-shirts)

---

## 2. Main window layout

```
┌──────────────────────────────────────────────────────────────────────┐
│ Categories: you own 128 things                              [_][□][x]│
├───────────────────┬──────────────────────────────────────────────────┤
│ [Categories][Cal.]│  Clothing › T-shirts                             │
│───────────────────│  [New] [Delete] [Log event ▾]  [Search...       ]│
│ ▾ All items       │ ┌──────┬────────────┬────────┬────────┬────────┐ │
│   ▾ Clothing      │ │ img  │ Name ▲     │ Brand  │ Colour │ Size   │ │
│     ▸ T-shirts    │ ├──────┼────────────┼────────┼────────┼────────┤ │
│     ▸ Trousers    │ │ [▣]  │ Black tee  │ Uniqlo │ Black  │ M      │ │
│   ▸ Books         │ │ [▣]  │ Band tee   │ —      │ White  │ L      │ │
│   ▸ Tech          │ │ ...  │            │        │        │        │ │
│                   │ │  ＋ Add item                               │ │
│                   │ └──────┴────────────┴────────┴────────┴────────┘ │
│ [+ Category]      │ ┌ Preview ─────────────────────────────────────┐ │
│                   │ │ [ large image ]   Black tee                  │ │
│                   │ │                   Bought 2026-03-02 · Worn 14│ │
│                   │ └──────────────────────────────────────────────┘ │
└───────────────────┴──────────────────────────────────────────────────┘
```

- **Left sidebar** (fixed width ~240px, resizable via `ttk.PanedWindow`): a `ttk.Notebook` with two tabs, **Categories** and **Calendar**.
- **Right pane**: shows the **Table view** when the Categories tab is active, and the **Calendar view** when the Calendar tab is active. Switching tabs swaps the right pane.
- **Window title**: `Categories: you own X things`, where X = number of items whose status is not `discarded` (stored items still count as owned). Updates immediately after any add, delete, or status change.

---

## 3. Categories tab (sidebar)

- `ttk.Treeview` in tree mode showing the category hierarchy.
- A root node **All items** is always present and cannot be renamed or deleted; selecting it shows every item.
- **Selecting a category** immediately refreshes the table on the right to that category (plus descendants), and the breadcrumb above the table updates (`Clothing › T-shirts`).
- **[+ Category]** button at the bottom: adds a new category as a child of the selected one (or top-level if "All items" is selected), then starts inline rename.
- **Right-click menu** on a category:
  - Add subcategory
  - Rename (also: double-click / F2)
  - Delete — only allowed if the category and its descendants contain no items; otherwise prompt: "Move N items to parent category and delete?"
- Optional (later): drag-and-drop a category to re-parent it; drag items from the table onto a category to move them.
- Expanded/collapsed state and last selected category are remembered between sessions.

---

## 4. Table view (right pane)

### 4.1 Structure
- `ttk.Treeview` in headings mode, with vertical and horizontal scrollbars.
- **Column 0 (`#0`)**: item thumbnail (32×32), blank if the item has no image.
- **Name column**: always first and always present. This is the row's name.
- Then inherited fields (ancestor-first order), then the category's own fields, each in their saved `position` order.

### 4.2 Toolbar (above the table)
- **Breadcrumb** of the current category.
- **[New]**: adds a new item to the current category and starts editing its Name cell.
- **[Delete]**: deletes the selected item(s) after confirmation. (Distinct from "got rid of" — Delete removes the record entirely, including its events. Getting rid of something is a status change, see §6.)
- **[Log event ▾]**: dropdown of the event types that apply to **all** selected items (see §6). Picking one opens the Log event dialog with the selection prefilled.
- **Search box** with scope dropdown (see §4.8).
- **[☐ Show discarded]**: discarded items are hidden by default; when shown, they appear greyed out.

### 4.3 Adding entries
- Two equivalent ways: the **[New]** button, or the **＋ Add item** pseudo-row pinned at the bottom of the table.
- A new item is created in the currently selected category (if "All items" is selected, prompt for a category).
- Creating an item automatically logs a `bought` event dated today; the date can be edited from the item's context menu ("Edit bought date…").

### 4.4 Editing
- **Rename a row (item name)**: double-click the Name cell or press F2 → inline `Entry` overlaid on the cell. Enter saves, Escape cancels.
- **Edit a cell**: double-click any field cell → inline `Entry`. Values are stored as text.
- **Rename a column**: double-click its heading → inline `Entry` over the heading. If the column is inherited from an ancestor, show a note: "This renames the column for `Clothing` and all its subcategories."
- **Right-click on a heading**:
  - Rename column
  - Add column (to the current category)
  - Delete column (confirm; deletes all values in it)
  - Move left / Move right
- **Right-click on a row**: Edit, Set image…, Remove image, Log event ▸ (submenu of applicable event types), Take out of storage, Got rid of…, Edit bought date…, Move to category…, Delete.

### 4.5 Images
- One image per item, set via "Set image…" (file dialog: png, jpg, jpeg, gif, webp).
- The image is **copied** into `data/images/<item_id>.<ext>`; the DB stores the relative path.
- Thumbnail shown in column 0; a larger version shown in the **Preview panel** below the table for the selected item.
- Per-cell images are **out of scope** — `ttk.Treeview` only supports one image per row.

### 4.6 Preview panel
- Below the table (collapsible). Shows the selected item's large image, name, category path, status (and storage location if stored), bought date, and its 5 most recent events (newest first).

### 4.7 Sorting
- Clicking a column heading sorts by that column; clicking again reverses. The active column shows ▲ or ▼ in its heading.
- Sort is case-insensitive; values that parse as numbers sort numerically; blanks sort last.
- The ＋ Add item row always stays at the bottom.
- Sort choice is remembered per category for the session.

### 4.8 Search
- Search box on the right of the table toolbar. **Ctrl+F** focuses it, **Escape** clears it.
- Filters the table live as you type (debounced ~200 ms).
- Case-insensitive substring match against item name, all field values, and storage location.
- **Scope dropdown** next to the box: `This category` (default, includes descendants) or `All items`.
- In `All items` scope, an extra **Category** column shows each result's category path.
- The breadcrumb shows the active search, e.g. `Search "tee" in Clothing — 12 results`.
- Respects **Show discarded**. Sorting, editing, and Log event all work on results as normal.
- Changing the selected category keeps the query and re-applies it.
- The title count is not affected by search.

### 4.9 Rotate table (optional, last phase)
- Toolbar toggle that transposes the table: items become columns and fields become rows. Read-only in rotated mode (no sorting, no ＋ row). Only implement if everything else is done.

---

## 5. Calendar tab

### 5.1 Layout
When the Calendar tab is active, the right pane shows a month grid instead of the table.

```
┌───────────────────┬──────────────────────────────────────────────────┐
│ [Categories][Cal.]│        ‹   October 2026   ›          [Today]     │
│───────────────────│ ┌──────┬──────┬──────┬──────┬──────┬──────┬────┐ │
│ Show:             │ │ Mon  │ Tue  │ Wed  │ Thu  │ Fri  │ Sat  │Sun │ │
│ ☑ Worn            │ ├──────┼──────┼──────┼──────┼──────┼──────┼────┤ │
│ ☑ Bought          │ │      │      │  1   │  2   │  3   │  4   │ 5  │ │
│ ☑ Stored          │ │      │      │worn 2│      │      │      │    │ │
│ ☑ Got rid of      │ ├──────┼──────┼──────┼──────┼──────┼──────┼────┤ │
│                   │ │  6   │ ...  │      │      │      │      │ 11 │ │
│ Category:         │ │      │      │      │      │      │      │worn│ │
│ [All items     ▾] │ │      │      │      │      │      │      │ 3 ▽│ │
│                   │ │      │      │      │      │      │      │bght│ │
│                   │ │      │      │      │      │      │      │ 1 ▽│ │
│                   │ └──────┴──────┴──────┴──────┴──────┴──────┴────┘ │
└───────────────────┴──────────────────────────────────────────────────┘
```

- **Sidebar (Calendar tab)**: filters.
  - One checkbox per event type (built-in and custom), generated from the `event_type` table.
  - A category dropdown (default All items) — limits counts to events involving at least one item in that category or its descendants.
  - **[Manage event types…]** button (see §6.3).
- **Header**: `‹` previous month, `Month Year` label, `›` next month, **[Today]** button.
- **Grid**: 7 columns (Mon–Sun, UK week start) × 5–6 rows, built from `Frame`s and `Label`s (not `tkcalendar` — it can't render custom multi-line cells).
- Days outside the current month are shown greyed out.
- Today's cell is highlighted.

### 5.2 Day cell
- Shows the day number, then one line per event type **with at least one event** that day, e.g. `worn: 1 ▽`, `bought: 2 ▽`, drawn in the type's colour.
- Counts are **events, not items**: one outfit of three things is `worn: 1`.
- Types with no events are hidden, and days with no events show only the number. If more types have events than fit, show the first three and `+N more`.
- Clicking **▽** next to a kind opens the Day popup filtered to that kind.
- Clicking anywhere else in the cell opens the Day popup showing all kinds for that day.

### 5.3 Day popup
- A `Toplevel` window titled e.g. `Sunday 11 October 2026`.
- Contains a tree-table:
  - Each **event** is a parent row: Type | Location | Note | item count.
  - Its **items** are child rows: thumbnail | Item | Category.
  - Events are expanded by default.
- A type filter at the top (All + one entry per event type), preset from whichever ▽ was clicked.
- **No event creation here.** Events are logged from the main table (§6); the popup is for reviewing and correcting.
- Right-click on an event row: Edit date / location / note, Delete event.
- Right-click on an item row: Remove from this event, Go to item. Go to item switches to the Categories tab and selects the item's category and the item.
- Closing the popup refreshes the calendar counts.

---

## 6. Events

### 6.1 Automatic events
Only two event types are ever created automatically:

| Type | When | Shape | Side effect |
|---|---|---|---|
| `bought` | An item is created. | One event per item, dated today. Editable via row menu "Edit bought date…". | — |
| `discarded` | User picks "Got rid of…" on one or more items. A dialog asks for the date and an optional note (sold, donated, binned…). | One event containing all selected items. | Status → `discarded`. Items are hidden unless "Show discarded" is on, and excluded from the title count. |

### 6.2 Manual events (logged from the main table)
Every other event is logged by hand from the table:

1. Select one or more rows.
2. Open **[Log event ▾]** in the toolbar, or right-click → **Log event ▸**, and pick a type.
3. Fill in the **Log event dialog**:
   - **Type** (prefilled).
   - **Date** (default today, editable). This is how past events are back-filled.
   - **Items**: the selection, shown as a list you can remove entries from.
   - **Location**: only for `stored`. Free text, with autocomplete from previously used locations.
   - **Note**: optional.
4. Saving creates **one** event linked to all listed items.

Only event types that apply to **every** selected item are offered (see scope in §6.3). If none apply, the menu shows "No event types for this selection".

Built-in manual types:
- `worn`: no side effects. It's unscoped by default; scope it to your clothing/accessories category in Manage event types.
- `stored`: requires a location. Each item's status → `stored` and its storage location is set. The row menu's "Take out of storage" sets status back to `owned` and clears the location.

### 6.3 Custom event types
- **Manage event types…** dialog, opened from the Calendar sidebar and from the bottom of the Log event menu. It lists each type with its name, scope and colour.
- **Add**: name, optional scope category, colour.
  - A scoped type applies to that category and all its descendants; an empty scope means all items.
  - The colour is used for the type's line in calendar cells.
- **Edit**: rename, change scope, change colour.
- **Delete**: if the type has events, confirm "Delete N events too?".
- Restrictions on built-ins:
  - `bought` and `discarded` can be renamed and recoloured, but not scoped or deleted.
  - `stored` can be renamed, scoped and recoloured, but not deleted, because it drives status.
  - `worn` behaves like a custom type and can be deleted.
- Custom types have no side effects; they're just dated records.

### 6.4 Deletion rules
- Deleting an item removes it from all its events; any event left with no items is deleted.
- Deleting a `stored` or `discarded` event offers to restore the affected items to `owned`.
- Deleting a category that scopes an event type makes that type unscoped.

---

## 7. Data model (SQLite)

Database file: `data/categories.db`. Enable `PRAGMA foreign_keys = ON` on every connection.

```sql
CREATE TABLE category (
    id         INTEGER PRIMARY KEY,
    name       TEXT    NOT NULL,
    parent_id  INTEGER REFERENCES category(id) ON DELETE RESTRICT,
    position   INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE item (
    id               INTEGER PRIMARY KEY,
    name             TEXT    NOT NULL,
    category_id      INTEGER NOT NULL REFERENCES category(id) ON DELETE RESTRICT,
    image_path       TEXT,
    status           TEXT    NOT NULL DEFAULT 'owned'
                     CHECK (status IN ('owned', 'stored', 'discarded')),
    storage_location TEXT,
    created_at       TEXT    NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE field (
    id           INTEGER PRIMARY KEY,
    category_id  INTEGER NOT NULL REFERENCES category(id) ON DELETE CASCADE,
    name         TEXT    NOT NULL,
    position     INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE field_value (
    item_id   INTEGER NOT NULL REFERENCES item(id)  ON DELETE CASCADE,
    field_id  INTEGER NOT NULL REFERENCES field(id) ON DELETE CASCADE,
    value     TEXT,
    PRIMARY KEY (item_id, field_id)
);

CREATE TABLE event_type (
    id           INTEGER PRIMARY KEY,
    key          TEXT    UNIQUE,       -- 'bought' | 'discarded' | 'stored' | 'worn' for built-ins, NULL for custom
    name         TEXT    NOT NULL,     -- user-editable display name
    category_id  INTEGER REFERENCES category(id) ON DELETE SET NULL,  -- scope; NULL = all items
    colour       TEXT    NOT NULL DEFAULT '#888888',
    position     INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE event (
    id        INTEGER PRIMARY KEY,
    type_id   INTEGER NOT NULL REFERENCES event_type(id) ON DELETE CASCADE,
    date      TEXT    NOT NULL,        -- ISO 'YYYY-MM-DD'
    location  TEXT,
    note      TEXT
);

CREATE TABLE event_item (
    event_id  INTEGER NOT NULL REFERENCES event(id) ON DELETE CASCADE,
    item_id   INTEGER NOT NULL REFERENCES item(id)  ON DELETE CASCADE,
    PRIMARY KEY (event_id, item_id)
);

CREATE INDEX idx_event_date ON event(date);
CREATE INDEX idx_event_item_item ON event_item(item_id);
CREATE INDEX idx_item_category ON item(category_id);
```

Key queries:
- **Descendant categories:** recursive CTE starting from the selected category id.
- **Items in a category view:** items whose `category_id` is in the descendant set (and `status != 'discarded'` unless showing discarded).
- **Visible columns:** fields on the selected category and its ancestors (recursive CTE upward) **plus** fields on its descendants, ordered ancestor-first then by `position`.
- **Calendar counts:** `SELECT e.date, e.type_id, COUNT(DISTINCT e.id) FROM event e [JOIN event_item ei ON ei.event_id = e.id WHERE ei.item_id IN (category item set) AND] e.date BETWEEN ? AND ? GROUP BY e.date, e.type_id`.
- **Applicable event types for a selection:** types where `category_id IS NULL`, or `category_id` is the category or an ancestor of the category of **every** selected item.
- **Search:** items in scope where `name LIKE ?` OR `storage_location LIKE ?` OR `EXISTS (SELECT 1 FROM field_value WHERE item_id = item.id AND value LIKE ?)`, using `%query%` with `COLLATE NOCASE`.

Built-in event types are seeded on first run. Code refers to them by `key`, never by id or name, because names are user-editable.
- **Title count:** `SELECT COUNT(*) FROM item WHERE status != 'discarded'`.

---

## 8. Project structure

```
categories/
├── CLAUDE.md
├── SPEC.md
├── pyproject.toml
├── data/                     # created at runtime, git-ignored
│   ├── categories.db
│   └── images/
├── src/categories/
│   ├── __main__.py           # entry point: python -m categories
│   ├── app.py                # root window, paned layout, tab switching, title updates
│   ├── db.py                 # connection, schema/migrations, ALL SQL lives here
│   ├── models.py             # dataclasses: Category, Item, Field, Event
│   ├── services.py           # business rules (logging events on status change, etc.)
│   ├── images.py             # copy/resize/thumbnail via Pillow, PhotoImage cache
│   └── ui/
│       ├── sidebar.py        # Notebook with Categories + Calendar tabs
│       ├── category_tree.py
│       ├── table_view.py     # toolbar + item table + preview panel
│       ├── editable_tree.py  # Treeview subclass with inline cell/heading editing
│       ├── calendar_view.py  # month grid
│       ├── day_popup.py
│       └── dialogs.py        # log event, got rid of, manage event types, move to category
└── tests/
    ├── test_db.py
    └── test_services.py
```

---

## 9. Build phases

Each phase should leave the app runnable.

1. **Skeleton** — window, PanedWindow, Notebook with two tabs, empty right pane that swaps with the tab. DB schema created on first run.
2. **Categories** — tree with All items root, add/rename/delete, persistence.
3. **Basic table** — Name column only, add via [New] and ＋ row, inline rename, delete, title count.
4. **Nesting** — parent categories show descendants' items.
5. **Custom columns** — add/rename/delete/reorder fields, inheritance and union rules, inline cell editing.
6. **Sorting.**
7. **Search** — live filter, scope dropdown, Category column in All items scope.
8. **Events & status** — automatic `bought` / `discarded`, Log event dialog with multi-item events, `worn`, `stored` + Take out of storage, Show discarded.
9. **Custom event types** — Manage event types dialog, scopes, colours, applicable-type filtering in Log event.
10. **Calendar view** — month grid, event counts, navigation, filters.
11. **Day popup** — event/item tree, type filter, edit/delete events, Go to item.
12. **Images** — set/remove, thumbnails, preview panel.
13. **Polish** — remember UI state, keyboard shortcuts (Ctrl+N new, Del delete, F2 rename, Ctrl+F search), drag-and-drop.
14. **(Optional) Rotate table.**

---

## 10. Out of scope (for now)

- Per-cell images
- Multiple images per item
- Typed fields (dates, numbers, dropdowns) — all field values are text
- Sync / cloud / multi-user
- Storage locations as structured data (they're free text for now)
- Field-specific search syntax (e.g. `size:M`)
- Side effects for custom event types

## 11. Open questions

- Should "Take out of storage" log its own `retrieved` event so it shows on the calendar?
- Should items be movable between categories by drag-and-drop in v1, or is the "Move to category…" menu enough?
- When several items are added at once (e.g. a shopping haul), should their `bought` events be grouped into one event?
