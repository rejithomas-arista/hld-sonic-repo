# ZMQ Queue Deduplication — Correctness Proof

## 1. Single-Buffer Swap-Out + Merge-Back (Initial Design)

This section documents the correctness proof for the initial single-buffer design with swap-out and merge-back. This design was replaced by the double-buffer design (§2) but is retained here for reference as it informed the design evolution.

### Context

When the number of pending keys exceeds the batch size (`m_popBatchSize`), `pops()` must **merge back** unprocessed entries into the shared structures. During the time `pops()` was processing (without the lock), the poll thread may have added new entries. The merge-back logic must preserve correctness.

### Data Structures

| Structure        | Type                          | Purpose                                  |
| ---------------- | ----------------------------- | ---------------------------------------- |
| `m_stateTable`   | `unordered_map<key, entry>`   | Latest SET value per key (O(1) lookup)   |
| `m_pendingKeys`  | `deque<key>`                  | Insertion-ordered keys to process        |
| `m_deletedKeys`  | `unordered_set<key>`          | Keys with pending DEL (O(1) lookup)      |

### Invariants

**INV1 (Completeness)**: Every key in `m_pendingKeys` has a corresponding entry in exactly one of `m_stateTable` or `m_deletedKeys`.

**INV2 (No duplicates)**: A key appears in `m_pendingKeys` at most once.

**INV3 (Disjointness)**: `m_stateTable` and `m_deletedKeys` are disjoint — a key is never in both simultaneously.

**INV4 (Recency)**: The state in `m_stateTable`/`m_deletedKeys` for any key reflects the **latest** operation received for that key.

### handleReceivedData() — Invariant Preservation

```cpp
bool isNewKey = (m_stateTable.find(key) == m_stateTable.end() &&
                 m_deletedKeys.find(key) == m_deletedKeys.end());

if (op == SET_COMMAND)
{
    m_stateTable[key] = kco;       // overwrite or insert
    m_deletedKeys.erase(key);      // remove from deleted if present
}
else if (op == DEL_COMMAND)
{
    m_stateTable.erase(key);       // remove from state if present
    m_deletedKeys.insert(key);     // mark as deleted
}

if (isNewKey)
{
    m_pendingKeys.push_back(key);  // add to queue only on first encounter
}
```

- **INV1**: If `isNewKey`, key is added to `m_pendingKeys` and to exactly one of the maps. If not `isNewKey`, key is already in `m_pendingKeys` and its map entry is updated in place. ✓
- **INV2**: `isNewKey` is true only when key is in neither map, meaning it's not in `m_pendingKeys` yet (by INV1). So no duplicate is added. ✓
- **INV3**: SET adds to `m_stateTable` and erases from `m_deletedKeys`. DEL erases from `m_stateTable` and adds to `m_deletedKeys`. Never in both. ✓
- **INV4**: Each operation overwrites the previous state for that key. ✓

### Merge-Back Algorithm

At merge time, **local** contains leftover unprocessed entries (older), **member** contains entries added by poll thread since swap-out (newer).

```cpp
// Step 1: Push remaining keys — check member maps BEFORE merging local state
for (remaining keys in reverse)
{
    if (key NOT in m_stateTable AND NOT in m_deletedKeys)
        m_pendingKeys.push_front(key);
}

// Step 2: Merge local SET entries — don't overwrite newer DEL
for (entry in localState)
{
    if (key NOT in m_deletedKeys)
        m_stateTable.emplace(key, entry);  // emplace = no-op if exists
}

// Step 3: Merge local DEL entries — don't overwrite newer SET
for (key in localDeleted)
{
    if (key NOT in m_stateTable)
        m_deletedKeys.insert(key);
}
```

**Critical ordering**: Step 1 runs before Steps 2-3. At Step 1, member maps contain only poll thread entries, allowing O(1) duplicate detection.

### Merge-Back Proof by Cases

| Scenario | Push K? | Winning state | All invariants hold? |
|----------|---------|---------------|----------------------|
| A: local only | Yes | local (only source) | ✓ |
| B: member only | N/A | member (untouched) | ✓ |
| C1: local SET, member SET | No | member SET (emplace no-op) | ✓ |
| C2: local SET, member DEL | No | member DEL (deletedKeys guard) | ✓ |
| C3: local DEL, member SET | No | member SET (stateTable guard) | ✓ |
| C4: local DEL, member DEL | No | member DEL (idempotent) | ✓ |
| C5: local SET, member DEL→SET | No | member SET (emplace no-op) | ✓ |
| C6: local SET, member SET→DEL | No | member DEL (deletedKeys guard) | ✓ |

### Limitations of Single-Buffer Design

- **O(R) lock hold time** during merge-back, where R = unprocessed keys beyond batch limit. At scale (100K+ routes), R can be large, stalling the poll thread for 5ms+.
- **Correctness complexity**: Three guarded loops with ordering constraints. Step 1 must run before Steps 2-3 to correctly distinguish poll thread entries from local entries.

---

## 2. Double-Buffer Design (Final Design)

### Architecture

Two complete sets of dedup structures. Poll thread writes to one buffer, main thread reads from the other. No shared data between threads during processing.

```
Buffer 0:  { stateTable, pendingKeys, deletedKeys }
Buffer 1:  { stateTable, pendingKeys, deletedKeys }

m_writeIdx:  index of buffer the poll thread writes to
m_readBuf:   index of buffer with unprocessed leftovers (-1 if none)
```

### Why It's Correct by Construction

The double-buffer design eliminates all merge-back correctness concerns because the two buffers are **fully independent**:

1. **No shared mutable state**: The poll thread writes to `m_buffers[m_writeIdx]`. The main thread reads from `m_buffers[readIdx]` where `readIdx != m_writeIdx`. They never access the same buffer simultaneously.

2. **Buffer flip is atomic**: The index flip (`m_writeIdx = 1 - m_writeIdx`) happens under the mutex. After the flip, the poll thread immediately starts writing to the other buffer. The old write buffer becomes the read buffer, frozen for the main thread.

3. **Each buffer independently satisfies all invariants**: `handleReceivedData()` maintains INV1-4 for whichever buffer it writes to. The read buffer is frozen — no concurrent modifications, so invariants trivially hold during processing.

4. **Order across buffers**: The read buffer contains entries accumulated before the flip. The write buffer contains entries accumulated after. Processing the read buffer first, then the write buffer on the next `pops()` call, preserves temporal ordering.

### Invariant Proof

**INV1 (Completeness)**: Each buffer maintains this independently. `handleReceivedData()` adds a key to `pendingKeys` only on first encounter in that buffer (checked via `stateTable` and `deletedKeys` lookups). Every key in `pendingKeys` has an entry in exactly one map. ✓

**INV2 (No duplicates)**: The `isNewKey` check in `handleReceivedData()` prevents duplicates within a buffer. Cross-buffer duplicates are impossible because the buffers are independent. ✓

**INV3 (Disjointness)**: SET erases from `deletedKeys`, DEL erases from `stateTable`. Within each buffer, they are always disjoint. ✓

**INV4 (Recency)**: Within each buffer, `handleReceivedData()` overwrites on every operation — last-writer-wins. Across buffers, the read buffer has older entries and the write buffer has newer entries. Both are processed in order. ✓

### Complexity

| Operation | Lock hold time |
| --------- | -------------- |
| `handleReceivedData()` — per entry | O(1) — hash operations on write buffer |
| `pops()` — step 1 (flip or continue) | O(1) — single index check/flip |
| `pops()` — step 2 (process) | **No lock** — iterates read buffer |
| `pops()` — step 3 (mark done) | O(1) — set `m_readBuf` |

Total lock hold time per `pops()` call: **O(1)** regardless of queue depth or batch size.

### Trade-off

Entries arriving during processing go to the **other** buffer, so there is no cross-buffer dedup for the same key. This matches the original queue behavior where entries in separate `pops()` calls were never deduplicated. The dedup win is within a single accumulation period, which is where the 7x duplication factor occurs.
