# Roam XML Features Research

Comprehensive catalog of Roam Research features, their JSON export representation, and proposed XML mappings for the roam-filesystem project.

## Sources

- **Real data**: Roam Help database JSON export (`MatthieuBizien/RoamResearch-offical-help`, `json/help-2024-12-10-05-06-53.json`)
- **Type definitions**: `roam-garden/roam-export` TypeScript interfaces (`RoamBlock`, `RoamPage`)
- **Parser reference**: `LuisThiamNye/roam-parser` (ClojureScript, comprehensive syntax coverage)
- **Official tools**: `Roam-Research/roam-tools` MCP server
- **Converter projects**: `rj2obs`, `Roam2Obsidian`, Obsidian importer

## Roam JSON Export Structure

### Page object

```json
{
  "title": "Page Title",
  "uid": "KjR9_d8HM",
  "children": [ /* block objects */ ],
  "create-time": 1614229626805,
  "edit-time": 1614769817480,
  ":create/user": { ":user/uid": "CHxhvJ..." },
  ":edit/user": { ":user/uid": "CHxhvJ..." },
  ":log/id": 1614231696948
}
```

### Block object

```json
{
  "string": "Block text with [[refs]] and ((uids))",
  "uid": "abc123def",
  "children": [ /* nested block objects */ ],
  "order": 0,
  "open": true,
  "heading": 2,
  "text-align": "center",
  "create-time": 1614229626805,
  "edit-time": 1614769817480,
  "create-email": "user@example.com",
  "edit-email": "user@example.com",
  ":create/user": { ":user/uid": "CHxhvJ..." },
  ":edit/user": { ":user/uid": "CHxhvJ..." },
  "refs": [ { "uid": "dZV0KPRqu" } ],
  ":block/refs": [ { ":block/uid": "dZV0KPRqu" } ],
  "props": {
    "image-size": {
      "https://example.com/img.png": { "width": 580, "height": null }
    }
  },
  ":block/props": {
    ":image-size": {
      "https://example.com/img.png": { ":width": 580, ":height": null }
    }
  }
}
```

**Key observations from real exports:**
- `heading` values: 0 (none), 1, 2, 3
- `text-align` values: "left", "center", "right", "justify" (rare in practice)
- `refs` and `:block/refs` are parallel arrays tracking referenced entities
- `props`/`:block/props` mirror each other with `:` prefix on namespaced version
- `:log/id` only appears on daily note pages
- `open` tracks collapsed/expanded state

---

## Tier 1: Must-Have Features

### A. Block metadata attributes

| JSON field | Values | XML representation | Decision |
|---|---|---|---|
| `heading` | 0, 1, 2, 3 | `<block uid="x" heading="2">` | Add. Omit attribute when 0. |
| `text-align` | left, center, right, justify | `<block uid="x" text-align="center">` | Add. Omit when "left". |
| `view` (expanded) | table, kanban, document, numbered | `<block uid="x" view="document">` | Expand existing. Add `document`, `numbered`. |
| `create-time` | ms epoch | Exclude from XML | Sync metadata, bloats files. |
| `edit-time` | ms epoch | Exclude from XML | Sync metadata, bloats files. |
| `:create/user` | user object | Exclude from XML | Internal, not agent-useful. |
| `:edit/user` | user object | Exclude from XML | Internal, not agent-useful. |
| `refs` / `:block/refs` | UID arrays | Exclude from XML | Derivable from content tags. |
| `:log/id` | ms epoch | Exclude from XML | Daily notes handled by filename. |
| `open` | boolean | Exclude from XML | UI state, not content. |
| `order` | integer | Exclude from XML | Determined by document order. |
| `props.image-size` | width/height | `<block uid="x" image-width="580">` | Add for lossless round-trip. |

### B. Aliased references

Roam supports markdown-link syntax to alias page refs and block refs:

| Roam syntax | XML representation |
|---|---|
| `[display text]([[Page]])` | `<page-ref title="Page">display text</page-ref>` |
| `[display text](((block-uid)))` | `<block-ref uid="block-uid">display text</block-ref>` |

**Real example from Help DB:**
```json
{ "string": "[community-built themes for inspiration]([[Themes]])" }
```
becomes:
```xml
<page-ref title="Themes">community-built themes for inspiration</page-ref>
```

**Design:** `<page-ref>` and `<block-ref>` become dual-form tags:
- Self-closing when no alias: `<page-ref title="X"/>`
- Container when aliased: `<page-ref title="X">display text</page-ref>`

The parser distinguishes aliases from regular markdown links by checking if the URL portion matches `[[...]]` or `((...))` patterns.

### C. Media embeds

| Roam syntax | XML representation | Real example |
|---|---|---|
| `{{[[video]]: URL}}` | `<video src="URL"/>` | `{{[[video]]: https://youtube.com/watch?v=kUgAqyzwGzw}}` |
| `{{youtube: URL}}` | `<video src="URL"/>` | Same tag, variant syntax |
| `{{pdf: URL}}` | `<pdf src="URL"/>` | PDF document embed |
| `{{iframe: URL}}` | `<iframe src="URL"/>` | `{{iframe: https://zsolt.blog/...}}` |

All `{{[[video]]:`, `{{video:`, and `{{youtube:` variants normalize to `<video>`.

### D. Page embeds (distinct from block embeds)

| Roam syntax | XML representation |
|---|---|
| `{{[[embed]]: ((uid))}}` | `<embed uid="uid"/>` (existing) |
| `{{[[embed]]: [[Page]]}}` | `<embed page="Page Title"/>` (new) |

Page embeds use a `page` attribute instead of `uid`.

### E. Inline embed content (new feature)

Embeds include the full content of the referenced block/page tree inline for readability:

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
  <block uid="beta1">Beta overview content
    <block uid="beta2">Beta detail</block>
  </block>
</embed>

<!-- Unresolvable embed (deleted block) falls back to bare tag -->
<embed uid="deleted999"/>
```

**Design rules:**
1. Daemon resolves embed UIDs/page-titles at serialization time
2. Content inside `<embed>` is **read-only** -- parser ignores it on write-back (only `uid`/`page` attribute matters)
3. Unresolvable embeds fall back to self-closing tag
4. Recursion cap: max depth 3 for nested embeds, then bare `<embed>` to prevent cycles
5. Embed content refreshed each sync cycle
6. Duplicate UIDs across files are intentional (source page + embedding pages)

### F. Backlinks section (new feature)

Each page includes a `<backlinks>` section listing every block in the graph that references it:

```xml
<page uid="KjR9_d8HM" title="Project Alpha">

<block uid="abc123">First block content</block>
<block uid="def456">Second block</block>

<backlinks count="2">
  <backlink page="Meeting Notes" page-uid="mtn001" uid="blk789">Discussed <page-ref title="Project Alpha"/> timeline</backlink>
  <backlink page="Daily Notes/2026-04-08" page-uid="dn0408" uid="blk456"><todo/> Follow up on <page-ref title="Project Alpha"/> deliverables</backlink>
</backlinks>

</page>
```

**Design rules:**
1. `<backlinks>` appears after last `<block>`, before `</page>`
2. Each `<backlink>` has: `page` (source page title), `page-uid` (source page UID), `uid` (source block UID)
3. Content is the full block text with inline XML tags
4. Sorted by page title, then block order within page
5. **Read-only** -- parser ignores on write-back
6. Rebuilt each sync cycle from reverse reference index
7. Includes refs via `[[Page]]`, `#Page`, `#[[Page]]`, alias `[text]([[Page]])`
8. Block-ref backlinks NOT included (block-to-block, not page-level)
9. Self-references included (page referencing itself)
10. `count` attribute on `<backlinks>` for quick agent inspection

**Efficiency:** Daemon builds reverse index during sync: O(total blocks). Trivial for graphs under 50k pages.

---

## Tier 2: Nice-to-Have

| Roam syntax | Proposed XML | Notes |
|---|---|---|
| `{{roam/render: ((uid))}}` | `<roam-render uid="uid"/>` | Custom React components. Content in child code block. |
| `{{roam/css}}` / `{{[[roam/css]]}}` | `<roam-css/>` | CSS marker. CSS in child code block. |
| `{{attr-table: [[Page]]}}` | `<attr-table page="Page"/>` | Structured data view. |
| `[[[[nested]] pages]]` | `<page-ref title="[[nested]] pages"/>` | Title preserves literal inner brackets. |
| `props.image-size` | `image-width` attribute on block | Only for blocks with images. |

---

## Tier 3: Skip for v1 (preserve as literal text)

These `{{...}}` patterns are preserved as-is in block text. They round-trip without loss since the parser treats unknown `{{...}}` as literal text.

| Feature | Roam syntax | Reason to skip |
|---|---|---|
| Calculator | `{{calc: expression}}` | Interactive widget, no persistent data |
| Slider | `{{slider}}` | Ephemeral UI widget |
| Pomodoro | `{{POMO}}` | Timer widget |
| Word count | `{{word-count}}` | Display-only |
| Character count | `{{character-count}}` | Display-only |
| Mentions | `{{mentions}}` | Dynamic query widget |
| Streak | `{{streak}}` | Habit tracker widget |
| Search | `{{search: term}}` | Dynamic search widget |
| Date picker | `{{date}}` | UI widget |
| Comment button | `{{comment-button}}` | UI widget |
| Conditional | `{{or: a \| b}}` | Conditional rendering, very rare |
| Conditional visibility | `{{=: a \| b}}` | Very rare |
| Delta calc | `{{Δ: expression}}` | Extremely rare |
| Hiccup | `:hiccup [...]` | Developer-only React/ClojureScript |
| Encrypted blocks | `{{encrypt}}` | Requires keys, security scope |
| Comments (threads) | N/A | Not in JSON export |
| TaoOfRoam | `{{TaoOfRoam}}` | Easter egg |
| Orphans | `{{orphans}}` | Dynamic query widget |

---

## Markdown kept as-is (no XML conversion)

These inline patterns stay as raw text in block content:

- Bold: `**text**`
- Italic: `__text__`
- Code: `` `text` ``
- Code blocks: ` ```lang ... ``` `
- Highlights: `^^text^^`
- Strikethrough: `~~text~~`
- LaTeX: `$$formula$$`
- Images: `![alt](url)`
- External links: `[text](url)`
- Blockquotes: `> quoted text`
- Horizontal rules: `---`
- Inline URLs (auto-detected)
- Date references like `[[December 30th, 2020]]` (these are regular `<page-ref>` -- the date format is just a page title convention)

---

## Updated Tag Vocabulary (complete)

Expands from 9 tags to 17 tags:

| Tag | Status | Self-closing? | Attributes |
|---|---|---|---|
| `<page>` | Existing | No | `uid`, `title` |
| `<block>` | Existing, updated | No | `uid`, `view`, `heading`, `text-align`, `image-width` |
| `<page-ref>` | Existing, updated | Either | `title`, `syntax` |
| `<block-ref>` | Existing, updated | Either | `uid` |
| `<embed>` | Existing, updated | Either | `uid` or `page` |
| `<todo>` | Existing | Yes | -- |
| `<done>` | Existing | Yes | -- |
| `<attr>` | Existing | No | `name` |
| `<query>` | Existing | No | -- |
| `<video>` | **New** | Yes | `src` |
| `<pdf>` | **New** | Yes | `src` |
| `<iframe>` | **New** | Yes | `src` |
| `<roam-render>` | **New** | Yes | `uid` |
| `<roam-css>` | **New** | Yes | -- |
| `<attr-table>` | **New** | Yes | `page` |
| `<backlinks>` | **New** | No | `count` |
| `<backlink>` | **New** | No | `page`, `page-uid`, `uid` |

---

## Error Checking & Verification Strategy

### A. Structural verification (JSON -> XML)

| Check | Method | Pass criteria |
|---|---|---|
| Page count | Count JSON top-level objects vs `.roam` files | Exact match |
| Block count | Recursive JSON block count vs `<block>` tags across files | Exact match |
| UID preservation | Extract all UIDs from JSON vs all `uid="..."` from XML | Set equality |
| Nesting depth | Per-block: JSON parent chain depth vs XML element depth | Match per block |
| Content fidelity | Per-block: JSON `string` vs XML block content (after reversing tag substitutions) | Semantic match |
| Heading preservation | Blocks with `heading` in JSON have `heading` attribute in XML | All match |
| View preservation | Blocks with view info have `view` attribute in XML | All match |

### B. Round-trip verification (JSON -> XML -> JSON)

1. Convert JSON export -> XML files
2. Parse XML files -> reconstruct JSON-equivalent structure
3. Normalize both:
   - Sort pages by UID
   - Strip metadata fields (`create-time`, `edit-time`, `:create/user`, `:edit/user`, `refs`, `:block/refs`) from JSON side
   - Normalize whitespace
4. Deep-compare normalized trees
5. Report per-page pass/fail with diff details

**Reverse mapping for round-trip:**
- `<page-ref title="X"/>` -> `[[X]]` (or `#X` if `syntax="hashtag"`)
- `<page-ref title="X">text</page-ref>` -> `[text]([[X]])`
- `<block-ref uid="X"/>` -> `((X))`
- `<block-ref uid="X">text</block-ref>` -> `[text](((X)))`
- `<embed uid="X">...</embed>` -> `{{[[embed]]: ((X))}}` (strip inlined content)
- `<embed page="X">...</embed>` -> `{{[[embed]]: [[X]]}}`
- `<todo/>` -> `{{[[TODO]]}}`
- `<done/>` -> `{{[[DONE]]}}`
- `<attr name="K">V</attr>` -> `K:: V`
- `<video src="URL"/>` -> `{{[[video]]: URL}}`
- `<pdf src="URL"/>` -> `{{pdf: URL}}`
- `<iframe src="URL"/>` -> `{{iframe: URL}}`
- `<query>Q</query>` -> `{{[[query]]: Q}}`
- `<roam-render uid="X"/>` -> `{{roam/render: ((X))}}`
- `<roam-css/>` -> `{{[[roam/css]]}}`
- `<attr-table page="X"/>` -> `{{attr-table: [[X]]}}`
- `<backlinks>...</backlinks>` -> stripped entirely (read-only, not in Roam data)

### C. Reference integrity checks

| Check | Method | Severity |
|---|---|---|
| Dangling block-refs | Every `<block-ref uid="X">` has a `<block uid="X">` somewhere | Warning (Roam allows refs to deleted blocks) |
| Dangling page-refs | Every `<page-ref title="X">` has a corresponding file | Warning (Roam allows refs to non-existent pages) |
| Embed resolution | Every `<embed uid="X">` resolves; self-closing = failed? | Warning |
| Backlink completeness | Every `<page-ref>` in block B -> target page's backlinks contains B | Error (bug in backlink builder) |
| Backlink accuracy | Every `<backlink>` corresponds to a real `<page-ref>` | Error (stale backlink) |

### D. Continuous sync verification

After each Roam -> local update:
1. Re-serialize fetched JSON to XML independently
2. Byte-compare with actual file written
3. Log discrepancies as sync integrity warnings

### E. Test harness

```
test/
  fixtures/
    comprehensive.json      # Synthetic, exercises all features
    help-database.json      # Real Roam help DB export
  verify.rs (or .ts)
    structural_check(json, xml_dir) -> Report
    round_trip_check(json, xml_dir) -> Report
    reference_integrity_check(xml_dir) -> Report
    backlink_completeness_check(xml_dir) -> Report
```

---

## Test Data Sources

### Primary: Roam Research Help Database

- **Repo**: `github.com/MatthieuBizien/RoamResearch-offical-help`
- **File**: `json/help-2024-12-10-05-06-53.json`
- **Features found**: `heading` (1,2,3), `props`/`:block/props` with `image-size`, `:log/id` on date pages, `{{[[video]]: URL}}`, `{{iframe: URL}}`, `{{[[roam/css]]}}`, `{{roam/render: ((uid))}}`, `[alias]([[Page]])`, `Key:: [[Value]]`, code blocks, `---`, `((block-ref))`
- **Not found** (need synthetic): `text-align`, `{{TODO}}`, `{{DONE}}`, `{{query:}}`, `{{pdf:}}`, kanban, table view, nested `[[[[page]]]]` refs

### Secondary: Synthetic test fixtures

Hand-crafted JSON exercising all features including those not in help DB.

### Tertiary: Converter projects

- `roam-garden/roam-export` -- TypeScript types
- `LuisThiamNye/roam-parser` -- Comprehensive syntax tests
- `Roam-Research/roam-tools` -- Official MCP, validates API understanding
- `obsidianmd/obsidian-importer` -- Exhaustive `{{}}` command list

---

## Edge Cases

1. **Alias vs markdown link**: `[text](url)` is markdown. `[text]([[Page]])` is alias. Parser checks if URL starts with `[[` or `((`.

2. **Nested page refs**: `[[[[nested]] pages]]` -- title is `[[nested]] pages`. Requires bracket-depth counting.

3. **Embed cycles**: A embeds B embeds A. Depth cap of 3 with visited-UID tracking.

4. **Self-referencing backlinks**: Page refs itself. Include -- useful info.

5. **Attribute values with page refs**: `Status:: [[Active]]` -> `<attr name="Status"><page-ref title="Active"/></attr>`.

6. **Multiple `::` in attributes**: `Description:: Has a key:: inside` -- split only on FIRST `::`.

7. **Video URL variants**: `{{[[video]]:`, `{{video:`, `{{youtube:` all normalize to `<video>`.

8. **`children/view-type` not in JSON export**: `document` and `numbered` views may only appear via API pull, not JSON export. Spec should note this gap.

9. **Multi-image blocks**: `props.image-size` can have multiple URLs. `image-width` attribute handles first image; multi-image is rare.

10. **Backlink volume**: Hub pages (e.g., "TODO") may have hundreds of backlinks. `count` attribute helps agents. Consider `max_backlinks` config.

11. **Firebase storage URLs**: Image URLs often use Firebase with long query params. Must preserve exactly.

12. **Empty pages/blocks**: Valid. `<page uid="x" title="Y"></page>` and `<block uid="x"></block>`.

13. **Date page title in backlinks**: `[[April 8th, 2026]]` produces `<page-ref title="April 8th, 2026"/>`. The `page` attribute in `<backlink>` uses the page title, not the filename.
