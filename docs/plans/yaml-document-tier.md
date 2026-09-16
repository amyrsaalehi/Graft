# Plan: the document tier — indexing YAML sections and their relations

Status: proposed · Owner: unassigned · Target: `graft` 0.19

This plan adds a fourth extraction tier to graft so that YAML files (knowledge
bases, catalogues, configs, specs) are indexed the same way code is: every
section becomes a node with an exact `file:line` span, and the relations
between sections (aliases, references, includes, links to code) become edges.
Once built, every existing surface — `graft ask`, `graft grep`, `graft
skeleton`, `graft callers`, `graft map`, the MCP tools, the cards — works on
YAML with no per-surface changes.

It is written so an implementer who has never seen this repo can do the work
phase by phase. Every phase names the files it touches, the functions it adds,
the tests that prove it, and the acceptance criteria. Read §1–§4 fully before
starting Phase 0. Do not skip a phase; each one is a working, shippable state.

---

## 0. Vocabulary

| Term | Meaning in this plan |
| --- | --- |
| **tier** | One of graft's extractor families. Today: *depth* (`src/graph/extract.ts`, hand-written tree-sitter), *breadth/generic* (`src/graph/generic.ts`, `tags.scm` over WASM grammars), *container* (`src/graph/container.ts`, `.vue`). This plan adds **document**. |
| **document** | A YAML file. Later: JSON, TOML through the same tier (out of scope here, but the registry is shaped for it). |
| **section** | A mapping entry whose value is itself a mapping or a sequence. The structural unit of a YAML file; becomes a node of kind `section`. |
| **entry** | A sequence item that is a mapping with an identity key (`name`, `id`, `key`, `slug`, `title`). Becomes a node of kind `entry`. |
| **key path** | Dotted path from the document root to a section: `database.replicas`. Used as the node id tail. |
| **wiring graph** | `graft/.graph/wiring.json` — the `GraphV1` of nodes + edges (`src/graph/types.ts`). |
| **RawEdge** | An unresolved edge intent emitted by an extractor (`src/graph/extract.ts:L90-L123`), settled into an `EdgeV1` by `resolveEdges` (`src/graph/resolve.ts:L101`). |

---

## 1. Why, and what "done" looks like

graft answers "where is X / how does X work / what depends on X" from a
prebuilt graph. Today a `.yml` file is invisible: `listSourceFiles`
(`src/graph/source-files.ts:L64`) keeps a file only if one of the three tier
routers claims its extension, and none claims `.yml`/`.yaml`. A knowledge base
kept in YAML therefore cannot be asked about, grepped by section, or traced.

**Done means**, for a repo containing `knowledge/auth.yml`:

```yaml
# knowledge/auth.yml
concepts:
  session:
    summary: A signed cookie that identifies a logged-in user.
    implemented_by: src/auth/session.ts
    related: [token, refresh]
  token: &token
    summary: A bearer JWT.
    handler: issueToken
  refresh:
    <<: *token
    summary: Rotates a token before expiry.
    see: "#/concepts/session"
policies:
  - name: max-session-age
    applies_to: session
    value: 30d
```

1. `graft skeleton knowledge/auth.yml` lists `concepts`, `concepts.session`,
   `concepts.token`, `concepts.refresh`, `policies`, `policies.max-session-age`,
   each with the exact `Lx-Ly` span.
2. `graft ask "how long may a session live"` ranks `policies.max-session-age`
   at the top, with the YAML lines inlined by `--source`.
3. `graft callers concepts.token` reports `concepts.refresh` (extends, via the
   merge key) and any other section that aliases `*token`.
4. `graft callers src/auth/session.ts` includes `concepts.session`
   (imports, from the `implemented_by:` path), and `graft callers issueToken`
   includes `concepts.token` (references, from the `handler:` value).
5. `graft grep "30d"` groups the hit under `policies.max-session-age`.
6. `graft check` is clean right after `graft build`, and `graft build` a second
   time reuses every YAML parse from the extraction cache.
7. `checkGraphInvariants` (`src/graph/invariants.ts:L52`) reports no problems.

---

## 2. Architecture

### 2.1 Where the tier plugs in

```
                 listSourceFiles (source-files.ts)
                          │  claims .yml/.yaml via documentLangOf()
                          ▼
   buildGraph (build.ts:L151)  ──── same 4-way switch in ──── checkGraph (check.ts:L106)
        │
        │  lang ? extractFile          (depth)
        │  : container ? extractContainer
        │  : document ? extractDocument   ◄── NEW  src/graph/document.ts
        │  : extractGeneric               (breadth)
        ▼
   nodes: NodeV1[]  (file + section + entry nodes, origin:"document")
   rawEdges: RawEdge[]  (contains / references / extends / imports)
        │
        ▼
   resolveEdges (resolve.ts:L101)
        └─ new branch for origin:"document" sources ──► src/graph/document-resolve.ts (NEW)
        ▼
   GraphV1  →  writeGraph · writeAskIndex · writeCards · writeFingerprint (unchanged)
        ▼
   ask / grep / skeleton / callers / map / MCP / viewer  (unchanged; see §2.6 for label tweaks)
```

The tier is deliberately shaped like the container tier: a small registry, a
router function, an `extractX(rel, source, lang)` that returns `ExtractResult`,
and one branch in the two places that switch on tiers. Nothing downstream needs
to know a node came from YAML except the resolver, which needs a
document-specific way to settle references.

### 2.2 Parser choice

Use the `yaml` npm package (`/eemeli/yaml`, v2). Not tree-sitter-yaml.

Reasons:
- **Exact offsets.** Every node carries `range: [start, valueEnd, nodeEnd]`
  (character offsets), and `LineCounter.linePos(offset)` maps them to
  1-based lines. Spans are graft's core promise; this makes them trivial to get
  right and easy to test.
- **Anchors, aliases and merge keys are first-class** (`Alias.source`,
  `node.anchor`, `<<` pairs), which is exactly the relation data we want.
  A tree-sitter grammar gives tokens; we would re-implement YAML's reference
  semantics.
- **Multi-document streams** (`---`) via `parseAllDocuments`.
- **Pure JS, no native build, no WASM warm-up.** `extractDocument` can be
  synchronous, so the parse loop in `build.ts` needs no new `await`.
- **Error tolerance.** `doc.errors` lists problems while `doc.contents` still
  holds what parsed; we can index a partially valid file instead of dropping it.

The dependency goes in `package.json` `dependencies`. It has no `postinstall`,
so `allowScripts` needs no entry.

### 2.3 Node model

All document-tier nodes have `origin: "document"` (new union member on
`NodeV1.origin`, `src/graph/types.ts:L69`).

| Node | `kind` | `id` | `name` | `signature` | `span` |
| --- | --- | --- | --- | --- | --- |
| the file | `file` | `knowledge/auth.yml` | `auth.yml` | `null` | `L1-L<last>` |
| a mapping entry whose value is a map or seq | `section` (new) | `knowledge/auth.yml#concepts.session` | `session` | `session: {summary, implemented_by, related}` | key line → last line of value |
| a seq item that is a map with an identity key | `entry` (new) | `knowledge/auth.yml#policies.max-session-age` | `max-session-age` | `- name: max-session-age {applies_to, value}` | `- ` line → last line of the item |
| a seq item with no identity key | *(not a node)* | — | — | — | folded into the parent's `body_text` |
| a scalar entry | *(not a node)* | — | — | — | folded into the parent's `body_text` |

Rules:
- **id** = `<path>#<key path>`. Key path segments are the raw keys joined by
  `.`; an entry's segment is its identity value. Keys containing `.` or
  whitespace are kept verbatim (ids only need uniqueness; `resolveSymbol`
  matches on the tail after the last `.`, which still works for the common
  case). Collisions (`a.b` under `a` vs key `a.b`) go through `mintId` from
  `extract.ts:L448` and get a `~2` ordinal — the same rule code uses, so
  `stripOrdinals` in `traverse.ts` already handles them.
- **Multi-document streams**: the first document uses the plain key path; the
  Nth (N ≥ 2) is prefixed `doc<N>.`, e.g. `file.yml#doc2.metadata`. Kubernetes
  manifests are the motivating case.
- **Depth cap** `MAX_SECTION_DEPTH = 4`: below it, nothing becomes a node; the
  text is still searchable via the enclosing node's `body_text`.
- **Per-file cap** `MAX_SECTIONS_PER_FILE = 400`: past it, the extractor keeps
  only top-level sections (depth 1) and records nothing else. This stops a
  20 000-key i18n locale file from minting 20 000 nodes. The cap is a constant
  in `document.ts` with a doc comment; not a flag.
- **signature**: for a section, `key: {child, keys, …}` listing up to 6 child
  keys (mapping) or `key: [N items]` (sequence). For an entry, `- name: <id>
  {other, keys}`. Whole signature capped at 100 chars. This is what
  `graft skeleton` and the cards print, so it must read like a table of
  contents.
- **span**: start = line of the pair's key token (`pair.key.range[0]`); end =
  line of `value.range[1] - 1` (value-end is exclusive). Trailing blank/comment
  lines are NOT included (that is what `range[1]` vs `range[2]` gives us). For
  an entry, start = the item's own `range[0]` (the line with `- `).
- **body_text**: the whitespace-collapsed source slice of the node's span,
  capped at 5000 chars (reuse the same cap as `generic.ts`), so `ask` can match
  on any scalar value inside a section. The file node's `body_text` is the
  residual (top-level scalars), computed exactly as `fileResidual` does in
  `extract.ts:L156` — export that function or duplicate the 10 lines with a
  comment pointing at the original.
- **body_hash**: `contentHash` of the raw slice (the Tier-2 summary cache key).
- **exported**: always `true`.
- **chars**: set on the file node only (byte length), like `generic.ts:L207`.

### 2.4 Edge model

Every edge uses an existing `Relation` (`types.ts:L95`). No new relation is
added; the vocabulary maps cleanly and every consumer (`WALK_RELATIONS` in
`relations.ts`, `traverse.ts`, `graphrank.ts`, `blast`, MCP descriptions)
keeps working.

| YAML construct | Edge | Confidence | Emitted as |
| --- | --- | --- | --- |
| file → top section; section → nested section / entry | `contains` | `extracted` | RawEdge with `targetId` (already resolved) |
| `*alias` anywhere inside section S, anchor `&a` defined inside section T | S `references` T | `extracted` | RawEdge `{relation:"references", name:"&a"}`, resolved **inside the extractor** (same file, deterministic) — emit with `targetId` |
| `<<: *base` (merge key) inside S, anchor in T | S `extends` T | `extracted` | RawEdge `{relation:"extends", name:"&base"}` → `targetId` |
| `$ref: "#/a/b"`, `see: "#/a/b"` (JSON-pointer into the same file) | S `references` `file#a.b` | `extracted` | `targetId` after pointer → key path conversion |
| `see: "a.b"`, `related: [x, y]`, `depends_on: …` (dotted key path or bare key) | S `references` matching section | `extracted` if same file; `inferred` if unique across all document nodes; dropped if ambiguous | RawEdge `{relation:"references", name:"a.b"}` (no specifier) |
| `extends: base`, `inherits: base`, `parent: base`, `base: …`, `template: …` | S `extends` target | same rule | RawEdge `{relation:"extends", name}` |
| `$ref: "other.yml#/a/b"`, `see: "../x.yml#a.b"` | S `references` `other.yml#a.b` | `extracted` | RawEdge `{relation:"references", specifier:"other.yml", name:"a.b"}` |
| `include: other.yml`, `imports: [a.yml]`, `!include other.yml` tag, any scalar that is an in-repo file path | S (or file) `imports` file node | `extracted` | RawEdge `{relation:"imports", specifier:"<path>"}` |
| scalar value under a code-ish key (`handler`, `function`, `callback`, `class`, `entrypoint`, `command`, `implemented_by`, `module`) that equals the name of exactly one non-file code node | S `references` code node | `inferred` | RawEdge `{relation:"references", name:"issueToken", kinds:[…all non-file kinds]}` |
| scalar value (any key) that equals the identity of exactly one `entry` node in another document | S `references` entry | `inferred` | RawEdge `{relation:"references", name}` — **Phase 3, gated by `RELATION_KEYS`** (see §2.5) |

Resolution philosophy is the repo's existing one (`resolve.ts` header):
same-file match is certain; a unique cross-file match is `inferred`; ambiguity
is dropped, never guessed. Self-loops are dropped for `references`/`extends`.

Because `familyOf(path)` (`resolve.ts:L66`) returns `null` for a `.yml` file,
`reachable()` never filters a document → code edge. That is intended: a
knowledge file may legitimately point at any language. No change to `FAMILIES`.

### 2.5 Which keys count as relational

Two configurable constant sets in `document.ts`:

```ts
/** Keys whose scalar/sequence value NAMES another section or entry. */
export const RELATION_KEYS: Record<string, Relation> = {
  $ref: "references", ref: "references", refs: "references",
  see: "references", see_also: "references", related: "references",
  links: "references", link: "references", uses: "references",
  depends_on: "references", dependencies: "references", requires: "references",
  applies_to: "references", target: "references", targets: "references",
  extends: "extends", inherits: "extends", parent: "extends",
  base: "extends", template: "extends",
  include: "imports", includes: "imports", import: "imports", imports: "imports",
};

/** Keys whose scalar value names a CODE symbol. */
export const CODE_KEYS = new Set([
  "handler", "function", "func", "fn", "callback", "class", "entrypoint",
  "entry_point", "command", "module", "implemented_by", "implementation",
  "symbol", "method", "hook",
]);
```

Matching is on the lower-cased key with `-` normalised to `_`. Values may be
a scalar or a sequence of scalars; each scalar yields one RawEdge. A value that
looks like a file path (contains `/` or ends with a known indexed extension,
see `supportedExtensions()`) under ANY key becomes an `imports` candidate;
`document-resolve.ts` keeps it only if the path resolves to a node in the
graph (relative to the referencing file first, then to the repo root).

Do not make this list user-configurable in this plan. Ship the constants, note
in the doc comment that a per-repo override is a follow-up.

### 2.6 Surfaces that need a one-line touch

| Surface | File | Change |
| --- | --- | --- |
| Kind union | `src/graph/types.ts:L15` | add `"section"` and `"entry"` with a comment |
| Origin union | `src/graph/types.ts:L69` | add `"document"` |
| Invariants | `src/graph/invariants.ts:L28` | add the two kinds to `KINDS` |
| Standalone quality script | `scripts/graph-quality.mjs:L19` | same two kinds (this file keeps its own copy on purpose) |
| Supported extensions | `src/graph/source-files.ts:L20` | union in `documentExtensions()` |
| File enumeration | `src/graph/source-files.ts:L64` | `documentLangOf(f) !== null` in the filter |
| Build switch | `src/graph/build.ts:L211-L221, L271` | 4-way branch + label |
| Check switch | `src/graph/check.ts:L106-L124` | identical 4-way branch, same order |
| Resolver | `src/graph/resolve.ts:L228` | new `origin === "document"` branch delegating to `document-resolve.ts` |
| Language label | `src/graph/extract.ts:L82` `languageLabelOf` | leave; instead in `build.ts` the label falls back to `document?.name` — and `map.ts:L132 sortedLanguages` must also consult `documentLangOf` so `graft map` reports `yaml` |
| Cards | `src/graph/cards.ts:L58 cardPathFor` | document-tier files keep their extension in the card name (`auth.yml.md`) — see §5 risk 2 |
| Viewer colour | `viewer/data.ts:L108` | map `section`/`entry` to an existing colour token (reuse `--k-file`-adjacent one; no new CSS var) |
| MCP descriptions | `src/mcp/tools.ts:L40-L131` | mention "code and YAML sections" in `graft_find_code`, `graft_file_api`, `graft_trace_calls` |
| Skill template | `src/claude/skill-template.ts` | one sentence: YAML/knowledge files are indexed; sections are symbols |
| README | `README.md:L198` | new bullet under *Supported languages*: **Documents** |
| CHANGELOG | `CHANGELOG.md` | `### Added` entry |

`extractorStamp()` (`src/graph/extract-cache.ts:L139`) content-hashes every
module in `src/graph/`, so adding `document.ts` automatically invalidates the
extraction cache on upgrade. Nothing to do there.

---

## 3. Module design

### 3.1 `src/graph/document.ts` (new)

```ts
/**
 * Document tier — YAML files indexed as sections + entries with exact spans,
 * and the relations between them (aliases, merge keys, $ref/see/extends
 * keys, includes, links to code) as edges. See docs/plans/yaml-document-tier.md.
 */
import { parseAllDocuments, LineCounter, isMap, isSeq, isPair, isScalar, isAlias,
         type Document, type Node, type Pair, type YAMLMap, type YAMLSeq } from "yaml";

export interface DocumentLang { name: string; exts: string[] }
export const DOCUMENT_LANGS: readonly DocumentLang[] = [
  { name: "yaml", exts: [".yml", ".yaml"] },
];
export function documentLangOf(path: string): DocumentLang | null;   // same shape as containerLangOf
export function documentExtensions(): string[];

export const MAX_SECTION_DEPTH = 4;
export const MAX_SECTIONS_PER_FILE = 400;
export const IDENTITY_KEYS = ["name", "id", "key", "slug", "title"] as const;
export const RELATION_KEYS: Record<string, Relation> = { /* §2.5 */ };
export const CODE_KEYS: ReadonlySet<string> = new Set([ /* §2.5 */ ]);

/** Synchronous; never needs warming. Throws only when NOTHING parsed
 *  (so build.ts records the error); partial documents are indexed. */
export function extractDocument(rel: string, source: string, lang: DocumentLang): ExtractResult;
```

Internal structure of `extractDocument` (keep these as separate top-level
functions so they are unit-testable and show up in `graft skeleton`):

1. `parseStream(source)` → `{ docs: Document[], lines: LineCounter }`. Calls
   `parseAllDocuments(source, { lineCounter, keepSourceTokens: false, merge: false, prettyErrors: false })`.
   `merge: false` keeps `<<` as an ordinary pair so we can see it. If every doc
   has `contents === null` and at least one has `errors.length > 0`, throw
   `Error("yaml parse failed — " + firstError.message)`.
2. `spanOf(node, lines)` → `"Lx-Ly"` per §2.3 rule. One function, one test file
   pinning it against hand-numbered fixtures.
3. `walkSections(doc, docIndex, ctx)` — depth-first over `YAMLMap.items` and
   `YAMLSeq.items`, minting nodes with `mkNode(keyPath, kind, name, whole, sig)`.
   Emits `contains` RawEdges (`targetId`) parent → child. Tracks
   `anchorOwner: Map<string, nodeId>` — when a node (or any descendant scalar)
   carries `.anchor`, record the innermost enclosing section/entry node id.
4. `collectAliasEdges(doc, ctx)` — second pass with `yaml`'s `visit`: every
   `Alias` becomes `references` (or `extends` when its parent pair's key is
   `<<`) from the innermost enclosing node to `anchorOwner.get(alias.source)`.
   Both ids are known → emit RawEdge with `targetId`. Skip when source == target.
5. `collectKeyEdges(doc, ctx)` — for every `Pair` whose key is in
   `RELATION_KEYS` or `CODE_KEYS` or whose value looks like a path: produce
   RawEdges per §2.4. Same-file `#/a/b` pointers are converted to a key path
   and, if that node exists in `minted`, emitted with `targetId`; otherwise
   emitted with `name` for the resolver.
6. `fileResidual` for the file node's `body_text` (top-level scalars).

The `RawEdge` interface already has every field this needs (`targetId`,
`specifier`, `name`, `kinds`). Add **no** fields to `RawEdge`.

### 3.2 `src/graph/document-resolve.ts` (new)

Called from `resolveEdges` for `references` / `extends` / `imports` RawEdges
whose source node has `origin === "document"`. Keeps YAML logic out of the
already-long resolver.

```ts
export interface DocumentIndex {
  /** "<path>#<keyPath>" → node, for every section/entry node. */
  byId: Map<string, NodeV1>;
  /** bare key path ("concepts.session") → nodes across ALL documents. */
  byKeyPath: Map<string, NodeV1[]>;
  /** last segment / entry identity ("session") → nodes across ALL documents. */
  byName: Map<string, NodeV1[]>;
}
export function buildDocumentIndex(nodes: NodeV1[]): DocumentIndex;

/** Resolve one document-origin RawEdge to an EdgeV1 target, or null to drop it. */
export function resolveDocumentEdge(
  e: RawEdge, byId: Map<string, NodeV1>, docIndex: DocumentIndex,
  globalName: Map<string, NodeV1[]>,   // resolve.ts's existing index, for code symbols
): { target: string; confidence: Confidence } | null;
```

Resolution order inside `resolveDocumentEdge`:

1. `targetId` set → return it, `extracted`.
2. `relation === "imports"` → `resolveDocumentPath(specifier, e.file, byId)`:
   try `posix.join(dirname(file), specifier)`, then `specifier` from repo root,
   with and without a leading `./`; must be a `file` node. Not found → **drop**
   (unlike code imports, an unresolved YAML path is noise, not an external
   package).
3. `specifier` + `name` (cross-file pointer): resolve the file as in (2), then
   look up `<thatPath>#<keyPath>`; pointer form `#/a/b` is converted with
   `pointerToKeyPath` (strip `#/`, split on `/`, unescape `~1`→`/`, `~0`→`~`).
   Missing → drop.
4. `name` only, source is a document node, key is relational:
   a. same file `<file>#<name>` exists → `extracted`.
   b. `byKeyPath.get(name)` has exactly one → `inferred`.
   c. `byName.get(name)` has exactly one → `inferred`.
   d. else drop.
5. `name` with `kinds` (code-ish key): `globalName.get(name)` filtered by
   `kinds`; exactly one → `inferred`; else drop. Also accept dotted
   `Class.method` by matching the id tail (reuse the suffix logic from
   `traverse.ts:L101 symbolMatches` — extract it into a shared helper if
   importing traverse from resolve creates a cycle; check with
   `graft callers symbolMatches`).

Never return `e.source` as the target.

### 3.3 Hook into `resolveEdges`

In `src/graph/resolve.ts`, before the loop, build `docIndex` once
(`buildDocumentIndex(nodes)` — cheap, only document nodes). Inside the loop add
one guard **at the top of** the `imports`, `extends/implements`, and
`references` branches:

```ts
const srcNode = byId.get(e.source);
if (srcNode?.origin === "document") {
  const hit = resolveDocumentEdge(e, byId, docIndex, globalName);
  if (hit && hit.target !== e.source) add(e.source, hit.target, e.relation, hit.confidence);
  continue;
}
```

This keeps every existing code path byte-for-byte untouched for non-document
sources (the same "provably untouched" gating the generic branch uses at
`resolve.ts:L260`).

---

## 4. Phases

Each phase ends with `npm test` green and `npm run build` clean. Commit per
phase. Phase titles double as commit subjects.

### Phase 0 — plumbing: `.yml` is a source file, file node only

Goal: the tier exists end to end but emits only a file node. Proves the
routing, cache, fingerprint and check paths agree before any YAML logic lands.

Steps:
1. `npm i yaml` (dependencies). Verify `import { parseAllDocuments } from "yaml"` type-checks under `tsconfig.json` (ESM, `moduleResolution`).
2. Create `src/graph/document.ts` with the registry, `documentLangOf`,
   `documentExtensions`, and an `extractDocument` that returns only the file
   node (copy the shape of `fileNode` in `generic.ts:L207`, `origin: "document"`).
3. `types.ts`: add `"document"` to `origin`.
4. `source-files.ts`: union `documentExtensions()` in `supportedExtensions()`;
   add `documentLangOf(f) !== null` to the filter in `listSourceFiles`.
5. `build.ts`: after the `container` line, `const document = lang || container ? null : documentLangOf(f.abs);` then `generic` only when none of the three; extend the label fallback and the extract ternary. **Order matters**: depth → container → document → generic, and `check.ts` must use the identical order (its comment at L101 says why).
6. `check.ts`: same branch.
7. `map.ts sortedLanguages`: add `documentLangOf(p)?.name`.

Tests — new file `test/document-yaml.test.ts` (mirror the header style of
`test/container-extract.test.ts`):
- registry: `documentLangOf("k/a.yml")?.name === "yaml"`, `.YAML` case-insensitive, `.ts` → null; `supportedExtensions()` includes `.yml` and `.yaml`.
- build: `tmpRepo` + one `.ts` + one `.yml`; `buildGraph` → graph has a file node for the yml with `origin:"document"`; `meta.languages` includes `"yaml"`; `checkGraph` is clean; second `buildGraph` reports `reused` including the yml.
- `checkGraphInvariants` has no problems.

Acceptance: all of the above green; `graft build` on this repo itself indexes
`.github`-external YAML (e.g. `deploy/*.yml` if present) without errors.

### Phase 1 — sections, entries, spans, contains

Goal: the node model of §2.3 with exact spans; `skeleton`, `grep`, cards work.

Steps:
1. Implement `parseStream`, `spanOf`, `walkSections`, `mkNode`, signature
   builder, depth and per-file caps, multi-document prefixing, `body_text`,
   file residual.
2. `types.ts` Kind: add `"section"`, `"entry"`. `invariants.ts` and
   `scripts/graph-quality.mjs`: add to `KINDS`.
3. `cards.ts cardPathFor`: if `documentLangOf(sourcePath)` → keep the full
   filename: `auth.yml` → `graft/knowledge/auth.yml.md`. Update the doc comment
   and `writeIndex`'s wording if it describes the mapping.
4. `viewer/data.ts` colour map: add the two kinds.

Tests (fixtures as line arrays, expected lines read off the array):
- spans: a 3-level mapping; assert each node's `Lx-Ly` exactly; a section
  followed by a blank line and a comment must NOT include them.
- entries: `- name:` items become `entry` nodes named by identity; an item
  without identity keys is not a node but its text is in the parent's
  `body_text`.
- identity precedence: `name` wins over `id` when both present.
- multi-doc: `---` separated; second doc ids start with `doc2.`.
- depth cap: level 5 keys produce no node; their text is in the level-4 node's `body_text`.
- per-file cap: 401 top-level-child sections → only depth-1 nodes exist; count asserted.
- key collisions: two `- name: x` items in one seq → `x` and `x~2`.
- malformed YAML with nothing parseable → `extractDocument` throws; `buildGraph` records it in `errors` and the file's cache entry has `error` (same as generic's crash test in `test/generic-extract.test.ts`).
- partially malformed → nodes for the valid part, no throw.
- `skeleton(dir, "knowledge/auth.yml")` lists sections in span order with signatures.
- `grepGraph` groups a hit under the innermost section.
- cards: `graft/knowledge/auth.yml.md` exists and lists each section as `- name · section · Lx-Ly — signature`.
- invariants clean; `checkGraph` clean.

Acceptance: `graft ask "<a phrase from a deep scalar>"` on the fixture repo
returns the enclosing section as hit 1 (use `ask()` from `src/ask/ask.ts` in
the test, `graphRank:false`).

### Phase 2 — intra-file relations

Goal: aliases, merge keys, same-file pointers and relational keys become edges.

Steps:
1. `collectAliasEdges` (anchors → owner node; `*alias` → `references`; `<<: *x` → `extends`).
2. `collectKeyEdges` for same-file targets: `#/a/b` pointers and dotted/bare key paths that exist in `minted` → `targetId`.
3. `document-resolve.ts` with `buildDocumentIndex` and steps 1 and 4a of `resolveDocumentEdge`; hook in `resolve.ts`.

Tests:
- alias inside `concepts.refresh` to `&token` under `concepts.token` → one `references` edge, `extracted`.
- `<<: *token` → `extends`, not `references`; a merge with a sequence of aliases (`<<: [*a, *b]`) → two `extends` edges.
- an anchor on a scalar deep inside section T is attributed to T (innermost node), not the file.
- alias to an anchor defined in the same section → no self-loop.
- `see: "#/concepts/session"` → references `…#concepts.session`; pointer escaping `~1`.
- `related: [token, refresh]` → two references resolved by bare name within the file.
- unknown target name → no edge, no invariant problem.
- `graft callers concepts.token` via `callersOf` returns `concepts.refresh` with relation `extends`; `impactOf` at depth 2 walks through.
- `structural()` in ask: query `"who references concepts.token"` returns caller hits (INCOMING regex already matches "references").

### Phase 3 — cross-file relations and links to code

Goal: knowledge ↔ knowledge across files, and knowledge → code.

Steps:
1. `collectKeyEdges`: cross-file pointer forms (`other.yml#/a/b`, `../x.yaml#a.b`) → `{specifier, name}`; path-looking scalars → `imports` with `specifier`; `CODE_KEYS` values → `references` with `kinds` = every `Kind` except `file`, `section`, `entry`.
2. `resolveDocumentEdge` steps 2, 3, 4b–d, 5.
3. Bare-name cross-file resolution (4b/4c) is **only** attempted for values under `RELATION_KEYS`; never for arbitrary scalars (a `value: session` under a non-relational key must not create an edge — assert this).

Tests:
- `implemented_by: src/auth/session.ts` → `imports` edge from the section to the `.ts` file node; `callersOf(fileNode)` includes the section. Relative form `./src/auth/session.ts` and a path relative to the yml's own directory both resolve; a non-existent path yields no edge.
- `handler: issueToken` with exactly one function `issueToken` in the repo → `references`, `inferred`; two functions of that name in different files → dropped; `handler: Auth.issue` resolves to the method by id tail.
- `see: "policies.yml#/rules/max-age"` across files → references `extracted`.
- `applies_to: session` in `policies.yml` where `concepts.session` exists only in `auth.yml` → `inferred`; if a second file also defines `session` → dropped.
- code → YAML direction is NOT produced (a TS string literal `"knowledge/auth.yml"` makes no edge); assert absence so nobody adds it by accident later.
- `graft callers src/auth/session.ts --depth 2` from the CLI (spawn `tsx src/cli.ts`, see how `test/graph-traverse-cli.test.ts` does it) prints the section.
- `blast`: a diff touching `src/auth/session.ts` lists `knowledge/auth.yml` in the radius (use the `blast` API the way `test/blast.test.ts` does).

### Phase 4 — surfaces, docs, guard rails

1. MCP tool descriptions (`src/mcp/tools.ts`) and `src/claude/skill-template.ts` wording; `src/mcp/instructions.ts` if it enumerates what is indexed.
2. README *Supported languages*: add a **Documents** bullet — "YAML (`.yml`, `.yaml`): every mapping section and named list entry is a node with an exact span; aliases, merge keys, `$ref`/`see`/`extends`-style keys, includes and path/handler values become edges to other sections and to code." Update the "Twenty-three languages" count sentence.
3. CHANGELOG `### Added` under the next version.
4. Opt-out: `graft build --no-documents`, persisted like `--no-follow-submodules` (`src/util/state.ts` `BuildConfig`; add `indexDocuments?: boolean`, `readIndexDocuments(root)` defaulting to `true`). `listSourceFiles` must read it the same way it reads `readIncludeDirs`, so the fingerprint probe and hooks path agree with the build. Record it in the fingerprint next to `onlyDirs` (`fingerprint.ts:L87 writeFingerprint`) so a toggle flips drift detection.
5. Performance check: build a scratch repo with 2 000 generated YAML files of ~50 sections each; parse phase must stay under 2 s on a laptop (yaml parses ~10 MB/s; this is ~2 MB). Record the number in the PR description.
6. `graft build` on this repository; inspect `graft/deploy/*.yml.md` cards by eye and fix anything that reads badly (signatures too long, wrong spans).

Tests:
- `--no-documents` persisted: build with the flag, then `listSourceFiles(root, outDir)` with no args excludes `.yml`; `checkGraph` stays clean; building again without the flag but with persisted state still excludes; `--documents` re-enables.
- README/CHANGELOG are prose; no test.

### Phase 5 (follow-ups, not in this plan)

- JSON (`.json`) and TOML (`.toml`) through the same tier: add a `parse` field
  to `DocumentLang` that yields the same intermediate `{ key, value, range }`
  tree; `document.ts` walks the tree, not `yaml` nodes directly. When doing
  Phase 1, keep the walker one level of abstraction above `yaml`'s node types
  (an internal `DocNode` shape) so this is a registry row later — but do not
  build JSON support now.
- Markdown headings as sections (would collide with concept nodes in
  `graft/*.md`; needs its own plan).
- Per-repo override of `RELATION_KEYS` / `CODE_KEYS` in `BuildConfig`.
- Tier-2 summaries for sections (`enrichGraph` already runs on every node with
  `summary_state:"pending"`; check the prompt in `src/ai/` reads well for YAML
  before enabling — until then, nothing needs to be done, it just works or
  produces bland summaries).

---

## 5. Risks and decisions

1. **Node explosion on locale / generated YAML.** Mitigated by
   `MAX_SECTION_DEPTH`, `MAX_SECTIONS_PER_FILE`, the existing 1 MB file cap
   (`ingest/fs.ts MAX_FILE_BYTES`), dot-directory skipping (`.github/` is never
   walked), and `--no-documents`. If a real repo still blows up, the next lever
   is a per-directory cap, not a smarter heuristic.
2. **Card path collision.** `cardPathFor` swaps the extension, so `config.yml`
   and `config.ts` in one directory would both map to `config.md`. Decision:
   document-tier cards keep the extension (`config.yml.md`). This also makes
   `grep config.yml graft/` land on the card, which the INDEX tells agents to do.
3. **False edges from bare-name matching.** Only values under `RELATION_KEYS`
   are name-resolved, and only unique matches win. A bare `value: session` is
   never an edge. Reviewers should push back on any widening of the key list
   without a fixture.
4. **Span drift on comments and blank lines.** `range[1]` (value-end) vs
   `range[2]` (node-end) is the difference. Tests pin exact lines; any change
   to `spanOf` must update the fixtures, never the other way around.
5. **`resolveSymbol` and dotted keys.** `graft callers concepts.token` works via
   the id-tail suffix match. `graft callers token` also works (bare `name`).
   A key containing `.` (`"v1.2": …`) will be matched on its last segment
   only; accepted.
6. **yaml package options.** Use `merge: false` so `<<` stays visible; use
   `keepSourceTokens: false` (we only need ranges); set `version: "1.2"`
   default but do not fail on `%YAML 1.1` directives. `uniqueKeys: false` so a
   duplicate key is a warning, not a dropped document.
7. **Two extractors disagreeing** (`build` vs `check`) is the classic failure
   in this codebase (#236). Both switches must be edited in the same commit,
   and the Phase 0 test builds then checks in the same test.

---

## 6. Appendix A — expected graph for the §1 fixture

Nodes (kind · id · span, for the file in §1):

```
file    knowledge/auth.yml                              L1-L17
section knowledge/auth.yml#concepts                     L2-L13
section knowledge/auth.yml#concepts.session             L3-L6
section knowledge/auth.yml#concepts.token               L7-L9
section knowledge/auth.yml#concepts.refresh             L10-L13
section knowledge/auth.yml#policies                     L14-L17
entry   knowledge/auth.yml#policies.max-session-age     L15-L17
```

Edges:

```
auth.yml                  contains    #concepts                          extracted
auth.yml                  contains    #policies                          extracted
#concepts                 contains    #concepts.session                  extracted
#concepts                 contains    #concepts.token                    extracted
#concepts                 contains    #concepts.refresh                  extracted
#policies                 contains    #policies.max-session-age          extracted
#concepts.session         imports     src/auth/session.ts                extracted   (implemented_by:)
#concepts.session         references  #concepts.token                    extracted   (related:)
#concepts.session         references  #concepts.refresh                  extracted   (related:)
#concepts.token           references  src/auth/token.ts#issueToken       inferred    (handler:, unique)
#concepts.refresh         extends     #concepts.token                    extracted   (<<: *token)
#concepts.refresh         references  #concepts.session                  extracted   (see: "#/concepts/session")
#policies.max-session-age references  #concepts.session                  extracted   (applies_to:)
```

(`#…` abbreviates `knowledge/auth.yml#…`. Line numbers assume the fixture is
written exactly as in §1 with the comment on line 1.)

## 7. Appendix B — files touched, by phase

| File | P0 | P1 | P2 | P3 | P4 |
| --- | :-: | :-: | :-: | :-: | :-: |
| `package.json` (dep `yaml`) | ✓ | | | | |
| `src/graph/document.ts` (new) | ✓ | ✓ | ✓ | ✓ | |
| `src/graph/document-resolve.ts` (new) | | | ✓ | ✓ | |
| `src/graph/types.ts` | ✓ | ✓ | | | |
| `src/graph/source-files.ts` | ✓ | | | | ✓ |
| `src/graph/build.ts` | ✓ | | | | ✓ |
| `src/graph/check.ts` | ✓ | | | | |
| `src/graph/map.ts` | ✓ | | | | |
| `src/graph/resolve.ts` | | | ✓ | | |
| `src/graph/invariants.ts` · `scripts/graph-quality.mjs` | | ✓ | | | |
| `src/graph/cards.ts` | | ✓ | | | |
| `src/graph/fingerprint.ts` · `src/util/state.ts` · `src/cli.ts` | | | | | ✓ |
| `viewer/data.ts` | | ✓ | | | |
| `src/mcp/tools.ts` · `src/claude/skill-template.ts` | | | | | ✓ |
| `README.md` · `CHANGELOG.md` | | | | | ✓ |
| `test/document-yaml.test.ts` (new) | ✓ | ✓ | ✓ | ✓ | ✓ |

## 8. Appendix C — how to work in this repo (for the implementer)

- Orientation: `graft map`, then `graft ask "<question>" --source`. Before
  editing a function: `graft callers <name>` to see every consumer.
- Run tests: `npm test` (all) or `node --import tsx --test test/document-yaml.test.ts` (one file).
- Type-check + build: `npm run build`. The `.scm` copy step in
  `scripts/build-viewer.mjs` is irrelevant to this tier (no query files).
- Style: file header doc-comment explaining *why* the module exists; comments
  explain decisions, not mechanics; tests are named after the behaviour they
  pin and reference the issue/plan that motivated them.
- Fixtures in tests are arrays of lines joined with `\n`, so expected line
  numbers are index + 1 (see `test/container-extract.test.ts` for the pattern).
