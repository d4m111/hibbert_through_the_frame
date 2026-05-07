# Double-click selection rules — Query Editor

Rules applied in priority order when the user double-clicks on a character.
Implemented in `QueryEditor.mouseDoubleClickEvent` and mirrored in
`JsonReadOnlyView.mouseDoubleClickEvent` (both in `hibbert/ui/query_editor.py`).

---

## 1. Quoted string (`_find_enclosing_quotes`)
- Selects the **content** inside the nearest enclosing `"..."` pair.
- The quotes themselves are **not** included in the selection.
- Handles escaped quotes (`\"`).
- Condition: `quote_start < pos <= closing_quote` (clicking the closing quote also triggers it).

## 2. Number (`_find_number_at`)
- Triggers when the cursor touches a digit or `.` (checks left side first, then right).
- Expands left and right over digits and `.`.
- Includes a leading `-` only when it looks like a negative sign, not a subtraction
  (checks the character before the `-`).
- Supports scientific notation: `e`, `E`, `e+`, `e-` followed by digits.
- Rejected if the token contains no digit (e.g. a bare `.` or `-`).

## 3. Array content (`_find_enclosing_pair` with `[` / `]`)
- Selects the **content** inside the innermost `[...]` pair enclosing the cursor.
- Brackets themselves are **not** included.
- Characters inside strings are ignored when searching for bracket pairs.
- When multiple nested pairs enclose the cursor, the **innermost** one wins.

## 4. Object content (`_find_enclosing_pair` with `{` / `}`)
- Same behaviour as rule 3 but for `{...}` pairs.

## 5. Key/value between brace and colon (`_find_between_brace_and_colon`)
- Scans **left** from the cursor until hitting `{`, `:`, or `}`.
- Scans **right** from the cursor until hitting `{`, `:`, or `}`.
- Selection is valid only for these two boundary pairs (either order):
  - `{` and `:` — targets an unquoted **key**: `{ field: ...`
  - `:` and `}` — targets an unquoted **value**: `... : value }`
- The two delimiter characters are **not** included.
- Leading and trailing spaces are stripped from the result.

## Fallback
If none of the above rules match, Qt's default word-selection behaviour is used.
