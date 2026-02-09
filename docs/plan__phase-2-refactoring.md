# Phase 2 Refactoring Plan: B10-B22

**Document:** plan__phase-2-refactoring  
**Version:** v1.0  
**Date:** 2026-02-07   
**Author:** Architect  
**Status:** Ready for Dev  
**Source brief:** `modules/Issues-FS/to_refactor-in/v0_2_34__brief-006a__gitgraph-issues__phase-2-backend__revised.md`

---

## 1. Executive Summary

This plan covers the refactoring of implementation files from `modules/Issues-FS/to_refactor-in/` into the `issues_fs/` package. The provided code was originally written against the `mgraph_ai_ui_html_transformation_workbench` namespace and must be rewritten to use `issues_fs.*` import paths. The work spans 13 tasks (B10-B22) covering recursive node discovery, legacy cleanup, MGraphDB integration, and hyphenated label support.

---

## 2. Task-by-Task Analysis

### B10: Recursive Node Discovery

**Objective:** Update `Graph__Repository` to find ALL `issue.json` files recursively, including nested `issues/` subfolders.

**Provided code:** Yes
**Target files:**
- NEW: `issues_fs/schemas/graph/Schema__Node__Info.py`
- MODIFY: `issues_fs/issues/graph_services/Graph__Repository.py`

**Changes required:**
1. Create `Schema__Node__Info` schema class (pure data container with `label`, `path`, `node_type` fields).
2. Add to `Graph__Repository`:
   - Class attribute `SKIP_LABELS` (set of system folder names)
   - Method `nodes_list_all(root_path)` -- recursive file scan returning `List[Schema__Node__Info]`
   - Method `is_path_under_root(file_path, root_path)` -- path containment check
   - Method `extract_node_type_from_file(file_path)` -- reads `node_type` from JSON
3. Replace existing `nodes_list_labels()` to delegate to `nodes_list_all()`.

**Import rewrites:**
- `mgraph_ai_ui_html_transformation_workbench.schemas.graph.Schema__Node__Info` --> `issues_fs.schemas.graph.Schema__Node__Info`
- `mgraph_ai_ui_html_transformation_workbench.schemas.graph.Safe_Str__Graph_Types` --> `issues_fs.schemas.graph.Safe_Str__Graph_Types`

**New import required in Graph__Repository:**
- `from osbot_utils.type_safe.primitives.domains.files.safe_str.Safe_Str__File__Path import Safe_Str__File__Path`

**Diff vs current:** Current `nodes_list_labels()` searches only `data/{type}/{label}/` and checks for both `issue.json` and `node.json`. The new version uses `nodes_list_all()` for recursive discovery across all paths and only looks for `issue.json`.

---

### B11: Node Loading by Path

**Objective:** Add methods to load a node by its full folder path rather than by type+label.

**Provided code:** Yes
**Target files:**
- MODIFY: `issues_fs/issues/graph_services/Graph__Repository.py`
- MODIFY: `issues_fs/issues/graph_services/Node__Service.py`

**Changes required:**
1. Add to `Graph__Repository`:
   - Method `node_load_by_path(folder_path)` -- loads `issue.json` from an explicit path
   - Method `node_find_path_by_label(label)` -- searches all paths for a given label
   - Method `node_load_by_label(label)` -- convenience wrapper combining the above two
2. Add to `Node__Service`:
   - Method `get_node_by_path(folder_path)` -- returns `Schema__Node__Response`

**New schema required:** `Schema__Node__Response` does not currently exist in `issues_fs/schemas/graph/`. It is referenced in both `Node__Service.py` and `Routes__Nodes.py` from the provided code. The Dev agent must create this schema or verify it already exists in the parent workbench schemas. Based on usage, it contains `success: bool`, `node: Schema__Node = None`, `message: str = ''`.

**Depends on:** B10 (uses `nodes_list_all` path infrastructure).

---

### B12: Remove node.json Write (Delete on Save)

**Objective:** When saving a node, delete any existing legacy `node.json` file.

**Provided code:** Yes
**Target files:**
- MODIFY: `issues_fs/issues/graph_services/Graph__Repository.py`

**Changes required:**
1. Update `node_save()` to call `delete_legacy_node_json()` after successful save.
2. Add method `delete_legacy_node_json(node_type, label)` -- deletes `node.json` if it exists.

**Diff vs current:** Current `node_save()` simply writes `issue.json`. The new version additionally removes the legacy `node.json` after a successful write.

**Depends on:** None (independent).

---

### B13: Remove node.json Read Fallback

**Objective:** Remove the fallback that reads `node.json` when `issue.json` is not found. Requires a migration script to be run first.

**Provided code:** Yes
**Target files:**
- MODIFY: `issues_fs/issues/graph_services/Graph__Repository.py`
- MODIFY: `issues_fs/issues/phase_1/Root__Selection__Service.py`
- MODIFY: `issues_fs/issues/phase_1/Issue__Children__Service.py`
- NEW: `issues_fs/scripts/migrate_node_to_issue_json.py`

**Changes required:**

*Graph__Repository:*
1. Update `get_issue_file_path()` -- remove the `node.json` fallback. Only check `issue.json`.

*Root__Selection__Service:*
1. Update `is_valid_root()` -- remove `node.json` check; only check `issue.json`.
2. Update `scan_for_issue_folders()` -- remove `node.json` from the path suffix check.
3. Update `load_issue_summary()` -- remove `node.json` fallback.
4. Update `count_top_level_issues()` -- remove `node.json` from the path suffix check.
5. Update `count_children_in_folder()` -- remove `node.json` from the path suffix check.
6. Remove import of `FILE_NAME__NODE_JSON` (it will no longer be needed).

*Issue__Children__Service:*
1. Update `parent_exists()` -- remove `node.json` check; only check `issue.json`.
2. Update `scan_child_folders()` -- remove `'node.json'` from the filename check.
3. Update `load_child_summary()` -- remove `node.json` fallback.

*Migration script:*
1. Create `issues_fs/scripts/migrate_node_to_issue_json.py` with the `Migration__Node_To_Issue_Json` class.

**Import rewrites in migration script:**
- `mgraph_ai_ui_html_transformation_workbench.service.issues.graph_services.Graph__Repository` --> not needed (script uses `storage_fs` directly in the refactored version)

**CRITICAL:** The migration script must be run BEFORE deploying B13 changes to production. This is an operational prerequisite, not a code dependency.

**Depends on:** B12 (progressive cleanup should happen first so saves clean up on the fly).

---

### B14: Path-Based Node Endpoint

**Objective:** Add an endpoint to load a node by hierarchical path (e.g., `Project-1/Version-1/Task-6`).

**Provided code:** Yes
**Target files:**
- MODIFY: `issues_fs/issues/graph_services/Node__Service.py`
- CROSS-REPO: `routes/Routes__Nodes.py` (belongs in Issues-FS__Service)

**Changes required:**
1. Add `root_selection_service` attribute to `Node__Service` class definition.
2. Add to `Node__Service`:
   - Method `resolve_hierarchical_path(path, current_root)` -- converts `Project-1/Version-1/Task-6` to filesystem path
   - Method `get_node_by_hierarchical_path(path)` -- resolves path and loads node
   - Method `get_node_by_path(folder_path)` -- loads by explicit path (also from B11)
   - Method `get_current_root_path()` -- extracts current root from `root_selection_service`

**Import rewrites:**
- `mgraph_ai_ui_html_transformation_workbench.service.issues.phase_1.Root__Selection__Service` --> `issues_fs.issues.phase_1.Root__Selection__Service`

**Depends on:** B10 (recursive discovery), B11 (path-based loading).

---

### B15: Config CRUD - Node Types (Update Method)

**Objective:** Add a PUT/update method for node types to the existing `Type__Service`.

**Provided code:** No (brief only, no implementation file)
**Target files:**
- MODIFY: `issues_fs/issues/graph_services/Type__Service.py`
- NEW: Schema `Schema__Node__Type__Update` (needs to be added to `issues_fs/schemas/graph/Schema__Node__Type.py` or a new file)
- CROSS-REPO: Routes update in Issues-FS__Service

**Changes required:**
1. Create `Schema__Node__Type__Update` schema class.
2. Add `update_node_type(name, updates)` method to `Type__Service`.

**Depends on:** None.

---

### B16: Config CRUD - Link Types (Update Method)

**Objective:** Add a PUT/update method for link types following the same pattern as B15.

**Provided code:** No (brief only, no implementation file)
**Target files:**
- MODIFY: `issues_fs/issues/graph_services/Type__Service.py`
- NEW: Schema `Schema__Link__Type__Update`
- CROSS-REPO: Routes update in Issues-FS__Service

**Changes required:**
1. Create `Schema__Link__Type__Update` schema class.
2. Add `update_link_type(name, updates)` method to `Type__Service`.

**Depends on:** None.

---

### B17: Root Scoping Verification

**Objective:** Ensure setting a root correctly filters all query results.

**Provided code:** Yes
**Target files:**
- MODIFY: `issues_fs/issues/graph_services/Node__Service.py`

**Changes required:**
1. Update `list_nodes()` to use `get_current_root_path()` and pass root filter to `list_nodes_for_type()`.
2. Rewrite `list_nodes_for_type()` to accept `root_path` parameter and use `nodes_list_all(root_path=root_path)` with `node_load_by_path()`.

**Diff vs current:** Current `list_nodes_for_type()` calls `nodes_list_labels(node_type)` then loads each by type+label. The new version calls `nodes_list_all(root_path=root_path)` then filters by type, using `node_load_by_path()`.

**Depends on:** B10 (recursive discovery with root filter), B14 (root_selection_service dependency).

---

### B18: MGraphDB Schema Design

**Objective:** Define MGraph schemas following the four-layer architecture (Schema + Domain layers).

**Provided code:** Yes
**Target files:**
- NEW: `issues_fs/mgraph/Schema__MGraph__Issues.py`
- NEW: `issues_fs/mgraph/MGraph__Issues__Domain.py`
- NEW: `issues_fs/mgraph/__init__.py`

**Changes required:**
1. Create `Schema__MGraph__Issue__Node` -- pure data container for issue nodes in the graph.
2. Create `Schema__MGraph__Issue__Edge` -- pure data container for edges.
3. Create `Schema__MGraph__Issues__Data` -- container holding nodes and edges dicts.
4. Create `MGraph__Issues__Domain` -- domain layer with indexes (`index_by_label`, `index_by_path`, `index_by_parent`) and query methods (`add_node`, `get_node_by_label`, `get_node_by_path`, `add_edge`, `get_children`, `get_ancestors`, `clear`, `all_nodes`).

**Import rewrites:**
- `mgraph_ai_ui_html_transformation_workbench.mgraph.Schema__MGraph__Issues` --> `issues_fs.mgraph.Schema__MGraph__Issues`
- `mgraph_ai_ui_html_transformation_workbench.schemas.graph.Safe_Str__Graph_Types` --> `issues_fs.schemas.graph.Safe_Str__Graph_Types`

**Depends on:** None.

---

### B19: MGraphDB Sync Service

**Objective:** Create a sync service that builds the MGraph from the filesystem, with lazy loading and root change support.

**Provided code:** Yes
**Target files:**
- NEW: `issues_fs/services/mgraph/MGraph__Issues__Sync__Service.py`
- NEW: `issues_fs/services/__init__.py`
- NEW: `issues_fs/services/mgraph/__init__.py`

**Changes required:**
1. Create `MGraph__Issues__Sync__Service` with:
   - Lazy loading via `ensure_loaded(root)`
   - Root change trigger via `on_root_change(new_root)`
   - Full sync via `full_sync(root)` -- rebuilds graph from filesystem
   - Edge building via `build_containment_edges()` -- infers parent-child from paths
   - Path inference via `infer_parent_path(child_path)`
   - Persistence via `save_to_disk()` and `load_from_disk()`

**Import rewrites:**
- `mgraph_ai_ui_html_transformation_workbench.mgraph.Schema__MGraph__Issues` --> `issues_fs.mgraph.Schema__MGraph__Issues`
- `mgraph_ai_ui_html_transformation_workbench.mgraph.MGraph__Issues__Domain` --> `issues_fs.mgraph.MGraph__Issues__Domain`
- `mgraph_ai_ui_html_transformation_workbench.service.issues.graph_services.Graph__Repository` --> `issues_fs.issues.graph_services.Graph__Repository`
- `mgraph_ai_ui_html_transformation_workbench.service.issues.storage.Path__Handler__Graph_Node` --> `issues_fs.issues.storage.Path__Handler__Graph_Node`
- `mgraph_ai_ui_html_transformation_workbench.schemas.graph.Safe_Str__Graph_Types` --> `issues_fs.schemas.graph.Safe_Str__Graph_Types`

**Depends on:** B10 (uses `nodes_list_all`), B11 (uses `node_load_by_path`), B18 (MGraph schemas).

---

### B20: MGraphDB Query Endpoints

**Objective:** Add API endpoints that query the MGraphDB.

**Provided code:** Yes
**Target files:**
- CROSS-REPO: `routes/Routes__Graph.py` (belongs in Issues-FS__Service, NOT Issues-FS core)

**Endpoints:**
- `GET /api/graph/nodes` -- all nodes
- `GET /api/graph/node/{label}` -- single node
- `GET /api/graph/node/{label}/children` -- direct children
- `GET /api/graph/node/{label}/ancestors` -- path to root
- `POST /api/graph/sync` -- force resync

**Note:** This file belongs in the Issues-FS__Service repo. The Dev agent should place it in a staging area or flag it for the service repo. The core `issues_fs` package should NOT contain FastAPI route definitions.

**Depends on:** B18, B19.

---

### B21: Graph Visualisation Endpoints

**Objective:** Return data formatted for D3/vis.js visualisation.

**Provided code:** Yes (in the same `Routes__Graph.py` file as B20)
**Target files:**
- CROSS-REPO: `routes/Routes__Graph.py` (belongs in Issues-FS__Service)

**Endpoints:**
- `GET /api/graph/viz/tree` -- D3 hierarchical tree format
- `GET /api/graph/viz/force` -- force-directed graph format

**Depends on:** B18, B19, B20.

---

### B22: Hyphenated Label Support

**Objective:** Generate labels like `User-Story-1` instead of `UserStory1`.

**Provided code:** Yes
**Target files:**
- MODIFY: `issues_fs/issues/graph_services/Node__Service.py`

**Changes required:**
1. Rewrite `label_from_type_and_index()` to produce hyphenated labels (e.g., `user-story` --> `User-Story-5`).
2. Add `type_to_label_prefix(node_type)` helper method.
3. Add `parse_label_to_type(label)` -- reverse operation, matching against known types (longest first).
4. Update `resolve_link_target()` (currently `_resolve_link_target()`) to use `parse_label_to_type()` instead of the simple `split('-', 1)[0].lower()` approach.
5. Rename `_traverse_graph` to `traverse_graph` and `_find_incoming_links` to `find_incoming_links` (removing underscore prefix per coding patterns).

**Diff vs current:** Current `label_from_type_and_index()` simply capitalises the first word. Current `_resolve_link_target()` assumes single-word types by splitting on the first hyphen. The new code handles multi-word types like `user-story`.

**Depends on:** None (can be done in parallel, but should be applied after B14/B17 since they also modify `Node__Service`).

---

## 3. File Mapping

### Source --> Target

| Source file (to_refactor-in/) | Target file (issues_fs/) | Action | Tasks |
|-------------------------------|--------------------------|--------|-------|
| `schemas/graph/Schema__Node__Info.py` | `issues_fs/schemas/graph/Schema__Node__Info.py` | CREATE | B10 |
| `Graph__Repository.py` | `issues_fs/issues/graph_services/Graph__Repository.py` | MODIFY | B10, B11, B12, B13 |
| `Node__Service.py` | `issues_fs/issues/graph_services/Node__Service.py` | MODIFY | B11, B14, B17, B22 |
| `Root__Selection__Service.py` | `issues_fs/issues/phase_1/Root__Selection__Service.py` | MODIFY | B13 |
| `Issue__Children__Service.py` | `issues_fs/issues/phase_1/Issue__Children__Service.py` | MODIFY | B13 |
| `mgraph/Schema__MGraph__Issues.py` | `issues_fs/mgraph/Schema__MGraph__Issues.py` | CREATE | B18 |
| `mgraph/MGraph__Issues__Domain.py` | `issues_fs/mgraph/MGraph__Issues__Domain.py` | CREATE | B18 |
| `services/mgraph/MGraph__Issues__Sync__Service.py` | `issues_fs/services/mgraph/MGraph__Issues__Sync__Service.py` | CREATE | B19 |
| `scripts/migrate_node_to_issue_json.py` | `issues_fs/scripts/migrate_node_to_issue_json.py` | CREATE | B13 |
| `routes/Routes__Graph.py` | CROSS-REPO (Issues-FS__Service) | STAGE | B20, B21 |
| `routes/Routes__Nodes.py` | CROSS-REPO (Issues-FS__Service) | STAGE | B14 |
| `Html_Transformation_Workbench__Fast_API.py` | CROSS-REPO (parent workbench) | REFERENCE ONLY | Wire-up |

### New files to create

| File | Purpose | Task |
|------|---------|------|
| `issues_fs/schemas/graph/Schema__Node__Info.py` | Node discovery result container | B10 |
| `issues_fs/schemas/graph/Schema__Node__Response.py` | Node response wrapper (success + node) | B11/B14 |
| `issues_fs/mgraph/__init__.py` | Package init | B18 |
| `issues_fs/mgraph/Schema__MGraph__Issues.py` | MGraph schema layer | B18 |
| `issues_fs/mgraph/MGraph__Issues__Domain.py` | MGraph domain layer | B18 |
| `issues_fs/services/__init__.py` | Package init | B19 |
| `issues_fs/services/mgraph/__init__.py` | Package init | B19 |
| `issues_fs/services/mgraph/MGraph__Issues__Sync__Service.py` | Filesystem-to-MGraph sync | B19 |
| `issues_fs/scripts/migrate_node_to_issue_json.py` | Migration script | B13 |

---

## 4. Import Path Rewriting Map

Every import from the provided files that starts with `mgraph_ai_ui_html_transformation_workbench` must be rewritten. The full mapping is:

| Old prefix | New prefix |
|-----------|-----------|
| `mgraph_ai_ui_html_transformation_workbench.schemas.graph.` | `issues_fs.schemas.graph.` |
| `mgraph_ai_ui_html_transformation_workbench.schemas.issues.phase_1.` | `issues_fs.schemas.issues.phase_1.` |
| `mgraph_ai_ui_html_transformation_workbench.service.issues.graph_services.` | `issues_fs.issues.graph_services.` |
| `mgraph_ai_ui_html_transformation_workbench.service.issues.phase_1.` | `issues_fs.issues.phase_1.` |
| `mgraph_ai_ui_html_transformation_workbench.service.issues.storage.` | `issues_fs.issues.storage.` |
| `mgraph_ai_ui_html_transformation_workbench.mgraph.` | `issues_fs.mgraph.` |
| `mgraph_ai_ui_html_transformation_workbench.services.mgraph.` | `issues_fs.services.mgraph.` |

**Note:** The old namespace uses `service.issues.graph_services` (singular `service`) but the target uses `issues.graph_services` (no `service` prefix). The old namespace uses `services.mgraph` (plural) for the sync service, which maps to `services.mgraph` in the target.

---

## 5. Dependency Graph and Execution Order

```
Priority 0 -- Critical Path (blocks UI):
  B10 --> B11 --> B14

Priority 1 -- Cleanup (no UI dependency):
  B12 --> B13 (migration script must run before B13 deploy)

Priority 2 -- Features (parallel OK):
  B15, B16, B17, B22

Priority 3 -- MGraphDB (can start after P0):
  B18 --> B19 --> B20 --> B21
```

### Detailed dependency edges

```
B10 (recursive discovery)
 +-- B11 (path loading)              [uses nodes_list_all infrastructure]
 |    +-- B14 (path endpoint)        [uses node_load_by_path, node_find_path_by_label]
 |         +-- B17 (root scoping)    [uses root_selection_service from B14]
 +-- B19 (sync service)              [uses nodes_list_all, node_load_by_path]

B12 (delete on save) --> B13 (remove fallback)

B18 (MGraph schema) --> B19 (sync service) --> B20 (query endpoints) --> B21 (viz endpoints)

B15, B16 (config CRUD) -- independent
B22 (hyphenated labels) -- independent but should be applied after B14/B17 due to shared file
```

### Recommended execution order for the Dev agent

1. **B10** -- Schema__Node__Info + recursive discovery in Graph__Repository
2. **B11** -- Path-based loading in Graph__Repository + Node__Service (+ create Schema__Node__Response)
3. **B12** -- Legacy node.json deletion on save
4. **B14** -- Hierarchical path resolution in Node__Service (add root_selection_service)
5. **B17** -- Root scoping in list_nodes
6. **B22** -- Hyphenated label support (apply last to Node__Service to minimise merge conflicts)
7. **B13** -- Remove node.json fallback everywhere + create migration script
8. **B18** -- MGraph schemas (can be done in parallel with B12-B22)
9. **B19** -- MGraph sync service
10. **B15** -- Config CRUD node types (can be done any time)
11. **B16** -- Config CRUD link types (can be done any time)
12. **B20** -- Graph query endpoints (cross-repo, stage for Issues-FS__Service)
13. **B21** -- Graph visualisation endpoints (cross-repo, stage for Issues-FS__Service)

---

## 6. Coverage Matrix

| Task | Has provided code? | Has tests in brief? | Target repo | Complexity |
|------|--------------------|--------------------:|-------------|------------|
| B10 | Yes | Yes | Issues-FS | Medium |
| B11 | Yes | Yes | Issues-FS | Medium |
| B12 | Yes | Yes | Issues-FS | Low |
| B13 | Yes | Partial | Issues-FS | Medium |
| B14 | Yes | Yes | Issues-FS + Service | Medium |
| B15 | No (brief only) | No | Issues-FS + Service | Low |
| B16 | No (brief only) | No | Issues-FS + Service | Low |
| B17 | Yes | Yes | Issues-FS | Low |
| B18 | Yes | No | Issues-FS | Medium |
| B19 | Yes | No | Issues-FS | High |
| B20 | Yes | No | Issues-FS__Service | Medium |
| B21 | Yes | No | Issues-FS__Service | Medium |
| B22 | Yes | Partial | Issues-FS | Medium |

---

## 7. Cross-Repo Concerns

### Files that belong in Issues-FS__Service (NOT Issues-FS core)

1. **`routes/Routes__Graph.py`** (B20, B21) -- FastAPI route definitions for MGraph query and visualisation endpoints. These depend on `osbot_fast_api` and should live in the service layer, not the core library.

2. **`routes/Routes__Nodes.py`** (B14) -- The provided file includes the new `node__get_by_path` endpoint. This is a modification to an existing service-layer route file.

3. **`Html_Transformation_Workbench__Fast_API.py`** -- This is the wire-up file for the parent workbench application. It shows how services should be connected but belongs in the workbench repo, not Issues-FS.

### Recommended approach

The Dev agent should:
- Implement core library changes (B10-B19, B22) in `issues_fs/`
- Place cross-repo route files in a staging directory (e.g., `to_refactor-in/staged-for-service/`) with import paths already rewritten to `issues_fs.*`
- Document the wire-up changes needed in Issues-FS__Service as comments in the task issues

---

## 8. Bugs and Issues in Provided Code

### 8.1 Missing Schema: Schema__Node__Response

The provided `Node__Service.py` imports `Schema__Node__Response` from `mgraph_ai_ui_html_transformation_workbench.schemas.graph.Schema__Node__Response`, but this class does not exist in `issues_fs/schemas/graph/`. It must be created. Based on usage, its definition should be:

```python
class Schema__Node__Response(Type_Safe):
    success : bool            = False
    node    : Schema__Node    = None
    message : str             = ''
```

### 8.2 Method naming convention change

The provided code renames `_traverse_graph` to `traverse_graph` and `_resolve_link_target` to `resolve_link_target` and `_find_incoming_links` to `find_incoming_links` (removing the underscore prefix). This follows the coding patterns guide which states "No underscore prefix". The Dev agent should apply this rename consistently.

### 8.3 Schema__MGraph__Issues__Data initialisation

The provided `Schema__MGraph__Issues__Data` uses `__init__` to default `nodes` and `edges` to `{}`. This is necessary because `Type_Safe` may not handle `Dict` defaults well. The pattern is consistent with `MGraph__Issues__Domain` which does the same for its index dicts.

### 8.4 Issue__Children__Service label generation change

The provided `Issue__Children__Service` changes the label generation for hyphenated types from `''.join(p.capitalize())` (e.g., `GitRepo-1`) to `'-'.join(p.capitalize())` (e.g., `Git-Repo-1`). This is intentional for B22 consistency but is a **breaking change** for any existing issues created with the old format. Existing `GitRepo-1` labels will not match the new `Git-Repo-1` pattern.

### 8.5 Potential performance concern in B17

The `list_nodes_for_type()` method in B17 calls `nodes_list_all(root_path=root_path)` for EACH type, which means it scans all filesystem paths multiple times. A more efficient approach would call `nodes_list_all()` once and partition by type. This is noted as an optimisation opportunity but is not blocking.

### 8.6 MGraph sync service -- Safe_Str__Text wrapping

In `MGraph__Issues__Sync__Service.full_sync()`, the `title` field is explicitly wrapped with `Safe_Str__Text(str(node_data.title))`. This extra wrapping may not be necessary if `node_data.title` is already a `Safe_Str__Text`. The Dev agent should verify type compatibility.

---

## 9. Risk Assessment

| Risk | Severity | Mitigation |
|------|----------|------------|
| `Schema__Node__Response` not defined | High | Create the schema before starting B11/B14 |
| Breaking label format change (B22) | Medium | Run B13 migration first; update any hardcoded label patterns in tests |
| B13 deployed before migration | High | Make migration script a prerequisite; add warning in task issue |
| Cross-repo files placed in wrong repo | Medium | Clear staging guidance; do not import `osbot_fast_api` in core `issues_fs` |
| Performance regression from repeated `files__paths()` scanning | Low | Acceptable for Phase 2; optimise in Phase 3 if profiling shows issues |
| `Node__Service` has four tasks modifying it (B11, B14, B17, B22) | Medium | Follow recommended execution order to minimise merge conflicts |

---

## 10. Testing Strategy

Each task should have:
1. **Unit tests** covering the new/modified methods (test templates provided in the brief for B10, B11, B12, B13, B14, B17).
2. **Integration tests** verifying the end-to-end flow (e.g., create nested issues, then verify `nodes_list_all` finds them).
3. **Regression tests** ensuring existing Phase 1 tests still pass after modifications.

For B13 specifically, the migration script should be tested with:
- Files that have only `node.json`
- Files that have only `issue.json`
- Files that have both `node.json` and `issue.json`
- Empty directories

---

## 11. Glossary

| Term | Definition |
|------|-----------|
| `issues_fs` | The core Issues-FS Python package (target namespace) |
| `mgraph_ai_ui_html_transformation_workbench` | The source namespace from which code is being refactored |
| `to_refactor-in/` | Directory containing the provided implementation files |
| `issue.json` | The current standard file for storing issue data |
| `node.json` | The legacy file format being removed in B12/B13 |
| `MGraph` | Memory-Graph database layer providing indexed access to issue data |
| Four-layer architecture | Schema (data) --> Model (CRUD) --> Domain (logic) --> Action (workflows) |

---

*Phase 2 Refactoring Plan v1.0*
*Architect Role -- Issues-FS Ecosystem*
*Date: 2026-02-07*
