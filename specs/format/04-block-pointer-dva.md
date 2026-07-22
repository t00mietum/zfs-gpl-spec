# ZFS on-disk format: block pointers and DVAs

**Provenance**

- OpenZFS commit studied for behavioral corroboration: `9b7642df` (reference only).
- Status: **APPROVED** 2026-07-21 (gatekeeper review complete). Authoritative.
- Each fact below is tagged with a provenance marker:
	- **[2006]** - corroborated by the public 2006 Sun "ZFS On-Disk Specification" (the durable core, independently published).
	- **[modern]** - source-only; a post-2006 addition or refinement observed in current OpenZFS. Required to interoperate with pools written by recent OpenZFS, but not part of the original published spec.

This document specifies the block pointer: the fixed 128-byte structure that every ZFS object uses to locate, size, checksum, and describe a block of data on disk. It answers request #4. Block pointers are the edges of the entire pool graph - the uberblock's root pointer (see `01-vdev-label-uberblock.md` section 5) is a block pointer, every indirect block is an array of block pointers, and every dnode embeds them. An implementer must read them byte-exactly to traverse the pool.

Object-type codes (the `type` field), the dnode and object-set structures that hold arrays of block pointers, and the details of the ZIL are specified elsewhere (DMU and later specs) and are only referenced here.

---

## 1. Size, shape, and word numbering

- A block pointer is exactly **128 bytes**: sixteen 64-bit words, numbered word 0 through word 15 (hex 0..f) in increasing address order. **[2006]**
- Every multi-byte field is stored in the **native byte order of the host that wrote the containing block** (the same rule as all other ZFS structures; see `01` section 4). A reader corrects byte order using the byteorder bit in the property word (section 4.8) or, for whole blocks, the enclosing structure's byte-order signal. **[2006]**
- The 16 words are grouped as: three **DVAs** (words 0..5, two words each), one **property word** (word 6), reserved/extra words (words 7..8), two **birth-txg** words (words 9..a), a **fill count** (word b), and a **256-bit checksum** (words c..f). **[2006]** for the DVA-triple / property / birth / fill / checksum grouping; **[modern]** for the exact assignment of words 7..a described in section 5.

Standard word/bit map (non-embedded, unencrypted block pointer). Bit 63 is most significant:

```
 word  63          48 47      40 39 38    32 31        16 15         0
  0     pad | vdev(hi)  |  reserved  ...  |  reserved  |    ASIZE      |   -- DVA 0 word 0
  1     G |                       offset 0 (63 bits)                    |   -- DVA 0 word 1
  2     ...                       DVA 1 word 0  (same layout as word 0) |
  3     ...                       DVA 1 word 1  (same layout as word 1) |
  4     ...                       DVA 2 word 0                          |
  5     ...                       DVA 2 word 1                          |
  6     B D X | level(5) | type(8) | checksum(8) | E | comp(7) | PSIZE(16) | LSIZE(16)
  7     R |                    reserved / extra                          |
  8     |                        reserved / extra                       |
  9     |                    physical birth txg                         |
  a     |                    logical  birth txg                         |
  b     |                        fill count                             |
  c     |                        checksum word 0                        |
  d     |                        checksum word 1                        |
  e     |                        checksum word 2                        |
  f     |                        checksum word 3                        |
```

The exact bit positions of each field are given in the sections below; the map above is orientation only.

---

## 2. The DVA (data virtual address)

Each block pointer carries up to **three** DVAs (words 0..1, 2..3, 4..5). **[2006]** Multiple DVAs are independent on-disk copies of the same logical block ("ditto blocks"); any valid one can be read. A DVA is two 64-bit words.

### 2.1 DVA word 0 - vdev, allocated size, reserved

Bit fields of the first DVA word (bit 0 = least significant):

| Field | Bits | Meaning | Provenance |
|-------|------|---------|------------|
| ASIZE | 0..23 (24 bits) | allocated size, in 512-byte sectors | **[2006]** |
| reserved ("grid") | 24..31 (8 bits) | historically RAID-Z layout hint; treated as zero | **[2006]** field exists / **[modern]** now unused |
| vdev id | 32..55 (24 bits) | top-level vdev index the block lives on | **[2006]** field / **[modern]** 24-bit width |
| reserved (pad) | 56..63 (8 bits) | zero | **[modern]** |

- **ASIZE** is the total space allocated for this copy, in units of 512 bytes (the minimum block-shift; `01` section 4.4). Actual bytes allocated = `ASIZE << 9`. There is **no** bias on ASIZE: a value of 0 means the DVA is empty/unused. **[2006]** ASIZE includes RAID-Z parity and gang-header overhead, so it can exceed the block's physical size (section 4.3). **[2006]**
- **vdev id** selects the top-level vdev (as ordered in the config nvlist vdev tree; see `01` section 3). The original format allotted the full high 32 bits (bits 32..63) to the vdev id; current implementations use only the low 24 of those (bits 32..55) and keep bits 56..63 zero. Reading a 24-bit field is interop-safe because the high 8 bits are zero on disk in practice. **[2006]** for the field, **[modern]** for the 24-bit narrowing.
- A DVA is **valid** iff its ASIZE is non-zero; a DVA with both words zero is empty. **[2006]**

### 2.2 DVA word 1 - gang flag and offset

| Field | Bits | Meaning | Provenance |
|-------|------|---------|------------|
| offset | 0..62 (63 bits) | vdev-relative block address, in 512-byte sectors | **[2006]** |
| G (gang) | 63 (1 bit) | 1 = this DVA points to a gang block header, not data | **[2006]** |

- **offset** is the location of the block **within the addressable data area of the vdev**, measured in 512-byte sectors. The byte offset within that area = `offset << 9`. **[2006]**
- The addressable data area begins **4 MiB** (`0x400000`) into the vdev, after the two front labels (2 x 256 KiB) and the reserved boot region (3.5 MiB) documented in `01` sections 2.2-2.3. Therefore the **physical device byte offset** of a (non-RAID-Z) block copy is:

  ```
  device_byte_offset = (offset << 9) + 0x400000
  ```
  **[2006]** (For RAID-Z vdevs the mapping from this vdev-relative offset to child-disk offsets and the placement of parity is a separate geometry, specified with RAID-Z; the `offset << 9` vdev-relative address and the 4 MiB skip are unchanged.)
- **G (gang)**: when set, the target of this DVA is not the block's data but a **gang block header** - a small self-checksummed block holding an array of block pointers whose targets, concatenated, form the logical block. See section 6.2. **[2006]**

---

## 3. Sizes and the shift/bias encoding

Three distinct sizes describe a block; all three are stored as **sector counts** (units of 512 bytes) in the standard block pointer, not as raw byte counts. **[2006]**

| Size | Where | Encoding | Actual bytes |
|------|-------|----------|--------------|
| **LSIZE** logical size | property word bits 0..15 | stored value + 1, then x512 | `(LSIZE_raw + 1) << 9` |
| **PSIZE** physical size | property word bits 16..31 | stored value + 1, then x512 | `(PSIZE_raw + 1) << 9` |
| **ASIZE** allocated size | DVA word 0 bits 0..23 | x512, no bias | `ASIZE_raw << 9` |

- **LSIZE** is the logical (uncompressed) size of the block; **PSIZE** is its physical (post-compression) on-disk size. If no compression is in effect, `PSIZE == LSIZE`. **[2006]**
- LSIZE and PSIZE carry a **+1 bias**: the stored value is `(bytes >> 9) - 1`, so a stored 0 means 512 bytes and the maximum stored value `0xFFFF` means `65536 << 9` = **32 MiB**. A standard block can never be zero-sized, which is why the bias is used. **[2006]** for the sector encoding; **[modern]** for the exact +1 bias and 32 MiB ceiling.
- **ASIZE** has **no** bias (its zero value is meaningful - an empty DVA), and is per-DVA (each copy may have a different ASIZE, e.g. different RAID-Z widths). **[2006]**
- Relationship: `PSIZE <= LSIZE` (compression only shrinks) and, for a single non-RAID-Z copy, `ASIZE >= PSIZE` (allocation rounds up to the vdev's alignment and adds any gang overhead). **[2006]**

Embedded block pointers encode their sizes differently (in bytes, not sectors); see section 6.3.

---

## 4. The property word (word 6)

Word 6 packs the block's type, level, compression, checksum identity, sizes, and several flag bits. Bit 0 is least significant.

| Field | Bits | Width | Meaning | Provenance |
|-------|------|-------|---------|------------|
| LSIZE | 0..15 | 16 | logical size (section 3) | **[2006]** |
| PSIZE | 16..31 | 16 | physical size (section 3) | **[2006]** |
| compression | 32..38 | 7 | compression identity (section 7.2) | **[2006]** |
| E (embedded) | 39 | 1 | 1 = embedded block pointer (section 6.3) | **[modern]** |
| checksum | 40..47 | 8 | checksum identity (section 7.1) | **[2006]** |
| type | 48..55 | 8 | DMU object type (specified with DMU) | **[2006]** |
| level | 56..60 | 5 | indirection level | **[2006]** |
| X (crypt) | 61 | 1 | 1 = encryption/authentication parameters present | **[modern]** |
| D (dedup) | 62 | 1 | 1 = block is deduplicated | **[2006]** |
| B (byteorder) | 63 | 1 | writer's byte order (section 4.8) | **[2006]** |

### 4.1 LSIZE / PSIZE
As section 3. In embedded block pointers these two 16-bit slots are reallocated (section 6.3).

### 4.2 Compression (bits 32..38)
A 7-bit compression-identity code (section 7.2). Only the algorithm identity lives here; where an algorithm has a level (gzip, zstd), gzip levels are distinct enum codes while the zstd level is carried in the compressed payload, not in this field. **[2006]** for the field; identity values per section 7.2.

### 4.3 Checksum (bits 40..47)
An 8-bit checksum-identity code (section 7.1) naming the algorithm that produced the 256-bit checksum in words c..f. **[2006]**

### 4.4 Type (bits 48..55)
An 8-bit DMU object-type code. The enumeration is specified with the DMU dnode format and is not reproduced here. **[2006]**

### 4.5 Level (bits 56..60)
Indirection level. **Level 0** is a leaf (the block holds the object's actual data). A **level N > 0** block holds an array of block pointers pointing at level N-1 blocks; this is how large objects fan out into an indirect tree. **[2006]**

### 4.6 Embedded / crypt flags (bits 39, 61)
- **E (bit 39)**: the block pointer carries its data payload inline and has no DVAs (section 6.3). **[modern]**
- **X (bit 61)**: the block uses ZFS native-encryption handling; encryption parameters occupy space normally used by the third DVA, part of the fill-count word, and half the checksum field. The detailed encrypted layout is deferred to the native-encryption spec. **[modern]**

### 4.7 Dedup (bit 62)
Set when the block participates in deduplication (a dedup-table entry references it). Does not change the block's on-disk data layout. **[2006]**

### 4.8 Byteorder (bit 63) and the byteswap rule
The byteorder bit records the endianness of the host that wrote the block: **0 = big-endian writer, 1 = little-endian writer**. **[2006]**

A reader compares this bit to its own host order and byte-swaps the block's structured contents when they differ:

```
should_byteswap = (byteorder_bit != host_byteorder)      host_byteorder: 0 on BE, 1 on LE
```

So a block written big-endian (bit 0) read on a little-endian host (host order 1) is swapped, and vice versa; matching orders are read as-is. This mirrors the label/uberblock endianness detection in `01` section 7, but the per-block signal is this bit rather than a magic value. **[2006]** for the convention and the not-equal test; **[modern]** for the exact bit polarity confirmation (0=BE, 1=LE).

---

## 5. Birth txgs, fill count, checksum (words 7..f)

| Word | Field | Meaning | Provenance |
|------|-------|---------|------------|
| 7 | reserved / rewrite | high bit is a "rewrite" flag; remainder reserved | **[modern]** |
| 8 | reserved | extra space, zero | **[2006]** padding / **[modern]** exact split |
| 9 | physical birth txg | txg in which the first DVA was physically written; 0 if equal to the logical birth | **[modern]** |
| a | logical birth txg | txg in which the block was logically born (last modified) | **[2006]** |
| b | fill count | number of non-zero blocks beneath this pointer | **[2006]** |
| c..f | checksum | 256-bit checksum, four 64-bit words | **[2006]** |

- **Birth txg.** The original format stored a single birth txg (the transaction group that produced the block). Current OpenZFS splits it into a **logical** birth (word a; when the data last changed) and a **physical** birth (word 9; when the copy was physically written, which can differ after device removal or rewrite). Word 9 is **0** when the physical birth equals the logical birth, in which case the logical birth (word a) is the effective birth. **[2006]** for the single birth txg concept and word a; **[modern]** for the physical/logical split into word 9 and the "0 means same as logical" rule.
- **Fill count (word b).** For a level-0 pointer, 1 if the block is non-empty. For an indirect (level > 0) pointer, the total number of non-hole blocks in the subtree it roots. This lets a reader know how much live data hangs below a pointer without descending. **[2006]** (In encrypted block pointers the upper 32 bits of this word hold part of the IV; see the native-encryption spec.) **[modern]**
- **Checksum (words c..f).** The 256-bit checksum of the data the pointer describes, produced by the algorithm named in the property word's checksum field (section 4.3). It is four 64-bit words in increasing address order. Fletcher checksums fill all 256 bits; a 256-bit cryptographic hash (e.g. SHA-256) fills all four words. The digest-to-word packing used for the self-checksummed label/uberblock is specified in `02-label-checksum-sha256-packing.md`; block-data checksums are stored as the algorithm's four output words in native byte order and are byte-swapped on read per section 4.8. **[2006]**

---

## 6. Block-pointer variants: hole, gang, embedded

Three variants deviate from the standard "three DVAs pointing at data" shape. A reader must test for them in this order: embedded first (it repurposes the whole structure), then hole, then gang.

### 6.1 Hole
A **hole** is a block that was never written or is entirely zero-filled; ZFS stores no data for it. **[2006]**

- Detection: the block pointer is **not embedded** (E bit clear) **and** its first DVA is entirely zero (both words of DVA 0 == 0). **[2006]**
- A hole may still carry a logical size, level, type, and birth txg (a region punched after having held data), so those fields are not necessarily zero; only the DVAs are. A read of a hole returns zeros for the whole logical size. **[2006]**

### 6.2 Gang block
When a single contiguous allocation of the needed size is unavailable, ZFS allocates several smaller pieces and stitches them together with a **gang block**. **[2006]**

- A DVA with its **gang bit set** (section 2.2) points not at data but at a **gang block header**: a small block containing an array of block pointers followed by a self-checksumming trailer. The classic gang header is **512 bytes** and holds up to **three** block pointers plus padding and a 40-byte checksum trailer (the same `{magic, 256-bit checksum}` embedded-checksum trailer shape used elsewhere; see `01` section 6). **[2006]** for the gang concept and 512-byte 3-pointer header; **[modern]** for the note that newer gang blocks may be larger and hold more pointers.
- The gang header is self-checksummed with the dedicated **gang-header** checksum identity (section 7.1), verified like any embedded-checksum structure. To read the logical block, a reader validates the header, then reads each non-empty child pointer in order and concatenates their data. Gang trees may nest (a child pointer may itself be a gang pointer). **[2006]**

### 6.3 Embedded block pointer
A very small block can be stored **inside** the block pointer itself, eliminating the pointed-to block entirely. **[modern]**

- Detection: the property word's **E bit (bit 39) is set**. An embedded block pointer has **no DVAs** and is never a hole or gang. **[modern]**
- **Payload.** The data occupies **14 of the 16 words = 112 bytes**, using every word *except* the property word (word 6) and the logical-birth word (word a). Concretely the payload words are 0..5, 7..9, and b..f, read in that address order to reconstruct the (possibly compressed) payload. **[modern]**
- **Sizes are in bytes, not sectors,** and use different bit positions in the property word than a standard pointer:

  | Field | Property-word bits | Width | Encoding |
  |-------|--------------------|-------|----------|
  | LSIZE (bytes) | 0..24 | 25 | stored value + 1 = bytes |
  | PSIZE (bytes) | 25..31 | 7 | stored value + 1 = bytes |
  | embedded-type (etype) | 40..47 | 8 | how to interpret the payload |

  PSIZE (<= 112) is the compressed payload length actually present in the 14 words; LSIZE is its logical size after decompression. The compression field (bits 32..38), level, type, and the B/D/X/E flags keep their standard meaning and positions. **[modern]**
- **Embedded-type (etype)** codes, occupying the 8 bits where a standard pointer keeps its checksum identity: **[modern]**

  | Code | Name | Meaning | Provenance |
  |------|------|---------|------------|
  | 0 | DATA | payload is ordinary (optionally compressed) file/metadata data | **[modern]** |
  | 1 | (reserved) | reserved | **[modern]** |
  | 2 | REDACTED | block was redacted (send/receive redaction); no real data | **[modern]** |

- An embedded pointer has no separate checksum field or DVA checksum; its integrity rides on the checksum of the block that contains it. Encrypted blocks can never be embedded (they need the payload space for encryption parameters). **[modern]**

---

## 7. Checksum and compression identity codes

Both fields in the property word hold a small integer that indexes a fixed, ordered enumeration. The numeric codes are on-disk facts: a reader must map the stored integer to the algorithm below.

### 7.1 Checksum identities (property-word bits 40..47)

| Code | Algorithm / role | Provenance |
|------|------------------|------------|
| 0 | inherit (from parent; not seen on disk in a written block) | **[2006]** |
| 1 | on (policy alias; resolves to a concrete algorithm) | **[2006]** |
| 2 | off (no checksum) | **[2006]** |
| 3 | label (self-checksummed label/uberblock; see `02`) | **[2006]** |
| 4 | gang header (self-checksummed gang block; section 6.2) | **[2006]** |
| 5 | ZIL log | **[2006]** |
| 6 | fletcher-2 | **[2006]** |
| 7 | fletcher-4 | **[2006]** |
| 8 | SHA-256 | **[2006]** |
| 9 | ZIL log v2 | **[modern]** |
| 10 | no-parity (skip verification; special ZIL use) | **[modern]** |
| 11 | SHA-512/256 | **[modern]** |
| 12 | Skein | **[modern]** |
| 13 | Edon-R | **[modern]** |
| 14 | BLAKE3 | **[modern]** |

Codes 0..8 are the original ordered set; 9..14 were appended in the order shown and are gated by pool feature flags (specified with feature flags and the extended-checksum specs). The default data checksum for a modern pool is fletcher-4 (code 7); the dedup checksum is SHA-256 (code 8). **[2006]** for codes 0..8 and their order; **[modern]** for codes 9..14 and the exact appended order. The low 8 bits are the identity; a separate in-core "verify" flag (bit 8 of the wider field) is not stored on disk.

### 7.2 Compression identities (property-word bits 32..38)

| Code | Algorithm | Provenance |
|------|-----------|------------|
| 0 | inherit | **[2006]** |
| 1 | on (policy alias; resolves to a concrete algorithm) | **[2006]** |
| 2 | off (no compression; PSIZE == LSIZE) | **[2006]** |
| 3 | LZJB | **[2006]** |
| 4 | empty (all-zero payload marker) | **[2006]** |
| 5..13 | gzip level 1 through gzip level 9 (code = 4 + level) | **[2006]** |
| 14 | ZLE (zero-length encoding) | **[2006]** |
| 15 | LZ4 | **[modern]** |
| 16 | Zstandard (zstd) | **[modern]** |

Codes 0..14 are the original ordered set; LZ4 (15) and zstd (16) were appended and are gated by pool feature flags (specified with the compression-identity spec). When "on" (code 1) is resolved, a legacy pool uses LZJB and a modern pool uses LZ4. **[2006]** for codes 0..14 and their order; **[modern]** for codes 15..16. The 7-bit field is wide enough for all current identities; the gzip level is part of the identity code, not a separate stored field. **[2006]**

---

## 8. Reader recipe

To follow a block pointer and read the block it describes:

1. **Embedded?** If the property word's E bit (bit 39) is set, the data is inline: gather payload words 0..5, 7..9, b..f (112 bytes), take PSIZE bytes (embedded encoding, section 6.3), decompress per the compression field to LSIZE bytes. Done - no device I/O. **[modern]**
2. **Hole?** If not embedded and DVA 0 is all-zero, the block is a hole: return LSIZE bytes of zero. **[2006]**
3. **Pick a DVA.** Choose a valid DVA (ASIZE != 0). For each: read vdev id, ASIZE, gang bit, and offset (section 2). **[2006]**
4. **Gang?** If the DVA's gang bit is set, read the gang header at that location, validate its self-checksum (gang-header identity), then recurse into each child pointer and concatenate. **[2006]**
5. **Locate.** Otherwise compute the device byte offset `(offset << 9) + 0x400000` on the selected vdev (RAID-Z vdevs remap this vdev-relative address to child disks per RAID-Z geometry). Read PSIZE bytes. **[2006]**
6. **Byte-swap** the block's structured contents if the property word's byteorder bit differs from the host (section 4.8). **[2006]**
7. **Verify** the 256-bit checksum (words c..f) using the checksum identity (section 4.3) over the PSIZE bytes read; on mismatch, try another DVA copy. **[2006]**
8. **Decompress** per the compression field (section 4.2) from PSIZE to LSIZE bytes. If level > 0, the result is an array of block pointers (recurse); if level 0, it is the object's data. **[2006]**

---

## 9. Constants and rules quick reference

| Item | Value / rule | Provenance |
|------|--------------|------------|
| Block pointer size | 128 bytes = sixteen 64-bit words | **[2006]** |
| DVAs per pointer | 3 (independent copies) | **[2006]** |
| Sector unit | 512 bytes (`<< 9`) for ASIZE, offset, LSIZE, PSIZE | **[2006]** |
| Vdev data start | 4 MiB (`0x400000`) into the vdev; device offset = `(offset << 9) + 0x400000` | **[2006]** |
| ASIZE | DVA word 0 bits 0..23; `raw << 9`, no bias; 0 = empty DVA | **[2006]** |
| Vdev id | DVA word 0 bits 32..55 (24 bits; high 8 zero) | **[2006]** field / **[modern]** width |
| Offset / gang | DVA word 1 bits 0..62 = offset; bit 63 = gang | **[2006]** |
| LSIZE / PSIZE | property word bits 0..15 / 16..31; `(raw + 1) << 9` bytes; max 32 MiB | **[2006]** core / **[modern]** +1 bias |
| Compression | property word bits 32..38 (7 bits); identity per 7.2 | **[2006]** |
| Embedded flag | property word bit 39 | **[modern]** |
| Checksum id | property word bits 40..47 (8 bits); identity per 7.1 | **[2006]** |
| Type | property word bits 48..55 (8 bits); DMU object type | **[2006]** |
| Level | property word bits 56..60 (5 bits); 0 = data, >0 = indirect | **[2006]** |
| Crypt / dedup / byteorder | property word bits 61 / 62 / 63 | **[2006]** (D,B) / **[modern]** (X) |
| Byteorder polarity | 0 = BE writer, 1 = LE writer; swap iff bit != host order | **[2006]** / **[modern]** polarity |
| Physical / logical birth | words 9 / a; word 9 == 0 means "same as logical" | **[2006]** birth / **[modern]** split |
| Fill count | word b; non-hole blocks below the pointer | **[2006]** |
| Checksum words | words c..f (256 bits, four words, native order) | **[2006]** |
| Embedded payload | 112 bytes in words 0..5,7..9,b..f; sizes in bytes | **[modern]** |
| Hole test | not embedded and DVA 0 all-zero | **[2006]** |
| Default data checksum / compression (modern) | fletcher-4 (7) / LZ4 (15) | **[modern]** |
