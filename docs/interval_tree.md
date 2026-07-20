# Interval Tree Using a BTreeMap

**Estimated reading time: ~5 minutes**

An **interval tree** is a data structure that holds intervals and efficiently answers queries like "which intervals overlap a given point or range?" A classic interval tree uses a balanced BST with augmented nodes, but we can build a practical variant using only a **BTreeMap** by exploiting its sorted-key property.

## Core Idea

Store intervals keyed by their **start** point in a BTreeMap. Alongside the tree, maintain a **running maximum end** value so that, for any prefix of keys ≤ some value, we know the largest endpoint seen. This lets us prune searches early.

### Data Layout

```
BTreeMap<K, Vec<Interval<K>>>
```

Each key in the map is the **start** of one or more intervals. The value is a list of intervals sharing that start. We also keep a separate global or per-node **max_end** to accelerate overlap queries.

A simpler (and fully BTreeMap-only) approach stores each interval as a `(start, end)` entry keyed by `start`, and scans candidates using range queries.

---

## Key Operations

### 1. Insert an interval `[lo, hi]`

1. Use `lo` as the key.
2. Append `(lo, hi)` to the vector at that key (or create a new entry).
3. Update `max_end = max(max_end, hi)`.

### 2. Delete an interval `[lo, hi]`

1. Look up key `lo`.
2. Remove the matching `(lo, hi)` from the vector.
3. If the vector is now empty, remove the key from the map.
4. Recompute `max_end` if the removed interval's `hi` was equal to the current `max_end` (a full scan of endpoints may be needed, or maintain a secondary structure).

### 3. Query — find all intervals overlapping a point `p`

Two intervals `[lo, hi]` and point `p` overlap when `lo <= p` **and** `hi >= p`.

1. Use a **range query** on the BTreeMap for all keys where `start <= p` (i.e., `..=p`).
2. For each candidate `(lo, hi)`, check `hi >= p`. Collect matches.
3. *Early termination*: if you maintain per-subtree `max_end` metadata and the `max_end` for a range of keys is less than `p`, skip that range entirely.

### 4. Query — find all intervals overlapping a range `[qlo, qhi]`

Two intervals `[lo, hi]` and `[qlo, qhi]` overlap when `lo <= qhi` **and** `hi >= qlo`.

1. Use a range query for all keys where `start <= qhi` (i.e., `..=qhi`).
2. Filter: keep entries where `hi >= qlo`.

---

## Pseudocode

```text
structure IntervalTree:
    tree   : BTreeMap<K, List<(K, K)>>   // key = start, value = list of (start, end)
    max_end: K                            // largest endpoint across all intervals

function INSERT(tree, lo, hi):
    if tree.contains_key(lo):
        tree[lo].append((lo, hi))
    else:
        tree[lo] = [(lo, hi)]
    max_end = MAX(max_end, hi)

function DELETE(tree, lo, hi):
    if NOT tree.contains_key(lo):
        return NOT_FOUND

    tree[lo].remove((lo, hi))

    if tree[lo] is empty:
        tree.remove(lo)

    // Recompute max_end (simple but O(n); can be optimized)
    max_end = 0
    for each key k in tree:
        for each (_, end) in tree[k]:
            max_end = MAX(max_end, end)

function QUERY_POINT(tree, p):
    // Quick bail-out
    if max_end < p:
        return []

    results = []

    // Iterate all keys from the smallest up to p
    for each key k in tree.range(..= p):
        for each (lo, hi) in tree[k]:
            if hi >= p:
                results.append((lo, hi))

    return results

function QUERY_RANGE(tree, qlo, qhi):
    if max_end < qlo:
        return []

    results = []

    // Only keys with start <= qhi can overlap the query range
    for each key k in tree.range(..= qhi):
        for each (lo, hi) in tree[k]:
            if hi >= qlo:
                results.append((lo, hi))

    return results
```

---

## Complexity

| Operation | Time |
|-----------|------|
| Insert | **O(log n)** (BTreeMap insertion) |
| Delete | **O(log n)** amortised; O(n) if `max_end` must be recomputed |
| Point query | **O(log n + k)** where k = number of intervals with `start <= p` |
| Range query | **O(log n + k)** where k = number of candidate intervals |

The BTreeMap gives us the `range(..)` operation in O(log n) to find the starting iterator position, then O(k) to walk the candidates.

## Trade-offs vs. a Full Augmented Interval Tree

| Aspect | BTreeMap-only | Augmented interval tree |
|--------|---------------|------------------------|
| Implementation complexity | Low | High |
| Library dependencies | Just `BTreeMap` | Custom node augmentation |
| Point query worst case | O(n) when all starts ≤ p | O(log n + m) |
| Bulk range scans | Very efficient (cache-friendly B-tree) | Depends on implementation |
| Delete + `max_end` update | May need O(n) recompute | O(log n) with augmentation |

### Optimising `max_end` Recomputation

To avoid the O(n) scan on delete, store endpoints in a **second BTreeMap** (or a `BTreeMap<K, usize>` counting occurrences). Then `max_end` is simply the last key in that map, retrievable in O(log n).

```text
end_counts: BTreeMap<K, usize>

on INSERT(lo, hi):
    end_counts[hi] += 1        // O(log n)

on DELETE(lo, hi):
    end_counts[hi] -= 1        // O(log n)
    if end_counts[hi] == 0:
        end_counts.remove(hi)
    max_end = end_counts.last_key()   // O(log n)
```

This keeps every operation at **O(log n)** while using nothing but BTreeMaps.
