# ZFS on-disk format: DSL directories, datasets, and snapshots

**Provenance**

- OpenZFS commit studied for behavioral corroboration: `9b7642df` (reference only).
- Status: **APPROVED** 2026-07-22 (gatekeeper review complete). Authoritative.
- Each fact below is tagged with a provenance marker:
	- **[2006]** - corroborated by the public 2006 Sun "ZFS On-Disk Specification" (the durable core, independently published).
	- **[modern]** - source-only; a post-2006 addition or refinement observed in current OpenZFS. Required to interoperate with pools written by recent OpenZFS, but not part of the original published spec.

This document specifies the Dataset and Snapshot Layer (DSL): the MOS objects that organize a pool into a tree of named filesystems, volumes, snapshots, and clones. It answers request #7. The DSL is built entirely from DMU objects living in the MOS (`05`), wired together by object numbers, with ZAP objects (`06`) providing every name-to-number map. Two record types carry the state: the **DSL directory** (one per filesystem/volume lineage; holds the namespace tree, properties, and space accounting) and the **DSL dataset** (one per head filesystem/volume and one per snapshot; holds the block pointer to that dataset's object set and the snapshot chain).

This spec builds on `05-dmu-dnode-objset.md` (dnodes, bonus buffers, the MOS, the object-type enumeration in its section 5.1 - the type numbers cited here are from that table), `06-zap.md` (every name map below is a micro or fat ZAP), and `04-block-pointer-dva.md` (the 128-byte block pointer embedded in the dataset record). The ZPL layout inside a filesystem's object set is a later document.

---

## 1. DSL objects and their wiring

All DSL objects live in the **MOS** (the object set of type 1, reached from the active uberblock's root block pointer; `05` section 6). The entry point is the **object directory**, the ZAP at MOS object number **1**. **[2006]**

Object-directory keys relevant to the DSL (string keys, uint64 values unless noted):

| Key | Value | Provenance
| :-- | :-- | :--
| `root_dataset` | object number of the pool's root DSL directory | [2006]
| `free_bpobj` | object number of the pool-wide free-list block-pointer object (section 6.1) | [modern]
| `com.delphix:deleted_clones` | ZAP of clones pending async destroy | [modern]
| `tmp_userrefs` | ZAP of temporary snapshot holds | [modern]

The object directory also carries pool-wide keys outside DSL scope (`config`, feature maps, error log, history, pool properties, scan state, ...); those are specified with their own topics.

The wiring, from the top:

1. `root_dataset` names the **root DSL directory** (section 2). Every other directory is reached from it through child-map ZAPs.
2. A directory's **head dataset** field names the **DSL dataset** (section 3) of the live filesystem or volume.
3. The head dataset's **snapshot-names ZAP** names its snapshots, each itself a DSL dataset, chained by prev/next pointers (section 4).
4. Each dataset's embedded **block pointer** points at that dataset's object set (`05`) - the actual filesystem or volume contents.

### 1.1 Record placement: bonus buffers

Both DSL record types live in the **bonus buffer** of their dnode (`05` section 1.3), not in object data:

- A **DSL directory** object has object type 12, bonus type 12, bonus length **256** bytes. Its dnode has no object data. **[2006]**
- A **DSL dataset** object has object type 16, bonus type 16, bonus length **320** bytes - exactly the classic maximum bonus (`05` section 1.3). Its dnode has no object data in the classic layout. **[2006]**
- With the extensible-dataset feature, either object may be **"zapified"**: its object type is rewritten to the bit-encoded metadata-ZAP type (`05` section 5.2) and its object *data* becomes a ZAP of named extension fields (sections 2.3, 3.3), while the bonus record keeps its layout and meaning. A reader detects this by the object type; the bonus is interpreted identically either way. **[modern]**

All multi-byte fields below are in the writer's native byte order (`01` section 4). Every "object number" field is a 64-bit MOS object number; 0 means "none" unless stated otherwise.

---

## 2. The DSL directory

One DSL directory exists per filesystem/volume **lineage**: it owns the head dataset and all its snapshots, the child namespace, the property store, and rolled-up space accounting. **[2006]**

### 2.1 The 256-byte directory record

Byte offsets within the bonus buffer:

| Offset | Size | Field | Meaning
| :-- | :-- | :-- | :--
| 0   | 8 | creation time | seconds since Unix epoch; recorded at creation, not consulted afterward. **[2006]**
| 8   | 8 | head dataset obj | DSL dataset object of the live (head) filesystem or volume. 0 only for special directories with no head (section 2.2). **[2006]**
| 16  | 8 | parent dir obj | parent DSL directory; 0 for the root directory. **[2006]**
| 24  | 8 | origin obj | for a clone, the DSL **dataset** object of the origin snapshot (section 4.3). (Named "clone parent" in the original spec.) **[2006]**
| 32  | 8 | child map obj | ZAP (type 13) mapping child directory name -> child DSL directory object (section 2.2). **[2006]**
| 40  | 8 | used bytes | space charged to this directory tree (head + snapshots + children). **[2006]**
| 48  | 8 | compressed bytes | compressed size of that space. **[2006]**
| 56  | 8 | uncompressed bytes | uncompressed size of that space. **[2006]**
| 64  | 8 | quota | byte limit on `used bytes`; 0 = no quota. **[2006]**
| 72  | 8 | reservation | bytes guaranteed to this tree from the parent's space. **[2006]**
| 80  | 8 | props ZAP obj | ZAP (type 15) of this directory's properties (section 5). **[2006]**
| 88  | 8 | delegation ZAP obj | ZAP (type 32) of delegated-administration permissions. Format deferred to its own topic. **[modern]**
| 96  | 8 | flags | bit 0: the used-breakdown fields at offset 104 are valid. Other bits reserved. **[modern]**
| 104 | 40 | used breakdown | five 64-bit words subdividing `used bytes`, in order: head dataset, snapshots, child directories, child reservations, refreservation. Valid only when flag bit 0 is set. **[modern]**
| 144 | 8 | clones ZAP obj | ZAP (type 52) on an origin's directory: the set of **head dataset** objects of clones whose origin snapshot lies in this directory (section 4.3). **[modern]**
| 152 | 104 | pad | thirteen reserved 64-bit words; zero. **[2006]** (three of the original pad words were consumed by the fields at 88..144)

The record is exactly **256 bytes**. The original record ended at offset 88 (fields 0..80 plus pad); the delegation, flags, breakdown, and clones fields were carved from the pad. **[2006]** for the 256-byte size.

### 2.2 The namespace tree and special directories

- The **child map** ZAP (offset 32) maps each child's single name component (the text between `/` separators) to the child's DSL directory object. Resolving a dataset name `pool/a/b` means: root directory -> child map lookup `a` -> that directory's child map lookup `b`. **[2006]**
- The **root directory's** child map also carries reserved entries whose names begin with `$` - not reachable by user naming: **[modern]**

| Name | Purpose
| :-- | :--
| `$MOS` | accounting directory for space used by the MOS itself
| `$FREE` | accounting directory for blocks pending async free
| `$ORIGIN` | carrier of the universal origin snapshot (section 4.3)
| `$LEAK` | accounting directory for leaked space, present only if leak recovery ran

### 2.3 Directory extension fields (zapified) [modern]

When the directory object is zapified (section 1.1), its object-data ZAP may carry (string keys; uint64 values unless the topic's own spec says otherwise):

| Key | Meaning
| :-- | :--
| `com.joyent:filesystem_count` | count of filesystems under this directory (limit enforcement)
| `com.joyent:snapshot_count` | count of snapshots under this directory
| `com.datto:crypto_key_obj` | object holding this tree's wrapped encryption key (format in the encryption spec)
| `com.delphix:livelist` | block-pointer object tracking a clone's diverged blocks (deletion optimization)
| `com.ixsystems:snapshots_changed` | timestamp of the last snapshot namespace change

Absence of a key means the feature is not active for this directory.

---

## 3. The DSL dataset

One DSL dataset object exists per **head** (live filesystem/volume) and per **snapshot**. The record is one object-set pointer plus the snapshot bookkeeping around it. **[2006]**

### 3.1 The 320-byte dataset record

Byte offsets within the bonus buffer:

| Offset | Size | Field | Meaning
| :-- | :-- | :-- | :--
| 0   | 8 | dir obj | owning DSL directory object. **[2006]**
| 8   | 8 | prev snapshot obj | next-older snapshot's dataset object; 0 if none (section 4.1). **[2006]**
| 16  | 8 | prev snapshot txg | creation txg of that snapshot. **[2006]**
| 24  | 8 | next snapshot obj | on a snapshot: the next-newer snapshot's dataset object, or the head's if this is the newest. 0 on a head. **[2006]**
| 32  | 8 | snapshot map obj | on a head: ZAP (type 14) mapping snapshot name -> snapshot dataset object. **0 on a snapshot** - this is the head/snapshot discriminator alongside child count. **[2006]**
| 40  | 8 | child count | 0 on a head. On a snapshot: number of on-disk references = 1 (its next-newer neighbor) + the number of clones whose origin it is (section 4.2). **[2006]**
| 48  | 8 | creation time | seconds since Unix epoch. **[2006]**
| 56  | 8 | creation txg | txg this dataset was created in. **[2006]**
| 64  | 8 | deadlist obj | this dataset's deadlist (section 6.2). **[2006]** (the modern deadlist *format* differs; see 6.2)
| 72  | 8 | referenced bytes | bytes reachable from this dataset's object set. **[2006]**
| 80  | 8 | compressed bytes | compressed size of that space. **[2006]**
| 88  | 8 | uncompressed bytes | uncompressed size of that space. **[2006]**
| 96  | 8 | unique bytes | bytes referenced by this snapshot only (freed if it is destroyed); maintained meaningfully for snapshots. **[2006]**
| 104 | 8 | fsid guid | runtime filesystem-id seed; unique per dataset within the pool. **[2006]**
| 112 | 8 | guid | globally unique dataset id (survives send/receive). **[2006]**
| 120 | 8 | flags | section 3.2. **[2006]** as a flag word (originally a single restore-in-progress indicator); the defined bit set is **[modern]**.
| 128 | 128 | block pointer | pointer (`04`) to this dataset's **object set** (`05`). The root of the dataset's contents. **[2006]**
| 256 | 8 | next-clones obj | on a snapshot with clones: ZAP (type 37) holding the **head dataset** objects of its clones (section 4.3). **[modern]**
| 264 | 8 | props obj | on a snapshot: ZAP (type 15) of snapshot-local properties (section 5.3). **[modern]**
| 272 | 8 | user-refs obj | ZAP (type 41) of user holds: hold tag name -> creation time (seconds since epoch). A held snapshot cannot be destroyed. **[modern]**
| 280 | 40 | pad | five reserved 64-bit words; zero. **[2006]** (three original pad words were consumed by the fields at 256..272)

The record is exactly **320 bytes**. **[2006]**

### 3.2 Dataset flags (offset 120)

| Bit | Mask | Meaning
| :-- | :-- | :--
| 0  | 0x01 | inconsistent: contents are mid-modification (e.g. an incomplete receive or interrupted destroy); not safe to interpret as a complete filesystem.
| 1  | 0x02 | this snapshot may not be promoted over.
| 2  | 0x04 | the `unique bytes` field is accurately maintained (set on all datasets created at or after the version introducing it).
| 3  | 0x08 | destroy was requested and deferred until holds/clones release the snapshot.
| 16 | 0x10000 | filenames within this dataset are case-insensitive (set at creation, immutable).

All defined bits are **[modern]** (bit 0 is the successor of the original restore-in-progress flag). Bits not listed are reserved; a reader must tolerate set unknown bits.

### 3.3 Dataset extension fields (zapified) [modern]

When the dataset object is zapified (section 1.1), its object-data ZAP may carry:

| Key | Meaning
| :-- | :--
| `com.delphix:bookmarks` | ZAP of bookmark name -> bookmark record (format deferred to the send/bookmark topic)
| `org.zfsonlinux:large_dnode` | dataset may contain multi-slot dnodes (`05` section 1.1)
| `com.delphix:remap_deadlist` | deadlist of blocks remapped by device removal
| `com.datto:ivset_guid` | encryption IV-set guid for this snapshot
| `com.delphix:resume_fromguid`, `com.delphix:resume_toguid`, `com.delphix:resume_toname`, `com.delphix:resume_object`, `com.delphix:resume_offset`, `com.delphix:resume_bytes`, `com.delphix:resume_largeblockok`, `com.delphix:resume_embedok`, `com.delphix:resume_compressok`, `com.datto:resume_rawok`, `com.delphix:resume_redact_book_snaps` | state of a partially received send stream (present only mid-receive)

Absence of a key means the feature/state is not present.

---

## 4. Snapshots, chains, and clones

### 4.1 The snapshot chain

Snapshots of one head form a doubly linked list through the dataset records, ordered by creation txg: **[2006]**

- The **head's** `prev snapshot obj` points at the **newest** snapshot (0 if none). The head's `next snapshot obj` is 0.
- Each **snapshot's** `prev snapshot obj` points at the next-older snapshot (0 for the oldest, or the origin snapshot for a clone's oldest; universal origin under section 4.3). Its `next snapshot obj` points at the next-newer snapshot; the **newest snapshot's** points at the **head**.
- `prev snapshot txg` mirrors the pointed-to snapshot's creation txg, letting a reader order and bound birth-txg ranges without following the pointer.

On snapshot creation the new snapshot inherits the head's previous prev-pointer, its `next snapshot obj` is set to the head, the formerly newest snapshot's next-pointer is rewritten to the new snapshot, and the head's prev-pointer/txg advance. The snapshot record's block pointer is a copy of the head's at that txg - the frozen root of the snapshot's contents.

### 4.2 Head vs snapshot discrimination

A dataset is a **snapshot** iff its `child count` is nonzero (equivalently: its `snapshot map obj` is 0 and it is the target of a next/prev pointer). A head has `child count` 0 and a snapshot-map ZAP. **[2006]** The snapshot's name is recorded only as its key in the head's snapshot-map ZAP - the record itself carries no name. **[2006]** Current OpenZFS creates the snapshot map with the case-fold (uppercase) normalization flag set (`06`, normalization flags), so case-insensitive snapshot lookup is possible where the dataset requires it. **[modern]**

Full-name resolution `pool/a/b@snap`: resolve `pool/a/b` per section 2.2 to a directory, follow `head dataset obj`, then look `snap` up in the head's snapshot-map ZAP. **[2006]**

### 4.3 Clones and origins

A **clone** is a head dataset whose directory's `origin obj` (section 2.1 offset 24) names the snapshot it branched from. **[2006]** Cross-links maintained with it:

- The origin snapshot's `child count` includes each clone. **[2006]**
- The origin snapshot's **next-clones ZAP** (type 37, dataset record offset 256) holds the clone **head dataset** object numbers. **[modern]**
- The origin's **directory's clones ZAP** (type 52, directory record offset 144) also holds the clone head dataset object numbers (per-directory rollup for iteration). **[modern]**
- Both clone sets use the integer-keyed ZAP convention of section 6.3, keyed by the stored object number itself.

**The universal origin.** Pools at or after the origin version carry a hidden `$ORIGIN` directory (section 2.2) whose head and single snapshot are both named `$ORIGIN`. That snapshot - empty, txg-bounded at pool creation - is the **universal origin**: every subsequently created non-clone filesystem records it as its origin (directory `origin obj` and initial `prev snapshot obj` both point at it), making dataset creation uniform with cloning. A reader therefore tests "is a real clone" as: `origin obj` nonzero **and** not the universal origin snapshot. Promotion (swapping a clone's and origin's roles) rewrites these links but preserves the invariants above. **[modern]**

---

## 5. Properties

### 5.1 The props ZAP

Each directory's `props ZAP obj` (type 15) stores locally set properties. Keys are property names; values: **[2006]**

- **Numeric property**: a single uint64 (ZAP integer size 8, count 1).
- **String property**: a byte string including its terminating NUL (ZAP integer size 1, count = length + 1).

Only *locally set* properties appear; an absent key means "inherit". Effective-value resolution walks `parent dir obj` upward taking the first directory that has the key, falling back to the property's default. **[2006]**

### 5.2 Key-name variants [modern]

Three suffix conventions extend the plain key:

| Key form | Meaning
| :-- | :--
| `<name>` | local value, set on this dataset
| `<name>$recvd` | value delivered by `zfs receive`; a plain local value overrides it
| `<name>$inherit` | explicit "revert to inherited" marker recorded over a received value
| `<name>$iuv` | a value from a sender using an index enumerator this pool's software predates

Effective-value resolution per key at each level: local beats received; the `$inherit` marker suppresses a received value.

### 5.3 Snapshot properties [modern]

Snapshot-local properties (e.g. per-snapshot user properties) live in the snapshot dataset record's `props obj` (offset 264), same encoding as 5.1. 0 = none.

---

## 6. Space maps of the dead: bpobj and deadlist

### 6.1 The block-pointer object (bpobj)

A **bpobj** (object type 5, bonus type 6) is an on-disk list of block pointers: its object data is a packed array of 128-byte block pointers (`04`); its bonus is a header of up to seven 64-bit words. The bonus **length** gates which fields exist - a reader must check it before touching later fields:

| Offset | Field | Present when bonus | Provenance
| :-- | :-- | :-- | :--
| 0  | count of block pointers in the object data | always (>= 16) | [2006]
| 8  | total allocated bytes those pointers reference | always (>= 16) | [2006]
| 16 | total compressed bytes | >= 32 | [modern]
| 24 | total uncompressed bytes | >= 32 | [modern]
| 32 | sub-objects obj: object (type 53) whose data is a packed uint64 array of child bpobj object numbers | >= 48 | [modern]
| 40 | count of entries in that sub-object array | >= 48 | [modern]
| 48 | count of freed pointers (livelist use) | >= 56 | [modern]

A bpobj's full content is its own pointer array plus, recursively, every sub-object bpobj. The 16-byte form is the original ("block pointer list") object that backed the 2006-era deadlist and the pool-wide `sync_bplist`; the pool-wide free list is now the `free_bpobj` of section 1. **[2006]** for the 16-byte form and pointer-array data; recursion and later fields **[modern]**.

### 6.2 The deadlist

A dataset's **deadlist** (record offset 64) lists the blocks that were referenced by its previous snapshot but are no longer referenced by the dataset itself - the blocks that "died" in it. Destroying a snapshot merges its deadlist into its next-newer neighbor's; the space a snapshot would free is computed from these. **[2006]** for the role; the current format is **[modern]**:

- **Modern format**: object type 50 (a ZAP) with bonus type 51, bonus length **320** bytes. Bonus header: total used bytes at offset 0, compressed at 8, uncompressed at 16, then thirty-seven reserved words (pad to 320). The ZAP maps a birth-txg bucket key (encoded per section 6.3) -> the object number of a **bpobj** (6.1) holding that bucket's dead block pointers. A dead pointer lands in the entry with the **largest key strictly less than its logical birth txg** - a pointer born exactly at a key txg belongs to the entry *below* that key. Keys are snapshot-creation txgs, so entry K holds the pointers with births in `(K, next key]`: the blocks born after snapshot K (blocks born in K's own txg are part of snapshot K and bucket below it). **[modern]**
- **Old format**: the deadlist object is itself a plain bpobj (type 5) - no ZAP, no bucketing. A reader discriminates the two formats **by the object's type**. **[2006]** (as the original bplist deadlist)

### 6.3 Integer-keyed ZAP encoding

Where a ZAP's key is semantically a uint64 (deadlist buckets, clone sets, and similar MOS sets), the key is stored as the **lowercase hexadecimal ASCII string** of the value - `printf "%llx"`: no `0x` prefix, no leading zeros, digits `0-9a-f` (e.g. txg 300 -> key `"12c"`). The value is a single uint64 (integer size 8, count 1). For membership *sets* (the clone sets of 4.3) the stored value equals the encoded key. Hashing/storage of the key follows the ordinary string-key rules of `06` (the NUL-exclusion rule there applies - the key hashes as a string). **[modern]**

---

## 7. Traversal summary

From the MOS to a filesystem's contents, all name maps being ZAPs (`06`):

1. Read the MOS object set from the active uberblock (`05` section 6). Read the **object directory** at MOS object 1.
2. Look up `root_dataset` -> root DSL directory object; read its 256-byte bonus record (section 2.1).
3. For each name component, follow the **child map** ZAP to the next directory (section 2.2).
4. Follow `head dataset obj` -> DSL dataset object; read its 320-byte bonus record (section 3.1). For `@snapshot`, indirect once more through the head's **snapshot map** ZAP.
5. The record's embedded **block pointer** (offset 128) points at the dataset's object set - decode per `04`/`05`. Its type field (`05` section 4.6) distinguishes filesystem from volume.

---

## 8. Constants quick reference

| Constant | Value | Provenance
| :-- | :-- | :--
| object-directory object number (MOS) | 1 | [2006]
| DSL directory record size (bonus) | 256 bytes | [2006]
| DSL dataset record size (bonus) | 320 bytes | [2006]
| dataset object-set block pointer offset | 128 | [2006]
| directory used-breakdown words | 5 | [modern]
| deadlist header bonus size | 320 bytes | [modern]
| bpobj bonus sizes | 16 / 32 / 48 (/ 56) bytes | 16 [2006]; rest [modern]
| snapshot discriminator | child count != 0 | [2006]
| head snapshot-map / snapshot marker | snapshot map obj = 0 on snapshots | [2006]
| special directory names | `$MOS`, `$ORIGIN`, `$FREE`, `$LEAK` | [modern]
| universal origin dataset | `$ORIGIN` dir, head + snapshot both named `$ORIGIN` | [modern]
| integer-ZAP key encoding | lowercase hex string, no prefix/padding | [modern]

Object types used (numbers per `05` section 5.1): DSL directory 12, child map 13, snapshot map 14, props 15, DSL dataset 16, delegation 32, next-clones 37, user-refs 41, deadlist 50 (header 51), clones 52, bpobj 5 (header 6), bpobj sub-objects 53.

---

## 9. Cross-references

- `01-vdev-label-uberblock.md` - the active uberblock roots the MOS; native-byte-order rule.
- `04-block-pointer-dva.md` - the 128-byte block pointer embedded in the dataset record, bpobj arrays, and deadlist buckets.
- `05-dmu-dnode-objset.md` - bonus buffers carrying both DSL records; the MOS; the object-type table; object-set types distinguishing filesystem/volume.
- `06-zap.md` - every name map here (object directory, child map, snapshot map, props, clone sets, deadlist buckets); string-key hashing rules for the integer-key encoding of section 6.3.
- ZPL layout, delegation ZAP contents, encryption key/bookmark record formats, livelist details - later documents.
