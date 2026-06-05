# Fractional Indexing

> **Thesis.** If you give each item in an ordered collection a *fraction-valued sort key* instead of a positional integer index, then any reorder or insert touches **exactly one key** and never renumbers its neighbours — because between any two distinct keys there is always room to mint a third. Encode those fractions as **lexicographically-sortable base-$N$ strings** and the "room to insert" property holds forever, with no floating-point precision wall.

**Source.** David Greenspan, *Implementing Fractional Indexing* (Observable notebook, `@dgreensp`), and the Figma engineering blog, *Realtime editing of ordered sequences*. The reference implementation is `rocicorp/fractional-indexing` (CC0), which packages Greenspan's algorithm with variable-length integers and a prepend/append optimisation.

The Observable notebook is the canonical write-up of the string encoding; the Figma post is the canonical motivation (why an editor reaches for this instead of array indices, linked lists, or a full sequence CRDT) and the canonical statement of its one real weakness: **interleaving under concurrency**.

> **Note on sources.** The Observable notebook renders its prose and code via client-side JavaScript and was not machine-readable through a plain fetch; its algorithm is reconstructed here from the reference implementation's source, README, and DeepWiki, all of which credit and track the notebook directly. The Figma blog was fully accessible.

---

## TL;DR

- **The problem.** Maintain an *ordered* list of objects (rows in a DB, layers in a design, items in a doc) where many clients insert / move / reorder concurrently and edits arrive in different orders on each replica. You want each edit to be a small, commutative-ish write to *one* item.
- **Array indices are the wrong primitive.** Storing `index = 0,1,2,…` means inserting at position $k$ rewrites every item after $k$. That is $O(n)$ writes, and worse, two clients inserting "at index 3" collide and corrupt the order.
- **The idea.** Give each item a **real-number key** in $(0,1)$ and define order as "sort by key." To insert between neighbours $a$ and $b$, store the **average** $\tfrac{a+b}{2}$. Insert at the front: average with $0$. At the back: average with $1$. **One write, no renumbering.**
- **Why floats fail.** 64-bit doubles run out of mantissa after ~50 successive midpoints in the same gap. The fix: represent the fraction as a **base-$N$ digit string** and compute the midpoint as **string surgery** — arbitrary precision, the string just grows a character when a gap gets tight.
- **Lexicographic = numeric.** Choose digits that sort the same as ASCII (`0-9A-Za-z`, base 62) so a plain string `<` comparison reproduces numeric order. Then your database's native string sort *is* the item order.
- **The real implementation.** `generateKeyBetween(a, b)` returns a key strictly between `a` and `b` (either may be `null` for "unbounded"); `generateNKeysBetween(a, b, n)` evenly distributes $n$ keys so bulk inserts stay short. Keys have a clever **variable-length integer part** (first char encodes its length) so prepend/append stays $O(1)$ in key length most of the time.
- **Two limits.** (1) Keys **grow unboundedly** under adversarial repeated insertion in the same spot. (2) Concurrent inserts into the *same* gap can **interleave** ("abc" + "xyz" → "axbycz"), and **fractional indexing is *not* a full CRDT** — it does not guarantee a clean concurrent merge. Mitigations: **jitter** the keys, or have a server hand out distinct positions.

---

## 1. The problem: ordering objects that many people edit at once

Almost every collaborative app has an *ordered* collection inside it: the layers of a Figma frame, the blocks of a document, the cards of a Kanban column, the rows of a shared table. The defining operations are:

- **insert** an item at a position,
- **move** / **reorder** an existing item,
- and read the items **in order**.

In a realtime setting the difficulty is concurrency. As the Figma post puts it: *"Each client instantaneously applies its edits locally and then sends them off to the server, which then sends the edits to other connected clients. This means edits may be applied in a different order on each client."* So whatever scheme encodes "position" has to be a value you can **set on a single object** and have every replica agree on the resulting order — regardless of the order the edits land.

### 1.1 Why the obvious encodings hurt

**Array indices (`0,1,2,…`).** The position is the item's offset. To insert at offset $k$ you must shift every later item: $O(n)$ writes, and a wall of update messages on the wire. Two clients that both "insert at index 3" produce conflicting renumberings; merging them is ad-hoc. Moving an item is delete-then-insert at a new offset — again $O(n)$.

**Linked lists (each item stores `next` / `prev`).** Insert and move become $O(1)$ pointer edits — but now an *edit touches two-to-three objects at once* (the new node and its neighbours' pointers), concurrent edits to the same pointer race, and reading "the list in order" requires a pointer walk you can't express as a database `ORDER BY`. Cycles and dangling pointers are easy to create under concurrency.

**Sequence CRDTs / OT.** These solve concurrency *correctly* (RGA, Logoot, Treedoc, OT…) but, in Figma's words, *"OT is hard to understand and hard to implement correctly,"* and OT typically encodes a move as *"a delete and insert instead of a move,"* which is wasteful for reordering. They also carry per-character/per-element metadata.

Fractional indexing sits deliberately *below* a full CRDT: it is a single sortable value per item, trivial to store and `ORDER BY`, at the price of weaker concurrent-merge guarantees (§7).

| Approach | Insert | Move | Read-in-order | Concurrent inserts | Per-item state |
|---|---|---|---|---|---|
| **Array indices** | $O(n)$ renumber | $O(n)$ | trivial (`ORDER BY idx`) | collide / corrupt | one int |
| **Linked list** | $O(1)$, edits neighbours | $O(1)$, edits neighbours | $O(n)$ pointer walk, no `ORDER BY` | pointer races, cycles | two pointers |
| **Sequence CRDT (RGA/Logoot)** | $O(\log n)$-ish | delete+insert | walk structure | **correct merge** | identifier + tombstones |
| **Fractional index** | **$O(1)$, one write** | **$O(1)$, one write** | trivial (`ORDER BY key`) | may **interleave** (not a CRDT) | one sortable string |

> **Mental model.** A fractional index is "a bookmark wedged *between* two pages," not "page number 5." Wedging a new bookmark never renumbers the pages, and you can always wedge one more between any two — paper permitting. The whole game is making the paper infinite (string keys) and keeping the bookmarks legible (lexicographic order).

---

## 2. The fractional key idea

Give every item a key drawn from the open interval $(0,1)$ and define the list order as the numeric order of keys. Figma: *"Every object has a real number as an index and the order of the children … is determined by sorting all children by their index. To insert between two objects, just set the index for the new object to the average index of the two objects on either side."*

Let the existing keys be $0 < k_1 < k_2 < \dots < k_n < 1$. The three insert operations are:

$$
\text{insert before } k_1:\quad k' = \frac{0 + k_1}{2}, \qquad
\text{insert between } k_i, k_{i+1}:\quad k' = \frac{k_i + k_{i+1}}{2}, \qquad
\text{insert after } k_n:\quad k' = \frac{k_n + 1}{2}.
$$

A **move** is the same as an insert: drop the moved item's old key, compute a fresh key for its destination gap, write it. One field, one write.

```
number line view — inserting between 0.2 and 0.3

   0 ┃·········0.2····?····0.3·············· 1
                     │
                     ▼   new key = (0.2 + 0.3)/2 = 0.25
   0 ┃·········0.2···0.25···0.3············· 1

   want it earlier still? average again:
                 (0.2 + 0.25)/2 = 0.225
   0 ┃·········0.2··0.225·0.25·0.3·········· 1
```

The defining property — the reason this works *forever* in exact arithmetic — is the **density of the rationals**:

$$
\forall\, a, b \in \mathbb{Q},\; a < b \;\Longrightarrow\; a < \frac{a+b}{2} < b .
$$

There is **always room to insert between two distinct keys**. No "the list is full" state can ever arise. The cost is that each midpoint can need one more digit of precision than its parents — which is exactly where naive floating point breaks.

> **Key gotcha.** *"Averaging between two identical indices doesn't work."* If two keys are ever equal, $\tfrac{a+a}{2}=a$ and you can't separate them. Keys must be kept **strictly distinct**; the encoding and the concurrency strategy both have to guarantee it.

---

## 3. Why floats fail, and the base-$N$ string fix

A 64-bit IEEE double has a 52-bit mantissa. Each midpoint between two *adjacent* representable values has no double between it and its parent — you simply *run out of bits*. After roughly 50 repeated "insert at the same spot" operations the average equals one of the endpoints and the scheme silently collapses. Figma's response is blunt: *"We use arbitrary-precision fractions instead of 64-bit doubles so that we can't run out of precision after lots of edits."*

The elegant encoding (Greenspan's) is to drop *numbers* entirely and represent the fraction as a **string of base-$N$ digits**, interpreted as the digits *after an implicit radix point*. A digit string $d_1 d_2 \dots d_m$ over an alphabet of size $N$ denotes

$$
0.d_1 d_2 \dots d_m \;=\; \sum_{j=1}^{m} d_j \, N^{-j} \;\in\; [0, 1).
$$

Two facts make this perfect for our purpose:

1. **Arbitrary precision for free.** Need more resolution in a tight gap? *Append a digit.* The string just gets one character longer; nothing rounds.
2. **Lexicographic order = numeric order**, *provided the digit alphabet is itself in ascending order*. Compare two digit strings character by character: the first position where they differ decides both the numeric and the lexicographic comparison, identically. (One subtlety: a *prefix* like `"2"` vs `"23"` — i.e. $0.2$ vs $0.23$ — sorts correctly because the shorter string is treated as the smaller, matching $0.2 < 0.23$. The encoding forbids a *trailing zero digit*, so there is no ambiguity such as `"20"` vs `"2"`.)

Choose the alphabet so a plain ASCII string sort already matches. The reference library's default is **base 62**:

```
BASE_62_DIGITS = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
                  └ digit 0                                          digit 61 ┘
```

These 62 characters are in strict ASCII-ascending order, so native `<` on the *whole key string* reproduces item order. Figma pushes the same idea harder — *"using the entire ASCII range instead of just the numbers 0–9 (base 95 instead of base 10)"* — a larger base means each digit subdivides the gap more finely, so keys grow more slowly.

> **Key gotcha.** Sort with the **native byte/codepoint comparator**, never a locale comparator. As the README warns, `localeCompare` is case-insensitive and will interleave `A` with `a`, **destroying the ordering**. Base-62 keys are case-sensitive by design (`Z` < `a` in ASCII).

```
the midpoint, as string surgery — between "2" (0.2) and "3" (0.3) in base 10:

   "2" ──┐                      gap of one digit-step:
         ├─ digits differ by 1   no integer digit strictly between 2 and 3
   "3" ──┘                       ⇒ borrow precision: descend a level
                                 take "2" and append the midpoint of (next digit-of-2 .. 10)
   result: "25"  (= 0.25)        — exactly the float average, but exact and unbounded
```

---

## 4. The real key format: a variable-length integer part

A subtlety the toy "fraction in $(0,1)$" model hides: **prepending** repeatedly (insert at the very front, again and again) would force ever-tinier fractions $0.0\dots0x$ and keys that grow one character per prepend. Greenspan's implementation, mirrored by `rocicorp/fractional-indexing`, removes that cost by letting the key carry an **integer part** as well as a fraction, and by making the integer part **self-describing in length**. A key is:

```
            ┌── head: 1 char, encodes the *length* of the integer digits
            │   ┌── integer digits (length given by head)
            │   │   ┌── optional fractional digits (the "between" precision)
            ▼   ▼   ▼
key  =     "a"  "0"  ""        →  integer part "a0",  no fraction      (this is the very first key)
key  =     "a"  "1"  "V"       →  integer part "a1",  fraction "V"     (between "a1" and "a2")
```

The first character (the **head**) is itself a base-$N$ digit, and it encodes **how many integer digits follow**, so a parser can split head / integer / fraction with no separators. In the reference code, lowercase heads `a..z` denote integer-lengths $2..27$ (i.e. `getIntegerLength('a') = 2`), and uppercase heads `A..Z` run the *other* direction for the *negative* range, so that smaller integers (including negatives, used when prepending past the start) still sort *before* larger ones lexicographically. This is why:

- the first-ever key is `"a0"` (head `a` ⇒ 2 integer digits → `a0`);
- appending past `"a0"` gives `"a1"`, then `"a2"`, `"a3"`, … — just **increment the integer**, fraction empty, key length constant;
- prepending before `"a0"` gives `"Zz"` — an uppercase head, i.e. **decrement into the negative-encoded range** — again constant length, no growing fraction.

```
   …  "Zz"   "a0"   "a1"   "a2"   "a3"  …      ← append/prepend: integer steps, O(1) length
            └──────┬─────┘
                   │ insert between a1 and a2 needs the fraction:
                   ▼
            "a1V"          ← head a, integer "1", fraction "V" (V ≈ middle of base-62)
```

Two helper operations make append/prepend cheap:

- **`incrementInteger`** — add one to the integer digits, right-to-left with carry; on overflow (`…z`) it grows the integer by one digit and **bumps the head** (which is why the head encodes length). Used by `generateKeyBetween(a, null)` (append).
- **`decrementInteger`** — symmetric, with borrow; underflow shrinks/bumps the head downward. Used by `generateKeyBetween(null, b)` (prepend).

The **fraction** is only used when you must wedge *between* two keys that share an integer part — that's where the §3 digit-string midpoint runs.

---

## 5. The algorithms

### 5.1 The digit-string midpoint

The heart of everything: given two fractional digit strings $a < b$ (no integer prefix; $a$ may be `""` meaning the smallest value, $b$ may be `null` meaning no upper bound), return a string strictly between them.

```pseudocode
# All digit chars come from `digits` (e.g. BASE_62_DIGITS); index(c) = position of c.
midpoint(a, b, digits):
    # 1. Peel off a shared prefix — it can't help separate a from b.
    if b != null and a >= b: error("a must be < b")
    n = length of the longest common prefix of a and b
    if n > 0:
        return a[0..n] + midpoint(a[n:], b[n:] if b else null, digits)

    # 2. No common prefix. Look at the leading digits.
    digitA = (a == "")  ? 0          : index(a[0])
    digitB = (b == null)? len(digits): index(b[0])

    if digitB - digitA > 1:
        # There's a whole digit of room between them: pick the rounded middle.
        midDigit = round(0.5 * (digitA + digitB))
        return digits[midDigit]          # a single middle character fits the gap
    else:
        # Digits are adjacent (e.g. 2 and 3). Keep b's gap closed off and
        # *descend into a*: recurse on a's tail against "end of range".
        if length(b) > 1:
            # b like "23" — anything starting with b[0] and < b works
            return b[0..1]
        else:
            # borrow a digit of precision from a's side
            return digits[digitA] + midpoint(a[1:], null, digits)
```

The two cases are exactly the two pictures in §3: if the leading digits differ by **more than one**, a single middle digit fits (`0.2`/`0.5` → `0.3`); if they're **adjacent** (`0.2`/`0.3`), no integer digit fits, so you *borrow precision* — keep the lower digit and recurse one level deeper, yielding `0.25`. The recursion is what makes the key grow by a character precisely (and only) when a gap is that tight.

> **Mental model.** The midpoint routine is "binary-ish search inside the digit alphabet." Wide gap → answer in one digit. Pinhole gap → append a digit and search again inside it. Keys are short exactly as long as inserts are spread out, and lengthen exactly where inserts pile up.

### 5.2 `generateKeyBetween(a, b)`

Returns one key strictly between `a` and `b`. Either bound may be `null` (`a = null` ⇒ "before everything", `b = null` ⇒ "after everything").

```pseudocode
generateKeyBetween(a, b, digits = BASE_62_DIGITS):
    if a != null: validate(a);  if b != null: validate(b)
    if a != null and b != null and a >= b: error("a >= b")

    if a == null:                                 # PREPEND
        if b == null: return "a0"                 #   empty list → the anchor key
        ib = integerPart(b)
        fb = fractionPart(b)
        if fb is nonempty:                        #   room below b within same integer
            return ib + midpoint("", fb, digits)
        return decrementInteger(ib, digits) + ""  #   step the integer down: "a0" → "Zz"

    if b == null:                                 # APPEND
        ia = integerPart(a); fa = fractionPart(a)
        i  = incrementInteger(ia, digits)         #   step the integer up: "a0" → "a1"
        if i == null:                             #   integer overflowed its width
            return ia + midpoint(fa, null, digits)#   fall back to extending the fraction
        return i

    # BETWEEN two real keys
    ia = integerPart(a); fa = fractionPart(a)
    ib = integerPart(b); fb = fractionPart(b)
    if ia == ib:                                  # same integer ⇒ split the fraction
        return ia + midpoint(fa, fb, digits)
    i = incrementInteger(ia, digits)              # different integers
    if i < b:  return i                           #   a whole integer fits between them
    return ia + midpoint(fa, null, digits)        #   else extend a's fraction upward
```

Worked outputs from the reference implementation (base 62):

| `a` | `b` | result | why |
|---|---|---|---|
| `null` | `null` | `"a0"` | first key, the anchor |
| `"a0"` | `null` | `"a1"` | append → increment integer |
| `"a2"` | `null` | `"a3"` | append → increment integer |
| `null` | `"a0"` | `"Zz"` | prepend → decrement into upper-head range |
| `"a1"` | `"a2"` | `"a1V"` | consecutive integers, none fit between → extend fraction (`V` ≈ mid of base 62) |

### 5.3 `generateNKeysBetween(a, b, n)` — bulk inserts stay short

Calling `generateKeyBetween` $n$ times in a row to drop $n$ items in one gap is wasteful: each new key wedges against the *previous* new key, so keys lengthen fast and unevenly (a "staircase"). `generateNKeysBetween` instead **subdivides the gap evenly** — conceptually a divide-and-conquer that places the middle key first, then recurses left and right:

```pseudocode
generateNKeysBetween(a, b, n, digits):
    if n == 0: return []
    if n == 1: return [ generateKeyBetween(a, b, digits) ]
    if a == null:                          # n keys before b: build down from b
        keys = generateNKeysBetween(null, b, n-1, digits) ; first ahead of them …
        # (implementations special-case the unbounded ends for short keys)
    if b == null:                          # n keys after a: a, a+1, a+2, …
        … incrementInteger repeatedly …
    mid = floor(n / 2)
    c   = generateKeyBetween(a, b, digits) # the median key
    return generateNKeysBetween(a, c, mid, digits)
         + [c]
         + generateNKeysBetween(c, b, n - mid - 1, digits)
```

The payoff (from the README): between `null` and `null` with $n=2$ you get `['a0', 'a1']` (two integer steps, length 2 each), and between `"a0"` and `"a1"` with $n=2$ you get `['a0G', 'a0V']` — evenly spaced thirds of the gap — instead of the lopsided `['a0V', 'a0l']` you'd get from two sequential single calls.

### 5.4 Average-case length growth

If you only ever **append**, keys stay roughly constant length (integer increments; a width bump every $N$ items, so length $\approx \log_N(\text{count})$). The pathological case is repeatedly inserting into the **same** gap: each insert can add up to one base-$N$ digit, so after $m$ such inserts a key can be $\Theta(m)$ characters. In bits, a key needs about

$$
\text{length} \;\approx\; \big\lceil \log_2(\text{number of items that have shared this gap}) \big\rceil \big/ \log_2 N \quad \text{base-}N\text{ digits,}
$$

i.e. logarithmic in the *contention on a gap*, linear only under truly adversarial "always insert in the same spot" workloads. Figma notes the growth is *"not problematic for typical usage,"* and `generateNKeysBetween` plus occasional re-balancing (re-spacing all keys) keeps it bounded in practice.

---

## 6. Jitter: surviving concurrent inserts into the same gap

The deterministic midpoint has a sharp edge under concurrency. If two offline/concurrent clients both insert "between $a$ and $b$", they both compute the **same** midpoint $\tfrac{a+b}{2}$ → **two items with identical keys**, and §2's "identical keys can't be averaged apart" pathology strikes: the order between them is now undefined and *unfixable by either client alone*.

**Jitter** breaks the symmetry by injecting randomness so two clients almost never pick the same key:

```pseudocode
generateJitteredKeyBetween(a, b, digits):
    mid = generateKeyBetween(a, b, digits)     # the deterministic midpoint
    # randomly nudge into the lower or upper half, repeatedly, for more entropy
    repeat jitterDepth times:
        if coinFlip():  mid = generateKeyBetween(a,   mid, digits)   # go lower
        else:           mid = generateKeyBetween(mid, b,   digits)   # go higher
    return mid
```

Each extra round of jitter adds entropy (lowering collision probability) at the cost of a slightly longer key — the same length/safety trade-off, now tuned by `jitterDepth`. This is what the `fractional-indexing-jittered` / `jittered-fractional-indexing` packages do.

> **Design lesson.** Jitter trades *key length* for *collision resistance*. It makes identical-key collisions astronomically unlikely, but it does **not** fix interleaving (§7) — two jittered concurrent inserts still land in arbitrary relative order. Jitter prevents *corruption* (equal keys); it does not impose *intent* (which insert should come first).

Figma takes a different, simpler route for the equal-key case: a **central authority**. *"The server prevents this anomaly by just generating and assigning a unique position to the second insert operation"* when two clients insert between the same pair. With a server in the loop you don't need jitter; in a serverless / P2P setting, jitter (or a tie-breaker like appending the client's unique ID) is the move.

---

## 7. Why fractional indexing is *not* a CRDT: interleaving

Fractional indexing gives **convergence of the *sorted result*** — every replica that holds the same set of (item, key) pairs sorts them identically, because string sort is deterministic. That is genuinely useful and is why it "feels" CRDT-like. But it does **not** guarantee that the *merged order matches what the users intended*, and that is the line between "a sortable key" and "a sequence CRDT."

The failure mode is **interleaving**. Suppose the list is `[L, R]` and two users, offline from each other, each insert a *run* of items into the same gap between `L` and `R`:

- User 1 inserts the run **a, b, c** (so a < b < c, all between L and R),
- User 2 inserts the run **x, y, z** (so x < y < z, all between L and R).

Each user's keys are internally well-ordered, but the two users chose keys **independently in the same interval**, so on merge the keys can shuffle together:

```
                 L                         R
   user 1 wanted:  L · a · b · c · R          (a contiguous block)
   user 2 wanted:  L · x · y · z · R          (a contiguous block)

   a possible merge by key-sort:
                 L · a · x · b · y · c · z · R
                       └─ interleaved! neither run stays together ─┘
```

Figma states the anomaly directly: *"Merging new elements from multiple clients may interleave them"* — inserting "abc" and "xyz" simultaneously *"might produce interleaved results like 'axbycz'."* No deterministic key assignment can prevent this in general, because each replica picked keys with no knowledge of the other's run. **This is precisely the guarantee a real sequence CRDT (RGA, Logoot, Treedoc, Fugue…) provides and fractional indexing does not**: those structures attach causal/identity information so concurrent runs stay contiguous (non-interleaved). Fractional indexing throws that information away in exchange for "one sortable scalar per item."

> **Key gotcha.** Fractional indexing converges on a *total order of keys*, not on a *conflict-free merge of intent*. Same keys everywhere ⇒ same sorted list (good). But concurrent insertions are **not commutative in their visual effect** — the merged order is a deterministic function of the *keys*, which were chosen non-deterministically by independent clients. That is the missing CRDT property.

Why is this acceptable in practice? Figma's judgement: *"Interleaving concurrently inserted elements in a design is usually fine because the new objects likely don't overlap. And if interleaving looks weird, users can just manually fix the ordering afterwards,"* concluding that *"it's much more beneficial for the Figma platform to use simple algorithms that are easy to understand and implement than to use the most advanced algorithms out there."* For *text* (where interleaved characters are gibberish) the calculus flips and a true sequence CRDT or OT is usually warranted; for *coarse-grained objects* (layers, list items, rows) fractional indexing's simplicity wins.

| Property | Fractional index | Sequence CRDT (RGA / Logoot / Fugue) |
|---|---|---|
| Per-item state | one sortable string | identifier + (often) tombstones |
| Read in order | native `ORDER BY` | traverse structure |
| Move an item | rewrite one key, $O(1)$ | typically delete + re-insert |
| Concurrent equal-spot inserts | may collide (need jitter/server) | safe by construction |
| Concurrent **runs** | may **interleave** | **stay contiguous** (no interleaving) |
| Implementation cost | tiny | substantial |

---

## 8. Practical checklist

- **Pick a base whose alphabet is ASCII-ascending** (base 62 `0-9A-Za-z`, or up to base 95 over printable ASCII). Bigger base ⇒ shorter keys, but mind your storage/transport's character constraints.
- **Sort with the byte/codepoint comparator.** Never `localeCompare` or a case-insensitive collation — it reorders `A`/`a` and corrupts the sequence.
- **Use `generateNKeysBetween` for bulk inserts** (paste, import) so keys stay short and even rather than a length-staircase.
- **Decide your concurrency story up front:** server-assigned uniqueness (Figma) *or* jitter / client-ID tie-breaker (serverless). Either prevents equal-key corruption; **neither prevents interleaving**.
- **Plan a re-balancing job.** Under heavy same-gap churn, keys grow; periodically re-space all keys (offline, or as a coordinated op) to reclaim length. Re-spacing is itself a bulk reorder — `generateNKeysBetween(null, null, n)`.
- **Keep keys strictly distinct.** Equal keys are the one state the math can't recover from; the entire concurrency design exists to avoid them.
- **If interleaving is unacceptable** (e.g. collaborative *text*, where contiguity is semantic), reach for a real sequence CRDT or OT instead — fractional indexing is the wrong tool there.

### The one-paragraph distillation

Replace positional integer indices with **fraction-valued sort keys**: order = "sort by key", insert = "store the average of your two neighbours," move = "recompute one key." Because the rationals are dense, there is always room to insert; because 64-bit floats are not dense enough, **encode the fraction as a lexicographically-sortable base-$N$ digit string** that you lengthen by a character whenever a gap gets tight, plus a self-describing **variable-length integer part** so append/prepend stays $O(1)$. The result is one tiny sortable scalar per item, a native-`ORDER BY` read path, and $O(1)$ single-write inserts and moves — at the cost of two honest limitations: keys **grow** under adversarial same-spot churn (mitigate with `generateNKeysBetween` and re-balancing), and concurrent inserts **collide or interleave** because fractional indexing is **not a full CRDT** (mitigate equal-key collisions with **jitter** or a **server-assigned** position; accept that contiguous runs may interleave, or use a real sequence CRDT where that matters).

---

## Glossary & notation

| Term | Meaning |
|---|---|
| **Fractional index / order key** | A sortable value (here a base-$N$ string) assigned to an item; list order = sorted order of keys. |
| **Midpoint / average** | The key strictly between two neighbours: numerically $\tfrac{a+b}{2}$, computed as digit-string surgery. |
| **Density of $\mathbb{Q}$** | $a<b \Rightarrow a<\tfrac{a+b}{2}<b$ — the "always room to insert" property. |
| **Base-$N$ digit string** | A string over an $N$-symbol alphabet interpreted as digits after a radix point: $\sum_j d_j N^{-j}$. |
| **Lexicographic = numeric** | With an ascending alphabet, plain string `<` reproduces fraction order — so DB sort = item order. |
| **`BASE_62_DIGITS`** | `0123456789A…Za…z`, the reference library's default ascending alphabet. |
| **Head** | The first character of a key, encoding the **length** of the integer part (lowercase → positive widths, uppercase → the negative/prepend range). |
| **Integer part** | The whole-number digits of a key; `incrementInteger`/`decrementInteger` step it for $O(1)$ append/prepend. |
| **Fraction part** | The "between-neighbours" precision digits, produced by the midpoint routine. |
| **`generateKeyBetween(a,b)`** | Returns one key strictly between `a` and `b`; `null` bound = unbounded (prepend/append). |
| **`generateNKeysBetween(a,b,n)`** | Returns $n$ evenly-spaced keys in the gap (divide-and-conquer) so bulk inserts stay short. |
| **Jitter** | Randomly nudging a generated key into a sub-half to make concurrent clients pick distinct keys (collision avoidance). |
| **Interleaving** | Two concurrently-inserted **runs** shuffling together on merge (`abc`+`xyz`→`axbycz`); the anomaly fractional indexing can't prevent. |
| **Re-balancing** | Periodically re-spacing all keys to reclaim length after same-gap churn. |
| **Not a CRDT** | Fractional indexing converges on a total key order but not on conflict-free **intent** under concurrency (interleaving) — unlike a sequence CRDT. |
