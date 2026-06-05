# DSON: a JSON CRDT built from delta-mutations

> **Thesis.** Treat a JSON document as a tree of *nested* δ-state CRDTs (maps, arrays, registers), give each one **Observed-Remove** semantics over a shared causal context, and the result converges with no synchronisation **and** stores metadata bounded by the document size — not by the number of updates. The missing piece prior work lacked was a *composable, causal, tombstone-free δ-based array*; DSON supplies it.

**Source.** Arik Rinberg, Tomer Solomon, Roee Shlomo, Guy Khazma, Gal Lushi, Idit Keidar, Paula Ta-Shma. *DSON: JSON CRDT Using Delta-Mutations For Document Stores.* Proceedings of the VLDB Endowment (PVLDB), Vol. 15, No. 5, pp. 1053–1065, 2022. doi:10.14778/3510397.3510403. (Technion / IBM Research. Source code: `crdt-ibm-research/json-delta-crdt`.)

This paper sits one rung *above* the CRDT-foundations paper: it assumes you accept δ-state CRDTs and asks "how do I build a *whole JSON document type* out of them, suitable for a long-lived document **store** rather than a collaborative text editor?" The novel technical core is a δ-based Observed-Remove **Array** with a clever position-as-a-forest encoding that supports concurrent **move** without tombstones.

---

## TL;DR

- **The problem.** Distributed JSON document stores (CouchDB, Couchbase, DynamoDB) default to *eventual* consistency and resolve conflicts ad-hoc ("most-mutations wins") or push merge logic onto the application developer — error-prone and hard to reason about.
- **The goal.** A *full* JSON CRDT giving **Strong Eventual Consistency** (SEC) + causal consistency + read-your-writes, with **well-defined, intuitive** conflict semantics, **and** metadata that stays bounded over an arbitrarily long document lifetime.
- **Why δ-state.** Op-based CRDTs need reliable, exactly-once, ordered delivery (hard even over TCP). State-based ship the whole state (too big). **δ-state CRDTs** (Almeida et al.) ship *deltas* — small messages that are themselves elements of the same join-semilattice as the state — and *naturally bound metadata growth*.
- **Why prior JSON CRDTs don't fit.** Automerge (RGA-based) and Yjs (YATA-based) optimise for *collaborative text editing*: they keep **tombstones** and history proportional to the **number of updates**, which cannot be safely garbage-collected. Unacceptable for long-lived documents.
- **JSON as nested CRDTs.** A document is recursively `Map | Array | Register`. DSON = **MVReg** (register) + **ORMap** (map, from Almeida et al.) + a **brand-new ORArray** — all composable, all sharing one causal context.
- **The novel ORArray.** Array = map from a *unique id* (a dot) to a `(value, set-of-positions)` pair. Positions are stable identifiers (real numbers à la Logoot/LSEQ). The position set is stored as a **depth-3 forest** inside a new dot-store, the **CompDotFun**, which is what makes `apply`/`move`/`delete` commute correctly and supports **concurrent move without data duplication or tombstones**.
- **Observed-Remove everywhere.** A `delete`/overwrite only removes state the operation has *observed*; a concurrent update therefore *survives*. Intuitive: "you can only remove what you've seen."
- **Metadata bound.** $O(D + n\log n)$ in steady state, worst case $O(k^2 D + n\log n)$, where $n$ = replicas, $D$ = document elements, $k \le n$ = concurrent operations. Crucially, **independent of the number of updates**.
- **Evaluation.** A 2500-LOC JavaScript prototype (Automerge-style API). Versus Automerge and Yjs: DSON's metadata stays **constant** as updates accumulate, where the others grow linearly or unboundedly. Worst-case $O(k^2)$ confirmed empirically but shown to be rare.

---

## 1. The problem: document stores want availability *and* sane conflicts

NoSQL document stores are dominant, and most adopt a **JSON** data model — it maps to programming-language data structures and relaxes schema. They are deployed at global scale and in mobile/edge settings, where **high availability under disconnected operation** is paramount. So they default to **eventual consistency**.

But EC only promises that state *will* converge "at some unspecified future point if updates stop." It says nothing about *what* the converged value is. Real systems paper over this with arbitrary policies:

- Couchbase: "the replica with the most mutations wins."
- Many stores: hand the developer an API to *manually* merge conflicts — time-consuming, error-prone, hard to reason about.

> **The pitch.** Adopt CRDTs so that *any two nodes that received the same set of updates are in the same state* — with **predetermined, intuitive** conflict resolution baked into the data type, not bolted on per-application. Add causal consistency and read-your-writes on top. The developer stops reasoning about conflict resolution entirely.

The new constraint that distinguishes a document *store* from a collaborative *editor*: documents have **arbitrarily long lifetimes**, so the CRDT metadata must be **reasonably bounded** — it cannot grow forever with edits.

---

## 2. Why δ-state, and why existing JSON CRDTs fail the store

### 2.1 The three CRDT replication styles

| Style | Ships | Pro | Con |
|---|---|---|---|
| **Op-based** [Baquero et al.] | individual operations | simple, small messages | assumes **reliable, exactly-once, ordered** delivery — hard to maintain even over TCP |
| **State-based** [Letia et al.] | the entire state | robust to lossy/dup/reordered channels | messages are the **whole state** — prohibitively large |
| **δ-state** [Almeida et al.] | a **delta** (an element of the same lattice as the state) | small messages **and** lattice robustness; **naturally bounds metadata** | needs **causal** delivery guarantees; designing causal δ-CRDTs is non-trivial |

DSON chooses **δ-state** because it is the only style that gives both small messages and a path to *bounding* metadata. The price is that you must engineer causal consistency yourself — which is exactly the technical contribution.

> **Mental model.** A δ-CRDT update is "compute the *increment* from old state to new, ship only the increment." Old state $X$, mutator produces delta $X^\delta$, new state is $X \sqcup X^\delta$. Because deltas live in the *same join-semilattice* as states, "apply a delta" and "merge a peer's state" are the **same** join operation — idempotent, commutative, associative. Duplicate, reordered, or re-sent deltas are all harmless.

### 2.2 Why Automerge and Yjs are wrong for a store

Both are excellent *collaborative editors*, but their array cores keep history/tombstones proportional to edit count:

- **Automerge** (op-based, [Kleppmann & Beresford]): array is **RGA** — a linked list where each node has an id, a next-id, a value, and a **tombstone flag**. Nesting is hacked in by storing nested-CRDT ids in value fields. RGA *inherently* needs tombstones; **garbage collection cannot be done without compromising consistency**. Also, when two replicas concurrently write the same key, Automerge picks one arbitrarily and dumps the rest into a `conflicts` structure as **raw values (not CRDTs)**.
- **Yjs** (YATA, [Nicolaescu et al.]): array is a doubly-linked list; an `update` is faked by inserting the new value at the array's front (a "Replace Manager"). Deletes are handled with a **state-based tombstone CRDT** plus a **time-based GC** and ad-hoc optimisations.

> **Design lesson the paper drives home.** Tombstones + history-keeping = metadata that grows with the **number of updates** and resists garbage collection. For a long-lived store that is fatal. δ-based CRDTs with a **causal context** need **no GC at all** — the causal context *is* the mechanism that lets removed state vanish safely.

DSON's contributions in one breath: a formal δ-based **OR-JSON** CRDT, built on a **novel composable, causal, tombstone-free δ-based ORArray** that supports **concurrent move** and **arbitrary nesting** without data duplication; a proof of its convergence and the correctness of its semantics; and an $O(k^2 D + n\log n)$ metadata bound verified empirically.

---

## 3. The δ-state machinery you need first

### 3.1 Model and happens-before

Nodes $\mathcal{I}$ host replicas; "node" and "replica" are interchangeable. They communicate by **broadcasting over unreliable, unordered links** with two assumptions: (1) every delivered message was previously broadcast (no forgery), and (2) a message broadcast infinitely often is eventually delivered by every node. Notation: $\mathit{bcast}_i(m)$, $\mathit{deliver}_j(m,i)$.

An **execution** $\sigma$ is an interleaving of events and states; its **trace** $Tr(\sigma)$ is the event subsequence. Lamport's **happens-before** $e_1 \prec_\sigma e_2$ holds if $e_1$ precedes $e_2$ at the same replica, or $e_1$ broadcasts a message that $e_2$ delivers, or by transitivity. $\mathit{Ops}(\sigma)$ = the API invocations. Concurrent: $e_1 \,||_\sigma\, e_2$ when neither precedes the other.

CRDT semantics are specified over the **poset** $(\mathit{Ops}(\sigma), \prec_\sigma)$ — the operations and their happens-before order — *not* over a concrete execution. This is the move that lets DSON say "what does a `get` return" abstractly, independent of message timing.

**The reference object — MVReg.** A Multi-Value Register exposes `write(x)` and `read()`. Over $\sigma$:

$$
\mathit{read}() = \{\, v \mid \exists\,\mathtt{write}(v) \in \mathit{Ops}(\sigma)\ \wedge\ \nexists\,\mathtt{write}(u): \mathtt{write}(v) \prec_\sigma \mathtt{write}(u) \,\}
$$

i.e. **read returns every value whose write was not overwritten by a *causally later* write**. Concurrent writes both survive: $\mathtt{write}_i(v_1) \,||\, \mathtt{write}_j(v_2)$ → a later read returns $\{v_1, v_2\}$. A subsequent `write(v3)` collapses it back to $\{v_3\}$. This is the seed of all OR semantics in the paper.

### 3.2 Join-semilattice, dots, and the causal context

A **join-semilattice** (here just "lattice") is $(S, <, \sqcup)$ with $\sqcup$ associative, commutative, idempotent. In a δ-CRDT **both** the state **and** the broadcast deltas are elements of $S$.

Operations get unique ids. Replica $i$ mints the sequence $(i,1),(i,2),\dots$; each such pair is a **dot**. A dot is **observed** by $i$ if $i$ generated it or delivered a message containing it. The set of observed dots at a replica is its **causal context** $c$:

$$
\text{CausalContext} \in P(\mathcal{I} \times \mathbb{N})
$$
$$
\max_i(c) = \max(\{n \mid (i,n) \in c\} \cup \{0\}), \quad
\mathrm{next}_i(c) = (i, \max_i(c)+1), \quad
\mathrm{insert}_i(c,d) = c \cup \{d\}
$$

The causal context is a **grow-only set**, so unbounded in principle — but **compressible**: since dots are sequential, store a `prefix : I ↦ N` (if `prefix[i] = n` then all $(i,m), m\le n$ are present) plus a small uncompressed set of out-of-order dots beyond the prefix. The unreliable network can leave gaps (you may have $(i,1)$ and $(i,3)$ but not $(i,2)$), which is why the extra set exists. Compressed, the causal context is $O(n \log n)$.

> **Key idea.** The causal context replaces tombstones. To "remove" something, you add its dot to the causal context; the join then makes any copy of that dot **fail to survive**. Nothing has to be kept as a gravestone — the *absence* of a dot from a map, combined with its *presence* in the causal context, encodes "deleted."

### 3.3 Dot stores: DotFun and DotMap

A **DotStore** holds data-type-specific state and exposes `dots(s)` = the set of dots it currently stores. Two primitives from Almeida et al.:

$$
\textbf{DotFun}\langle V:\text{Lattice}\rangle : \mathcal{I}\times\mathbb{N} \mapsto V, \qquad \mathrm{dots}(s) = \mathrm{dom}\,s
$$
$$
\textbf{DotMap}\langle K, V:\text{DotStore}\rangle : K \mapsto V, \qquad \mathrm{dots}(m) = \bigcup_{k\in\mathrm{dom}\,m}\mathrm{dots}(m[k])
$$

A **DotFun** maps dots → values (e.g. an MVReg is a DotFun mapping each surviving write's dot to its value). A **DotMap** maps keys → dot-stores (e.g. an ORMap).

### 3.4 The join: `Causal⟨DotStore⟩`

State and deltas live in `Causal⟨DotStore⟩` = a **(DotStore, CausalContext)** pair. The partial order is *defined through* the join: $X \sqcup \bot = X$, and $X_1 < X_2$ iff $\exists X \neq \bot:\ X_1 \sqcup X = X_2$.

The **`Causal⟨DotFun⟩`** join keeps a mapping iff it is in both maps (then merge the values), **or** present in one map and *not yet known* to the other's causal context (i.e. genuinely new, not deleted):

$$
(m,c) \sqcup (m',c') =
\Big(
\{\, d \mapsto m[d]\sqcup m'[d] \mid d \in \mathrm{dom}\,m \cap \mathrm{dom}\,m' \,\}
$$
$$
\cup\ \{(d,v)\in m \mid d \notin c'\}\ \cup\ \{(d,v)\in m' \mid d \notin c\},\ \ c\cup c' \Big)
$$

Read the two unions carefully: a dot present in $m$ but **absent from $m'$** survives **only if** $m'$'s causal context $c'$ has *not* seen it. If $c'$ *has* seen $d$ but $d$ is gone from $m'$, that means $d$ was **deleted/overwritten** — so it must *not* come back. This single rule is how OR semantics and tombstone-free deletion fall out of the algebra.

The **`Causal⟨DotMap⟩`** join recurses per key and keeps every non-$\bot$ value:

$$
(m,c)\sqcup(m',c') = \big(\{\, k\mapsto v(k) \mid k\in\mathrm{dom}\,m\cup\mathrm{dom}\,m' \wedge v(k)\neq\bot \,\},\ c\cup c'\big)
$$
$$
\text{where } v(k) = \mathrm{fst}\big((m(k),c)\sqcup(m'(k),c')\big)
$$

> **Worked intuition (why an old write dies).** Write $w_1 \prec_\sigma w_2$. The dot of $w_1$ is included in $w_2$'s delta's causal context. So when the join runs, $w_1$'s mapping is in one side but its dot is *already in the other side's causal context* → it does **not** survive → the old value is overwritten and disappears from the map's range. No tombstone, no GC.

### 3.5 MVReg as a δ-CRDT (the template for all OR types)

```text
Algorithm 1 — MVReg at node i.   State (m, c) : Causal⟨DotFun⟩
  procedure write_i(v):
      d  ← next_i(c)                 # fresh dot
      c' ← dom m                     # dots currently stored = writes to overwrite
      return ({ d ↦ v }, c' ∪ {d})   # delta X^δ
  procedure read_i:
      return ran m
```

`write` creates a delta whose **map** is the single fresh `{d ↦ v}` and whose **causal context** is *all currently-stored dots plus the new one*. Joining that delta into the state pulls every previously-stored dot into the causal context → those old writes fail the survival test → they vanish. Concurrent writes (different replicas, neither's dot in the other's delta context) **both survive** → multi-value. Trace:

```
i: write_i(v) → δ = ({(i,1)↦v}, {(i,1)})            ; j delivers → state ({(i,1)↦v}, {(i,1)})
i: write_i(u) ∥ j: write_j(w)
   δ_u = ({(i,2)↦u}, {(i,1),(i,2)})                 # (i,1) listed ⇒ v overwritten
   δ_w = ({(j,1)↦w}, {(i,1),(j,1)})
after all merges, both replicas:
   ({(i,2)↦u, (j,1)↦w}, {(i,1),(i,2),(j,1)})        # holds BOTH u and w
```

---

## 4. Observed-Remove semantics, formalised

> **Mental model — "you can only remove what you've observed."** A delete/overwrite carries the set of dots it has *seen*. Anything concurrent (a fresh, unseen dot) is invisible to it and therefore **survives**. Concurrent update-vs-remove → **update wins**.

Figure 1 of the paper contrasts this with **Remove-Wins (RW)** via a shopping list: replica A updates an item while replica B concurrently deletes it. Under **OR**, B's delete only removes the state it observed, so A's concurrent update survives — the item stays. Under **RW**, the delete wins and the update is lost. OR is the intuitive choice: *unobserved* state can't be silently destroyed.

### 4.1 δ-based ORMap (composable, from Almeida et al.; semantics newly formalised here)

An **ORMap** maps keys $K$ → arbitrary δ-CRDTs, *possibly ORMaps themselves* → **composable**, **schema-less** (each key may hold a different CRDT type). API:

- `apply(k, oδ_i)` — invoke embedded mutator $o^\delta_i$ on the CRDT at key $k$.
- `delete(k)` — remove key $k$ (recursively, via `dots`).
- `get(k)` — return the embedded value at $k$.

**Semantics.** Given $\sigma$, let $\mathcal{X}_{apply}(\sigma)$ be the deltas of `apply` ops on a key **not subsequently deleted**, and $X(\sigma)$ their join (or $\bot$ if empty):

$$
\mathcal{X}_{apply}(\sigma) \triangleq \{\, X^\delta \mid \exists\,\mathtt{apply}_i(k, o^\delta_i)\in\mathit{Ops}(\sigma)\text{ returning }X^\delta\ \wedge\ \nexists\,\mathtt{delete}_j(k): \mathtt{apply}_i(k,o^\delta_i)\prec_\sigma \mathtt{delete}_j(k) \,\}
$$

`get(k)` returns `Value(X(σ), k)`. Consequence: a `delete` concurrent with an `apply` removes only the *causally-preceding* applies and **leaves the concurrently-written value**. A delete is an **"undo" of everything leading up to it** — but not of what it never saw.

```text
Algorithm 2 — ORMap at node i.   State (m, c) : Causal⟨DotMap⟩  (causal context SHARED across keys)
  procedure apply_i(k, oδ_i):
      (v, c') ← oδ_i(m[k], c)        # run embedded mutator on (value-at-k, shared ctx)
      return ({ k ↦ v }, c')
  procedure delete_i(k):
      c' ← dots(m[k])                # ALL embedded dots under k
      return (⊥, c')                 # empty store, but ctx carries the deleted dots
```

The shared causal context is the trick: `apply` threads it into the embedded mutator; `delete` harvests *all* dots beneath the key (via the recursive `dots`) into the delta's context so the join erases the whole subtree.

---

## 5. The ORArray — the paper's novel core

An array is harder than a map because elements have **order**, and order must survive concurrent insert/delete/**move** without renumbering everything and without tombstones.

### 5.1 Stable position identifiers

Don't use integer indices (an insert/delete would shift every later element). Use **stable identifiers** drawn from a dense order (the reals), à la Logoot/LSEQ: to insert between positions $p_1$ and $p_2$, pick $(p_1+p_2)/2$. The array is just the values **sorted by position ascending**. E.g. `[a,b,c]` ≈ `{(a,1.0),(b,7.5),(c,42.7)}`.

But concurrency makes a single position *ambiguous* (like an MVReg's value), so a position is a **set** of candidate positions. If $b$ is concurrently moved before $a$ and after $c$, its position set might be $\{0.3, 54\}$. To sort deterministically, every replica picks the **same** representative (e.g. the maximum) from the set.

To track an element across position changes, each element gets a **unique id** = the **dot of its creation** (creation points are globally unique). So the ORArray is:

$$
\textbf{map: unique-id} \;\mapsto\; (\text{value},\ \text{set-of-positions})
$$

```text
{ (i,1) ↦ (a, {1.0}),  (j,1) ↦ (b, {0.3, 54}),  (i,2) ↦ (c, {42.7}) }
```

Values can be **any** δ-CRDT, including another ORArray → composable, schema-less.

### 5.2 API: low-level (uid/position) vs. wrapper (integer index)

Low-level API operates on uids + stable positions:

- `apply(uid, oδ_i, p)` — run mutator on the element's value **and** overwrite its position set with $p$.
- `move(uid, p)` — overwrite position only (value untouched).
- `delete(uid)` — remove the element.
- `get(uid)` — return `(value, position-set)`.

A **wrapper API** exposes the familiar integer-index view (`insert`, `update`, `move(old_idx,new_idx)`, `delete(idx)`, `get(idx)`) by sorting the array and translating an index to the uid/position of the element there. The wrapper (Algorithm 4) is **decoupled** from the state, so any stable-identifier scheme (Logoot, LSEQ) plugs in.

### 5.3 ORArray semantics — value and position separately

**Value** mirrors the ORMap: $X(\sigma)$ is the join of applies to a uid not subsequently deleted; `fst(get(uid)) = Value(X, uid)`.

**Position** is subtler. Let `maximal(σ, uid)` = the operations altering `uid` (`apply`, `move`, `delete`) that do **not** precede another op altering `uid`. Define:

$$
P_{apply}(uid,\sigma) \triangleq \{ p \mid \exists\,\mathtt{apply}(uid,o^\delta_i,p)\in\text{maximal}(\sigma,uid)\}
$$
$$
P_{move}(uid,\sigma) \triangleq \{ p \mid \exists\,\mathtt{move}(uid,p)\in\text{maximal}(\sigma,uid)\}
$$

Then if `fst(get(uid)) ≠ ⊥`, the returned position set $P(uid)$ satisfies:

$$
P_{apply}(uid,\sigma)\ \subseteq\ P(uid)\ \subseteq\ \big(P_{apply}(uid,\sigma)\cup P_{move}(uid,\sigma)\big)
$$

Reading the bound: positions set by **`apply` always survive** (an apply changes the *value*, so its position must not be lost), while positions set by **`move` may or may not** survive (a move concurrent with another update touches no data, so it may be dropped). And a non-$\bot$ element **always has at least one position** (lower-bounded — if $P_{apply}$ is empty, at least one $P_{move}$ position remains).

### 5.4 The CompDotFun: a *composable* DotFun

To realise "an apply always wins over a concurrent move while a move can be overwritten," the position needs a dot store where **a removed root never reappears** (unlike a DotMap key, which can linger if there's a concurrent update). The paper introduces the **CompDotFun** — a DotFun whose *range is itself a dot store*, recursing `dots` into the range:

$$
\textbf{CompDotFun}\langle V:\text{DotStore}\rangle : \mathcal{I}\times\mathbb{N}\mapsto V, \qquad
\mathrm{dots}(m) = \mathrm{dom}\,m\ \cup\ \bigcup_{v\in\mathrm{ran}\,m}\mathrm{dots}(v)
$$

with join (keys must be in *both* and survive as in DotFun, **or** present in one and not in the other's context):

$$
(m,c)\sqcup(m',c') = \Big( \{ d\mapsto v(d) \mid d\in\mathrm{dom}\,m\cap\mathrm{dom}\,m' \wedge v(d)\neq\bot \}
$$
$$
\cup\ \{(d,v)\in m \mid d\notin c'\}\ \cup\ \{(d,v)\in m' \mid d\notin c\},\ \ c\cup c'\Big),\quad v(d)=\mathrm{fst}((m(d),c)\sqcup(m'(d),c'))
$$

Because this rests on set union (commutative/associative/idempotent) over a lattice range:

> **Corollary 5.1.** `Causal⟨CompDotFun⟩` is a join-semilattice. → the whole ORArray is a valid δ-CRDT.

### 5.5 The position-as-a-forest encoding

The ORArray state is:

$$
\text{Pos} \triangleq \textbf{CompDotFun}\langle \textbf{DotFun}\langle P\rangle\rangle,\qquad
\text{State} : \textbf{Causal}\langle\textbf{DotMap}\langle \mathcal{I}\times\mathbb{N},\ (\text{DotStore}, \text{Pos})\rangle\rangle
$$

A position set is stored as a **forest of depth-3 trees**: outer DotMap key (uid) → `(value, Pos)`, and `Pos` is a CompDotFun (the **roots**) whose range is a DotFun (the **children**) holding the actual position numbers. Each `root ↦ {child ↦ position}`:

```
ORArray
└── uid (dot)  ───►  ( value : any CRDT ,  Pos )
                                            │
                     Pos = CompDotFun  ◄── roots
                       ├── d1 ──► DotFun ── d2 ──► 0.3      ┐ one tree = one position
                       └── d3 ──► DotFun ── d4 ──► 54       ┘   (set {0.3, 54})
```

How the operations manipulate the forest:

- **`apply`** adds **one new tree** (a fresh root + child holding $p$) and **removes all old roots** (by listing them in the delta's causal context). Because roots live in a **CompDotFun**, a removed root **never reappears**. → an apply *decisively wins*.
- **`move`** keeps the existing roots but **swaps every root's child** for a new child holding $p$. Children live in a **DotFun**, so old children don't survive. A move can therefore be **overwritten by a concurrent apply or delete** (which remove the roots entirely), but never leaves an element value-ful yet position-less (apply always re-adds a root).
- **`delete`** removes **all roots** → no position, no value.

```text
Algorithm 3 — ORArray at node i.
State (m, c) : Causal⟨DotMap⟨ I×N , (DotStore, CompDotFun⟨DotFun⟨P⟩⟩) ⟩⟩

  procedure apply_i(uid, oδ_i, p):
      d     ← next_i(c)
      (v,c')← oδ(fst(m[uid]), c ∪ {d})              # run value mutator
      roots ← { d' | d' ∈ dom scnd(m[uid]) }        # all current roots
      m'    ← { uid ↦ (v, { d ↦ { d ↦ p } }) }      # ONE new tree
      c'    ← c' ∪ {d} ∪ roots                       # new dot + KILL old roots
      return (m', c')

  procedure move_i(uid, p):
      d        ← next_i(c)
      roots    ← { d' | d' ∈ dom scnd(m[uid]) }      # keep roots
      children ← ⋃_{child ∈ dom(ran scnd(m[uid]))} dots(child)
      position ← { r ↦ { d ↦ p } | r ∈ roots }       # new child under EACH root
      m'       ← { uid ↦ ({}, position) }            # value untouched ({} = no value delta)
      c'       ← {d} ∪ children                       # kill old children only
      return (m', c')

  procedure delete_i(uid):
      c' ← dots(m[uid])                               # all dots under uid
      return ({}, c')                                 # kill everything
```

> **Why two levels (roots vs children)?** The root level is a CompDotFun (remove-is-permanent) so **apply/delete dominate**. The child level is a DotFun nested under each root so **move can rewrite the position in place** without creating a new root — that's what lets a *concurrent* move be cleanly subordinate to a concurrent apply. The whole thing achieves **concurrent move with no tombstones and no value duplication**.

### 5.6 Why it's correct (proof intuitions)

The value side reduces to the ORMap (Algorithm 3's `apply`/`delete` are, modulo position handling, Algorithm 2), giving:

> **Observation 1.** Element *values* under Algorithm 3 obey OR semantics.

The position bounds are two containments:

- **Lemma 5.2 ($P_{apply}\subseteq P$).** A maximal `apply(uid,·,p)` mints a fresh dot $d$ unseen by anyone, so `{d ↦ {d ↦ p}}` is not in any causal context → it **survives every join** → $p\in P$.
- **Lemma 5.3 ($P\subseteq P_{apply}\cup P_{move}$).** If $p$ was set by a *non-maximal* op (some later op $o$ alters the same uid), then $o$ kills it: an apply or delete adds the position's roots to the context (CompDotFun → gone); a move adds the children to the context (DotFun → gone). Either way $p\notin P$.
- **Lemma 5.4 (value $\neq\bot \Rightarrow P\neq\emptyset$).** If only moves are maximal, the maximal move sits *after* an apply whose dot $d$ is still alive, and the move re-attaches `{d ↦ {d' ↦ p}}` with a fresh child $d'$ → at least one position survives.

> **Mental model for the proof.** "Maximal" = "no later op clobbered it." A position survives **iff** it was placed by a maximal apply (always) or rides on a root that no later apply/delete removed (move, sometimes). Survival is entirely governed by whether a *later* operation pulled your dots into the causal context.

---

## 6. DSON: assembling the JSON CRDT

A JSON document is recursive:

$$
\text{Register} = \text{String}\mid\text{Number}\mid\text{null}, \quad
\text{Map} = \{\text{String}:\text{Value},\dots\}, \quad
\text{Array} = [\text{Value},\dots], \quad
\text{Value} = \text{Map}\mid\text{Array}\mid\text{Register}
$$

A Value's **identifier is its path from the root** `doc`. For `{k1:v1, k2:[v2,v3], k3:{k4:v4}}`: `v1` is `doc.k1`, `v3` is `doc.k2[1]`, `v4` is `doc.k3.k4`.

**OR-JSON CRDT** = register via **MVReg** + map via **ORMap** + array via **ORArray**, all sharing one causal context, communicating via a **causal anti-entropy** algorithm, each replica sequential.

```mermaid
graph TD
  Doc["doc : ORMap (top level)"]
  Doc -->|k1| A["ORArray"]
  Doc -->|k2| R1["MVReg (Register)"]
  Doc -->|k3| M2["ORMap"]
  A -->|uid=(i,1)| Av["(value, positions)"]
  Av --> R2["MVReg / ORMap / ORArray …"]
  M2 -->|k4| R3["MVReg"]
  classDef m fill:#eef,stroke:#88a;
  classDef a fill:#efe,stroke:#8a8;
  classDef r fill:#fee,stroke:#a88;
  class Doc,M2 m; class A a; class R1,R2,R3 r;
```
*A document is a tree of nested CRDTs; one shared causal context threads through all of them.*

### 6.1 Guarantees and how they're earned

| Guarantee | How DSON achieves it |
|---|---|
| **Read-your-writes** | each replica is **sequential** and merges a new delta into local state *before* any later read |
| **Causal consistency** | a **causal** anti-entropy algorithm delivers a delta only after all causally-preceding deltas |
| **Strong Eventual Consistency** | (1) anti-entropy ensures every update eventually reaches all replicas; (2) states & deltas are elements of a **join-semilattice**, so same updates ⇒ same join ⇒ identical state |

### 6.2 Delta-mutation propagation

```mermaid
sequenceDiagram
  participant A as Replica A
  participant B as Replica B
  Note over A: change(doc, f) → mutator produces δ = X^δ
  Note over A: X ← X ⊔ X^δ  (apply locally first → read-your-writes)
  A-)B: broadcast X^δ  (small; element of the lattice)
  Note over B: causal anti-entropy holds δ until its causal deps arrive
  Note over B: X_B ← X_B ⊔ X^δ   (join = idempotent/comm/assoc)
  Note over A,B: same set of deltas ⇒ identical state (SEC)
```

### 6.3 Type conflicts and the MVReg/Array/Map triple

JSON is weakly typed: a variable may be `1` on one replica and `{}` on another, **concurrently**. But the δ-CRDTs **don't define joins across dot-store kinds** (a DotFun ⊔ DotMap is undefined). DSON's fix: **wrap every variable in an ORMap with three reserved keys** — one Map, one Array, one MVReg — and impose a deterministic total order $\text{Map} > \text{Array} > \text{MVReg}$. A type change adds the new-type element and removes the others; the CRDT **eventually converges to a single type**, though values written under a previous type **linger until overwritten** (possibly arbitrarily long). **Value conflicts** (same type, different value, e.g. two timestamps) are handled directly by the MVReg's multi-value behaviour.

> **Key gotcha.** "Converges to a single type" is *eventual*, and stale-typed values are not eagerly purged — they survive until something overwrites them. This is the JSON-specific wrinkle that pure map/array/register CRDTs don't have to face.

---

## 7. Metadata bound — the whole point

State is $(m, c)$: document state plus causal context. Analysis:

**Steady state** (all conflicts resolved; maximal updates not concurrent with anything):

- Causal context (compressed prefix map): $O(n\log n)$.
- **ORMap stores no dots** → zero overhead.
- **MVReg**: one dot per value → $O(1)$ per register.
- **ORArray element**: 1 dot for the uid + 2 dots for the position (one root, one child) → $O(1)$ per element.

$$
\boxed{\ O(D + n\log n)\ }\quad D = \#\text{registers} + \#\text{array elements}
$$

**With $k$ concurrent operations** (worst case): $k/2$ concurrent `apply`s to the *same* array element create $k/2$ roots; everyone delivers them; then $k/2$ concurrent `move`s add $k/2$ children **to each root** → $O(k^2)$ position dots. Causal context stays $O(n\log n)$ (causal anti-entropy). So:

$$
\boxed{\ O(k^2 D + n\log n)\ }\qquad k \le n
$$

> **The headline.** Metadata depends on document size $D$, replica count $n$, and *transient* concurrency $k$ — **never on the number of updates**. That is precisely what Automerge/Yjs can't promise, and it is what makes DSON suitable for long-lived stores.

### DSON vs prior JSON CRDTs

| | **DSON** | **Automerge** (RGA, op-based) | **Yjs** (YATA, op-based) |
|---|---|---|---|
| Array core | ORArray (CompDotFun forest) | RGA linked list | doubly-linked list |
| Tombstones | **none** | **required** | required (state-based) |
| Garbage collection | **not needed** | can't GC without breaking consistency | time-based GC + ad-hoc opts |
| Metadata vs #updates | **bounded / constant** | grows linearly | grows (key objects until parent deleted) |
| Concurrent move | **yes, no duplication** | no | no |
| Concurrent same-key conflict | all subtrees kept as **valid CRDTs** (MVReg-like) | one picked arbitrarily; rest as **raw** values | Replace-Manager front-insert |
| Optimised for | **document stores** (long lifetime) | collaborative editing + full history/undo | collaborative **text** editing |

---

## 8. Implementation & evaluation

**Implementation.** ~2500 LOC JavaScript. Mimics Automerge's external API: `init()`, `from(obj)`, `change(doc, f)` (apply `f` locally + stash the delta), `applyChanges(doc, delta)` (= $X \sqcup X^\delta$), `getChanges(doc)` (coalesce deltas to send). Persistence, networking, and the anti-entropy algorithm are separate libraries. To make nested delta-mutators ergonomic, DSON uses **JavaScript Proxies** (like Automerge) so users write `doc.k1[0] = {k2:1}` and a mechanism translates it into a delta mutator down the path (Figure 2: build the LHS path mutator, create an MVReg for the literal, wrap it in an ORMap under `k2`). The wrapper (Algorithm 4) maps integer indices to uids/stable positions, keeping index logic decoupled so any Logoot/LSEQ scheme drops in.

**Evaluation.** Compared to **Automerge** and **Yjs** (the only known full-JSON CRDT libraries; Riak lacks arrays). A research prototype, unoptimised, so the focus is *metadata trends* vs. number of operations, on a 4-core i7 laptop, documents starting empty.

| Benchmark | DSON | Automerge | Yjs |
|---|---|---|---|
| (a) Update same map key repeatedly | **constant** | grows **linearly** | constant |
| (b) Insert+delete a map key repeatedly | **constant** | unbounded | **unbounded** (GC keeps per-key object until parent deleted) |
| (c) Update same array index repeatedly | **constant** | — | — |
| (d) Insert+delete a *character* in an array | **constant** | — | constant (YATA optimised for chars) |
| (e) Insert+delete a *map* in an array | **constant** | — | **grows** (char optimisation breaks down) |
| (f) Insert+delete an *array* in an array | **constant** | — | **grows** |

DSON keeps metadata **constant** across all workloads, never needing GC. Yjs only stays constant for *character* insert/delete; with complex nested values its optimisations break.

**Worst case** ($O(k^2)$): all $n$ replicas update every array element, sync, then sort — producing $k$ roots × $k$ children. Empirically matches the theoretical $O(k^2)$ in $n$ (Figure 4), but the paper argues this is **rare**. A randomised 5-replica micro-benchmark (200 steps; per step a random replica updates with prob $p_u$ else sorts, then syncs with prob $p_s$; 10 runs, average of per-run maxima) shows document size stays **far below** the worst case (Figure 5). **Sort** is cheap because `move` changes only positions, never copies values, and concurrent sorts converge to similar states.

---

## 9. Limitations & future work

- **Anti-entropy is wasteful.** Enes et al. showed naive δ-CRDT anti-entropy can perform no better than shipping full state; **join decomposition** yields optimal deltas, but constructing it for DSON's types is **future work**.
- **Causal context grows with replica count, not updates** — but *every* replica that ever connects leaves a permanent $O(n\log n)$ footprint. Fine for a bounded set of **long-lived** replicas (e.g. one per cloud region, strong consistency within a region), bad for **unbounded short-lived clients**. Proposed fix: **local vs. global dots** — a client mints local dots, ships them with its delta, and its serving replica **rewrites** them to global replica-dots and returns the mapping. Drawback: a lost/delayed mapping can cause a **double commit** if the client retries against another replica.
- **Efficient replica removal is open.** Adding replicas is free; **removing** one is hard — its dots persist in the causal context, and naively deleting them can break causality (unwanted re-application of updates). A safe metadata-cleanup method is left for future work.
- **Stale-typed values linger.** After a concurrent type change, old-typed values survive until overwritten — possibly indefinitely.

### One-paragraph distillation

Model a JSON document as a **tree of nested δ-state CRDTs** — MVReg, ORMap, and a **new ORArray** — all sharing a single **causal context**, and give them all **Observed-Remove** semantics so that *only observed state can be removed* (concurrent updates win over concurrent deletes). The ORArray's trick is to store each element's ambiguous **position set as a depth-3 forest** inside a **composable DotFun (CompDotFun)**: `apply` plants a new permanent root (dominates), `move` rewrites a child under existing roots (subordinate), `delete` clears the roots — yielding **concurrent move with no tombstones, no duplication, and no garbage collection**. Because the causal context (not tombstones) encodes deletion, metadata is bounded by $O(D + n\log n)$ in steady state and $O(k^2 D + n\log n)$ under $k$ concurrent ops — **independent of the number of updates** — which is exactly the property a long-lived document store needs and which Automerge and Yjs cannot give.

---

## Glossary & notation

| Term | Meaning |
|---|---|
| **δ-state CRDT** | CRDT that ships *deltas* — elements of the same join-semilattice as the state; apply-delta = merge-state. |
| **SEC** | Strong Eventual Consistency: same delivered updates ⇒ already-identical state. |
| **OR semantics** | Observed-Remove: a remove/overwrite affects only *observed* state; concurrent update wins. |
| **RW semantics** | Remove-Wins: a concurrent remove beats a concurrent update (contrast to OR). |
| **dot** $(i,n)$ | unique operation id minted sequentially by replica $i$. |
| **observed** | a dot is observed by $i$ if $i$ minted it or delivered a message containing it. |
| **causal context** $c$ | grow-only set of observed dots, $\in P(\mathcal{I}\times\mathbb{N})$; compressible to a prefix map ($O(n\log n)$). Encodes deletion *in lieu of tombstones*. |
| **DotStore** | data-type-specific container exposing `dots(s)`. |
| **DotFun**$\langle V\rangle$ | dot → lattice value; `dots = dom`. |
| **DotMap**$\langle K,V\rangle$ | key → dot-store; `dots = ⋃` of range. |
| **CompDotFun**$\langle V\rangle$ | **new**: dot → dot-store, recursing `dots` into the range; *removed keys never reappear* → composable DotFun. |
| **Causal⟨DotStore⟩** | (DotStore, CausalContext) pair forming the join-semilattice for state & deltas. |
| **MVReg** | Multi-Value Register: concurrent writes all survive; later writes overwrite earlier. |
| **ORMap** | composable Observed-Remove Map (Almeida et al.); semantics formalised here. |
| **ORArray** | **new** composable Observed-Remove Array; position-set stored as a depth-3 forest. |
| **stable position id** | dense-order (real-number) identifier (Logoot/LSEQ) letting insert/delete avoid renumbering. |
| **position set** | the set of candidate positions of an array element under concurrency; sort picks a deterministic representative. |
| **roots / children** | forest levels in `Pos`: roots in a CompDotFun (apply/delete win), children in a DotFun (move rewrites in place). |
| **maximal(σ, uid)** | ops altering `uid` that are not causally followed by another op altering `uid`. |
| $X^\delta$ | a delta: increment from old state $X$ to new state $X\sqcup X^\delta$. |
| $\sqcup$ | join (LUB); associative, commutative, idempotent. |
| $D,\ n,\ k$ | document elements; replica count; number of concurrent operations ($k\le n$). |
