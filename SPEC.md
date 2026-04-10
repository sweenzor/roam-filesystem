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

<block uid="abc123">First block with <page-ref title="Project Beta"/> link and a <page-ref title="research" syntax="hashtag"/></block>

<block uid="def456" heading="2">Second block with **formatting** and `code`
  <block uid="ghi789">Nested child block with <attr name="status"><page-ref title="Active"/></attr></block>
  <block uid="jkl012">References <block-ref uid="mno345"/> from another page</block>
  <block uid="jkl013">See <page-ref title="Project Beta">the beta project</page-ref> for details</block>
</block>

<block uid="pqr678"><todo/> Review the design doc
  <embed uid="xyz555">
    <block uid="xyz555">Design principles:
      <block uid="xyz556">Keep it simple</block>
      <block uid="xyz557">Optimize for readability</block>
    </block>
  </embed>
</block>

<block uid="vwx234"><done/> Ship the prototype</block>

<block uid="med001"><video src="https://www.youtube.com/watch?v=dQw4w9WgXcQ"/></block>

<block uid="lmn567"><query>[:find ?title :where [?p :node/title ?title] [?p :block/refs ?r] [?r :node/title "Project Alpha"]]</query></block>

<block uid="opq890" view="table">Table header row
  <block uid="rst111">Row 1 data</block>
  <block uid="uvw222">Row 2 data</block>
</block>

<backlinks count="2">
  <backlink page="Meeting Notes" page-uid="mtn001" uid="blk789">Discussed <page-ref title="Project Alpha"/> timeline</backlink>
  <backlink page="Daily Notes/2026-04-08" page-uid="dn0408" uid="blk456"><todo/> Follow up on <page-ref title="Project Alpha"/> deliverables</backlink>
</backlinks>

</page>
```

### Tag vocabulary

| Roam construct | XML representation | Attributes |
|---|---|---|
| Page | `<page>...</page>` | `uid`, `title` |
| Block | `<block>...</block>` | `uid`, optional `view`, `heading`, `text-align`, `image-width` |
| Page reference `[[Page]]` | `<page-ref title="Page"/>` | `title` |
| Aliased page ref `[text]([[Page]])` | `<page-ref title="Page">text</page-ref>` | `title` |
| Hashtag `#tag` | `<page-ref title="tag" syntax="hashtag"/>` | `title`, `syntax` |
| Hashtag `#[[multi word]]` | `<page-ref title="multi word" syntax="hashtag"/>` | `title`, `syntax` |
| Block reference `((uid))` | `<block-ref uid="uid"/>` | `uid` |
| Aliased block ref `[text](((uid)))` | `<block-ref uid="uid">text</block-ref>` | `uid` |
| Block embed `{{embed: ((uid))}}` / `{{[[embed]]: ((uid))}}` | `<embed uid="uid">...</embed>` | `uid` |
| Page embed `{{embed: [[Page]]}}` / `{{[[embed]]: [[Page]]}}` | `<embed page="Page">...</embed>` | `page` |
| TODO `{{[[TODO]]}}` | `<todo/>` | — |
| DONE `{{[[DONE]]}}` | `<done/>` | — |
| Attribute `key:: value` | `<attr name="key">value</attr>` | `name` |
| Query `{{[[query]]: ...}}` | `<query>[:find ...]</query>` | — |
| Video `{{[[video]]: URL}}` | `<video src="URL"/>` | `src` |
| PDF `{{pdf: URL}}` | `<pdf src="URL"/>` | `src` |
| iframe `{{iframe: URL}}` | `<iframe src="URL"/>` | `src` |
| Render component `{{roam/render: ((uid))}}` | `<roam-render uid="uid"/>` | `uid` |
| CSS block `{{roam/css}}` / `{{[[roam/css]]}}` | `<roam-css/>` | — |
| Attribute table `{{attr-table: [[Page]]}}` | `<attr-table page="Page"/>` | `page` |
| Backlinks section | `<backlinks>...</backlinks>` | `count` |
| Backlink entry | `<backlink>text</backlink>` | `page`, `page-uid`, `uid` |

Hashtags and wikilinks both normalize to `<page-ref>`. The `syntax` attribute (present only for hashtags) preserves the original form for round-tripping back to Roam. When `syntax` is absent, the reference was a `[[wikilink]]`. This means `grep '<page-ref title="foo"'` finds all references to page "foo" regardless of original syntax.

`<page-ref>` and `<block-ref>` are dual-form tags: self-closing when unaliased (`<page-ref title="X"/>`), container when aliased (`<page-ref title="X">display text</page-ref>`). The parser distinguishes aliased references from regular markdown links by checking if the URL portion matches `[[...]]` or `((...))` patterns.

### Block attributes

Blocks support the following optional XML attributes beyond `uid`:

| Attribute | Values | Source | Notes |
|---|---|---|---|
| `view` | `table`, `kanban`, `document`, `numbered` | `:children/view-type` | Child block display mode. `document` and `numbered` may only appear via API, not JSON export. |
| `heading` | `1`, `2`, `3` | `heading` field | Omit when 0 (no heading). |
| `text-align` | `center`, `right`, `justify` | `text-align` field | Omit when "left" (default). Rare in practice. |
| `image-width` | pixel number | `props.image-size.*.width` | Only on blocks containing images. |

Example:

```xml
<block uid="abc123" heading="2" text-align="center">A centered heading</block>
<block uid="def456" image-width="580">![screenshot](https://example.com/img.png)</block>
```

### Inline embed content

Embeds include the full content of the referenced block or page tree inline for readability. This duplicates data but makes each file self-contained — an agent reading a page sees all embedded content without opening other files.

```xml
<!-- Block embed with inlined content -->
<embed uid="xyz555">
  <block uid="xyz555">Design principles:
    <block uid="child1">Keep it simple</block>
    <block uid="child2">Optimize for readability</block>
  </block>
</embed>

<!-- Page embed with inlined content -->
<embed page="Project Beta">
  <block uid="beta1">Beta overview
    <block uid="beta2">Detail</block>
  </block>
</embed>

<!-- Unresolvable embed (deleted block) falls back to bare tag -->
<embed uid="deleted999"/>
```

Rules:
- The sync daemon resolves embed UIDs/page-titles at serialization time and includes the block tree
- Content inside `<embed>` is **read-only** — the parser ignores it on write-back. Only the `uid` or `page` attribute matters for Roam.
- If the referenced block/page doesn't exist, fall back to self-closing `<embed uid="X"/>`
- Nested embeds (embed of an embed) are resolved to max depth 3, then fall back to bare `<embed>` to prevent infinite recursion
- Embed content is refreshed on each sync cycle, so it stays current with source changes

### Backlinks

Each page file includes a `<backlinks>` section listing every block in the graph that references this page. This eliminates the need to `grep` all files to find incoming references.

```xml
<backlinks count="2">
  <backlink page="Meeting Notes" page-uid="mtn001" uid="blk789">Discussed <page-ref title="Project Alpha"/> timeline</backlink>
  <backlink page="Daily Notes/2026-04-08" page-uid="dn0408" uid="blk456"><todo/> Follow up on <page-ref title="Project Alpha"/> deliverables</backlink>
</backlinks>
```

Rules:
- `<backlinks>` appears after the last `<block>`, before `</page>`
- Each `<backlink>` has: `page` (source page title), `page-uid` (source page UID), `uid` (source block UID)
- Content is the full block text with inline XML tags preserved
- Sorted by page title, then block order within page
- **Read-only** — the parser ignores the entire `<backlinks>` section on write-back
- Rebuilt on each sync cycle from a reverse reference index
- Includes references via `[[Page]]`, `#Page`, `#[[Page]]`, and alias `[text]([[Page]])`
- Block-ref backlinks are NOT included (those are block-to-block, not page-level)
- Self-references (page referencing itself) are included
- `count` attribute on `<backlinks>` enables quick agent inspection of reference volume

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
- Blockquotes: `> text`
- Horizontal rules: `---`

Additionally, Roam `{{...}}` patterns that are interactive widgets (calc, slider, POMO, word-count, etc.) are preserved as literal text. They round-trip without loss since the parser treats unknown `{{...}}` as literal text.

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

### Parsing model: lenient, not strict XML

**These files are NOT valid XML.** They use XML-like tags for structure, but block content is unescaped natural text. This is a deliberate design choice.

The problem: markdown content routinely contains `<`, `>`, and `&` (code examples, comparisons, HTML). Requiring agents to escape these as `&lt;`, `&gt;`, `&amp;` defeats the purpose of making files easy to read and write.

The solution: the parser knows the **finite tag vocabulary** — `page`, `block`, `page-ref`, `block-ref`, `embed`, `todo`, `done`, `attr`, `query`, `video`, `pdf`, `iframe`, `roam-render`, `roam-css`, `attr-table`, `backlinks`, `backlink` — and treats any `<` that doesn't match a known tag as literal text. This is analogous to how HTML parsers work: lenient by default, strict only at the boundaries that matter.

Example of valid content that is NOT valid XML:

```xml
<block uid="abc123">Check if a < b && c > d, see <page-ref title="Results"/></block>
```

The parser sees:
- `<block uid="abc123">` — known tag, start of block
- `Check if a < b && c > d, see ` — literal text (the `<` and `&&` don't start known tags)
- `<page-ref title="Results"/>` — known tag, inline reference
- `</block>` — known tag, end of block

Agents can write natural text without escaping. The tradeoff is that standard XML parsers (`ElementTree`, `quick-xml`) cannot parse these files directly — the daemon uses a custom parser.

### Special characters in attributes

Attribute values (`uid="..."`, `title="..."`) DO require escaping for `"` since it terminates the attribute. Use `&quot;` or single-quote the attribute. Page titles containing `"` are rare in practice.

### Agent authoring guidelines

When an agent creates or edits a `.roam` file:
- Write block content as natural text — no XML escaping needed
- Use the known tags for structure and references
- Do not invent new tags — the parser will treat them as literal text
- Preserve existing `uid` attributes — they are identity anchors
- Omit `uid` on new blocks/pages — the daemon assigns them
- Do not edit content inside `<embed>` tags — it is read-only (edits are ignored; modify the source page instead)
- Do not edit the `<backlinks>` section — it is read-only and rebuilt each sync cycle

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

### Atomic file writes

The daemon writes updated files atomically: write to a temporary file in the same directory, then `rename()` to the target path. This prevents agents from reading a partially-written file. The file watcher ignores writes initiated by the daemon itself (tracked via a write-lock flag) to avoid echo loops.

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

## Agent Usage Patterns

Common operations an agent would perform against the filesystem:

```bash
# List all pages
ls pages/

# Read a page
cat pages/meeting-notes.roam

# Find all references to a page
grep '<page-ref title="Project Alpha"' pages/*.roam

# Find all open TODOs
grep '<todo/>' pages/*.roam

# Find which page defines a block UID
grep 'uid="mno345"' pages/*.roam

# Find all pages with a specific attribute value
grep '<attr name="status">active</attr>' pages/*.roam

# Create a new page
cat > pages/new-idea.roam << 'EOF'
<page title="New Idea">

<block>This is a new page created by an agent</block>

<block><page-ref title="Project Alpha"/> inspired this idea</block>

</page>
EOF

# Add a block to an existing page (append before </page>)
sed -i '' '/<\/page>/i\
<block>New block appended by agent</block>
' pages/meeting-notes.roam

# Find all pages modified recently (by filesystem mtime)
find pages/ -name "*.roam" -mmin -30

# Full-text search across all pages
grep -r "specific phrase" pages/

# See what links TO a page (no grep needed — just read the page)
# The <backlinks> section at the bottom lists all incoming references
cat pages/Project/Alpha.roam

# Find pages with most incoming references
grep -c '<backlink' pages/*.roam | sort -t: -k2 -rn | head -20

# Find all TODO blocks that reference a specific page
grep '<backlink.*<todo/>' pages/Project/Alpha.roam
```

---

## Resolved Design Decisions

1. **Hashtags**: `#tag` → `<page-ref title="tag" syntax="hashtag"/>`. Normalizes all page references to one tag. The `syntax` attribute preserves original form for round-tripping.

2. **Attributes**: `key:: value` → `<attr name="key">value</attr>`. Enables structured queries via grep. Attribute values can contain inline tags (e.g., `<page-ref>`). Only the first `::` is treated as the separator.

3. **Queries**: `{{[[query]]: ...}}` → `<query>...</query>`. Datalog content is opaque but bounded by tags.

4. **Tables**: Parent block gets `view="table"` attribute. Child blocks are rows. No special table syntax.

5. **Kanban**: Parent block gets `view="kanban"` attribute. Child blocks are columns, grandchildren are cards.

6. **Aliases**: `[text]([[Page]])` → `<page-ref title="Page">text</page-ref>`. Distinguished from regular markdown links by checking if URL matches `[[...]]` or `((...))`.

7. **Media embeds**: `{{[[video]]: URL}}`, `{{pdf: URL}}`, `{{iframe: URL}}` → dedicated self-closing tags. All video variants (`{{video:`, `{{youtube:`, `{{[[video]]:`) normalize to `<video>`.

8. **Inline embed content**: `<embed>` tags include the full block tree of the referenced content. Read-only on write-back. Max recursion depth 3 for nested embeds. Unresolvable embeds fall back to self-closing. Both `{{embed: ((uid))}}` and `{{[[embed]]: ((uid))}}` syntax variants are supported (same for page embeds).

9. **Backlinks**: Each page includes a `<backlinks>` section with full block text of every referring block. Read-only, rebuilt each sync cycle. Sorted by page title.

10. **Block metadata**: `heading` and `text-align` become block attributes. Timestamps, user info, and refs arrays are excluded (derivable or sync-internal).

11. **Widget commands**: `{{calc:}}`, `{{slider}}`, `{{POMO}}`, `{{word-count}}`, etc. are preserved as literal text. No XML tags needed — they round-trip safely.

12. **Version history**: Out of scope for v1. Roam's block-level history is not exposed via API anyway.

13. **Permissions**: Out of scope for v1. Single-user graphs only.

---

## Verification

The sync daemon includes verification checks to confirm proper JSON→XML translation:

### Structural checks (run after initial sync)

- **Page count**: JSON top-level objects == `.roam` file count
- **Block count**: Recursive JSON block count == `<block>` tags across all files
- **UID preservation**: All UIDs from JSON present in XML (set equality)
- **Content fidelity**: Per-block JSON `string` matches XML block content (after reversing tag substitutions)

### Round-trip verification

1. Convert JSON → XML files
2. Parse XML → reconstruct JSON-equivalent structure
3. Normalize both (strip excluded metadata, sort by UID)
4. Deep-compare — report per-page pass/fail with diff details

### Reference integrity (run periodically)

- **Dangling block-refs**: `<block-ref uid="X">` without matching `<block uid="X">` → warning
- **Dangling page-refs**: `<page-ref title="X">` without matching file → warning
- **Embed resolution**: Self-closing `<embed>` with no content → check if intentional
- **Backlink completeness**: Every `<page-ref>` in the graph has a matching `<backlink>` in the target page → error if missing
- **Backlink accuracy**: Every `<backlink>` corresponds to a real `<page-ref>` in the source block → error if stale

### Test data

- **Primary**: Roam Help database JSON export (`MatthieuBizien/RoamResearch-offical-help`)
- **Secondary**: Synthetic test fixture (`test/fixtures/comprehensive.json`) exercising all features
- **Tertiary**: Converter project test cases (`roam-garden/roam-export`, `LuisThiamNye/roam-parser`)
