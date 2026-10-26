---
sessionId: session-261025-200718-14k3
---

# Requirements

### Overview
Add **Volume** as a computed furniture property (width × depth × height) that displays in the furniture table using the application's existing length-based number formatting. Volume should be sortable (ascending/descending), toggleable as a display column, and available in both the main menu (Furniture > Sort by / Display column) and the furniture table context menu.

### Scope

**In Scope:**
- Add `VOLUME` to `SortableProperty` enum in `HomePieceOfFurniture.java` with a comparator
- Add Volume column rendering in `FurnitureTable.java`
- Add `SORT_HOME_FURNITURE_BY_VOLUME` and `DISPLAY_HOME_FURNITURE_VOLUME` actions in `HomeView.java` and `HomePane.java`
- Add Volume to sort and display property menus in `HomePane.java`
- Include Volume in default visible columns for new homes in `Home.java`
- Add English localization strings in `package.properties`

**Out of Scope:**
- Volume in the furniture editing panel (previously discussed and rejected)
- Localized strings for non-English languages (only English as requested)
- Backward compatibility for serialized homes (VOLUME is a new computed property)

### Functional Requirements
1. **Default visibility**: Volume column is shown by default for new homes alongside NAME, WIDTH, DEPTH, HEIGHT, and VISIBLE.
2. **Computed value**: Volume = width × depth × height, calculated on the fly for both catalog and home furniture.
3. **Formatting**: Uses the existing `SizeRenderer` pattern with `preferences.getLengthUnit().getFormat().format()` for number formatting.
4. **Sorting**: Ascending and descending sort by volume via menu and context menu.
5. **Menu integration**: Volume appears in Furniture > Sort by (radio button) and Furniture > Display column (checkbox).
6. **Context menu**: Same sort and display options available in the furniture table's right-click context menu.

# Technical Design

### Current Implementation
The existing pattern for sortable/displayable furniture properties flows through these layers:

1. **Model** (`HomePieceOfFurniture.java`): `SortableProperty` enum drives everything. Each value gets a comparator in `SORTABLE_PROPERTY_COMPARATORS` static block.
2. **Table** (`FurnitureTable.java`): `FurnitureTableColumnModel` iterates `SortableProperty.values()` to create columns. Switch statements in `getColumnName()`, `getColumnPreferredWidth()`, and `getColumnRenderer()` handle each property.
3. **Actions** (`HomeView.java`): `ActionType` enum defines `SORT_HOME_FURNITURE_BY_*` and `DISPLAY_HOME_FURNITURE_*` entries.
4. **Menus** (`HomePane.java`): Actions are created in the constructor, then wired into `createFurnitureSortMenu()` and `createFurnitureDisplayPropertyMenu()`.
5. **Defaults** (`Home.java`): New homes initialize `furnitureVisibleProperties` with default columns (NAME, WIDTH, DEPTH, HEIGHT, VISIBLE).
6. **Localization** (`package.properties`): Strings for column headers, menu names, descriptions, and mnemonics.

### Key Decisions
- **Volume is computed, not stored**: A `getVolume()` method in `HomePieceOfFurniture` returns `width * depth * height` on the fly, avoiding synchronization issues.
- **Shared SizeRenderer pattern**: VOLUME uses the same `getSizeRenderer()` switch as WIDTH/DEPTH/HEIGHT (line 1319-1382), with a new `case VOLUME` that calls `getVolume()` — no custom anonymous renderer needed.
- **Not backward compatible**: VOLUME returns `false` in `isFurnitureSortedPropertyBackwardCompatible()` so old home files don't break.
- **No editing panel**: Volume is read-only; no changes to `HomeFurnitureController` or `HomeFurniturePanel`.

### Proposed Changes

#### 1. `HomePieceOfFurniture.java` (model)
- Add `VOLUME` to `SortableProperty` enum (after `HEIGHT`) — **already done**
- Add comparator in static block that computes `width * depth * height` for each piece — **already done**
- Add `public float getVolume()` method returning `this.width * this.depth * this.height` (near `getWidth()`, `getDepth()`, `getHeight()`)

#### 2. `FurnitureTable.java` (table column)
- Add `case VOLUME` in `getColumnName()` → returns localized `volumeColumn` string
- Add `case VOLUME` in `getColumnPreferredWidth()` → returns `70` (wider than dimensions for larger numbers)
- Add `case VOLUME` in `getSizeRenderer()` switch (lines 1319-1382) returning an anonymous `SizeRenderer` that calls `((HomePieceOfFurniture)value).getVolume()` — same pattern as WIDTH/DEPTH/HEIGHT
- `getColumnRenderer()` already calls `getSizeRenderer(VOLUME, preferences)` at line 1222-1223 — no change needed there
- Add `case VOLUME` in `exportRowToCsv()` (lines 888-955) → `writer.write(sizeFormat.format(copiedPiece.getVolume()))` — same pattern as WIDTH/DEPTH/HEIGHT

#### 3. `HomeView.java` (action types)
- Add `SORT_HOME_FURNITURE_BY_VOLUME` to `ActionType` enum (after `SORT_HOME_FURNITURE_BY_HEIGHT`)
- Add `DISPLAY_HOME_FURNITURE_VOLUME` to `ActionType` enum (after `DISPLAY_HOME_FURNITURE_HEIGHT`)

#### 4. `HomePane.java` (menus)
- Create sort action: `createAction(ActionType.SORT_HOME_FURNITURE_BY_VOLUME, preferences, furnitureController, "toggleFurnitureSort", HomePieceOfFurniture.SortableProperty.VOLUME)`
- Create display action: `createAction(ActionType.DISPLAY_HOME_FURNITURE_VOLUME, preferences, furnitureController, "toggleFurnitureVisibleProperty", HomePieceOfFurniture.SortableProperty.VOLUME)`
- Add `addActionToMap()` call in `createFurnitureSortMenu()` after HEIGHT
- Add `addActionToMap()` call in `createFurnitureDisplayPropertyMenu()` after HEIGHT

#### 5. `Home.java` (default visible columns)
- Add `HomePieceOfFurniture.SortableProperty.VOLUME` to the `furnitureVisibleProperties` list in the constructor (line 156-161) and in the `init()` method (line 316-324)
- VOLUME returns `false` in `isFurnitureSortedPropertyBackwardCompatible()` (default case already handles this)

#### 6. `package.properties` (English localization)
- Add `FurnitureTable.volumeColumn=Volume`
- Add `HomePane.SORT_HOME_FURNITURE_BY_VOLUME.Name=Volume` and associated ShortDescription/MnemonicKey
- Add `HomePane.DISPLAY_HOME_FURNITURE_VOLUME.Name=Volume` and associated ShortDescription/MnemonicKey

### Architecture Diagram

``` mermaid
graph LR
    A[HomePieceOfFurniture.SortableProperty.VOLUME] --> B[Comparator: w*d*h]
    A --> C[FurnitureTable column model]
    C --> D[SizeRenderer: format volume]
    A --> E[HomePane sort menu]
    A --> F[HomePane display menu]
    E --> G[FurnitureController.toggleFurnitureSort]
    F --> H[FurnitureController.toggleFurnitureVisibleProperty]
    G --> I[Home.furnitureSortedProperty]
    H --> J[Home.furnitureVisibleProperties]
    J --> C
```

### File Structure
| File | Change |
|------|--------|
| `model/HomePieceOfFurniture.java` | Add `getVolume()` method (enum + comparator already done) |
| `model/Home.java` | Add VOLUME to default visible properties |
| `swing/FurnitureTable.java` | Add VOLUME case in getColumnName, getColumnPreferredWidth, getSizeRenderer switch, and exportRowToCsv |
| `viewcontroller/HomeView.java` | Add 2 ActionType enum entries |
| `swing/HomePane.java` | Create 2 actions + add to 2 menus |
| `swing/package.properties` | Add 7 localization keys |

### Risks
- **Column width**: Volume values can be large (e.g., 100×100×200 = 2,000,000). A preferred width of 70px (same as PRICE) should accommodate formatted numbers.
- **Serialization**: Old home files won't have VOLUME in their visible properties list, which is fine — they'll load without the column until the user enables it.
- **Null dimensions**: If any of width/depth/height is 0 or missing, volume will be 0. The SizeRenderer already handles null values by displaying nothing.

# Delivery Steps

###   Step 1: Add `getVolume()` to model and default visible columns
Add a computed `getVolume()` method to `HomePieceOfFurniture` and include VOLUME in default visible columns.
- Add `public float getVolume()` method in `HomePieceOfFurniture.java` returning `this.width * this.depth * this.height` (near `getWidth()`, `getDepth()`, `getHeight()`)
- The `VOLUME` enum value and comparator are already present
- Add `VOLUME` to default `furnitureVisibleProperties` in `Home.java` constructor and `init()` method so it shows by default for new homes

###   Step 2: Add VOLUME column to FurnitureTable
Wire VOLUME into the furniture table using the shared `getSizeRenderer()` pattern, same as WIDTH/DEPTH/HEIGHT.
- Add `case VOLUME` in `getColumnName()` returning `preferences.getLocalizedString(FurnitureTable.class, "volumeColumn")`
- Add `case VOLUME` in `getColumnPreferredWidth()` returning `70`
- Add `case VOLUME` in the `getSizeRenderer()` switch (lines 1319-1382) returning an anonymous `SizeRenderer` that calls `((HomePieceOfFurniture)value).getVolume()` — identical pattern to WIDTH/DEPTH/HEIGHT
- `getColumnRenderer()` already routes VOLUME to `getSizeRenderer()` at line 1222-1223, no change needed
- Add `case VOLUME` in `exportRowToCsv()` (lines 888-955) calling `writer.write(sizeFormat.format(copiedPiece.getVolume()))` — same pattern as WIDTH/DEPTH/HEIGHT

###   Step 3: Add VOLUME actions and menu entries
Register VOLUME sort and display actions, then wire them into the sort and display property menus.
- Add `SORT_HOME_FURNITURE_BY_VOLUME` and `DISPLAY_HOME_FURNITURE_VOLUME` to `ActionType` enum in `HomeView.java`
- Create both actions in `HomePane.java` constructor using `createAction()` with `toggleFurnitureSort` and `toggleFurnitureVisibleProperty` respectively
- Add `addActionToMap()` calls in `createFurnitureSortMenu()` and `createFurnitureDisplayPropertyMenu()` after HEIGHT entries

###   Step 4: Add English localization strings
Add all English labels, tooltips, and mnemonics for Volume controls in `package.properties`.
- Add `FurnitureTable.volumeColumn=Volume`
- Add `HomePane.SORT_HOME_FURNITURE_BY_VOLUME.Name=Volume`, `.ShortDescription=Sort furniture by volume`, `.MnemonicKey=O` (or another non-conflicting key)
- Add `HomePane.DISPLAY_HOME_FURNITURE_VOLUME.Name=Volume`, `.ShortDescription=Display volume column`, `.MnemonicKey=V` (or another non-conflicting key)