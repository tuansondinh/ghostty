# Cursor Jumping Bug Analysis - Input Display Overlay Issue

## Executive Summary

The cursor jumping bug stems from a **coordinate system mismatch** between `cursor.viewport.y` (which is in terminal-viewport coordinates) and the screen-space rendering coordinates used in `rebuildInputDisplayRow()`. When an input display overlay is placed at row 0 and `main_row_offset > 0` (due to sticky scroll), the cursor glyph and its rendering state get corrupted.

---

## Question 1: Is cursor.viewport.y in terminal-viewport coordinates or screen coordinates?

**ANSWER: `cursor.viewport.y` is in TERMINAL-VIEWPORT coordinates (0 = top of terminal content view).**

**Evidence:**
- File: `/Users/sonwork/Workspace/ghostty/src/terminal/render.zig`, lines 438-439
- In the `update()` function:
```zig
self.cursor.viewport = .{
    .y = y,  // <-- 'y' is the loop variable in rowIterator, which is viewport-relative
    .x = s.cursor.x,
    ...
};
```

The variable `y` is set from the `rowIterator` which iterates through the viewport starting at y=0. This is viewport space, not screen space. The cursor viewport coordinate system is:
- y=0 means the top row of the currently visible viewport
- It does NOT account for screen-level offsets like `main_row_offset`

---

## Question 2: rebuildRow() coordinate mismatch check

**ANSWER: YES, there is a critical coordinate system mismatch.**

**The Problem:**

In `rebuildRow()` at line 2884:
```zig
if (vp.y != y) break :cursor_x null;
```

Where:
- `vp.y` = `cursor.viewport.y` (viewport-relative coordinate, 0-based)
- `y` = the parameter passed to `rebuildRow()` (screen-relative coordinate after `main_row_offset` adjustment)

**When main_row_offset > 0, these are INCOMPATIBLE:**

Example scenario:
1. Sticky scroll enabled: `sticky.rows = 3`, so `main_row_offset = 3`
2. Cursor is at terminal viewport row 5
3. This cursor is rendered at screen row 8 (5 + 3)

In the main loop (line 2460):
```zig
const render_y: terminal.size.CellCountInt = y + main_row_offset;
```

So `rebuildRow()` is called with `y=8` (screen-adjusted).

But `cursor.viewport.y = 5` (viewport-relative, unchanged).

The comparison `if (vp.y != y)` becomes `if (5 != 8)` which is TRUE, so the cursor shaping is disabled with `break :cursor_x null`.

**This causes the cursor to NOT get font-shaped correctly on the row where it actually appears.**

---

## Question 3: Does setCursor at cursor_vp.y followed by cells.clear(0) erase the cursor?

**ANSWER: YES, this is the core corruption issue.**

**The Flow:**

1. **Line 2623-2627: `addCursor()` is called**
   - This calls `self.cells.setCursor()` at line 3515-3518
   - Sets cursor glyph at `grid_pos = .{ x, cursor_vp.y }`
   - If `cursor_vp.y == 0`, the cursor glyph is placed at row 0

2. **Line 2543-2553: `rebuildInputDisplayRow()` is called for the overlay**
   - When input display is at position `.top`, `input_y = 0`
   - Inside `rebuildInputDisplayRow()` at line 2814:
     ```zig
     self.cells.clear(y);  // y = 0
     ```
   - This clears row 0, **erasing the cursor glyph if it was placed there**

**The Critical Issue:**
If the cursor happens to be at row 0 of the viewport (rare but possible), the sequence is:
1. Cursor placed at screen position (x, 0) via `setCursor()`
2. Input display overlay calls `self.cells.clear(0)`
3. Cursor glyph is destroyed

---

## Question 4: Ordering of cursor block vs. rebuildInputDisplayRow

**ANSWER: YES, the ordering creates a data race that corrupts cursor state.**

**Current Order (lines 2543-2627):**
```zig
// 1. Render the input display row FIRST (line 2543-2553)
if (input_display) |overlay| {
    const input_y = switch (overlay.position) {
        .top => 0,  // <-- Places overlay at row 0
        ...
    };
    self.rebuildInputDisplayRow(input_y, overlay);  // Calls cells.clear(0)
}

// 2. Render the cursor AFTER (line 2556-2627)
cursor: {
    const cursor_vp = state.cursor.viewport orelse break :cursor;
    ...
    self.addCursor(...);  // Places cursor via setCursor()
}
```

**Problem:** The overlay is rendered BEFORE the cursor. If:
- Input display is at `.top` (row 0)
- Cursor is also at row 0
- The sequence is: overlay clears row 0 → cursor tries to use row 0 → corrupted state

However, looking more carefully at line 2814 in `rebuildInputDisplayRow()`:
```zig
self.cells.clear(y);
try self.rebuildRow(...);
```

The cursor glyph is set via `setCursor()` in the `cursor` block. The `rebuildRow()` call in `rebuildInputDisplayRow()` processes the overlay text. But `setCursor()` is called AFTER `rebuildInputDisplayRow()` returns, so actually the cursor should be placed AFTER the overlay.

**BUT** the issue is that `cells.clear(0)` happens at line 2814, which might also clear cursor state if cursors are stored in the same buffer.

---

## Question 5: Row dirty flags and row 0 inconsistency

**ANSWER: YES, there is potential inconsistency in how row 0 is marked dirty.**

**Analysis:**

In the main render loop (lines 2451-2492):
```zig
for (
    0..,
    row_raws[0..row_len],
    row_cells[0..row_len],
    row_dirty[0..row_len],
    ...
) |y_usize, row, *cells, *dirty, ...| {
    const y: terminal.size.CellCountInt = @intCast(y_usize);
    const render_y: terminal.size.CellCountInt = y + main_row_offset;

    // If this row is shifted outside our viewport, skip it.
    if (render_y >= self.cells.size.rows) continue;

    if (!rebuild) {
        if (!dirty.*) continue;
        self.cells.clear(render_y);
    }
    ...
    self.rebuildRow(render_y, ...);
}
```

Then separately (lines 2543-2553):
```zig
if (input_display) |overlay| {
    const input_y = switch (overlay.position) {
        .top => 0,
        .bottom => state.rows -| 1,
    };
    self.rebuildInputDisplayRow(input_y, overlay);
}
```

**The Inconsistency:**
- When `overlay.position == .top`, the overlay always renders to screen row 0
- But the main loop may or may not process row 0 depending on:
  - Whether main row 0 is marked dirty
  - Whether `rebuild` flag is set
  - The value of `main_row_offset`

If `main_row_offset > 0` and row 0 of the main content is not dirty:
1. Row 0 is SKIPPED in the main loop
2. But input display STILL renders to row 0
3. If the cursor was placed at screen row 0 from sticky content, it's corrupted without being re-rendered
4. Row 0's dirty flag was never cleared by the main loop (because it skipped the row)

---

## Question 6: preedit_range.y coordinate system mismatch

**ANSWER: YES, this creates a subtle bug when called from rebuildInputDisplayRow.**

**The Problem:**

At line 2434:
```zig
.y = @intCast(cursor_vp.y),
```

`cursor_vp.y` is viewport-relative. This is stored in `preedit_range.y`.

Later in `rebuildRow()` at line 2903:
```zig
if (range.y != y) break :preedit;
```

Where `y` is the screen-space row parameter.

**When rebuildInputDisplayRow calls rebuildRow:**
- Called at line 2815 with `y = input_y` (which is 0 for `.top` position)
- But `preedit_range = null` (line 2819), so this is safe

**However, in the main loop:**
- `rebuildRow()` is called with screen-space `y` values
- `preedit_range.y` is viewport-space
- When `main_row_offset > 0`, these don't match
- Preedit processing will be skipped even when it should apply

Example:
- Cursor at viewport row 3, screen row 6 (main_row_offset = 3)
- `preedit_range.y = 3`
- `rebuildRow()` called with `y = 6`
- Check: `if (3 != 6)` → true, so preedit is skipped
- **Preedit text will not be rendered on the correct screen row**

---

## Summary of Root Causes

| Issue | Location | Problem | Impact |
|-------|----------|---------|--------|
| Viewport vs Screen Coords | render.zig:438, generic.zig:2434, 2884 | `cursor.viewport.y` is viewport-relative; used in screen-space comparisons | Cursor shaping disabled; preedit not rendered |
| Overlay Row Clearing | generic.zig:2814 | `cells.clear(0)` erases whatever was at screen row 0 | Cursor glyph corrupted if at row 0 |
| Ordering Issue | generic.zig:2543-2627 | Input display rendered before cursor placement | Potential state corruption |
| Dirty Flag Inconsistency | generic.zig:2451-2553 | Overlay row might not be marked dirty in main loop | Row not re-rendered; stale state persists |
| Preedit Coordinate Mismatch | generic.zig:2434, 2903 | `preedit_range.y` is viewport-space; compared to screen-space `y` | Preedit text misplaced with sticky scroll |

---

## Recommended Fixes

1. **Convert cursor.viewport.y to screen coordinates** before comparisons in `rebuildRow()`
   - After `render_y = y + main_row_offset` is computed, use that for comparisons
   - Or: adjust `preedit_range.y` by `main_row_offset` before use

2. **Protect cursor glyph from being erased**
   - Don't call `cells.clear(0)` unconditionally in `rebuildInputDisplayRow()`
   - Or: set cursor BEFORE calling `rebuildInputDisplayRow()` and protect it

3. **Mark overlay row as dirty** after rendering it
   - Ensure row 0 (when overlay is at `.top`) is properly tracked
   - Or: always rebuild overlay row to ensure consistency

4. **Reorder operations**
   - Consider rendering sticky rows, then main content, then cursor, then input display
   - This ensures no overlay clears cursor state

5. **Add explicit coordinate system documentation**
   - Mark which coordinates are viewport-relative vs screen-relative
   - Add assertions to catch future mismatches
