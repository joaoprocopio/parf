# Peritext: A CRDT for Collaborative Rich-Text Editing

> **Thesis.** Store formatting as a *separate, append-only set of marks* — each anchored to the stable identifiers of the first and last character it covers, with a per-boundary "expand / don't-expand" rule — and *derive* the visible formatting from those marks by a deterministic, per-mark-type last-writer-wins flattening. Because the marks live beside the text rather than inside it, concurrent formatting operations commute, the document converges, and the merge result matches what a human would expect.

**Source.** Geoffrey Litt, Sarah Lim, Martin Kleppmann, and Peter van Hardenberg. *Peritext: A CRDT for Collaborative Rich-Text Editing.* Proc. ACM Hum.-Comput. Interact. **6**, CSCW2, Article 531 (November 2022), 35 pages. https://doi.org/10.1145/3555644. Prototype: https://github.com/inkandswitch/peritext.

This paper does two distinct things. First it builds a **specification** — a test suite of concurrent-editing scenarios that pins down what "intent-preserving" rich-text merge *means*. Then it gives **Peritext**, the first published CRDT algorithm that satisfies that spec. The specification is as much a contribution as the algorithm, because before Peritext there was no agreed answer to "what is the *correct* merge of two concurrently-formatted documents?"

---

## TL;DR

- **The problem.** Real-time rich-text editors (Google Docs, etc.) merge concurrent edits, but all known rich-text algorithms are **Operational Transformation (OT)** and require a **central server**. That blocks peer-to-peer, offline editing, and Git-like branch/merge. CRDTs make concurrent ops commute and need no server — but there was **no published CRDT for rich text**, and the few open-source attempts fail to preserve user intent.
- **Why the naive approaches fail.** Three obvious designs each break:
  - **Tree CRDTs** (model formatting as deleting text and re-inserting it inside a bold/italic node) → concurrent bold + italic **duplicates the text**.
  - **Per-character attributes** (Ritzy: each character stores its own `{bold, italic, …}`) → a comment gets **split** by a concurrent insertion; heavy per-character metadata.
  - **Control characters** (Yjs: embed hidden `<b>`/`</b>` markers in the text) → concurrent overlapping bolds **leak formatting** to the rest of the document; "count starts vs ends" patches some cases but **breaks others**. The root flaw: control characters say *where* a span begins/ends but not *which formatting is newer*, so they cannot represent formatting changing over time.
- **The intent-preservation spec.** Eight worked scenarios define desired behavior, and a 2×2 taxonomy of marks: **can marks overlap?** and **do marks expand?** (bold/italic: no-overlap, expand; links: no-overlap, no-expand; comments: overlap, no-expand).
- **The Peritext model.** Plain text is an ordinary character-id CRDT (RGA). Formatting is a separate **append-only set of marks**. Each mark has a **start anchor** and **end anchor**, each pointing *before* or *after* a specific character id. `addMark` / `removeMark` are operations; nothing is ever deleted, only superseded.
- **Internal state — op-sets.** Each anchor position carries an **op-set**: exactly the marks that overlap that position. Applying a mark only touches the local region (`Algorithm 1`), so it's efficient.
- **Flattening.** To render, walk the spans; for each span and each `markType` independently, apply **last-writer-wins** by `opId` (a Lamport timestamp). Comments don't use LWW — they all coexist. This function is **deterministic and order-independent ⇒ convergence**.
- **Proof.** All concurrent operation pairs commute (case analysis over insert/remove/addMark/removeMark), so any two replicas with the same op-set reach the same state — **Strong Eventual Consistency**. Validated with property-based testing over 5,000-op random traces.
- **Scope.** Inline formatting only (bold, italic, font, color, links, comments). Block elements (headings, lists, tables) are explicitly future work.

---

## 1. The problem: decentralized rich-text merge

A collaborative editor needs a **collaboration algorithm** that merges concurrent edits to a shared document. Two families exist:

| | **Operational Transformation (OT)** | **CRDT** |
|---|---|---|
| Idea | Apply local op immediately; *transform* incoming ops against concurrent ones | Design ops so that concurrent ops **commute** — any order converges |
| Rich-text status (2022) | Widely deployed (Google Docs/Jupiter, CKEditor, Quill, ProseMirror) | **No published algorithm**; only buggy OSS attempts |
| Topology | All known rich-text OT needs a **central server** (Jupiter-style) | Peer-to-peer, decentralized |
| Offline | Hard: re-transform each op against every concurrent op — slow | Natural: merge whenever |
| Branch/merge | Many OT algorithms don't support arbitrary 3-way merges | Git-like: versions side-by-side, merge any pair |

```
OT model:                          CRDT model (what Peritext enables):

  userA  userB  userC                 v0
    \      |     /                    /  \
     \     |    /                   A      B        ← independent branches
      v  central  v               /  \    /  \
       server (linear)           AB   AC BA   ...   ← merge ANY two versions
       single timeline                \  /
                                     merged
```
*(Adapted from Fig. 1.)* The headline cost of OT is the central server; the headline gap in CRDTs is rich text. Peritext fills that gap. Note: the CKEditor team reports real-time rich-text collaboration took them ~**42 person-years** — this is genuinely hard.

> **Mental model:** plain-text CRDTs are a solved problem (Treedoc, WOOT, RGA, Causal Trees, Logoot, LSEQ, YATA) — they all give each *character* a unique id. The open question Peritext answers is: where does the **formatting** live, and how does it merge?

---

## 2. Why the obvious designs fail

The paper's first real contribution is showing — concretely — that the three intuitive ways to add formatting to a text CRDT all produce anomalies. These failures *motivate* the design and double as test cases.

### 2.1 Tree CRDTs duplicate text

Represent rich text as a tree (HTML/XML/JSON). To bold a word, you **delete** the word from its text node and **insert** a new `bold` node containing it. Now run concurrent bold + italic on the same word:

```
Initial:                 The fox jumped.

Alice bolds "jumped":    The fox [bold: jumped].
Bob italicizes "jumped": The fox [italic: jumped].

Merge (generic tree algorithm):
                         The fox [bold: jumped][italic: jumped].
                                       ^^^^^^^         ^^^^^^^
                                    DUPLICATED TEXT
```

The merge produces **`jumped` twice**. The tree manipulation modeled a *formatting* change as *delete + insert*, falsely signaling the user wanted to change the text. (Observed for real in Convergence.io + Froala.)

### 2.2 Per-character attributes split spans (Ritzy)

Ritzy stores formatting *on each character*: every character carries an `attributes` property like `{bold: true}`. Concurrent attribute writes to the same character resolve last-writer-wins. Two problems:

- **A concurrent insertion splits a comment.** If Alice attaches a *comment* to "fox jumped" while Bob inserts text in the middle, the comment is now stored on two non-contiguous runs of characters. A comment is conceptually a single contiguous annotation; splitting it is surprising. (Bold tolerates the split; comments do not.)
- **Cost.** Every character carries its full attribute set.

### 2.3 Control characters leak formatting (Yjs)

Yjs embeds **hidden control characters** — `start-bold` / `end-bold` (write them `<b>` / `</b>`) — into the character sequence. Text between `<b>` and `</b>` renders bold. This fixes the "insert in the middle" case (inserted text falls between the markers, so it's bold). But it breaks elsewhere.

**Overlapping bolds (Example, §2.3.2).** Alice bolds the first two words, Bob bolds the last two:

```
Alice:  <b>The fox </b>jumped.
Bob:    The <b>fox jumped.</b>

Merge (interleaving both users' markers):
        <b>The <b>fox</b> jumped.</b>
            └Alice └Bob  └Alice └Bob

Yjs renders by toggling bold on each marker, left to right:
        <b>The </b>fox jumped.      ← only "The fox " is bold!
```

Toggling on/off as you scan, the document ends up **less bold than either user intended** — Alice and Bob *together* bolded every character, so the whole sentence should be bold.

**The "count markers" patch — and why it also fails (§2.3.3).** You could instead count start-markers vs end-markers and render bold where starts exceed ends. This fixes the case above, but now:

```
Start:  <b>The fox jumped</b> over the dog.
Alice unbolds "jumped":     <b>The fox </b>jumped over the dog.
Bob unbolds "fox jumped"
   and bolds "dog":         <b>The </b>fox jumped over the <b>dog.</b>

Merge: "dog" is preceded by EQUAL starts and ends ⇒ NOT bold.
       The bolding of "dog" is silently LOST.
```

If you duplicate the initial start-marker to recover "dog", you lose the un-bolding of "fox". *No layering of fixes works.*

> **Key gotcha:** control characters record *where* a span begins and ends, but **not which formatting is older and which is newer**. When overlapping spans are toggled bold/non-bold over time, and several users do it concurrently, you cannot reconstruct intent from boundary markers alone. This is the central insight that drives the whole design: **formatting changes over time, and you must record that history, not just the current boundaries.**

(A fourth system, **Papyrus**, was discovered after publication; it independently took a similar marks-outside-the-text approach, but does **not** support per-boundary expand policies — §3.3.)

---

## 3. The specification: criteria for intent preservation

Before designing an algorithm, the paper writes down what a *correct* merge should do. The spec is **subjective by necessity** — it's a model of "least surprising to users," informed by how Word/Google Docs/Pages behave — but making it explicit lets you evaluate *behavior* separately from *implementation*. It is a **test suite** any rich-text merge algorithm can be run against.

A caveat carried over from plain-text CRDTs: this preserves **low-level syntactic intent** only. Semantic intent (does the merged sentence still *make sense*?) still needs human review. But maximizing low-level intent minimizes manual cleanup.

### 3.1 Insertion inside a concurrent format (Example 1)

Alice bolds the whole sentence; Bob concurrently inserts "brown" in the middle.

```
Alice: [bold: The fox jumped.]
Bob:   The brown fox jumped.

Desired merge:  [bold: The brown fox jumped.]
```

> **Rule:** formatting applies to any text **inside the range between two characters**, even text that did not exist when the formatting was applied. A span is a *range*, not a *snapshot of characters*.

### 3.2 Overlapping formatting

**Example 2 — same mark type overlapping.** Alice bolds "The fox", Bob bolds "fox jumped." They overlap on "fox", which both set to bold ⇒ **the whole text is bold**. (This is exactly the case Yjs got wrong.)

**Example 3 — different mark types overlapping.** Alice bolds "The fox", Bob italicizes "fox jumped." Bold and italic *coexist*, so the overlap is both:

```
   The      fox      jumped.
 [bold ][bold+italic][italic ]
```

This is the canonical figure: three spans — bold-only, bold-and-italic, italic-only.

**Example 4 — conflicting overlap (colors).** Alice colors "The fox" red, Bob colors "fox jumped" blue. A character **cannot be both** red and blue. There is no intent-preserving merge:

```
Alice: [red: The fox] jumped.
Bob:   The [blue: fox jumped].

Merge: in the overlap ("fox"), arbitrarily but DETERMINISTICALLY pick one color:
       [red: The fox][blue: jumped]    or    [red: The][blue: fox jumped]
```

The choice must be the **same on every replica** (deterministic) and must respect **last-writer-wins** if the color is later changed. This is the Thomas write rule [49]. Rejected alternatives: dropping one user's edit (too restrictive), or blending colors (invents a color nobody chose).

**Example 5 — conflicting bold/non-bold.** Even plain bold conflicts. Alice bolds everything then un-bolds "fox jumped"; Bob bolds only "jumped." "The" → bold, "fox" → non-bold, but "jumped" was set non-bold by Alice and bold by Bob ⇒ **conflict on "jumped"**, resolved LWW. Important subtlety: if a user toggled the word several times, **only the latest state** of each side is part of the conflict; earlier intermediate states are not.

**Example 6 — multiple instances of the same mark type (comments).** Alice and Bob comment on overlapping spans. Comments have *different content* — you can't merge them into one. And unlike color, a single character **can carry multiple comments**. So comments are rendered as **overlapping highlight regions**, both retained.

```
   The      fox      jumped.
 [-- Alice's comment ---]
        [---- Bob's comment ----]
```

### 3.3 Text insertion at span boundaries — the "expand" question (Examples 7–8)

When you type new text, what formatting does it get? Inside a span (Example 1) it inherits the span. At a **boundary** it's subtler.

**Example 7 — bold span.** Document: `The [bold: fox jumped]. ` Alice inserts "quick " before the bold span and " over the dog" before the final period.

```
The quick [bold: fox jumped over the dog].
     ^^^^^                  ^^^^^^^^^^^^^
   NON-bold                    bold
```

> **Rule (most marks — bold, italic, underline, font, size, color):** an inserted character **inherits the bold/non-bold status of the *preceding* character**. So text before a bold span stays non-bold; text appended at the *end* of a bold span becomes bold — the span **expands**. (Exception: at the very start of a paragraph, inherit from the *following* character.)

**Example 8 — link / comment span.** If "fox jumped" is a *link* and Alice inserts the same way, major editors put the new text **outside** the link on both ends:

```
The quick [link: fox jumped] over the dog.
                              ^^^^^^^^^^^^^^
                           NOT part of link
```

> **Rule (links, comments):** the span does **not** expand at its end. Word/Docs/Pages are remarkably consistent here, suggesting it's deliberate. (If a link-end and bold-end fall on the same character and you insert after it, Word makes the new text bold-not-linked; Docs/Pages make it neither.)

### 3.4 The mark taxonomy (Table 1)

Two independent axes characterize every mark type:

| Mark types | **Can marks overlap?** (one char, multiple marks of this type) | **Do marks expand?** (typing at the end grows the mark) |
|---|---|---|
| **Bold, italic, colored text** | No | **Yes** |
| **Links** | No | No |
| **Comments** | **Yes** | No |

To a collaboration algorithm, a mark type is just *a configuration of these two flags*; rendering is the UI layer's concern. A developer is free to reconfigure (e.g. allow colors to overlap and blend).

---

## 4. The Peritext algorithm

Four parts: (1) the underlying plain-text CRDT; (2) generating formatting operations; (3) applying them into internal state; (4) deriving a displayable document.

The guiding principle: **capture user input — and thus intent — as literally as possible.** Typing/pasting → `insert`; backspace/delete/overwrite → `remove`; selecting text and choosing a format → `addMark` / `removeMark`. A **mark** is any property applied to a contiguous substring (a bold word, a commented sentence).

### 4.1 The text layer (RGA) and opIds

The text is an ordinary character-id sequence CRDT — Peritext uses **RGA / Causal Trees** but any would do. Every operation gets a unique immutable **opId**, a **Lamport timestamp** written `counter@nodeId`:

- `counter` = one greater than the max counter of any existing operation in the document (from any client);
- `nodeId` = a unique client id (UUID).

Ordering: `c1@n1 < c2@n2` iff `c1 < c2`; ties broken by string comparison of node ids. (Concurrent ops can share a counter, but `counter+nodeId` is globally unique because a client never reuses a counter.)

**Insert** references the id of the character to insert *after* (ids are stable; indices are not):

```json
{ "action": "insert", "opId": "2@alice", "afterId": "1@alice", "character": "x" }
```

`afterId: null` inserts at the start. Concurrent inserts with the same `afterId` are ordered by opId (RGA tie-break).

**Remove** marks a character deleted — it leaves a **tombstone**, never physically removed, so later inserts that reference the deleted id still resolve:

```json
{ "action": "remove", "opId": "5@alice", "removedId": "2@alice" }
```

```
State per character = (opId that inserted it, the char, deleted-flag)

  1@A  2@A  3@A  4@A  5@A  6@A  7@A   8@B✗  9@B  10@B ...
   t    h    e   ' '   f    o    x     t    T   ' j'...
                                       └ tombstone (Bob deleted 't', typed 'T')
  Visible text:  "The fox jumped."
```
*(Fig. 2.)*

### 4.2 Marks and anchors

> **The core idea.** Formatting is **not** stored in or on the characters. It is a separate, **append-only set of marks**. Each mark is an `addMark` or `removeMark` operation that names a span by the **ids** of its endpoint characters — plus, crucially, *which side* of each endpoint it attaches to.

**Anchors.** Each character has **two anchor positions**: one **before** it and one **after** it. A formatting change always happens *in the gap between two characters*, and the gap is named by an anchor. The before/after choice is exactly what encodes the **expand** behavior of §3.3.

```
   ┌───┬───┬───┬───┬───┐
 · │ T │ h │ e │   │ f │ ·          ← each '·' / '│' is an anchor gap
   └───┴───┴───┴───┴───┘
 before(T)  ...  before(f) after(f)

A mark's start/end each = { type: "before"|"after", opId }   (or startOfText/endOfText)
```
*(Fig. 3. Each character has a before-anchor and an after-anchor; which one a mark uses decides whether it expands when text is inserted on the boundary.)*

**addMark** — bold "fox jumped" (Fig. 3). Start `before 5@A` (the `f`), end `before 17@B` (the final period). The span **includes the start character, excludes the end character** — i.e. `[start, end)`:

```json
{
  "action": "addMark", "opId": "18@A",
  "start": { "type": "before", "opId": "5@A" },
  "end":   { "type": "before", "opId": "17@B" },
  "markType": "bold"
}
```

**removeMark** — we **never delete** an operation; ever. To un-bold, generate a `removeMark` over a span. It sets that span to non-bold (and can start/end on *any* character regardless of current formatting):

```json
{
  "action": "removeMark", "opId": "20@A",
  "start": { "type": "before", "opId": "10@A" },
  "end":   { "type": "before", "opId": "17@B" },
  "markType": "bold"
}
```

**Encoding the expand rules with before/after.** This is where the two anchors earn their keep:

| Mark behavior | `addMark` start / end | `removeMark` start / end |
|---|---|---|
| **Expands** (bold, italic, …) | `before` / `before` | `before` / `before` |
| **Doesn't expand** (link, comment) | `before` / `after` | `after` / `before` |

- A **bold** addMark ending `before` the period means: text inserted between the last bold char and the period falls *inside* the span → becomes bold (span expands). Text inserted before the start char falls outside → non-bold.
- A **link** addMark ending `after` its last character means: text inserted after that character falls *outside* the link → non-linked (no expansion). See Fig. 5.
- For removeMark the roles flip so the boundary stays consistent whether bold ends because an addMark ended or a removeMark began.

To change a link's URL: emit a new `addMark` with a new `url` and the *same* start/end. A comment stores only a unique `commentId` as its `markType`; the comment body/author/timestamp live in a separate CRDT.

> **Design lesson:** by giving every character *two* attachable positions and letting each mark choose, the same uniform machinery expresses bold (expands) and links (doesn't) — the expand/no-expand axis of Table 1 becomes a one-bit choice per anchor, not special-case code.

**Tombstone subtlety at link ends (§4.2.2, Fig. 6).** If the character a link's end-anchor points to gets deleted (becomes a tombstone) and you then insert at that position, RGA by default places the new char *before* tombstones — which would pull it *inside* the link. Fix: when inserting where tombstones exist, scan them; if any tombstone's after-anchor is the start/end of a mark, insert *after the last such tombstone*. This keeps end-of-link insertions outside the link. (In rare multi-anchor cases no ideal spot exists; the user may have to fix formatting manually.)

### 4.3 Applying operations: op-sets

Insert/remove use plain RGA. The new machinery is for `addMark` / `removeMark`, and it must be **commutative** (any order → same state).

> **Op-set.** At each anchor position we may store an **op-set**: *exactly the set of addMark/removeMark operations that overlap that anchor* — i.e. every op that starts **at or before** this position and ends **after** it.
>
> An op-set may be **absent** (`null`), meaning "same as the nearest present op-set to the left." Absent ≠ present-but-empty. Op-sets are stored only at *boundaries where formatting changes* — a compression: the same op-set conceptually applies to the whole contiguous run of absent positions following it.

**Algorithm 1 — applying a mark.** Update the op-set at the **start**; add the op to every present op-set strictly **within** the span; create an op-set at the **end** position (without the op) to mark the formatting boundary. `FindPrevious` copies the nearest left op-set when a boundary is newly created.

```python
def apply_op(op):                       # op is an addMark or removeMark
    start, end = op.start, op.end        # anchor positions

    # 1. start position: ensure an op-set exists, then add op
    if ops_at[start] is None:            # absent → seed from the left, then add
        ops_at[start] = find_previous(start) | {op}
    else:
        ops_at[start] = ops_at[start] | {op}

    # 2. interior: add op to every PRESENT op-set strictly inside (start, end)
    for pos in positions where start < pos < end:
        if ops_at[pos] is not None:
            ops_at[pos] = ops_at[pos] | {op}

    # 3. end position: mark a formatting boundary WITHOUT this op
    if ops_at[end] is None:
        ops_at[end] = find_previous(end)      # note: op is NOT added here

def find_previous(pos):                  # nearest present op-set to the left
    while pos is not None:
        if ops_at[pos] is not None:
            return ops_at[pos]
        pos = pos.prev
    return set()                          # none found → empty set
```

**Invariant maintained:** each op-set contains exactly the mark operations pertaining to the span of characters that follows it.

**Worked merge (Examples 2–3, Figs. 7–10).** Start: `The fox jumped.`, unformatted.

```
Alice bolds "The fox" (addMark 18@A, start=before 9@B 'T', end=before 10@B):

  [{bold18}] T h e   f o [{}] x   j u m p e d .
   └ op-set at before-T    └ empty op-set at the boundary

Bob concurrently italicizes "fox jumped." (addMark, start=before 'f', end=endOfText):

  T h e   [{ital}] f o x   j u m p e d . [{}]

Alice applies Bob's italic op (Fig. 9). Result op-sets:

  before-T:  {bold}          → span "The "        : bold
  before-f:  {bold, ital}    → span "fox"         : bold + italic
  before-' ':{ital}          → span " jumped."    : italic
```

Each op-set now records the live marks for the run that follows it. Mirrors Example 3 exactly: bold-only, bold+italic, italic-only. The application is **commutative** (proved in §6) and **efficient** — it only scans the affected region, never the whole document.

> **Performance note (footnote):** the very first mark forces `FindPrevious` back to the document start. To bound this, *seed* a copy of the nearest preceding op-set onto ~1 in 1,000 randomly-chosen characters at insert time. Correctness is unaffected, but backward scans now stop after ~1,000 characters on average.

### 4.4 Producing the final document: flattening

Op-sets subdivide the document into spans, each with a *set* of mark operations — but a span's op-set may hold many adds/removes of the *same* `markType` (toggled bold/non-bold repeatedly). We must collapse each set into a **current formatting state**, deterministically and order-independently.

```
For each span, for each markType independently:

   bold:   { addMark 19@A,  removeMark 23@B }   ── LWW by opId ──▶
           23@B > 19@A  ⇒  removeMark wins  ⇒  NON-bold

   italic: { addMark 12@A }                     ⇒  italic
```

```python
def flatten_span(op_set):
    fmt = {}
    by_type = group_by(op_set, key=lambda op: op.markType)
    for mark_type, ops in by_type.items():
        if mark_type_allows_overlap(mark_type):        # comments
            # keep every comment with no corresponding removeMark
            fmt[mark_type] = [o for o in ops if not removed(o, ops)]
        else:                                          # bold, italic, link, color
            winner = max(ops, key=lambda op: op.opId)  # last-writer-wins
            if winner.action == "addMark":
                fmt[mark_type] = winner.value          # e.g. True, or the url
            # else removeMark wins → attribute absent
    return fmt
```

Output for the worked example:

```json
[
  { "text": "The ",     "format": { "bold": true } },
  { "text": "fox",      "format": { "bold": true, "italic": true } },
  { "text": " jumped.", "format": { "italic": true } }
]
```

Why this converges: **each `markType` is handled independently**, and within a type the winner is the op with the **maximum opId** — a total order computed identically everywhere. Since `counter` always exceeds every prior op, toggling bold/non-bold repeatedly leaves the *latest* toggle as winner. **Comments don't use LWW** — they're overlap-allowed, so all comments lacking a matching removeMark are retained, keyed by unique commentId so they never collapse into one.

> **Mental model:** the op-set is the *history*; flattening is a pure function `history → current format` that any replica can run at any time and get the same answer. Convergence reduces to "the flatten function depends only on the *set* of ops and a total order on opIds — not on arrival order."

A consequence noted by the authors: you don't *strictly* need to keep the whole history to render — the latest value + opId per (span, markType) suffices. But keeping the full op history is what would later enable **version diffing** (visualizing what changed between versions), a planned feature.

Optimizations while rendering: skip spans containing only tombstones (invisible in a WYSIWYG editor), and merge adjacent spans with identical formatting.

### 4.5 Incremental patches

Re-flattening the whole document on every keystroke is too slow, and editor UIs (ProseMirror) are stateful — they want *what changed*, not a fresh document. So applying an op also emits **patches**:

- **insert**: use RGA to find the position, compute the visible index (count non-deleted predecessors), search backward for the nearest op-set, flatten it to get the inserted char's formatting, emit `{type:"insert", char, index, format}`.
- **remove**: find by opId; if already deleted, no-op; else mark deleted and emit a delete-at-index patch.
- **addMark/removeMark**: for each span touched, flatten the *old* and *new* op-set; if formatting changed, emit a formatting patch. A neat consequence: bolding the whole document may emit patches for **only some sub-spans**, if concurrent ops already negate the effect elsewhere.

```
Patch (index-based, UI-facing):     { "type":"insert", "char":"x", "index":6, "format":{ "bold":true } }
Operation (opId-based, network):    { "action":"insert", "opId":"…", "afterId":"…", "character":"x" }
```

> **Key gotcha:** **patches use indices; operations use opIds.** Indices are fine for the UI because the CRDT runs on the same thread as the editor — no concurrency between them. Operations sent to *other users* must use opIds, since indices are not stable under concurrent edits. Implementations are checked by accumulating incremental patches and comparing against the simpler from-scratch flatten.

### 4.6 Prototype and performance

TypeScript prototype atop a simplified Automerge; editor UI on ProseMirror. **Property-based testing**: random edit traces of **5,000+ operations** across **3 peers**, asserting convergence — this caught several real bugs in earlier versions. Marks tested: bold, italic, link, comment.

Simplicity was chosen over performance in places: one object per character (memory-heavy), all tombstones and all formatting history kept forever ⇒ **unbounded storage growth**. Mitigations:

- Store only the *value + highest opId* per (anchor, markType) instead of the full op-set (drop overwritten ops).
- RGA's tombstone GC applies; when a tombstone is purged, move its attached marks to the nearest surviving neighbor; drop a span once all its chars are purged.
- Use a tombstone-free text CRDT (Logoot) instead of RGA — formatting still attaches to character ids. A span can only be dropped once **causal stability** guarantees no future concurrent insertion can resurrect it.
- Even without GC, Automerge's compressed format stores history at **< 1 byte per operation**, so "keep everything" is cheaper than it sounds.

---

## 5. Correctness model

Peritext targets the standard collaborative-editing correctness triple of Sun et al. [45]:

| Property | Statement | How Peritext gets it |
|---|---|---|
| **Convergence** | Same set of ops applied ⇒ identical document state | All concurrent ops commute (§6) — i.e. **Strong Eventual Consistency** |
| **Causality preservation** | If `o0 → o1` then `o0` is applied before `o1` everywhere | Standard causal-broadcast machinery (vector clocks or ordered logs) |
| **Intention preservation** | Each op's effect matches its intent and doesn't disturb concurrent ops | The §3 examples (subjective; argued case-by-case) |

### 5.1 Causality preservation

Peritext is a **monotonically growing set of operations**: deleting text/formatting only *adds* `remove`/`removeMark` ops; nothing ever shrinks the set. A document **version** = the set of all ops applied to it from the empty start; by convergence that set uniquely determines the state. **Merging two versions = union of their op-sets.**

Define the causal order: `o0 → o1` iff `o1` was generated in the context of a version `V` and `o0 ∈ V`. While the op-set grows monotonically this `→` is a strict partial order. Enforce "apply `o0` before `o1` whenever `o0 → o1`; concurrent ops in any order" by either vector clocks on ops, or an ordered op-log merged by appending the other log's not-yet-seen ops in their original order. These are well-established; the paper defers to the literature.

### 5.2 Intention preservation

"Intention" (per Sun et al.) = the effect achievable by applying op `o` on the state it was generated from. The definition is admittedly loose — it leaves ambiguous cases (e.g. an insert between two characters that were both concurrently deleted; or bold vs non-bold on the same word, where *no* convergent state preserves both intents). The authors' operational reframing:

> **The intention-preservation question, restated:** *when two documents are merged, which outcome is least surprising to users?* This is what drove the §3 example exploration; the paper argues each example's chosen outcome is plausible, and shows Peritext achieves all eight (Examples 1–8). A finite example set is not a proof of universal intent-preservation, but it builds confidence.

A mapping of how each scenario resolves: Ex.1 range-not-snapshot; Ex.2 three spans all bold; Ex.3 bold/italic independent → bold+italic middle; Ex.4 LWW color in overlap; Ex.5 LWW bold in overlap; Ex.6 comments coexist via distinct markTypes; Ex.7 before/before anchors expand bold; Ex.8 before/after anchors keep links from expanding (plus the tombstone-scan rule).

---

## 6. The convergence proof

**Theorem A.1.** For any two versions of a Peritext document, if the **same set of operations** has been applied to both, the documents are in the **same state**.

**Proof skeleton.** Let `L1`, `L2` be the two application orders. Same set, each op applied once ⇒ `L2` is a **permutation** of `L1`. Causality preservation ⇒ if `o0 → o1` then `o0` precedes `o1` in *both* logs. Lemma A.9: you can transform `L1` into `L2` by repeatedly swapping **adjacent concurrent** ops only. Lemma A.2: concurrent ops **commute**, so each swap leaves the resulting state unchanged. Hence both orders yield the same document. ∎

The work is **Lemma A.2** — every pair of concurrent operations commutes — proved by exhaustive case analysis over the four operation types. Let `V` contain all causal dependencies of the two concurrent ops `o0`, `o1`:

| Concurrent pair | Why they commute |
|---|---|
| **insert ∥ insert** | RGA insertions commute (proved in prior work [13, 26]); formatting additions don't touch insertion logic |
| **remove ∥ remove** | Each just sets a deleted-flag; setting flags is order-independent whether same or different chars |
| **insert ∥ remove** | If concurrent, the remove can't target the just-inserted char (else `→`, not concurrent) ⇒ disjoint effects |
| **(addMark\|removeMark) ∥ (addMark\|removeMark)** | Lemma A.8 (the op-set `Algorithm 1` is order-independent) |
| **remove ∥ (addMark\|removeMark)** | Remove only flags deletion; formatting effect is independent of the deleted-flag ⇒ commute |
| **insert ∥ (addMark\|removeMark)** | See below — the key case |

**The insert ∥ mark case (intuition).** Because `V` holds the mark's dependencies, the mark's start/end **anchor characters already exist** in `V`. Since `o0` (insert) and `o1` (mark) are concurrent, the inserted character is **not** an anchor of `o1`. Two sub-cases:

- **Insert outside the mark's span** → the mark only touches chars within its span, so the two don't interact; order is irrelevant.
- **Insert inside the span.** If the mark runs first then the insert: the new char lands with **absent op-sets** (inserts carry no formatting), unaffected by the mark. If the insert runs first then the mark: the mark scans its span — but `Algorithm 1` **only modifies op-sets that are present**, and the freshly inserted character's op-sets are *absent*, so the mark **does not modify** the new character, and acts identically on all other characters as if the insert hadn't happened. Either order ⇒ same state. ∎

> **Why "absent" matters.** The whole proof of the hardest case hinges on the absent-vs-present distinction. A freshly inserted character has *absent* op-sets; `Algorithm 1` skips absent positions in its interior loop. That single design choice is what makes concurrent insert-and-format commute — the inserted char simply *inherits* the surrounding op-set at flatten time, regardless of operation order.

(Anchors get a **total order** (Def. A.3–A.4): for a character id `c`, `before(c) < after(c)`; and `after(c1) < before(c2)` whenever `c1` precedes `c2` in the document — this underpins the op-set lemmas.)

---

## 7. Limitations and future work

- **Inline formatting only.** Bold, italic, font, size, color, links, comments — all *within a paragraph*. **Block elements** (headings, bullet/numbered lists, blockquotes, tables) are explicitly out of scope and raise new intent questions: what happens when users concurrently *split, join, and move* block elements? Future paper.
- **Not a complete collaboration system.** It merges two versions automatically, but a full asynchronous-collaboration system also needs **version-diff visualization**, **conflict highlighting** for manual resolution, etc. The op-log basis is meant to support these later.
- **Moving / duplicating text.** If two users concurrently cut-paste the *same* text to different places then edit it, the most sensible outcome is unclear — open problem.
- **Conflicts are real and sometimes unresolvable.** Bold vs non-bold, red vs blue on the same character have no intent-preserving convergent answer; Peritext picks deterministically (LWW) and leaves manual correction to the user. Intention preservation is **subjective** — the spec is "least surprising," not provably optimal.
- **Storage growth.** Keeping all operations + tombstones forever is unbounded; GC needs causal stability and is only fully live when replicas are reachable (the usual CRDT tax).
- **Semantic intent.** Like all CRDTs, Peritext preserves *syntactic* low-level intent; whether the merged prose still *makes sense* is the human's job.

---

## 8. The one-paragraph distillation

Don't put formatting *in* the text (control characters leak; they can't tell old from new formatting) and don't put it *on* each character (comments split; metadata bloats). Put it **beside** the text: an append-only set of **marks**, each a span named by the **stable ids** of its endpoint characters, each endpoint attached to a **before- or after-anchor** that encodes whether the mark **expands** when you type at the boundary. Keep, at every formatting boundary, an **op-set** of the marks overlapping there (`Algorithm 1` updates only the local region). To render, **flatten** each span: handle every `markType` independently and pick the **highest-opId** operation (last-writer-wins) — except overlap-allowed marks like comments, which all survive. Because flattening depends only on the *set* of operations and the *total order* of their opIds, replicas converge; because every concurrent operation pair commutes (the crux: a freshly inserted character has *absent* op-sets, so a concurrent format leaves it untouched), Peritext is a true CRDT — decentralized, offline-capable, branch-and-merge rich text with no central server.

---

## Glossary & notation

| Term | Meaning |
|---|---|
| **Mark** | A formatting property (`bold`, `italic`, `link`, `comment`, …) applied to a contiguous substring; recorded as an `addMark`/`removeMark` operation, stored *outside* the text. |
| **markType** | The kind of a mark (`bold`, `link`, a unique `commentId`, …). Each markType is flattened independently. |
| **addMark / removeMark** | Operations that set / unset a mark over a span; never deleted, only superseded. |
| **opId** | Globally-unique operation id, a Lamport timestamp `counter@nodeId`; `counter` = (max existing counter)+1; total-ordered (counter, then nodeId). |
| **Anchor (position)** | A point *before* or *after* a specific character id (or `startOfText`/`endOfText`) where a span endpoint attaches. Every character has a before- and an after-anchor. |
| **before / after** | Which anchor a span endpoint uses; encodes whether the mark **expands** when text is inserted at the boundary. |
| **Expand** | Whether typing at a mark's end grows the mark (bold: yes; link/comment: no) — Table 1, axis 2. |
| **Overlap (allowed)** | Whether one character may carry multiple marks of the same type (comments: yes; bold/link: no) — Table 1, axis 1. |
| **Span** | A maximal run of characters with the same active op-set / formatting. |
| **Op-set** | The set of mark operations overlapping a given anchor (start ≤ here, end > here). Stored only at formatting boundaries; **absent** (`null`) means "same as nearest left op-set." |
| **Absent vs empty op-set** | Absent = inherit from the left (e.g. a freshly inserted char); empty present = explicitly no marks. The distinction makes concurrent insert∥format commute. |
| **Tombstone** | A character marked deleted but kept, so insert positions referencing it still resolve (RGA). |
| **Flattening** | Pure function: an op-set → current `{markType: value}`, via last-writer-wins (max opId) per type; comments retained as a list. |
| **LWW / Thomas write rule** | Conflict resolution: the operation with the maximum opId wins, deterministically, on every replica. |
| **Patch** | An **index-based** UI-facing edit describing what changed; distinct from an **opId-based** network operation. |
| **RGA** | The plain-text sequence CRDT Peritext builds on (≈ Causal Trees); any character-id text CRDT would work. |
| **Causal stability** | The condition under which no future concurrent op can affect a span/tombstone, enabling safe GC. |
| **Convergence / SEC** | Same set of ops applied ⇒ identical state; Strong Eventual Consistency, proved via all-concurrent-ops-commute. |
| **Intention preservation** | Subjective: the merge is "least surprising to users," validated against the eight §3 examples. |
| `o0 → o1` | `o0` happened-before `o1` (was in the version `o1` was generated against). |
| `o0 ∥ o1` | concurrent: neither happened-before the other. |
