# Plan: Support for MongoDB Collection Types

## Context

MongoDB supports three collection types: standard `collection`, `timeseries` (since MongoDB 5.0),
and `view`. The app currently treats all collections identically — no type info is fetched, stored,
or displayed.

---

## Scope of changes

### 1. `hibbert/services/mongo_service.py`

**Method:** `list_collections()` (line 127)

Replace `list_collection_names()` with `list_collections()` from PyMongo, which returns full
metadata per collection including the `type` field.

```python
# Before
return self._client[db_name].list_collection_names()

# After
result = self._client[db_name].list_collections(session=None)
return [(c["name"], c.get("type", "collection")) for c in result]
```

Return type changes from `List[str]` to `List[tuple[str, str]]`.

**Note:** Views don't support `collStats` — the worker must skip stats for `type == "view"`.

---

### 2. `hibbert/ui/workers.py`

**Class:** `CollectionListWorker` (line 127)

- Update `run()` to unpack `(col_name, col_type)` from the new `list_collections()` return.
- For views (`col_type == "view"`), skip both `count_documents()` and `get_collection_stats()` —
  use `None` for count and stats values instead. Views have no physical storage so neither call
  is valid.
- Append `col_type` to the result tuple:
  `(col_name, count, size_str, total_size, col_type)`
- Update the `loaded` signal docstring accordingly.

---

### 3. `hibbert/ui/connection_tree.py`

**Method:** `_on_collections_loaded()` (line 469)

- Unpack the 5-element tuple `(col_name, count, size_str, total_size, col_type)`.
- For views, `count` and `size_str` will be `None` — display `—` in those positions in the tree label.
- Assign icon and label suffix based on `col_type`:

| Type | Icon | Label suffix |
|------|------|--------------|
| `collection` | `mdi.table-large` | _(none)_ |
| `timeseries` | `mdi.chart-timeline-variant` | `[TS]` |
| `view` | `mdi.eye-outline` | `[view]` |

- Use themed colors:
  - `tree_collection_ts_icon` — icon color for timeseries
  - `tree_collection_view_icon` — icon color for views
  - (standard collections keep the existing `icon` color)

- Store `col_type` in `ROLE_DATA`:
  `(conn_id, db_name, col_name, col_type)`

---

### 4. Theme files — `hibbert/themes/ui/*.json`

Add two new color keys to **all 9 theme files**:

```json
"tree_collection_ts_icon":   "<color>",
"tree_collection_view_icon": "<color>"
```

Suggested values per theme family:

| Theme | `tree_collection_ts_icon` | `tree_collection_view_icon` |
|-------|--------------------------|----------------------------|
| dark variants | `#4fc3f7` (light blue) | `#a5d6a7` (light green) |
| light variants | `#0277bd` (dark blue) | `#2e7d32` (dark green) |
| norton | match existing accent | match existing accent |

These must also be read in `reload_theme()` of `connection_tree.py`.

---

### 5. `hibbert/mcp_server.py`

**Function:** `list_collections()` (line 89)

Update to return collection type alongside name, so AI assistants have that info:

```python
# Before: returns a plain list of names
return json.dumps(service.list_collections(database))

# After: returns list of {name, type} objects
collections = service.list_collections(database)
return json.dumps([{"name": name, "type": col_type} for name, col_type in collections])
```

---

## Documentation updates

### README.md

- In the **Features** section, update the collection browser bullet:
  > Database & collection tree browser with collection type indicators (standard, timeseries, view)

- In the **MCP Server — Available tools** table, update the `list_collections` description:
  > List collections in a database (includes collection type)

### CLAUDE.md

- No structural changes needed.
- Consider adding a note under **Cosas a tener en cuenta** about skipping `collStats`
  for views, since it's a non-obvious MongoDB constraint that could trip up future devs.

---

## Order of implementation

1. `mongo_service.py` — change the data source (everything else depends on this)
2. `workers.py` — adapt the worker to the new shape
3. `connection_tree.py` — show the type in the UI
4. Theme files — add the two new color keys
5. `mcp_server.py` — expose type via MCP
6. README.md — update features and MCP tools table
7. CLAUDE.md — add note about views and collStats

---

### 6. `hibbert/ui/query_tab.py`

Add a `forced_read_only` parameter to `__init__` (default `False`). When `True`, `set_read_only()`
ignores any attempt to set read-only to `False`, keeping the tab permanently locked.

```python
def __init__(self, ..., read_only=False, forced_read_only=False):
    ...
    self._forced_read_only = forced_read_only
    ...

def set_read_only(self, value: bool):
    if self._forced_read_only and not value:
        return  # views are always read-only, ignore connection-level toggle
    ...
```

This ensures that when the user toggles read-only at the connection level, view tabs remain locked
regardless.

---

### 7. `hibbert/ui/main_window.py`

When opening a collection tab (lines ~507 and ~665), check `col_type` from `ROLE_DATA` and pass
`forced_read_only=True` when the type is `"view"`:

```python
is_view = col_type == "view"
tab = QueryTab(..., read_only=is_read_only or is_view, forced_read_only=is_view)
```

---

## Order of implementation

1. `mongo_service.py` — change the data source (everything else depends on this)
2. `workers.py` — adapt the worker to the new shape
3. `connection_tree.py` — show the type in the UI
4. Theme files — add the two new color keys
5. `mcp_server.py` — expose type via MCP
6. `query_tab.py` — add `forced_read_only` support
7. `main_window.py` — pass `forced_read_only=True` when opening views
8. README.md — update features and MCP tools table
9. CLAUDE.md — add note about views and collStats

---

## Out of scope

- Showing timeseries-specific metadata (timeField, granularity, etc.) in a detail panel.
- Filtering/sorting the tree by collection type.

These can be addressed in follow-up tasks.
