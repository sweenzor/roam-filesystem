# roam-filesystem Specification

A bidirectional filesystem mirror of Roam Research, enabling AI agents to interact with a Roam graph as a directory of files.

## Goals

1. Agents can read any Roam page by reading a file
2. Agents can create/edit pages by writing files
3. Changes sync bidirectionally — edits in Roam appear as file changes, file edits appear in Roam
4. Roam's graph structure (block UIDs, page references, block references, embeds) is preserved with full fidelity through round-trips
5. The file format is optimized for LLM consumption — XML for structure, markdown for content

## Non-Goals

- Real-time collaborative editing (eventual consistency is fine)
- Replacing Roam's UI for human use
- Rendering files in markdown previewers
- Supporting graphs with >50,000 pages (optimize for personal/team graphs)

---

## Architecture

```
AI Agent / CLI  ←→  local files on disk  ←→  sync daemon  ←→  Roam Research API
   POSIX I/O        ~/roam-graph/pages/      roam-fsd         api.roamresearch.com
```

The system has two components:

1. **Local files**: Real files on disk in a watched directory. No FUSE, no NFS. Agents interact with them using standard filesystem operations.
2. **Sync daemon** (`roam-fsd`): A long-running process that watches the local directory for changes and polls Roam for remote changes. Syncs bidirectionally.

### Why real files (not FUSE/NFS)

- A personal Roam graph is typically <10,000 pages. Materializing all of them as files is trivial.
- No kernel extensions, no mount commands, no Docker privilege escalation.
- Every tool works: grep, cat, find, sed, your editor, git — no filesystem driver bugs.
- FUSE/NFS can be added later as an optimization layer if lazy hydration becomes necessary for very large graphs.

---

## Directory Layout

```
~/roam-graph/                           # Configurable root
  pages/                                # All Roam pages as .roam files
    Daily Notes/                        # Daily notes namespace
      2026-04-08.roam
      2026-04-07.roam
    Project/                            # Roam namespace "Project/" → directory
      Alpha.roam                        # Page titled "Project/Alpha"
      Beta.roam
    meeting-notes.roam                  # Page titled "meeting-notes"
    Books I Want to Read.roam           # Spaces preserved
  .roam/                                # Metadata directory
    config.toml                         # Auth token, graph name, sync settings
    state.json                          # Sync cursor, last poll timestamp
    conflicts/                          # Conflict losers (Roam wins)
      2026-04-08T14:30:00Z_meeting-notes.roam
    logs/                               # Sync daemon logs
```

### Page title to filename mapping

| Roam page title | File path |
|---|---|
| `meeting-notes` | `pages/meeting-notes.roam` |
| `Project/Alpha` | `pages/Project/Alpha.roam` |
| `April 8th, 2026` | `pages/Daily Notes/2026-04-08.roam` |
| `Books I Want to Read` | `pages/Books I Want to Read.roam` |
| `page with <angle> brackets` | `pages/page with %3Cangle%3E brackets.roam` |

Rules:
- `/` in page titles creates directory nesting (Roam namespace convention)
- Daily notes pages are detected by title pattern and placed in `Daily Notes/` with ISO date filenames
- Characters illegal in filenames (`<>:"\|?*`) are percent-encoded
- Filenames are truncated at 200 characters; a `-<uid-prefix>` suffix is appended if truncation causes collisions
- The mapping is deterministic — no lookup table needed for the common case

---

## File Format

Files use `.roam` extension. The format is XML-structured with markdown inline content.

### Example

```xml
<page uid="KjR9_d8HM" title="Project Alpha">

<block uid="abc123">First block with <page-ref title="Project Beta"/> link</block>

<block uid="def456">Second block with **formatting** and `code`
  <block uid="ghi789">Nested child block</block>
  <block uid="jkl012">References <block-ref uid="mno345"/> from another page</block>
</block>

<block uid="pqr678"><todo/> Review the design doc
  <block uid="stu901"><embed uid="xyz555"/></block>
</block>

<block uid="vwx234"><done/> Ship the prototype</block>

</page>
```

### Tag vocabulary

| Roam construct | XML representation | Attributes |
|---|---|---|
| Page | `<page>...</page>` | `uid`, `title` |
| Block | `<block>...</block>` | `uid` |
| Page reference `[[Page]]` | `<page-ref title="Page"/>` | `title` |
| Block reference `((uid))` | `<block-ref uid="uid"/>` | `uid` |
| Block embed `{{[[embed]]: ((uid))}}` | `<embed uid="uid"/>` | `uid` |
| TODO `{{[[TODO]]}}` | `<todo/>` | — |
| DONE `{{[[DONE]]}}` | `<done/>` | — |

### What stays as markdown

All inline formatting remains as-is within block text content:

- Bold: `**text**`
- Italic: `__text__`
- Code: `` `text` ``
- Code blocks: ` ```lang ... ``` `
- Highlights: `^^text^^`
- Strikethrough: `~~text~~`
- LaTeX: `$$formula$$`
- Images: `![alt](url)`
- External links: `[text](url)`
- Hashtags: `#tag` (these are page references in Roam but kept as shorthand since they're unambiguous)

Rationale: these are all inline decorations that every LLM and text tool handles fine. XML-ifying them would add noise without improving parseability.

### Block nesting

Nesting is expressed by XML tree structure. A `<block>` inside a `<block>` is a child:

```xml
<block uid="parent">Parent content
  <block uid="child1">First child
    <block uid="grandchild">Deep nesting</block>
  </block>
  <block uid="child2">Second child</block>
</block>
```

Indentation is cosmetic (2 spaces per level) for human readability. The parser uses XML tree structure, not indentation, to determine nesting.

### Block ordering

Blocks appear in document order. The order of `<block>` elements within their parent determines the `:block/order` in Roam. No explicit order attribute is needed.

### New blocks (no UID)

When an agent creates a new block, it may omit the `uid` attribute:

```xml
<block>New block created by an agent</block>
```

The sync daemon assigns a temporary local UID, creates the block in Roam, and rewrites the file with the Roam-assigned UID.

### New pages (no UID)

An agent can create a new page by creating a new `.roam` file:

```xml
<page title="My New Page">

<block>First block of new page</block>

</page>
```

The `uid` attribute on `<page>` is omitted. The sync daemon creates the page in Roam and rewrites the file with the assigned UID.

### Special characters in block content

Block text content is XML text, so:
- `<` → `&lt;` (unless it's a recognized tag like `<page-ref>`)
- `>` → `&gt;`
- `&` → `&amp;`
- `"` inside attributes → `&quot;`

The sync daemon handles escaping/unescaping transparently. Agents writing raw content should use standard XML escaping, or the daemon can accept unescaped content and fix it on next sync cycle (best-effort tolerance).

---

## Sync Protocol

### Overview

Inspired by the S3 Files "stage and commit" model:

- **Roam is authoritative.** On conflict, Roam's version wins.
- **Changes are batched.** Local file changes accumulate and commit to Roam periodically.
- **The boundary is explicit.** The sync daemon manages the boundary between the two representations rather than hiding it.

### Sync cycle

The daemon runs a continuous loop:

```
loop {
    1. Poll Roam for remote changes (pages modified since last sync)
    2. Apply remote changes to local files
    3. Detect local file changes (via filesystem watcher)
    4. Batch local changes and commit to Roam
    5. Handle conflicts
    6. Update sync state
    sleep(poll_interval)
}
```

### Remote change detection (Roam → local)

The daemon queries Roam for pages modified since the last sync timestamp:

```clojure
[:find ?title ?uid ?edit-time
 :in $ ?since
 :where
 [?page :node/title ?title]
 [?page :block/uid ?uid]
 [?page :edit/time ?edit-time]
 [(> ?edit-time ?since)]]
```

For each changed page, the daemon pulls the full block tree:

```clojure
[:find (pull ?page [:node/title :block/uid :edit/time :create/time
                    {:block/children ...}])
 :in $ ?page-uid
 :where
 [?page :block/uid ?page-uid]]
```

The pulled tree is converted to `.roam` XML format and written to the corresponding file, overwriting the previous version.

### Local change detection (local → Roam)

The daemon watches the `pages/` directory using platform-native file events:
- macOS: FSEvents
- Linux: inotify

On file change:
1. Parse the modified `.roam` file
2. Diff against the last-known state (stored in memory / `.roam/state.json`)
3. Compute minimal Roam write operations:
   - New `<block>` without `uid` → `create-block`
   - Modified block content → `update-block`
   - Removed `<block>` → `delete-block`
   - Reordered blocks → `move-block`
   - New file → `create-page` + `create-block` for each block
   - Deleted file → `delete-page`
4. Queue operations for next commit

### Commit batching

Local changes are committed to Roam every **60 seconds** (configurable via `commit_interval` in `config.toml`). This:
- Debounces rapid edits (agent writing multiple files in sequence)
- Reduces API calls
- Creates a clear "save point" semantic

On commit:
1. Drain the write queue
2. Group operations by page
3. Execute via Roam's write API (`POST /api/graph/{name}/write`)
4. On success: update sync state, rewrite files with Roam-assigned UIDs
5. On failure: log error, re-queue with exponential backoff (max 5 retries)

### Conflict resolution

A conflict occurs when the same block is modified both locally and in Roam within the same sync window.

Detection:
- The daemon tracks `last_synced_edit_time` per page
- On poll, if a page has `edit_time > last_synced_edit_time` AND has pending local changes, check block-level overlap
- If any block UID appears in both the remote diff and the local write queue → conflict

Resolution:
1. **Roam wins.** The remote version is written to the local file.
2. **Local version is preserved.** The pre-resolution local file is copied to `.roam/conflicts/<ISO-timestamp>_<filename>.roam`
3. **Conflict is logged.** Entry added to daemon logs with: timestamp, page title, conflicting block UIDs, paths to both versions.
4. **The write queue entry for conflicting blocks is dropped.** Non-conflicting changes on the same page still commit.

### Initial sync

On first run (no `.roam/state.json` exists):

1. Query Roam for all page titles, UIDs, and edit times
2. For each page, pull the full block tree
3. Convert to `.roam` format and write to disk
4. Record sync state

This is a full materialization. For a 5,000-page graph, expect this to take a few minutes depending on API rate limits.

### Incremental sync

On subsequent runs, only pages with `edit_time > last_sync_timestamp` are fetched. Typical sync cycle processes 0-10 pages.

---

## Roam API

### Authentication

```toml
# .roam/config.toml
[auth]
api_token = "roam-graph-token-xxxxx"
graph_name = "my-graph"

# Optional: use local API instead of remote
# api_mode = "local"
# local_port = 3000
```

Tokens are created in Roam: Settings → Graph → API Tokens.

### Endpoints used

| Operation | Endpoint | Method |
|---|---|---|
| Query (Datalog) | `/api/graph/{name}/q` | POST |
| Pull entity | `/api/graph/{name}/pull` | POST |
| Write actions | `/api/graph/{name}/write` | POST |

### Write actions used

- `create-page`: `{action: "create-page", page: {title: "..."}}`
- `create-block`: `{action: "create-block", block: {string: "...", uid: "..."}, location: {parent-uid: "...", order: N}}`
- `update-block`: `{action: "update-block", block: {uid: "...", string: "..."}}`
- `move-block`: `{action: "move-block", block: {uid: "..."}, location: {parent-uid: "...", order: N}}`
- `delete-block`: `{action: "delete-block", block: {uid: "..."}}`
- `delete-page`: `{action: "delete-page", page: {uid: "..."}}`

### Rate limiting

Roam does not document rate limits. The daemon defaults to conservative behavior:
- Poll interval: 30 seconds
- Max concurrent API calls: 1 (serial execution)
- Backoff on 429/5xx: exponential, starting at 5s, max 5 minutes
- Configurable via `config.toml`

---

## CLI Interface

### `roam-fsd` — the sync daemon

```bash
# Start the daemon
roam-fsd start --config ~/.roam-graph/.roam/config.toml

# Start in foreground (for debugging)
roam-fsd start --foreground

# Stop the daemon
roam-fsd stop

# Check status
roam-fsd status
# Output: syncing | idle | error | stopped
# Last sync: 2026-04-08T14:30:00Z
# Pending local changes: 3
# Conflicts: 1

# Force sync now (don't wait for next cycle)
roam-fsd sync

# Initial setup
roam-fsd init --graph my-graph --token roam-graph-token-xxxxx --dir ~/roam-graph
```

### `roam-fsd init` workflow

1. Creates `~/roam-graph/pages/` and `~/roam-graph/.roam/`
2. Writes `.roam/config.toml` with provided credentials
3. Tests API connectivity
4. Runs initial full sync
5. Prints summary: N pages synced, ready to use

---

## Configuration

```toml
# .roam/config.toml

[auth]
api_token = "roam-graph-token-xxxxx"
graph_name = "my-graph"
api_mode = "remote"             # "remote" (api.roamresearch.com) or "local" (desktop app)
local_port = 3000               # Only used if api_mode = "local"

[sync]
poll_interval = 30              # Seconds between Roam polls
commit_interval = 60            # Seconds between local change commits
max_retries = 5                 # Max retries on API failure
retry_backoff_base = 5          # Base seconds for exponential backoff

[files]
root_dir = "~/roam-graph"       # Root directory for synced files
daily_notes_dir = "Daily Notes" # Subdirectory for daily notes
daily_notes_format = "%Y-%m-%d" # Date format for daily note filenames
max_filename_length = 200       # Truncate filenames longer than this

[cache]
eviction_days = 0               # 0 = never evict (keep all files)
```

---

## Build and Development

All build toolchains run via Docker. No local installation of Rust or system dependencies.

### Build

```bash
docker build -t roam-filesystem .
docker cp $(docker create roam-filesystem):/usr/local/bin/roam-fsd ./roam-fsd
```

### Run

```bash
# Direct binary (extracted from Docker)
./roam-fsd init --graph my-graph --token xxx --dir ~/roam-graph
./roam-fsd start

# Or run inside Docker
docker run --rm -v ~/roam-graph:/data roam-filesystem start --config /data/.roam/config.toml
```

### Test

```bash
docker build --target test -t roam-filesystem-test .
docker run --rm roam-filesystem-test
```

---

## Implementation Phases

### Phase 1: Read-only mirror

- Roam API client (query + pull)
- XML serializer (Roam block tree → `.roam` format)
- Initial full sync
- CLI: `roam-fsd init`, `roam-fsd start --readonly`
- Docker build pipeline

**Milestone**: `ls pages/` shows all Roam pages, `cat pages/foo.roam` shows correct XML content.

### Phase 2: Live read sync

- Poll-based change detection
- Incremental file updates (only rewrite changed pages)
- Sync state persistence
- File watcher setup (detect external changes for Phase 3)

**Milestone**: Edit a page in Roam, see the file update within 30 seconds.

### Phase 3: Bidirectional sync

- XML parser (`.roam` format → Roam write operations)
- Write queue and commit batching
- New page/block creation (UID assignment)
- Block deletion and reordering
- CLI: `roam-fsd sync` (force), `roam-fsd status`

**Milestone**: Create a `.roam` file on disk, see the page appear in Roam within 60 seconds.

### Phase 4: Conflict resolution and robustness

- Per-block conflict detection
- Conflict preservation in `.roam/conflicts/`
- Retry logic with exponential backoff
- Graceful shutdown (flush pending writes)
- Crash recovery (replay write queue)

**Milestone**: Edit the same block in Roam and locally simultaneously. Roam wins, local version is in `conflicts/`.

---

## Open Questions

1. **Hashtag page refs**: `#tag` is shorthand for `[[tag]]` in Roam. Should it map to `<page-ref title="tag"/>` or stay as `#tag`? Current decision: keep as `#tag` for readability.

2. **Attributes**: Roam attributes (`key:: value`) are block-level metadata. Should they get their own XML tag (`<attr key="key">value</attr>`) or stay as text content? Leaning toward XML tag for parseability.

3. **Queries**: Roam query blocks (`{{[[query]]: ...}}`) contain Datalog. Should these be a tag (`<query>...</query>`) or text? Leaning toward tag.

4. **Tables**: Roam has a table construct. Needs investigation on how to represent.

5. **Kanban boards**: Roam kanban views are structural. May not need filesystem representation.

6. **Version history**: Roam tracks block-level version history. Out of scope for v1.

7. **Permissions**: Multi-user graphs have page-level permissions. Out of scope for v1.
