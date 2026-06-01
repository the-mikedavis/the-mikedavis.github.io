+++
title = "Hash Array Mapped Tries in Rust"
date = 2026-05-31
taxonomies.tags = ["Rust", "Erlang", "data structures"]

[extra]
repo_view = true
+++

This post is part two in a series focused on a specific data structure design for a type in Rust: a Hash Array Mapped Trie (HAMT). While the details are Rust-focused, we'll also discuss HAMTs as they appear in Erlang's map type. HAMTs are an interesting data structure providing a hash-map interface without chaining or open addressing. Notably they are favored in functional programming languages, especially garbage-collected ones.

In [part one](@/posts/sparse-array-dst.md) we built `SparseArray<T>`: a compact array that only stores occupied slots, tracking which indices are live with a 64-bit bitmap. Today we'll use it to build a [Hash Array Mapped Trie](https://en.wikipedia.org/wiki/Hash_array_mapped_trie) (HAMT), a persistent hash map where `clone` is O(1) and modifications share structure between versions.

Erlang and Elixir's `Map` type is a HAMT (for maps with more than 32 elements), and so is Clojure's `PersistentHashMap`. This post looks at how they work and why. The full implementation lives [here](https://github.com/the-mikedavis/hamt).

### Persistent data structures

Most data structures are _ephemeral_: when you `insert` into a `HashMap`, the old state is gone. Keeping a copy before modifying requires cloning it first - and cloning a map is O(n).

_Persistent_ data structures work differently. A modification produces a new version while leaving the old one intact. Both versions share the nodes they have in common; only the parts that changed are newly allocated. This is _structural sharing_. Concretely: each insertion costs O(log n) new allocations, not O(n). And `clone` is O(1) - just a reference count bump.

This is useful in a few situations. Concurrent reads need no locking: hand out references to the current version, let readers hold them as long as they need, and publish new versions with a pointer swap. Undo/redo is cheap because each historical state shares structure with its neighbors. And you can pass a map into a function and know for certain it hasn't been mutated.

### In the wild

HAMTs are the hash map implementation in several functional programming languages.

Erlang has used a HAMT for its built-in map type since OTP 17 (2014). Erlang and Elixir are functional languages: values are immutable, and modifying a map produces a new version rather than mutating the old one. A HAMT makes that cheap - only the nodes on the modified path are newly allocated, and the rest is shared with the previous version. Elixir's `Map` type is the same underlying structure; Elixir runs on BEAM and shares Erlang's data types.

It's worth knowing that BEAM maps aren't always HAMTs. Maps with 32 or fewer elements are stored as _flat maps_: a sorted array of key-value pairs. Flat maps are faster for small collections since there's no trie to navigate - lookups are a binary search. Only once a map grows past 32 elements does BEAM promote it to a HAMT. Most Erlang and Elixir developers don't think about this distinction day-to-day, but it has observable consequences: flat map iteration is sorted by key, while HAMT iteration follows hash order.

Clojure's [`PersistentHashMap`](https://github.com/clojure/clojure/blob/master/src/jvm/clojure/lang/PersistentHashMap.java) is the other canonical example. Rich Hickey adapted Phil Bagwell's Array Mapped Trie (2001) when designing Clojure (2007). Haskell's [`unordered-containers`](https://hackage.haskell.org/package/unordered-containers) and Scala's `scala.collection.immutable.HashMap` are HAMTs too.

### Iteration order

A useful (and hazardous) property of HAMTs is that iteration order is determined entirely by the hash function, not by insertion order. In an open-addressing hash table, inserting key A before key B can leave them in different slots than inserting B first, because collision probe sequences depend on what is already in the table. A HAMT has no such dependency - each key's position in the trie is set by its hash bits alone. Two maps built from the same set of keys always produce the same trie structure, regardless of insertion order.

This sounds like a useful guarantee, but languages are careful not to promise it. The hash function can change between releases, and when it does, iteration order changes with it. Erlang's internal map hash has been replaced more than once: from a variant of Bob Jenkins' classic hash, to CRC32-C, to the current MurmurHash3. Each switch reshuffled ordering and ordering changes could cause subtle bugs in Erlang applications.

Erlang's approach has been to document order-independence explicitly from the start. When maps were introduced in OTP 17 (2014), `maps:fold/3` replaced the older `maps:foldl/3` and `maps:foldr/3` - dropping the directional names to signal that callers should not depend on order. When `maps:iterator/1` was added in OTP 21 (2017), the default ordering was `undefined`.

To cover the needs of ordered traversal, OTP 26 (2023) added `maps:iterator/2` with an `ordered` option:

```erlang
1> Map = #{b => 2, a => 1, c => 3}.
#{a => 1, b => 2, c => 3}
2> maps:to_list(maps:iterator(Map, ordered)).
[{a, 1}, {b, 2}, {c, 3}]
```

This sorts by Erlang's standard term order, which is stable across releases. It costs more - sorting is O(n log n) vs. O(n) for a plain traversal - but it's the right tool when you actually need a deterministic sequence.

### Hash tries

A [trie](https://en.wikipedia.org/wiki/Trie) is a tree whose structure encodes its keys. In a classic string trie each edge represents one character and the path from root to leaf spells out a key. A HAMT takes the same idea but navigates by hash: we hash the key and consume the result a few bits at a time, taking a branch at each level based on the next chunk.

We'll use 6 bits per level, giving 64 possible branches at each interior node. With a 64-bit hash that's at most 10 full levels (6 * 10 = 60 bits). In practice tries are much shallower - a map with thousands of entries rarely needs more than two or three. The exception is when two distinct keys produce the same 64-bit hash, which we handle with a dedicated collision node containing a short linear list.

The three node types look like this:

```rust
#[derive(Clone)]
enum Node<K, V> {
    Leaf { hash: u64, key: K, value: V },
    Interior(SparseArray<Arc<Node<K, V>>>),
    Collision { hash: u64, entries: Vec<(K, V)> },
}
```

The branch nodes (`Interior`) leverage `SparseArray<T>` to save space. A straightforward trie node at each level would hold 64 child pointers, most of them null. An `Interior` stores only the children that are actually present, densely packed, with the bitmap tracking which slots are occupied. The trie itself wraps an optional root:

```rust
pub struct Hamt<K, V, S = RandomState> {
    root: Option<Arc<Node<K, V>>>,
    len: usize,
    hash_builder: S,
}
```

Every `Node` lives behind an `Arc`. We'll get to why shortly.

### The rank operation

The connection between a hash and a slot in the sparse array is the `rank` operation from `Bitmap`. Recall from part one that `SparseArray<T>` stores entries densely: a sparse array with entries at logical indices 2 and 5 physically stores them at `entries[0]` and `entries[1]`. Going from a logical index to its physical position is `rank`'s job:

```rust
pub fn rank(self, bit: usize) -> Result<usize, usize> {
    let idx = (self.0 & ((1u64 << bit) - 1)).count_ones() as usize;
    if self.has(bit) { Ok(idx) } else { Err(idx) }
}
```

`Ok(idx)` means the slot is occupied and the entry is at `entries[idx]`. `Err(idx)` means the slot is empty and `idx` is where a new entry would go to keep entries in order. The count comes from masking off all bits at or above `bit` and calling `count_ones`. That's a POPCNT instruction on any modern CPU, a single clock cycle.

Say a bitmap has bits 1 and 3 set:

```
Bitmap: 0b...00001010
rank(3) = popcount(0b...00000010) = 1   ->  entries[1]
rank(5) = popcount(0b...00001010) = 2   ->  Err(2), slot is empty
```

And `bit_index` extracts the 6-bit chunk from the hash at the current level:

```rust
const BITS: u32 = 6;
const MASK: u64 = (1u64 << BITS) - 1;

fn bit_index(hash: u64, level: u32) -> usize {
    ((hash >> (level * BITS)) & MASK) as usize
}
```

Together, these two get us from a hash value to a physical entry with nothing but bitwise arithmetic.

### Looking up a key

With that in place, lookup is a clean iterative descent:

```rust
pub fn get<Q>(&self, key: &Q) -> Option<&V>
where
    K: Borrow<Q>,
    Q: Hash + Eq + ?Sized,
{
    let hash = compute_hash(&self.hash_builder, key);
    let mut current = self.root.as_deref()?;
    let mut level = 0u32;
    loop {
        match current {
            Node::Leaf { hash: h, key: k, value: v } => {
                return if *h == hash && k.borrow() == key { Some(v) } else { None };
            }
            Node::Interior(arr) => {
                current = arr.get(bit_index(hash, level))?;
                level += 1;
            }
            Node::Collision { hash: h, entries } => {
                return if *h == hash {
                    entries.iter().find(|(k, _)| k.borrow() == key).map(|(_, v)| v)
                } else {
                    None
                };
            }
        }
    }
}
```

At each `Interior` node we call `arr.get(bit_index(hash, level))`. That extracts the next 6 bits of the hash, runs `rank` to find the physical index, and returns the child at that slot. If the slot is empty, `arr.get` returns `None` and we're done: the key isn't in the map. At a `Leaf` we compare both the hash and the key. At a `Collision` we scan the short list linearly.

### Inserting a key

Insertion has more cases. The simple ones are inserting into an empty slot (create a new `Leaf`) or finding the same key (update its value). The interesting case is landing in a slot already occupied by a `Leaf` for a _different_ key.

#### Splitting a leaf

Two different keys that share the same 6-bit prefix at the current level need to be separated at the next level. We create a new `Interior` node to hold the existing leaf and let the loop continue:

```rust
if existing_bit == new_bit {
    // Same prefix: wrap the existing leaf in a new Interior
    // and descend again at level + 1.
    let existing = current.clone();
    *current = Arc::new(Node::Interior(
        SparseArray::default().with_insert(existing_bit, existing),
    ));
} else {
    // Bits differ: create the final Interior with both leaves.
    let existing = current.clone();
    let new_leaf = Arc::new(Node::Leaf { hash, key, value });
    *current = Arc::new(Node::Interior(
        SparseArray::default()
            .with_insert(existing_bit, existing)
            .with_insert(new_bit, new_leaf),
    ));
    return true;
}
```

When the bits diverge we're done: one new `Interior`, two children. When they keep colliding, we create a single-child `Interior` and try again one level deeper. Two keys that are genuinely distinct but share all 60 hash bits (extraordinarily unlikely with any reasonable hasher) would eventually form a `Collision` node.

#### A raw pointer cursor

The full `insert_node` function is iterative: it advances through `Interior` nodes level by level without recursion. The wrinkle is that we need a `&mut Arc<Node>` that steps into the trie one level at a time, and we need to call `Arc::make_mut` on whatever we're currently pointing at. The borrow checker won't let us hold a reference into an `Arc` while also doing a `make_mut` on it in a loop.

A raw pointer cursor threads the needle:

```rust
fn insert_node<K, V>(arc: &mut Arc<Node<K, V>>, hash: u64, key: K, value: V) -> bool
where
    K: Clone + Eq,
    V: Clone,
{
    // SAFETY: ptr always points into an Arc<Node> kept alive by the trie.
    // Exclusive access is guaranteed by the original &mut arc.
    let mut ptr: *mut Arc<Node<K, V>> = arc;
    let mut level = 0u32;

    loop {
        let current: &mut Arc<Node<K, V>> = unsafe { &mut *ptr };
        match current.as_ref() {
            // ...
            Node::Interior(arr) => {
                let bit = bit_index(hash, level);
                let occupied = arr.bitmap().has(bit);
                let Node::Interior(arr) = Arc::make_mut(current) else { unreachable!() };
                if occupied {
                    let Ok(idx) = arr.bitmap().rank(bit) else { unreachable!() };
                    ptr = &raw mut arr.entries_mut()[idx];
                    level += 1;
                } else {
                    // ...
                }
            }
            // ...
        }
    }
}
```

`ptr` starts pointing at the root `Arc` and advances to point at a slot in the entries slice of each `Interior` node. The key insight is that `&mut arc` at the call site guarantees exclusive access to the entire trie for the duration of the call, so `ptr` is just a raw-pointer view into that same exclusive borrow, letting us move the cursor forward without fighting the loop-carried lifetime.

### Persistence

Because every node is behind an `Arc`, cloning a `Hamt` is O(1):

```rust
impl<K, V, S: Clone> Clone for Hamt<K, V, S> {
    fn clone(&self) -> Self {
        Self {
            root: self.root.clone(), // just increments a reference count
            len: self.len,
            hash_builder: self.hash_builder.clone(),
        }
    }
}
```

When you then insert into the clone, `Arc::make_mut` on each node along the modified path checks the reference count. If the node is uniquely owned (count of one) it hands back `&mut T` directly, no allocation needed. If it's shared (count of two or more), it clones the node first. Either way only the nodes _on the path being modified_ are ever copied. Everything else stays shared.

```rust
let mut a = Hamt::new();
a.insert("x", 10);

let mut b = a.clone();  // O(1): one reference count bump
b.insert("y", 20);

assert_eq!(a.get("y"), None);        // a is unchanged
assert_eq!(b.get("x"), Some(&10));  // b shares a's unmodified subtree
```

This property (modifications share all unaffected structure with previous versions) is what makes the data structure _persistent_. An ephemeral mutation (like `HashMap::insert`) destroys the previous state. A persistent one produces a new version while the old one remains valid for as long as anything holds a reference to it.

### Applying to Spellbook

I originally wanted a HAMT for [Spellbook](https://github.com/helix-editor/spellbook), a Rust spell-checking library I wrote for the [Helix editor](https://github.com/helix-editor/helix). Spellbook's core data structure is a word list, essentially a `HashMap<Stem, FlagSet>`. A persistent word list would be cheap to share across threads and safe to hand to consumers without copying.

I swapped out the `hashbrown::HashTable` backing the word list for the HAMT and ran the benchmarks against the American English dictionary (~50,000 stems):

| Test                          | `HashMap` | HAMT       | Change
|-------------------------------|-----------|------------|---
| `in_dictionary_word`          | 64.6 ns   | 77.8 ns    | +20%
| `word_with_suffix`            | 65.5 ns   | 80.9 ns    | +23%
| `word_with_prefix`            | 184.1 ns  | 206.5 ns   | +12%
| `word_with_prefix_and_suffix` | 355.6 ns  | 421.8 ns   | +19%
| `breaks`                      | 889.0 ns  | 1,589.7 ns | +79%
| compile `en_US`               | 6.8 ms    | 22.2 ms    | +228%

Lookup is 15-25% slower across the board. No surprise: Hashbrown uses SIMD to compare multiple hash fingerprints simultaneously within a single cache line; a HAMT pointer-chases through multiple independent allocations per lookup. Even at three or four levels deep it can't match the cache locality of a flat table.

The compilation benchmark is where it really hurts: building the word list is 3x slower. Each insertion allocates along the entire path from root to new leaf, whereas hashbrown amortizes resizing across many insertions. For tens of thousands of stems this adds up quickly.

For Spellbook, a HAMT is the wrong tool. Lookup needs very low latency since it's always used in a tight loop. Concurrent modification is most likely better handled by a `RwLock` or `Mutex`.

### Wrapping up

A Hash Array Mapped Trie is a trie where interior nodes are `SparseArray<Arc<Node>>`s indexed by 6-bit chunks of the key's hash. The `rank` operation (one POPCNT) maps a logical slot in the bitmap to a physical index in the dense entries array. `Arc::make_mut` handles copy-on-write during insertion so that only the O(depth) path being modified is ever newly allocated, leaving all other nodes shared between versions.

The full code lives [here](https://github.com/the-mikedavis/hamt).
