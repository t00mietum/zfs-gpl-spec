# ZFS on-disk format: ZPL znodes, system attributes, and directory entries

**Provenance**

- OpenZFS commit studied for behavioral corroboration: `9b7642df` (reference only).
- Status: **APPROVED** 2026-07-22 (gatekeeper review complete). Authoritative.
- Each fact below is tagged with a provenance marker:
	- **[2006]** - corroborated by the public 2006 Sun "ZFS On-Disk Specification" (the durable core, independently published).
	- **[modern]** - source-only; a post-2006 addition or refinement observed in current OpenZFS. Required to interoperate with pools written by recent OpenZFS, but not part of the original published spec.

This document specifies the ZFS POSIX Layer (ZPL) on-disk structures inside a filesystem's object set: the master node, the **znode** (per-file metadata) in both its legacy fixed-record and modern **system-attribute (SA)** forms, the SA registration/layout machinery, and the **directory entry** encoding. It answers request #8. It builds on `05-dmu-dnode-objset.md` (dnodes, bonus/spill buffers, the object-type table in its section 5.1 - type numbers cited here are from that table), `06-zap.md` (the master node, directories, and SA registry/layout objects are all ZAPs), `07-dsl-dataset-snapshot.md` (reaching a filesystem's object set), and `03-config-nvlist-xdr-encoding.md` (the XDR nvlist encoding reused by SA-stored extended attributes).

Everything here lives inside one filesystem's **object set** (`05` section 4), reached from its DSL dataset's embedded block pointer (`07` section 3.1). All multi-byte fields are in the writer's native byte order (`01` section 4) unless stated otherwise.

---

## 1. The master node and ZPL versions

### 1.1 The master node

Object number **1** of a filesystem object set is the **master node**, a ZAP of object type 21. It is the entry point for everything else in the filesystem. **[2006]**

Master-node keys (string keys, uint64 values):

| Key | Value | Provenance
| :-- | :-- | :--
| `VERSION` | ZPL version of this filesystem (section 1.2) | [2006]
| `ROOT` | object number of the root directory's znode | [2006]
| `DELETE_QUEUE` | object number of the unlinked set (section 6.4) | [2006]
| `SA_ATTRS` | object number of the SA master node (section 5.1); present when version >= 5 | [modern]
| `FUID` | object number of the FUID domain table (section 3.4); created on first non-POSIX id | [modern]
| `SHARES` | object number of a hidden directory holding share-metadata files | [modern]
| `normalization` | unicode normalization mode for filenames (0 = none); set at creation, immutable | [modern]
| `utf8only` | 1 = only valid UTF-8 filenames accepted; set at creation, immutable | [modern]
| `casesensitivity` | 0 = sensitive, 1 = insensitive, 2 = mixed; set at creation, immutable | [modern]
| `userquota@`, `groupquota@`, `userobjquota@`, `groupobjquota@`, `projectquota@`, `projectobjquota@` | each names the object of a quota ZAP (type 40) mapping id -> limit in bytes (or objects); created on first quota set | [modern]
| `FSID` | reserved name defined by the interface; not written by current OpenZFS | [modern]

Inside a quota ZAP, the key is the lowercase-hex string of the 64-bit id - for a plain POSIX id just the id (the integer-key convention of `07` section 6.3), for a domain-qualified id the full FUID encoding of section 3.4 (domain index in the high 32 bits). **[modern]**

### 1.2 ZPL versions

The `VERSION` value gates which structures below may appear: **[2006]** for the version key; the version list runs 1..5 and versions 2..5 are **[modern]**.

| Version | Adds
| :-- | :--
| 1 | base format: legacy znode record, directories, master node
| 2 | file-type nibble in directory-entry values (section 6.1)
| 3 | FUID ids (section 3.4), filename normalization, system-attribute groundwork
| 4 | user/group space accounting (quota ZAPs of 1.1; used-space tracking per `05` section 4)
| 5 | SA-based znodes (section 5): per-object attribute layouts, spill blocks

Current OpenZFS creates filesystems at version 5. A version-N reader must reject (or read-only tolerate) a filesystem whose `VERSION` exceeds what it implements. **[modern]**

---

## 2. File-type objects

Every file, directory, symlink, device node, socket, and FIFO is one DMU object - its **znode**. The object's dnode carries: **[2006]**

- **Object type**: 20 (directory contents; the object data is the directory ZAP of section 6) for directories, 19 (plain file contents; the object data is the file's bytes) for everything else.
- **Bonus type** selecting the znode form:
	- **17** (ZPL znode): the legacy fixed 264-byte record of section 4, bonus length 264. Written by ZPL versions 1-4. **[2006]**
	- **44** (system attributes): the SA form of section 5, bonus length variable up to the dnode's bonus capacity. Written at ZPL version 5. **[modern]**
- Directories are created with the ZAP normalization flags (`06`) matching the filesystem's `normalization`/`casesensitivity` settings, so name matching in a case-insensitive or normalizing filesystem happens inside the ZAP. **[modern]**

A filesystem upgraded in place from an older ZPL version may contain both bonus types simultaneously; a reader must dispatch on the bonus type per object. Modifying a legacy object under version 5 rewrites it to the SA form. **[modern]**

---

## 3. The znode attribute set

Both znode forms carry the same logical attributes. Encodings common to both:

### 3.1 Scalar attributes

| Attribute | Size | Encoding | Provenance
| :-- | :-- | :-- | :--
| access time | 16 | two uint64s: seconds since Unix epoch, then nanoseconds | [2006]
| modification time | 16 | same | [2006]
| change time | 16 | same | [2006]
| creation time | 16 | same | [2006]
| generation | 8 | txg in which the object was created | [2006]
| mode | 8 | POSIX file mode: low 12 bits permission/setuid/setgid/sticky, bits 12..15 the file-type code (values in section 6.1) | [2006]
| size | 8 | file: byte length. Symlink: link-target length. Directory: number of entries plus 2 (the two counting for the conventional self/parent entries, which are **not** stored - section 6.2) | [2006]
| parent | 8 | object number of the parent directory's znode. The root directory names itself. For a hard-linked file this records one (not necessarily current) parent; it is a hint, authoritative only for directories | [2006]
| links | 8 | link count (directories: 2 + subdirectory count) | [2006]
| xattr | 8 | object number of the extended-attribute directory (section 6.5); 0 = none | [2006]
| rdev | 8 | device id for block/character nodes (section 3.5) | [2006]
| flags | 8 | persistent flag word (section 3.2) | [2006] as a field; the full bit set is [modern]
| uid | 8 | owner id (section 3.4) | [2006]
| gid | 8 | owning-group id (section 3.4) | [2006]
| project id | 8 | project-quota id; SA form only, present when flag bit 47 is set | [modern]

### 3.2 Persistent flags

The flags word, low half: **[modern]** (the word position is **[2006]**; the original format defined no bits here)

| Bit | Mask | Meaning
| :-- | :-- | :--
| 0 | 0x1 | object is part of an extended-attribute tree (an xattr directory or xattr value file)
| 1 | 0x2 | ACL contains inheritable entries
| 2 | 0x4 | ACL is trivial (equivalent to the mode bits)
| 3 | 0x8 | ACL contains a complex object entry (section 7.3)
| 4 | 0x10 | ACL is protected (inheritance blocked)
| 5 | 0x20 | ACL was defaulted (no inherited or explicit source)
| 6 | 0x40 | ACL should auto-inherit to children
| 7 | 0x80 | legacy form only: an anti-virus scanstamp sits in the bonus tail (section 4.2)
| 8 | 0x100 | no ACE denies execute to anyone (execute-check shortcut)

High half (file-level attributes):

| Bit | Mask (<<32) | Meaning | Bit | Mask (<<32) | Meaning
| :-- | :-- | :-- | :-- | :-- | :--
| 32 | 0x1 | read-only | 40 | 0x100 | opaque
| 33 | 0x2 | hidden | 41 | 0x200 | AV quarantined
| 34 | 0x4 | system | 42 | 0x400 | AV modified
| 35 | 0x8 | archive | 43 | 0x800 | reparse point
| 36 | 0x10 | immutable | 44 | 0x1000 | offline
| 37 | 0x20 | no-unlink | 45 | 0x2000 | sparse
| 38 | 0x40 | append-only | 46 | 0x4000 | children inherit project id
| 39 | 0x80 | no-dump | 47 | 0x8000 | project-id attribute present

### 3.3 Timestamps

All four timestamps are the two-word seconds/nanoseconds pair of section 3.1; the nanoseconds word is a full 64-bit field holding 0..999999999. **[2006]**

### 3.4 Owner ids and FUIDs

At ZPL version >= 3 the 64-bit uid/gid fields are **FUIDs**: the low 32 bits are the rid (the POSIX id, or the final component of a Windows SID), the high 32 bits are an index into the filesystem's FUID **domain table** (1-based). Index 0 means "no domain": the value is a plain POSIX id, which is also the only form at versions 1-2. **[modern]** (plain 64-bit ids are **[2006]**)

The domain table (master-node key `FUID`) is an object of type 35 whose data is a packed nvlist mapping domain strings to indices, with its byte size in an associated size record (type 36 semantics per `05` section 5.1). Its internal nvlist schema is deferred to a later topic; a reader handling only POSIX ids never needs it (index is 0). **[modern]**

### 3.5 Device ids

The device field packs major and minor numbers: major in bits 32..63, minor in bits 0..31. **[2006]** Caveat: the Linux port currently stores the Linux kernel's own 32-bit packed dev encoding in this field instead; a cross-platform reader should treat nonzero high-word as the portable major/minor form and otherwise interpret per the writing platform. **[modern]**

---

## 4. The legacy znode record (bonus type 17)

### 4.1 The 264-byte record

Byte offsets within the bonus buffer (bonus length 264 = 0x108; all attribute encodings per section 3):

| Offset | Size | Field | Provenance
| :-- | :-- | :-- | :--
| 0   | 16 | access time | [2006]
| 16  | 16 | modification time | [2006]
| 32  | 16 | change time | [2006]
| 48  | 16 | creation time | [2006]
| 64  | 8  | generation | [2006]
| 72  | 8  | mode | [2006]
| 80  | 8  | size | [2006]
| 88  | 8  | parent | [2006]
| 96  | 8  | links | [2006]
| 104 | 8  | xattr directory obj | [2006]
| 112 | 8  | rdev | [2006]
| 120 | 8  | flags | [2006]
| 128 | 8  | uid | [2006]
| 136 | 8  | gid | [2006]
| 144 | 32 | pad; written zero (the first word was once defined as an extra-attributes object reference; current writers never populate it) | [2006]
| 176 | 88 | embedded ACL record (section 7.1) | [2006]

The record is exactly **264 bytes**. **[2006]**

### 4.2 The bonus tail

A legacy znode's dnode may declare a bonus length **greater than 264**; the bytes after the record are data owned by the file type: **[2006]**

- **Symlink**: if the link target fits (264 + target length <= max bonus length), the target bytes (no NUL) follow the record and `size` gives their length. Otherwise the target is the first `size` bytes of the object's data block. **[2006]**
- **Regular file with scanstamp**: when flag bit 7 is set, a 32-byte anti-virus scanstamp follows the record. **[modern]**

### 4.3 Layout invariants

Legacy znode objects never use a spill pointer, and under the SA read machinery they are always interpreted as the fixed layout above (layout number 0, section 5.3) - there is no header of any kind in the bonus buffer; data starts at offset 0. **[2006]**

---

## 5. System attributes (bonus type 44)

At ZPL version 5, znode metadata is stored as **system attributes**: self-describing sets of typed attributes packed into the bonus buffer and, on overflow, a **spill block**. The machinery is generic; the ZPL attribute set is section 5.4. **[modern]** (all of section 5)

### 5.1 The SA master node, registry, and layouts

The master-node key `SA_ATTRS` names the **SA master node**, a ZAP of type 45, holding (uint64 values):

| Key | Value
| :-- | :--
| `REGISTRY` | object number of the attribute registry, a ZAP of type 46
| `LAYOUTS` | object number of the layout table, a ZAP of type 47

Both are created lazily on first use; a freshly created empty filesystem may not yet have them.

**The registry** maps each attribute's name string to a uint64 describing it:

| Bits | Field
| :-- | :--
| 0..15 | attribute number (the id used everywhere else)
| 16..23 | byteswap class (section 5.7)
| 24..39 | fixed length in bytes; **0 = variable length**
| 40..63 | unused (zero)

**The layout table** maps a layout number - stored as its **decimal** ASCII string (note: decimal here, unlike the hex convention of `07` section 6.3) - to that layout's attribute-number list: a ZAP value of integer size **2**, count = attribute count, i.e. an array of uint16 attribute numbers in storage order.

### 5.2 Attribute numbering

- For filesystem object sets, attribute numbers **0..15** are permanently pre-assigned to the sixteen legacy record fields, in the exact order of the section 4.1 table (access time = 0, modification time = 1, change time = 2, creation time = 3, generation = 4, mode = 5, size = 6, parent = 7, links = 8, xattr = 9, rdev = 10, flags = 11, uid = 12, gid = 13, pad = 14, embedded ACL record = 15). These assignments hold even before (or without) a registry - a reader must know them implicitly, and for these sixteen names they take precedence over any registry content.
- The registration sync nonetheless records **all** not-yet-registered attributes, the fixed sixteen included, so on a real version-5 filesystem the registry normally lists every name in section 5.4 with the values above.
- All further attributes get the next free number at first registration, recorded in the registry. Numbers are **per-dataset**: two filesystems may assign different numbers to the same attribute name. A reader must resolve names through the registry, never assume numbers beyond the fixed sixteen.

### 5.3 Reserved layout numbers

- **Layout 0**: the implicit legacy-znode layout - the sixteen fixed attributes in numbering order (exactly section 4.1). Used only for bonus type 17; never stored in the layout table.
- **Layout 1**: the implicit empty layout (no attributes). Also never stored. In an SA buffer whose header (section 5.5) carries layout value 0, a reader treats it as layout 1.
- Stored layouts number from **2** upward, allocated sequentially (each new layout's number is the count of layouts known so far, including the two implicit ones).

### 5.4 The ZPL attribute names

The name strings below are on-disk registry keys (interface facts). Sizes and byteswap classes as registered by current OpenZFS:

| Name | Size | Class | Content
| :-- | :-- | :-- | :--
| `ZPL_ATIME` | 16 | 0 | access time
| `ZPL_MTIME` | 16 | 0 | modification time
| `ZPL_CTIME` | 16 | 0 | change time
| `ZPL_CRTIME` | 16 | 0 | creation time
| `ZPL_GEN` | 8 | 0 | generation
| `ZPL_MODE` | 8 | 0 | mode
| `ZPL_SIZE` | 8 | 0 | size
| `ZPL_PARENT` | 8 | 0 | parent
| `ZPL_LINKS` | 8 | 0 | links
| `ZPL_XATTR` | 8 | 0 | xattr directory obj
| `ZPL_RDEV` | 8 | 0 | rdev
| `ZPL_FLAGS` | 8 | 0 | flags
| `ZPL_UID` | 8 | 0 | uid
| `ZPL_GID` | 8 | 0 | gid
| `ZPL_PAD` | 32 | 0 | reserved
| `ZPL_ZNODE_ACL` | 88 | 3 | embedded ACL record (legacy container, section 7.1); present only on objects upgraded from the legacy form
| `ZPL_DACL_COUNT` | 8 | 0 | number of ACL entries
| `ZPL_SYMLINK` | var | 3 | symlink target bytes (no NUL)
| `ZPL_SCANSTAMP` | 32 | 3 | anti-virus scanstamp
| `ZPL_DACL_ACES` | var | 4 | the ACL entry stream (section 7.2)
| `ZPL_DXATTR` | var | 3 | SA-stored extended attributes (section 6.6)
| `ZPL_PROJID` | 8 | 0 | project id
| `ZPL_SEQ` | 8 | 0 | modification sequence number (recent addition; absent on most pools)

The first sixteen rows are the fixed numbers of section 5.2; the rest are registered on first use. Absence of an attribute from an object's layout means the attribute is not set for that object (e.g. no `ZPL_RDEV` on a regular file, no `ZPL_PROJID` when project quotas are off).

### 5.5 The SA header

Every SA buffer (bonus and spill) begins with a header:

| Offset | Size | Field
| :-- | :-- | :--
| 0 | 4 | magic **0x2F505A**
| 4 | 2 | layout info: bits 0..9 = layout number (0..1023); bits 10..15 = header size in 8-byte units (size in bytes = value << 3)
| 6 | 2... | variable-length array: one uint16 per **variable-size** attribute in the layout, in layout order, giving that attribute's byte length in this buffer

The header size is `8 + 2*(V-1)` rounded up to a multiple of 8, where V is the number of variable-size attributes in the buffer's layout (the base 8-byte header holds the first length slot; V = 0 or 1 gives 8). Fixed-size attribute lengths come from the registry and are never stored per-object.

### 5.6 Attribute data placement

- Attribute data begins at the header-size offset and follows the buffer's **layout order**.
- Each attribute starts at the next 8-byte-aligned offset: after an attribute of length L, the cursor advances by L rounded up to a multiple of 8. (Equivalently: every attribute's start offset is 8-aligned; sub-8 tails are padding.)
- An attribute never spans buffers: it is entirely in the bonus or entirely in the spill block.
- Layout numbers describe **one buffer**: a spilled object has two independent layouts, one for the attributes remaining in the bonus and one for those in the spill block, each buffer carrying its own full header.

### 5.7 Byteswap classes

The registry's byteswap class tells a foreign-endian reader how to swap the attribute's bytes: 0 = array of uint64, 1 = array of uint32, 2 = array of uint16, 3 = array of uint8 (no swap), 4 = ACL entry stream (structured swap per section 7.2 field widths). As everywhere, data is written in native order and swapped only by an opposite-endian reader; SA buffers are additionally swapped lazily in place on first access (the header magic's apparent byte order tells the reader whether the buffer needs swapping).

### 5.8 Spill blocks

When an object's attribute set outgrows its bonus capacity:

- The dnode gains a **spill pointer** (its final 128 bytes become a block pointer and dnode flag bit 2 is set - `05` sections 1.3/1.4), reducing the usable bonus accordingly.
- The writer splits the attribute list at the first attribute that no longer fits (accounting for the shrunken bonus and header growth); that attribute and all after it move to the spill block. In practice the large ACL stream is the usual spill tenant.
- The spill block is a single block sized to fit its contents, rounded up to a multiple of 512 bytes, between 512 bytes and 128 KiB.
- Both buffers are laid out per sections 5.5-5.6 with their own layouts. Lookup order for a reader: check the bonus layout for the attribute; if absent, check the spill layout.

### 5.9 Layout discipline for readers and writers

Layouts are an optimization vocabulary, not a schema: any attribute set in any order is legal if described by a stored layout. Current OpenZFS writes new files with a small set of conventional layouts (scalar core first, timestamps, then link count, then optional rdev/project id, then ACL count and stream), but none of that is guaranteed. A reader must: read the header, resolve the layout number through the layout table (or the implicit 0/1), resolve lengths through the registry plus the header's length array, and walk per section 5.6. A writer adding an attribute set with no matching stored layout must add the layout to the table under the next free number.

---

## 6. Directories and name storage

### 6.1 Directory entries

A directory's object data is a ZAP (`06`); its entries are the directory's filenames. Each entry: key = the filename (a single path component); value = one uint64 (integer size 8, count 1): **[2006]**

| Bits | Field | Provenance
| :-- | :-- | :--
| 0..47 | object number of the entry's znode | [2006]
| 48..59 | unused; zero | [2006]
| 60..63 | file-type code, = mode bits 12..15 (below); written at ZPL version >= 2, zero at version 1 | [modern]

File-type codes (the POSIX mode-format nibble, matching the BSD directory-entry type values):

| Code | Type | Code | Type
| :-- | :-- | :-- | :--
| 1 | FIFO | 8 | regular file
| 2 | character device | 10 | symlink
| 4 | directory | 12 | socket
| 6 | block device | 14 | event port / whiteout (platform-specific)

A reader needing the type on a version-1 filesystem (or defensively) fetches the target znode's mode; the nibble is a lookup-avoidance hint that always matches mode bits 12..15 when present. **[modern]**

### 6.2 No self/parent entries

`.` and `..` are **not stored**. The self object number is known from arrival; the parent comes from the znode's parent attribute. The directory `size` attribute counts stored entries plus 2 by convention (section 3.1). **[2006]**

### 6.3 Name matching

Directory ZAPs are created with the normalization flags (`06`) derived from the filesystem's `normalization`/`utf8only`/`casesensitivity` settings, so equality under normalization and/or case folding is evaluated by ZAP lookup itself. In a case-insensitive or normalizing filesystem, entry keys store the creator's original spelling; matching is fold-equivalent. **[modern]**

### 6.4 The unlinked set

The master-node key `DELETE_QUEUE` names the **unlinked set**, a ZAP of type 22 holding znodes whose last name was removed while the file was still held open (or whose teardown was interrupted). Integer-keyed membership set per the `07` section 6.3 convention: key = the object number's lowercase-hex string, value = the object number. Mount-time recovery deletes every listed object. **[2006]** for the object and role; the hex-string key convention as in `07`.

### 6.5 Extended-attribute directories

An object's extended attributes (the named-xattr model) form a hidden subtree: the znode's xattr attribute names a **directory object** (type 20, normal directory encoding per 6.1) whose entries name **regular-file objects** holding each xattr's value as file contents. All objects in the subtree carry flag bit 0 (section 3.2); the xattr files' parent attribute names the xattr directory, whose own parent names the owning file. The xattr directory is reachable only through the xattr attribute, never by a directory entry. **[2006]**

### 6.6 SA-stored extended attributes

With the SA form, small xattrs may instead live directly in the znode's `ZPL_DXATTR` attribute: an **XDR-encoded nvlist** (the encoding of `03`, including its 4-byte bootstrap header) mapping xattr name -> byte-array value. A filesystem may use both mechanisms on one object (dxattr is consulted first by current software); the on-disk source of truth is: dxattr holds what is in the nvlist, the xattr directory holds the rest. **[modern]**

### 6.7 Symlink storage

Three historical homes for the link target, in reading precedence:

1. SA form: the `ZPL_SYMLINK` attribute (bonus or spill). **[modern]**
2. Legacy form, small targets: the bonus tail after the 264-byte record (section 4.2). **[2006]**
3. Either form, large targets: the first `size` bytes of the object's data block. **[2006]**

The size attribute always gives the target length; no NUL terminator is stored in any form.

---

## 7. ACL storage containers

This section specifies the storage **containers and entry framing** only; the semantic catalog of access-mask bits, inheritance flags, and evaluation rules is deferred to a later document.

### 7.1 The embedded ACL record (legacy znode, and `ZPL_ZNODE_ACL`)

The 88-byte record at legacy-record offset 176 (also attribute number 15 in the SA world, for upgraded objects):

Version-1 framing (ZPL version >= 3): **[modern]**

| Offset | Size | Field
| :-- | :-- | :--
| 0  | 8 | external ACL object; 0 = ACL fully embedded
| 8  | 4 | total ACL byte length
| 12 | 2 | ACL version: 0 = fixed-size entries (below), 1 = variable-size entries (section 7.2)
| 14 | 2 | entry count
| 16 | 72 | embedded entry bytes

Version-0 framing (the original layout, same 88 bytes): external object at 0, entry **count** as 4 bytes at 8, version at 12, 2 bytes pad at 14, then six 12-byte fixed entries at 16. **[2006]**

- If the entries fit in the 72 embedded bytes (version 1: byte length <= 72; version 0: count <= 6), they live there and the external object is 0.
- Otherwise **all** entries live in an external object - type 33 for version-1 entries, type 18 for version-0 entries - as a packed entry stream in its object data, and the embedded bytes are unused. **[2006]** for the external-object mechanism.

### 7.2 Entry (ACE) framing

**Version-0 entry** - fixed 12 bytes: **[2006]**

| Offset | Size | Field
| :-- | :-- | :--
| 0 | 4 | who (32-bit id)
| 4 | 4 | access mask
| 8 | 2 | flags (inheritance and identity bits)
| 10 | 2 | entry type (0 allow, 1 deny, 2 audit, 3 alarm)

**Version-1 entry** - variable size, FUID-capable: **[modern]**

| Offset | Size | Field
| :-- | :-- | :--
| 0 | 2 | entry type
| 2 | 2 | flags
| 4 | 4 | access mask
| 8 | 8 | who (FUID, section 3.4) - **present only when needed** (below)
| 16 | 32 | two 16-byte object GUIDs - only in object entries

Size rules, keyed by type and by the identity bits in flags:

- Identity bits in flags: 0x1000 = owner, 0x2000 = owning group (always together with 0x40), 0x4000 = everyone, 0x40 = "who is a group". When type is allow (0) or deny (1) and the identity bits say owner, owning group, or everyone, the entry is the **8-byte header alone** - no who field.
- Any other allow/deny entry, and all audit (2) / alarm (3) entries, carry the 8-byte who: **16 bytes**.
- Object entry types 0x05 (allow-object), 0x06 (deny-object), 0x07 (audit-object), 0x08 (alarm-object) carry who plus two 16-byte GUIDs (object type and inherited object type): **48 bytes**. Their presence in an ACL is flagged by znode flag bit 3.

### 7.3 The SA-form ACL

An SA znode stores its ACL as two attributes: `ZPL_DACL_COUNT` (entry count) and `ZPL_DACL_ACES` (the version-1 entry stream, back to back with no padding between entries; byteswap class 4). No 88-byte container and no external object: an oversized stream simply spills (section 5.8). Upgraded objects may additionally retain a `ZPL_ZNODE_ACL` container per section 7.1. **[modern]**

---

## 8. Traversal summary

From a mounted dataset's object set (`07` section 7) to a file's bytes:

1. Read the **master node** (object 1); fetch `VERSION`, `ROOT`, and (v5) `SA_ATTRS` -> registry and layout ZAPs (cache both).
2. Read the root directory znode (dnode + bonus per its bonus type: section 4 or 5).
3. For each path component: look the name up in the directory's ZAP (fold rules per 6.3); mask the value to bits 0..47 for the child object number (6.1).
4. Read the target znode the same way; fetch mode/size/uid/gid/times per its form.
5. File contents are the object's data blocks (`05` section 3); symlink targets per 6.7; xattrs per 6.5/6.6.

---

## 9. Constants quick reference

| Constant | Value | Provenance
| :-- | :-- | :--
| master node object number | 1 | [2006]
| legacy znode record size (bonus) | 264 bytes | [2006]
| embedded ACL record size / offset | 88 bytes at offset 176 | [2006]
| embedded ACL entry space | 72 bytes (six 12-byte v0 entries) | [2006]
| SA header magic | 0x2F505A | [modern]
| SA layout-number field / header-size field | bits 0..9 / bits 10..15 (x8 bytes) | [modern]
| reserved SA layouts | 0 = legacy znode, 1 = empty; stored layouts start at 2 | [modern]
| fixed SA attribute numbers | 0..15, section 4.1 order | [modern]
| SA attribute alignment | 8 bytes | [modern]
| spill block size range | 512 bytes .. 128 KiB, 512-byte granularity | [modern]
| directory-entry object-number width | 48 bits | [2006]
| directory-entry type nibble | bits 60..63, = mode >> 12 (ZPL >= 2) | [modern]
| current ZPL version | 5 | [modern]

Object types used (numbers per `05` section 5.1): plain file contents 19, directory contents 20, master node 21, unlinked set 22, znode bonus 17, SA bonus 44, SA master node 45, SA registry 46, SA layouts 47, old ACL 18, ACL 33, FUID table 35, FUID table size 36, quota ZAPs 40.

---

## 10. Cross-references

- `01-vdev-label-uberblock.md` - native byte order rule.
- `03-config-nvlist-xdr-encoding.md` - the XDR nvlist encoding reused by SA-stored xattrs (6.6) and the FUID table (3.4).
- `05-dmu-dnode-objset.md` - dnode bonus/spill mechanics, object types, object-set accounting objects backing v4 quotas.
- `06-zap.md` - master node, directories, registry/layout tables, quota ZAPs, unlinked set; normalization flags for 6.3.
- `07-dsl-dataset-snapshot.md` - reaching the filesystem object set; the integer-key hex convention (its 6.3) used by the unlinked set and quota ZAPs.
- Deferred to later documents: ACL access-mask/inheritance semantics and evaluation, FUID table nvlist schema, share-directory contents, intent-log (ZIL) record formats.
