# ZFS on-disk format: DMU dnode and object set

**Provenance**

- OpenZFS commit studied for behavioral corroboration: `9b7642df` (reference only).
- Status: **APPROVED** 2026-07-22 (gatekeeper review complete). Authoritative.
- Each fact below is tagged with a provenance marker:
	- **[2006]** - corroborated by the public 2006 Sun "ZFS On-Disk Specification" (the durable core, independently published).
	- **[modern]** - source-only; a post-2006 addition or refinement observed in current OpenZFS. Required to interoperate with pools written by recent OpenZFS, but not part of the original published spec.

This document specifies the two structures at the heart of the Data Management Unit (DMU): the **dnode**, the fixed-layout record that describes a single DMU object, and the **object set**, the container that groups a numbered array of dnodes and roots them at a single block pointer. It answers request #5. Every file, directory, ZAP, dataset, and pool-metadata object is a DMU object described by a dnode; every dataset and the pool-wide metadata (the MOS) is an object set. Traversing a pool means: block pointer -> object set -> meta-dnode -> dnode array -> per-object dnode -> that object's block pointers -> data.

This spec builds directly on `04-block-pointer-dva.md` (the 128-byte block pointer, and the `type`/`checksum`/`compress` identity codes it references) and `01-vdev-label-uberblock.md` (the root block pointer in the active uberblock, and the native-byte-order rule). The DMU **object-type** enumeration is defined here in section 5 because the dnode's `type` byte selects from it. ZAP internals, DSL dataset structures, and ZPL/znode layout are named where an object type points at them but are specified in their own later documents.

---

## 1. The dnode

A **dnode** describes one DMU object: its type, its block geometry (block sizes, indirection depth), its up-to-three top-level block pointers, an optional inline "bonus" attribute area, and an optional spill pointer. Dnodes are stored packed in the data blocks of a special meta-dnode (section 4).

### 1.1 Size, slots, and blocks

- The dnode is built from fixed **512-byte slots**. The minimum (and classic) dnode is exactly one slot = **512 bytes**. **[2006]**
- Dnodes are stored 32 to a **16 KiB dnode block** (block shift 14; `16384 / 512 = 32` slots per block). **[2006]**
- A dnode may span **multiple consecutive slots**: its physical size is `(extra_slots + 1) * 512` bytes, where `extra_slots` is the byte at offset 12 (section 1.2). `extra_slots = 0` yields a 512-byte dnode and is on-disk-compatible with software that predates variable-length dnodes. Maximum size is one full dnode block = **16384 bytes** (32 slots). **[modern]** (the multi-slot "large dnode" capability; a single 512-byte slot is **[2006]**)
- When a dnode occupies more than one slot, the trailing slots it consumes are **interior** slots: they are not independently addressable object numbers (section 4.3). **[modern]**

### 1.2 The 64-byte dnode core

The first **64 bytes** of every dnode are a fixed header. All offsets are byte offsets from the start of the dnode; multi-byte fields are in the writer's native byte order (see `01` section 4). **[2006]** for the field set and order except where a field is tagged **[modern]**.

| Offset | Size | Field | Meaning
| :-- | :-- | :-- | :--
| 0  | 1 | type            | DMU object type of this object (section 5). 0 = unallocated dnode. **[2006]**
| 1  | 1 | indirect shift  | ln2 of the indirect-block size in bytes (section 3.2). **[2006]**
| 2  | 1 | levels          | number of block-pointer indirection levels; 1 means the dnode's block pointers point directly at data (section 3.2). **[2006]**
| 3  | 1 | nblkptr         | count of block pointers in the dnode's tail (section 1.3); 1..3 for a 512-byte dnode. **[2006]**
| 4  | 1 | bonus type      | DMU object type describing the bonus buffer's contents (section 5); 0 = none. **[2006]**
| 5  | 1 | checksum        | checksum identity override for this object's blocks (`04` section 7.1); 0 = inherit. **[2006]**
| 6  | 1 | compress        | compression identity override for this object's blocks (`04` section 7.2); 0 = inherit. **[2006]**
| 7  | 1 | flags           | bit flags, section 1.4. (In the original format this byte was reserved/unused; the flag semantics are **[modern]**, the byte position is **[2006]**.)
| 8  | 2 | data block sectors | data (level-0) block size in 512-byte sectors; block size in bytes = value << 9 (section 3.1). **[2006]**
| 10 | 2 | bonus length    | length in bytes of the bonus buffer (section 1.3). **[2006]**
| 12 | 1 | extra slots     | number of additional 512-byte slots this dnode consumes beyond the first (section 1.1). **[modern]** (in the original format offset 12 was part of the reserved padding: **[2006]** as pad)
| 13 | 3 | reserved        | pad; zero. **[2006]**
| 16 | 8 | max block id    | largest allocated logical block id in this object (its highest data block index). **[2006]**
| 24 | 8 | used            | disk space charged to this object; in bytes if flag bit 0 is set, else in 512-byte sectors (section 1.4). **[2006]**
| 32 | 32 | reserved       | four reserved 64-bit words; zero. Protected by the block MAC when the dnode block is encrypted. **[2006]** as reserved (MAC protection **[modern]**)

The core is exactly 64 bytes (offset 0 through 63). The remaining bytes of the dnode (offset 64 to the end of its last slot) are the **tail region** (section 1.3).

### 1.3 The tail region: block pointers, bonus, spill

The tail begins at offset **64** and runs to the end of the dnode's last slot (448 bytes for a 512-byte dnode). It holds, in this order:

1. **Block pointers**: `nblkptr` block pointers, each 128 bytes (`04`), starting at offset 64. These are the object's top-level pointers (section 3.2). For a 512-byte dnode `nblkptr` is at most 3 (`(512 - 64) / 128 = 3`). **[2006]**
2. **Bonus buffer** (optional): immediately after the block pointers, `bonus length` bytes of inline attribute data, interpreted per the **bonus type** byte. Used for small per-object metadata (for example a ZPL znode or a DSL dataset record) without a separate data block. **[2006]**
3. **Spill block pointer** (optional): when flag bit 2 is set (section 1.4), the **last 128 bytes** of the dnode's last slot are a block pointer to a "spill" block that holds attribute data too large for the bonus area. Its position is `dnode_start + (extra_slots + 1) * 512 - 128`. **[modern]**

For a classic 512-byte dnode the three layouts the tail can take are: (a) up to three block pointers filling the tail; (b) one block pointer plus a bonus buffer up to 320 bytes; (c) one block pointer, a smaller bonus buffer, and a spill pointer in the final 128 bytes. The maximum classic bonus length is **320 bytes** (`512 - 64 - 128`). **[2006]** for (a) and (b); the spill variant (c) is **[modern]**. Larger dnodes extend the bonus region proportionally. **[modern]**

### 1.4 Flag byte (offset 7)

Bit 0 is least significant. All flag semantics are **[modern]** (the byte was reserved in the original format).

| Bit | Mask | Meaning
| :-- | :-- | :--
| 0 | 0x01 | `used` field (offset 24) is in bytes; if clear, `used` is in 512-byte sectors.
| 1 | 0x02 | user-used accounting has been computed for this object.
| 2 | 0x04 | a spill block pointer is present in the tail (section 1.3 item 3).
| 3 | 0x08 | user/group/project object accounting has been computed for this object.

Bits 4..7 are reserved. A reader that does not understand a set bit should preserve it and must still honor bit 0 (unit of `used`) and bit 2 (presence of the spill pointer), which affect on-disk interpretation.

---

## 2. Reading a dnode: worked field extraction

Given the 512-byte (or larger) dnode image `D` in native byte order:

1. `type = D[0]`. If 0, the dnode is unallocated - stop.
2. `nblkptr = D[3]`, `bonus_type = D[4]`, `bonus_len = u16(D[10])`, `flags = D[7]`, `extra_slots = D[12]`.
3. Data block size = `u16(D[8]) << 9` bytes. Indirect block size = `1 << D[1]` bytes. Indirection levels = `D[2]`.
4. Block pointers occupy `D[64 .. 64 + nblkptr*128)`; pointer `i` starts at `64 + i*128`.
5. Bonus buffer, if `bonus_len > 0`, occupies `D[64 + nblkptr*128 .. + bonus_len)`.
6. If `flags & 0x04`, the spill pointer is the 128 bytes ending at offset `(extra_slots + 1) * 512`.
7. `used` bytes = `u64(D[24])` if `flags & 0x01`, else `u64(D[24]) << 9`.

---

## 3. Object block geometry and indirection

### 3.1 Data (level-0) blocks

- The object's data-block size is `data_block_sectors << 9` bytes (offset 8), a power of two from 512 bytes up to the pool's maximum block size. **[2006]**
- Logical block ids run `0 .. max_block_id` (offset 16). A hole (never-written block) is represented by a hole block pointer (`04` section 6.1). **[2006]**

### 3.2 The indirection tree

- If `levels == 1`, the dnode's `nblkptr` block pointers point **directly** at data blocks 0..nblkptr-1. **[2006]**
- If `levels > 1`, each dnode block pointer points at an **indirect block**: an array of block pointers, each 128 bytes, filling the indirect block. The number of pointers per indirect block is `indirect_block_size / 128 = 1 << (indirect_shift - 7)`. Indirect blocks nest `levels - 1` deep before reaching data blocks. **[2006]**
- To locate data block `blkid`, split `blkid` into `(indirect_shift - 7)`-bit digits, most-significant digit selecting within the dnode's top-level pointer array, each lower digit indexing the next indirect level. **[2006]**
- `indirect_shift` is bounded to 12..17 (4 KiB..128 KiB indirect blocks) in current OpenZFS. **[modern]** for the exact bounds; the shift field itself is **[2006]**.

The block-pointer format, birth-txg fill semantics, and checksum/compression identity codes used at every level here are specified in `04`.

---

## 4. The object set

An **object set** is the container that turns a single block pointer into a numbered universe of objects. It is a fixed-layout header whose first member is the **meta-dnode**, a dnode whose data blocks are the packed array of every dnode in the set.

### 4.1 The object-set header layout

The object-set header is a power-of-two block (1024, 2048, or 4096 bytes; section 4.4). Offsets are byte offsets from the start of the block, native byte order.

| Offset | Size | Field | Meaning
| :-- | :-- | :-- | :--
| 0    | 512 | meta-dnode      | a dnode (section 1) whose object data is the dnode array (section 4.2). **[2006]**
| 512  | 192 | intent-log header | the ZIL header for this object set (section 4.5). **[2006]**
| 704  | 8   | os type         | object-set type (section 4.6). **[2006]**
| 712  | 8   | os flags        | object-set flags (section 4.7). **[2006]**
| 720  | 32  | portable MAC    | message authentication code for encrypted object sets. **[modern]**
| 752  | 32  | local MAC       | message authentication code for encrypted object sets. **[modern]**
| 784  | 240 | reserved (pad)  | pad to the 1024-byte boundary; zero. **[2006]** as reserved (the two MAC fields above were carved from this original pad region: **[modern]**)
| 1024 | 512 | user-used dnode | dnode for user-space accounting (section 4.3). **[modern]**
| 1536 | 512 | group-used dnode | dnode for group-space accounting. **[modern]**
| 2048 | 512 | project-used dnode | dnode for project-space accounting. **[modern]**
| 2560 | 1536 | reserved (pad) | pad to the 4096-byte boundary; zero. **[modern]**

The original object set is the first **1024 bytes** (meta-dnode + ZIL header + type + flags + pad): the meta-dnode, ZIL header, `os type`, and `os flags` are **[2006]**. The two MAC fields occupy 64 bytes carved from the original trailing pad. The accounting dnodes at 1024/1536/2048 are later extensions (section 4.4).

### 4.2 The meta-dnode and the dnode array

- The **meta-dnode** (offset 0) is an ordinary dnode whose logical object data is the packed array of all dnodes in this set. Its data-block/indirection geometry (section 3) is how the dnode array is spread across disk. **[2006]**
- Object number `N` occupies the dnode slot at byte offset `N << 9` (`N * 512`) within the meta-dnode's logical data. Reading object `N` means reading `512` (or more, for a large dnode) bytes at that offset through the meta-dnode's block pointers. **[2006]**
- **Object 0 is the meta-dnode itself** (it is self-describing: object 0's dnode is the meta-dnode). **[2006]**

### 4.3 Special and interior object numbers

- Object number **0** is the meta-dnode (section 4.2). **[2006]**
- The **user-used (-1)**, **group-used (-2)**, and **project-used (-3)** accounting objects do not live in the meta-dnode array; their dnodes are the dedicated slots in the object-set header at offsets 1024/1536/2048 (section 4.1). Object numbers `<= 0` are "special" and are not looked up through the meta-dnode array. **[modern]** (the negative accounting objects; object 0 special-ness is **[2006]**)
- When a large dnode (section 1.1) occupies object slot `N` and spans `k` slots, slots `N+1 .. N+k-1` are **interior** and are not valid object numbers. **[modern]**

### 4.4 Object-set versions and size detection

- Three header sizes exist: **1024** (original), **2048** (adds the user-used and group-used dnodes), and **4096** (adds the project-used dnode). **[2006]** for 1024; the 2048 and 4096 sizes are **[modern]**.
- A reader detects which fields are present from the size of the decompressed object-set block: `>= 2048` bytes means the user-used and group-used dnodes are present; `>= 4096` means the project-used dnode is present. **[modern]**

### 4.5 The intent-log (ZIL) header

The 192-byte ZIL header at offset 512 records the object set's intent-log chain. Its fields (each a 64-bit value except the embedded block pointer), native byte order: **[2006]**

| Offset (within header) | Size | Field | Meaning
| :-- | :-- | :-- | :--
| 0   | 8   | claim txg      | txg in which the log blocks were claimed (0 if none). **[2006]**
| 8   | 8   | replay seq     | highest log sequence number replayed. **[2006]**
| 16  | 128 | log block ptr  | block pointer to the head of the log-block chain (`04`). **[2006]**
| 144 | 8   | claim block seq | highest claimed log-block sequence number. **[2006]**
| 152 | 8   | flags          | ZIL header flags (section 4.5 note). **[modern]**
| 160 | 8   | claim lr seq   | highest claimed log-record sequence number. **[modern]**
| 168 | 24  | reserved       | three reserved 64-bit words; zero. **[2006]**

ZIL header flags: bit 0 = replay-needed (internal, not required for a read-only interpreter); bit 1 = the claim-lr-seq field is valid. The full log-record and log-block-chain format is a separate topic and is not specified here. **[modern]**

### 4.6 Object-set type (offset 704)

A 64-bit code identifying what kind of object set this is: **[2006]**

| Code | Meaning
| :-- | :--
| 0 | none
| 1 | meta object set (the pool-wide MOS)
| 2 | ZFS filesystem (ZPL objects)
| 3 | ZVOL (a volume)

Codes 0..3 are the on-disk values. **[2006]** Two further enumerators exist in the source as in-memory sentinels ("other", for testing, and "any") and are not written to disk.

### 4.7 Object-set flags (offset 712)

A 64-bit flag word. All defined bits are **[modern]** (accounting-completion markers; the field itself is **[2006]**):

| Bit | Mask | Meaning
| :-- | :-- | :--
| 0 | 0x01 | user-accounting is complete and current.
| 1 | 0x02 | user-object accounting is complete and current.
| 2 | 0x04 | project-quota accounting is complete and current.

---

## 5. DMU object types

The dnode `type` byte (offset 0) and `bonus type` byte (offset 4), and the block-pointer `type` field (`04` section 4.5), all select from one enumeration of DMU object types. Two forms exist:

### 5.1 Classic (table-indexed) types

Values **0 .. N-1** are a fixed ordered enumeration. Each identifies both the object's role and how its data byteswaps. The ordered list below is the on-disk numbering; a reader keys off the number, not any name. **[2006]** for the numbering through the classic ZPL/ZVOL/test range; **[modern]** for the appended entries (roughly value 27 onward: error log, history, permissions, ACL/SYSACL, FUID, dedup/DDT, user-refs, SA-family, dead-list, clones, and BPOBJ-subobj types).

| # | Role | # | Role
| :-- | :-- | :-- | :--
| 0 | none (unallocated) | 28 | error log
| 1 | object directory (ZAP) | 29 | pool history (bytes)
| 2 | object array | 30 | pool history offsets
| 3 | packed nvlist (bytes) | 31 | pool properties (ZAP)
| 4 | packed nvlist size | 32 | delegated permissions (ZAP)
| 5 | block-pointer object | 33 | ACL
| 6 | block-pointer object header | 34 | system ACL
| 7 | space-map header | 35 | FUID table (packed nvlist bytes)
| 8 | space map | 36 | FUID table size
| 9 | intent log | 37 | next-clones (ZAP)
| 10 | dnode | 38 | scan queue (ZAP)
| 11 | object set | 39 | user/group used (ZAP)
| 12 | DSL directory | 40 | user/group quota (ZAP)
| 13 | DSL directory child map (ZAP) | 41 | user refs (ZAP)
| 14 | DSL dataset snapshot map (ZAP) | 42 | dedup table (ZAP)
| 15 | DSL properties (ZAP) | 43 | dedup stats (ZAP)
| 16 | DSL dataset | 44 | system attributes
| 17 | ZPL znode | 45 | SA master node (ZAP)
| 18 | old ACL | 46 | SA attr registration (ZAP)
| 19 | plain file contents (bytes) | 47 | SA attr layouts (ZAP)
| 20 | directory contents (ZAP) | 48 | scan translation (ZAP)
| 21 | master node (ZAP) | 49 | dedup (synthetic block pointer)
| 22 | unlinked set (ZAP) | 50 | dead list (ZAP)
| 23 | ZVOL data (bytes) | 51 | dead-list header
| 24 | ZVOL properties (ZAP) | 52 | DSL clones (ZAP)
| 25 | plain other (bytes, test) | 53 | block-pointer sub-object
| 26 | uint64 other (test) | | (values continue; end sentinel follows the last)
| 27 | ZAP other (test) | | 

The parenthetical after several entries (ZAP, bytes, UINT64, ...) is the byteswap/interpretation class the source records for that type; a byteswapping reader uses it, a native-order reader does not. The classic durable set runs through the ZPL/ZVOL/test range (values 0..27); the pool-metadata, delegation, ACL, FUID, accounting, dedup, SA, and dead-list types were added over time and are **[modern]**.

### 5.2 Bit-encoded ("new") types

Values with the **high bit (0x80) set** are self-describing rather than table indexed. The low bits carry: byteswap class in bits 0..4 (mask 0x1f), a **metadata** flag at 0x40, and an **encrypted** flag at 0x20. These encode "an object of byteswap class X that is/ isn't metadata, is/ isn't encrypted" without consuming a numbered table entry, and are used for generic typed objects. **[modern]**

The ten byteswap classes referenced by both forms are a fixed ordered set: uint8, uint16, uint32, uint64, ZAP, dnode, object-set, znode, old-ACL, ACL (in that order). A reader only needs them if it byteswaps foreign-endian data; the class selects which fixed-width swap to apply. **[2006]** for the uint*/ZAP/dnode/object-set/znode/ACL classes.

---

## 6. Traversal summary

To reach an arbitrary object's data from a root block pointer (for example the active uberblock's root pointer, `01` section 5, which points at the MOS object set):

1. Read the object-set header block through the root block pointer (`04` to decode the pointer). Detect its size (section 4.4).
2. Take the **meta-dnode** at offset 0 (section 4.2).
3. For target object number `N`: read the dnode at meta-dnode logical offset `N << 9` by walking the meta-dnode's indirection tree (section 3.2) to the right data block, then take the 512-byte (or larger) slot at that offset.
4. Interpret that dnode (section 2): its `type` (section 5) says what the object is; its block pointers and geometry (section 3) say where the object's data is.
5. Walk that dnode's own indirection tree to reach the object's data blocks.

The pool's named metadata (the object-directory ZAP at object 1 of the MOS, root-dataset pointer, feature maps, and so on) is reached by this procedure and is specified with ZAP and DSL in later documents.

---

## 7. Constants quick reference

| Constant | Value | Provenance
| :-- | :-- | :--
| dnode slot size | 512 bytes | [2006]
| dnode core size | 64 bytes | [2006]
| dnode block size | 16384 bytes (32 slots) | [2006]
| max block pointers, 512-byte dnode | 3 | [2006]
| max bonus length, 512-byte dnode | 320 bytes | [2006]
| max dnode size | 16384 bytes (32 slots) | [modern]
| indirect block shift range | 12..17 (4 KiB..128 KiB) | [modern]
| block pointers per indirect block | `indirect_size / 128` | [2006]
| object-set header sizes | 1024 / 2048 / 4096 bytes | 1024 [2006]; 2048, 4096 [modern]
| meta-dnode object number | 0 | [2006]
| user/group/project-used objects | -1 / -2 / -3 | [modern]
| ZIL header size | 192 bytes | [2006]
| MAC field length | 32 bytes each | [modern]

---

## 8. Cross-references

- `01-vdev-label-uberblock.md` - the active uberblock's root block pointer points at the MOS object set; native-byte-order rule.
- `04-block-pointer-dva.md` - the 128-byte block pointer embedded throughout this structure (dnode pointers, meta-dnode pointers, ZIL log pointer, spill pointer), and the checksum/compression identity codes referenced by the dnode's `checksum`/`compress` bytes.
- ZAP, DSL dataset, and ZPL layouts (later documents) - the contents of objects whose types are enumerated in section 5.
