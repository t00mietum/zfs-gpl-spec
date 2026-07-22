# ZFS on-disk format: ZAP (micro and fat)

**Provenance**

- OpenZFS commit studied for behavioral corroboration: `9b7642df` (reference only).
- Status: **APPROVED** 2026-07-22 (gatekeeper review complete). Authoritative.
- Each fact below is tagged with a provenance marker:
	- **[2006]** - corroborated by the public 2006 Sun "ZFS On-Disk Specification" (the durable core, independently published).
	- **[modern]** - source-only; a post-2006 addition or refinement observed in current OpenZFS. Required to interoperate with pools written by recent OpenZFS, but not part of the original published spec.

This document specifies the **ZAP** (ZFS Attribute Processor) on-disk format: the name-value store that ZFS uses for every dictionary-like object - the MOS object directory, dataset and snapshot name maps, ZPL directories, property lists, and many others (the ZAP-class object types are listed in `05-dmu-dnode-objset.md` section 5). It answers request #6. A ZAP object maps a **key** (normally a NUL-terminated string) to a **value** (an array of 1-, 2-, 4-, or 8-byte integers). Two on-disk forms exist: the **micro ZAP**, a single-block linear table for small maps with simple values, and the **fat ZAP**, a hashed, multi-block structure for everything larger. **[2006]**

A ZAP is a DMU object; its blocks are reached through its dnode's block pointers (spec `05`), and multi-byte fields follow the native-byte-order rule with the containing block's byteorder bit as the swap signal (spec `04` section 4.8), except where this spec says otherwise (fat-ZAP name/value bytes, section 5.4).

---

## 1. Form discrimination: the block-type word

The first 64-bit word of a ZAP object's **block 0** identifies the form. The high bit is always set; the low bits select the kind: **[2006]**

| Word 0 value | Kind |
|--------------|------|
| `0x8000000000000003` | micro ZAP (the whole object is this one block) |
| `0x8000000000000001` | fat ZAP header block |
| `0x8000000000000000` | fat ZAP leaf block (never block 0; see section 5) |

- The value `0x8000000000000002` is unused. **[2006]**
- The word is stored in writer-native byte order; a reader byte-swaps it (with the rest of the block's structured fields) per the containing block pointer's byteorder bit before comparing. **[2006]**
- Reader dispatch: read block 0 of the object, correct byte order, examine word 0: micro -> section 3; fat header -> section 4. Any other value is corruption. **[2006]**

---

## 2. The ZAP hash

Both forms store a 64-bit **salt**; the fat ZAP places every entry by a salted 64-bit hash of its key. The hash function is an interop fact: an independent writer must produce identical hash values or its entries land in the wrong leaves. **[2006]** for the salted-64-bit-hash concept; the exact algorithm below is **[modern]** (source-confirmed detail not published in 2006).

### 2.1 Algorithm

The hash is a table-driven CRC-64 seeded with the salt:

```
h = salt                                    (salt is nonzero; section 2.2)
for each key byte c (order per 2.3):
    h = (h >> 8) XOR table[(h XOR c) & 0xFF]
h = h & 0xFFFFFFF000000000                  (keep only the top 28 bits; see below)
```

- `table` is the 256-entry reflected CRC-64 table for the **ECMA-182 polynomial, reflected form `0xC96C5795D7870F42`**, generated as:

  ```
  for b in 0..255:
      c = b
      repeat 8 times:
          c = (c & 1) ? (c >> 1) XOR 0xC96C5795D7870F42 : (c >> 1)
      table[b] = c
  ```
  **[modern]**
- **Truncation:** after the CRC loop, all but the **top 28 bits** are cleared (`h &= ~((1 << 36) - 1)`). The stored/compared hash therefore has at most 28 significant bits, at the top of the word; the collision differentiator (section 2.4) absorbs collisions. Exception: a ZAP with the wide-hash flag set (section 4.2) keeps the **top 48 bits** instead (`h &= ~((1 << 16) - 1)`); the truncation always leaves the bottom bits zero so a CD space exists. **[modern]**

### 2.2 Salt

- The salt is a 64-bit value fixed at object creation and stored in the form's header (micro: offset 8, section 3.1; fat: offset 80, section 4.1). It is **nonzero**; writers conventionally force the low bit on to guarantee this. Its value is otherwise arbitrary - a reader must use the stored salt, never recompute or assume one. **[2006]** for the stored-salt requirement; **[modern]** for the nonzero/odd convention.
- When a micro ZAP is upgraded to fat (section 7), the salt (and normalization flags) carry over unchanged, so hashes stay stable. **[modern]**

### 2.3 Key bytes fed to the hash

- **String keys** (the normal case): the bytes of the name **excluding the terminating NUL** - even though the *stored* name (sections 3.2, 5.3) includes the NUL and the stored name length counts it. This asymmetry is interop-critical: hashing the NUL too yields a different hash for every key. **[modern]**
- **Normalized keys:** if the normalization-flags field is nonzero (section 4.2), the hash is computed over the unicode-*normalized* form of the name, so that all case/normalization variants of a name hash identically. The normalization algorithm is deferred to the ZPL casefold spec; default pools store 0 and hash the raw bytes. **[modern]**
- **64-bit-integer keys** (fat only, flag-gated; section 4.2): the key is an array of 64-bit words; each word is fed to the CRC **least-significant byte first**, in array order (8 bytes per word). **[modern]**
- **Pre-hashed keys** (flag-gated): the first word of the integer key *is* the hash; the CRC is not run. **[modern]**

### 2.4 Collision differentiator (CD)

Because the effective hash is only 28 bits, distinct keys collide routinely. Every entry therefore carries a **collision differentiator (CD)**: an integer distinguishing entries with equal hash. The identity of an entry is the pair (hash, CD). **[2006]**

- Writers assign the **lowest CD not already used** by another entry with the same hash (starting at 0); a lone entry has CD 0. This assignment rule is a write-side interop fact (readers echo stored CDs, e.g. in directory seek cursors). **[2006]** concept / **[modern]** exact lowest-unused rule.
- Both forms store CD as a 32-bit field (sections 3.2, 5.3); the fat leaf entry's CD was originally 16 bits followed by 2 pad bytes and was later widened into the pad, at the same offset. The CD ceiling is `2^32 - 1` normally, but only **65535** in a wide-hash ZAP (section 4.2). CDs survive micro-to-fat upgrade unchanged. **[2006]** 16-bit field / **[modern]** widening and ceilings.

---

## 3. Micro ZAP

A micro ZAP packs the whole map into **one block**: a 64-byte header followed by an array of 64-byte entries. It is used while every entry has a value that is exactly **one 64-bit integer**, a name of **49 bytes or fewer** (50 with NUL), no entry needs a larger encoding, and everything fits in one block. **[2006]** The block size is set at creation and is at most **128 KiB**. **[modern]**

### 3.1 Header (bytes 0..63)

| Offset | Size | Field | Meaning | Provenance |
|--------|------|-------|---------|------------|
| 0 | 8 | block type | `0x8000000000000003` (section 1) | **[2006]** |
| 8 | 8 | salt | hash salt (section 2.2) | **[2006]** |
| 16 | 8 | normalization flags | name-normalization flags; 0 = none (section 4.2) | **[modern]** (was pad) |
| 24 | 40 | pad | five words, zero | **[2006]** |

### 3.2 Entries (bytes 64..end, 64 bytes each)

Entry count = `blocksize/64 - 1`. Each entry: **[2006]**

| Offset | Size | Field | Meaning |
|--------|------|-------|---------|
| 0 | 8 | value | the entry's single 64-bit value |
| 8 | 4 | cd | collision differentiator (section 2.4) |
| 12 | 2 | pad | zero |
| 14 | 50 | name | NUL-terminated string, max 49 chars + NUL |

- A slot is **free** iff its first name byte is 0 (empty name). Free and live slots may be interleaved anywhere in the array: writers place entries at arbitrary slots (placement is a write-time heuristic, not a lookup structure), so a reader **must scan every slot** and match by name. **[2006]**
- The name is compared as bytes (or normalized per the normalization flags). The hash (section 2) is *not* needed to read a micro ZAP - only for CD assignment and upgrade. **[2006]**
- Byte order: value, cd, and the header words are writer-native (swap per container); name bytes are bytes. **[2006]**

---

## 4. Fat ZAP: header block

A fat ZAP is a hash table spread over the object's blocks: **block 0** is the header, and every other block is either a **leaf** (section 5) or an **external pointer-table block** (section 4.3). All blocks are the object's data block size (at most 128 KiB for ZAP objects). **[2006]**

### 4.1 Header layout (block 0)

| Offset | Size | Field | Meaning | Provenance |
|--------|------|-------|---------|------------|
| 0 | 8 | block type | `0x8000000000000001` (section 1) | **[2006]** |
| 8 | 8 | magic | `0x2F52AB2AB` | **[2006]** |
| 16 | 8 | table start | first block id of the external pointer table; 0 if embedded | **[2006]** |
| 24 | 8 | table blocks | external pointer-table block count; 0 = table is embedded (4.3) | **[2006]** |
| 32 | 8 | table shift | log2 of pointer-table entry count | **[2006]** |
| 40 | 8 | table next | new-table block id, nonzero only while the table is being doubled (4.4) | **[2006]** |
| 48 | 8 | table copied | doubling progress counter (4.4) | **[2006]** |
| 56 | 8 | free block | next never-used block id (allocation high-water mark) | **[2006]** |
| 64 | 8 | leaf count | count of leaf blocks | **[2006]** |
| 72 | 8 | entry count | total entries in the ZAP | **[2006]** |
| 80 | 8 | salt | hash salt (section 2.2) | **[2006]** |
| 88 | 8 | normalization flags | name-normalization flags; 0 = none | **[modern]** (was pad) |
| 96 | 8 | flags | ZAP flags (4.2) | **[modern]** (was pad) |
| 104 | to half | pad | zeros to the block's midpoint | **[2006]** |
| blocksize/2 | to end | embedded pointer table | second half of the block (4.3) | **[2006]** |

### 4.2 Flags and normalization flags

`flags` (offset 96) is a bit set; all bits are **[modern]**, gated by pool features where applicable:

| Bit | Meaning |
|-----|---------|
| 1 << 0 | wide hash: hash keeps its top 48 bits instead of 28, and CD is capped at 65535 (sections 2.1, 2.4) |
| 1 << 1 | integer keys: keys are arrays of 64-bit words, not strings (sections 2.3, 5.4) |
| 1 << 2 | pre-hashed keys: first key word is the hash; the CRC is not run (implies integer keys) |

The normalization-flags field (offset 88, and micro offset 16) is nonzero when names are unicode-normalized (and possibly case-folded) for hashing and matching; the encoding of the flag bits and the normalization itself are specified with the ZPL casefold material. Default pools store 0. **[modern]**

### 4.3 The pointer table

The pointer table maps the **top `shift` bits of the hash** to a leaf block id: `index = hash >> (64 - shift)`; each table entry is a 64-bit block id (within this object) of the leaf covering that hash range. **[2006]**

For block size `2^bs` bytes:

- **Embedded** (table-blocks field == 0): the table occupies the **second half of the header block** - `2^(bs-4)` entries (half the block's bytes, 8 bytes each). Its shift is then exactly `bs - 4`. A new fat ZAP starts this way. **[2006]**
- **External** (table-blocks field != 0): the table has outgrown the header block and lives in that many consecutive whole blocks starting at the table-start block id; entry `i` is word `i & (2^(bs-3) - 1)` of block `start + (i >> (bs-3))` (each block holds `2^(bs-3)` entries). The embedded area is then unused. **[2006]**
- Several **consecutive** table entries may name the same leaf: a leaf with `prefix_len < shift` is referenced by exactly `2^(shift - prefix_len)` consecutive entries (section 5.1). Every table entry names a valid leaf - there are no empty buckets. **[2006]**

### 4.4 Table-doubling state

When the table doubles (shift+1), the copy is incremental: the table-next field is the block id of the new (larger) table while copying is in progress, and the table-copied field counts old-table blocks already copied. In any quiesced pool image both are 0; a read-only consumer may treat nonzero mid-migration state as "use the old table entries not yet superseded", but will not normally encounter it. **[2006]** fields / **[modern]** quiesced-zero observation.

---

## 5. Fat ZAP: leaf blocks

A leaf holds the entries for one hash-prefix range. Layout: a **48-byte header**, then a **hash table** of 2-byte chunk references, then an array of **24-byte chunks**. **[2006]**

For a leaf block of size `2^bs` bytes:

| Region | Offset | Size |
|--------|--------|------|
| header | 0 | 48 bytes |
| leaf hash table | 48 | `2^(bs-5)` entries x 2 bytes = `2^(bs-4)` bytes |
| chunks | `48 + 2^(bs-4)` | `((2^bs - 2^(bs-4)) / 24) - 2` chunks x 24 bytes |

The `-2` in the chunk count accounts for the header's 48 bytes = two chunks' worth. (Example, 16 KiB leaf: 512 hash entries at byte 48, then 638 chunks at byte 1072, ending exactly at 16384.) **[2006]**

### 5.1 Leaf header (bytes 0..47)

| Offset | Size | Field | Meaning | Provenance |
|--------|------|-------|---------|------------|
| 0 | 8 | block type | `0x8000000000000000` (section 1) | **[2006]** |
| 8 | 8 | pad | originally a next-leaf chain link; now unused, zero | **[2006]** field / **[modern]** obsolete |
| 16 | 8 | prefix | the hash-prefix value this leaf covers | **[2006]** |
| 24 | 4 | magic | `0x2AB1EAF` | **[2006]** |
| 28 | 2 | nfree | count of free chunks | **[2006]** |
| 30 | 2 | nentries | count of entry chunks | **[2006]** |
| 32 | 2 | prefix len | significant bits of `prefix` | **[2006]** |
| 34 | 2 | freelist | head chunk id of the free-chunk list; `0xFFFF` = none | **[2006]** |
| 36 | 1 | flags | bit 0: entries in each chain are CD-sorted | **[modern]** |
| 37 | 11 | pad | zero | **[2006]** |

- **Prefix invariant:** the leaf contains exactly the entries whose hash's top `prefix_len` bits equal `prefix`, and `2^(shift - prefix_len)` consecutive pointer-table entries reference it (section 4.3). `prefix_len <= shift` always. (`prefix_len == 0` occurs only in a brand-new one-leaf ZAP, where every table entry points at the single leaf.) **[2006]**

### 5.2 Leaf hash table

`2^(bs-5)` entries of 2 bytes; entry values are **chunk ids** heading a chain of entry chunks, or **`0xFFFF`** meaning empty. The table is indexed by the hash bits **immediately below the leaf's prefix**:

```
index = (hash >> (64 - prefix_len - (bs - 5))) & (2^(bs-5) - 1)
```

i.e. take the `bs-5` bits following the top `prefix_len` bits. **[2006]**

### 5.3 Chunks

Chunk `N` occupies 24 bytes at leaf offset `48 + 2^(bs-4) + 24*N`; chunk ids index this array, and `0xFFFF` is the universal end-of-chain marker. Byte 0 of every chunk is its **type**: **[2006]**

| Type code | Kind |
|-----------|------|
| 251 | array chunk (name/value bytes) |
| 252 | entry chunk (one ZAP entry) |
| 253 | free chunk |

**Entry chunk** (type 252):

| Offset | Size | Field | Meaning |
|--------|------|-------|---------|
| 0 | 1 | type | 252 |
| 1 | 1 | value int size | bytes per value integer: 1, 2, 4, or 8 |
| 2 | 2 | next | next entry chunk in this hash chain; `0xFFFF` = end |
| 4 | 2 | name chunk | first array chunk of the key |
| 6 | 2 | name length | key length in integers (strings: bytes **including** NUL) |
| 8 | 2 | value chunk | first array chunk of the value |
| 10 | 2 | value length | value length in integers of value-int-size bytes |
| 12 | 4 | cd | collision differentiator (section 2.4; originally 16 bits + 2 pad, widened in place) |
| 16 | 8 | hash | the entry's full (truncated) hash |

**[2006]** for the layout; **[modern]** for the note that the name length counts 8-byte words when the integer-keys flag is set.

**Array chunk** (type 251): byte 0 = type; bytes **1..21** = 21 payload bytes; bytes 22..23 = next array chunk id (`0xFFFF` = end). Name and value byte streams are each stored as a chain of array chunks, filled in order, 21 bytes per chunk, last chunk zero-padded. **[2006]**

**Free chunk** (type 253): byte 0 = type; bytes 22..23 = next free chunk id; the freelist links all free chunks from the header's `freelist` field. Bytes 1..21 are ignored. **[2006]**

- An entry's chunks (the entry chunk, its name chain, its value chain) live **wholly within one leaf**; entries never span blocks. This bounds the maximum name+value size by the leaf's chunk capacity. **[2006]**

### 5.4 Name and value encoding in array chunks

- **Values** are stored **big-endian**, always, regardless of writer byte order: each value integer is written as value-int-size big-endian bytes, concatenated in array order into the chunk payload stream. Total value bytes = value length x value int size. **[2006]**
- **String keys** are stored as their bytes including the terminating NUL (name-length bytes, int size 1). **[2006]**
- **Integer keys** (flag-gated) are stored as name-length 64-bit words, each big-endian, in the same payload stream fashion. **[modern]**
- Because these payload bytes are order-fixed (or plain bytes), they are **never byte-swapped** on read; only the structural fields around them are (section 6.4). **[2006]**

---

## 6. Reader recipe (fat ZAP lookup by key)

1. Read block 0, byte-swap per the container's byteorder bit, verify block type `0x8000000000000001` and magic `0x2F52AB2AB` (section 4.1). **[2006]**
2. Compute the hash `h` of the key with the stored salt (section 2; normalize the key first iff the normalization flags are nonzero; apply the flag variants of section 4.2). **[modern]** for variants, else **[2006]**.
3. Index the pointer table with the top `shift` bits of `h` (embedded or external per the table-blocks field; section 4.3) to get a leaf block id. **[2006]**
4. Read that leaf, byte-swap, verify block type `0x8000000000000000` and magic `0x2AB1EAF`, and check the prefix invariant (`h`'s top `prefix_len` bits == `prefix`). **[2006]**
5. Index the leaf hash table with the next `bs-5` hash bits (section 5.2); walk the entry-chunk chain via `next`. **[2006]**
6. For each entry chunk: compare its stored hash to `h`; if equal, compare the key to the entry's name chain byte-for-byte (or normalized per the normalization flags). CD does not participate in a by-name match - names are unique; (hash, CD) identity matters for cursor/iteration use. **[2006]**
7. On match, gather value-length x value-int-size bytes from the value chain and decode each integer as big-endian (section 5.4). **[2006]**

Iteration (readdir-style) walks leaves via the pointer table (skipping duplicate consecutive entries), then each leaf's hash table and chains; entries within a chain may be CD-sorted per the leaf flag bit. Cursors encode position as (hash, CD). **[2006]** concept / **[modern]** cursor encoding detail deferred to the ZPL spec.

---

## 7. Micro-to-fat upgrade (writer facts)

A writer creates the micro form by default and must convert to fat when an insert requires: **[2006]** for the two-form design; conditions **[modern]** in exact detail:

- a value that is not exactly one 64-bit integer (any other int size or count), or
- a name longer than 49 bytes (50 with NUL), or
- a slot when the single block is full (and the block is already at its maximum size), or
- ZAP flags (section 4.2) at creation - flagged ZAPs are fat from birth.

Upgrade re-inserts every micro entry into a new fat structure in the same object, preserving the **salt**, the **normalization flags**, and each entry's **CD**; hashes are computed with the same function, so the map's identity is unchanged. The block-type word of block 0 changes from micro to fat header. **[modern]**

---

## 8. Byte-swap rules

Structured fields are writer-native and swapped when the container's byteorder bit differs from the host (spec `04` section 4.8). Field-by-field: **[2006]** for the rule; the per-kind lists follow the layouts above:

- **Micro block:** header words (type, salt, normalization flags, pads); per entry: the 64-bit value and 32-bit cd. Name bytes untouched.
- **Fat header block:** all header words (4.1) and every pointer-table entry (embedded or external blocks).
- **Leaf block:** header fields (each at its own width, 5.1); every 16-bit leaf-hash-table entry; then per chunk by type byte: entry chunks swap their 16-bit fields, the 32-bit cd, and the 64-bit hash (type and int size are single bytes); array chunks swap only the 16-bit next field - **payload bytes are never swapped**; free chunks swap only the 16-bit next field.

---

## 9. Constants and rules quick reference

| Item | Value / rule | Provenance |
|------|--------------|------------|
| Block-type words | micro `0x8000000000000003`, fat header `0x8000000000000001`, leaf `0x8000000000000000` | **[2006]** |
| Fat header magic | `0x2F52AB2AB` | **[2006]** |
| Leaf magic | `0x2AB1EAF` | **[2006]** |
| Hash | CRC-64, reflected ECMA-182 poly `0xC96C5795D7870F42`, seeded with salt | **[modern]** |
| Hash truncation | keep top 28 bits (`h &= ~((1<<36)-1)`); top 48 with the wide-hash flag | **[modern]** |
| String-key hashing | excludes the terminating NUL (stored name/length include it) | **[modern]** |
| Salt | stored, nonzero, arbitrary; preserved across upgrade | **[2006]** / **[modern]** details |
| CD | lowest unused among same-hash entries, from 0; 32-bit field; max 65535 if wide-hash | **[2006]** / **[modern]** rule |
| Micro header / entry | 64 bytes / 64 bytes; entries = `blocksize/64 - 1` | **[2006]** |
| Micro name limit | 49 bytes + NUL (50-byte field) | **[2006]** |
| Micro value | exactly one uint64 | **[2006]** |
| Micro free slot | first name byte == 0; reader scans all slots | **[2006]** |
| Micro max block | 128 KiB | **[modern]** |
| Pointer-table index | `hash >> (64 - shift)`; entry = leaf block id | **[2006]** |
| Embedded table | second half of header block; `shift = bs - 4` | **[2006]** |
| External table | `2^(bs-3)` entries per block, table-blocks blocks from table-start | **[2006]** |
| Leaf regions | 48-byte header; `2^(bs-5)` x 2-byte hash table; 24-byte chunks | **[2006]** |
| Leaf chunk count | `((2^bs - 2^(bs-4)) / 24) - 2` | **[2006]** |
| Leaf hash index | `(hash >> (64 - prefix_len - (bs-5))) & (2^(bs-5)-1)` | **[2006]** |
| Chunk types | 251 array, 252 entry, 253 free | **[2006]** |
| Array chunk payload | 21 bytes (bytes 1..21), next at 22..23 | **[2006]** |
| End-of-chain | `0xFFFF` (chains, freelist, leaf hash table) | **[2006]** |
| Value byte order | always big-endian in array chunks, int size in {1,2,4,8} | **[2006]** |
| Entry span | an entry's chunks never cross a leaf boundary | **[2006]** |
| Flags | 1<<0 wide hash, 1<<1 integer keys, 1<<2 pre-hashed | **[modern]** |
| Prefix invariant | top `prefix_len` hash bits == `prefix`; `2^(shift-prefix_len)` consecutive table refs | **[2006]** |
