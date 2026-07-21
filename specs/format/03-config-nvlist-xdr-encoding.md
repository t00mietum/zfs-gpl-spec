# ZFS on-disk format: config nvlist - XDR wire encoding

**Provenance**

- OpenZFS commit studied for behavioral corroboration: `9b7642df` (reference only).
- Status: **APPROVED** 2026-07-21 (gatekeeper review complete). Authoritative.
- Each fact below is tagged with a provenance marker:
	- **[2006]** - corroborated by the public 2006 Sun "ZFS On-Disk Specification" (the durable core, independently published). The XDR name/value-list format itself is the pre-existing Solaris libnvpair format the 2006 document builds on.
	- **[modern]** - source-only; a post-2006 addition or refinement observed in current OpenZFS. Required to interoperate with pools written by recent OpenZFS, but not part of the original published spec.

This document refines section 3.1 of `01-vdev-label-uberblock.md`. Spec 01 establishes *that* the label config area (region 3) holds an XDR-encoded name/value list and catalogs the key strings (01 sections 3.2-3.6). This document specifies the byte-exact XDR wire format of that serialized list, so an implementer can pack and unpack it to a byte-for-byte match. The same wire format applies to every XDR-packed nvlist in ZFS (MOS config, `features_for_read`, embedded child lists), not only the label config.

The key catalog (which names appear, their value types and meanings) is in `01` sections 3.2-3.6 and is not repeated here. This document covers only the serialization: the bootstrap header, the list header, the per-pair framing, the primitive encodings, the type codes, and the terminator.

---

## 1. Two framing layers

A packed nvlist has two distinct layers:

1. A fixed **4-byte bootstrap header** at the very front, stored as raw bytes (not itself XDR-encoded). It declares which encoding and byte order follow. **[2006]**
2. The **XDR-encoded body**: a small list header, then a sequence of encoded pairs, then a terminator. Every integer in the body follows XDR rules (see section 4). **[2006]**

A reader consumes the 4 raw header bytes first, selects the decoder from byte 0, then decodes the body.

---

## 2. Bootstrap header (4 bytes, raw)

| Offset | Size | Field | Meaning |
|-------:|-----:|-------|---------|
| 0 | 1 | encoding | Serialization method of the body. `1` = XDR; `0` = native (host-endian) form. On-disk label/config lists use **`1` (XDR)**. **[2006]** |
| 1 | 1 | endian | Byte order of the writing host (a host-endianness tag byte). **[2006]** |
| 2 | 1 | reserved | Zero. **[2006]** |
| 3 | 1 | reserved | Zero. **[2006]** |

Key interoperability fact: for the **XDR** encoding the body is inherently big-endian (section 4), so a reader decoding XDR does **not** consult the endian byte at offset 1 - the XDR body decodes identically on any host. The endian byte governs only the *native* encoding, where a mismatch between the stored tag and the reader's host order makes the native blob unreadable. Because on-disk ZFS config uses XDR, the stored config list is portable across writer and reader architectures regardless of that byte. **[modern]** (behavioral corroboration; the 2006 format defines the header, the endian-ignored-on-XDR detail is source-confirmed).

---

## 3. XDR body

### 3.1 List header

Immediately after the 4 bootstrap bytes, each nvlist (top-level or embedded) begins with two 4-byte XDR integers:

| Order | Size | Field | Value |
|------:|-----:|-------|-------|
| 1 | 4 | version | List format version. On disk this is **`0`**. **[2006]** |
| 2 | 4 | flags | Persistent name-uniqueness flags. Bit `0x1` = names unique by name; bit `0x2` = names unique by (name, type). **[2006]** |

Note: an **embedded** nvlist (a value of type NVLIST or an element of an NVLIST_ARRAY, section 5) carries its own version+flags header and its own terminator, but **not** its own 4-byte bootstrap header - only the outermost packed list has the bootstrap header. **[2006]**

### 3.2 Encoded pair

Zero or more pairs follow the list header. Each pair is:

| Order | Size | Field | Meaning |
|------:|-----:|-------|---------|
| 1 | 4 | encoded size | Number of bytes this pair occupies in the packed stream (from this field through the end of the value, including padding). **[2006]** |
| 2 | 4 | decoded size | In-memory size of the reconstructed pair. A reader uses only whether this is zero (terminator, section 3.3); the value itself is advisory for buffer sizing. **[2006]** |
| 3 | var | name | XDR string (section 4.3): 4-byte length (excluding terminator) + name bytes + zero pad to a 4-byte multiple. **[2006]** |
| 4 | 4 | type | Value type code (section 6). **[2006]** |
| 5 | 4 | element count | Number of value elements: `1` for a scalar, `n` for an n-element array, `0` for a valueless BOOLEAN. **[2006]** |
| 6 | var | value | Encoded per the type (sections 4-5). Absent when element count is `0`. **[2006]** |

Both size fields are 4-byte XDR integers. The two leading size fields plus the name-length, type, and element-count fields are the "5 words" of per-pair overhead before any value or name payload. **[modern]** (exact framing confirmed against source; consistent with the 2006 description).

### 3.3 Terminator

The list ends with a pair whose leading two size fields are both **zero** (8 zero bytes: encoded size `0`, decoded size `0`). A reader stops at the first pair whose decoded size is zero; no name, type, or value follows. **[2006]**

---

## 4. XDR primitive encoding

The body follows standard XDR (fixed 4-byte units, big-endian, no host-order variants). **[2006]**

### 4.1 Integers and widening

- Every scalar occupies a whole number of 4-byte units, big-endian. **[2006]**
- 1-, 2-, and 4-byte signed/unsigned integers (BYTE, INT8/UINT8, INT16/UINT16, INT32/UINT32) are each widened to **one 4-byte big-endian unit**. **[2006]**
- 8-byte integers (INT64, UINT64) and the high-resolution time type (HRTIME) are encoded as an **XDR hyper**: **8 bytes big-endian**. **[2006]**
- A BOOLEAN_VALUE is a 4-byte big-endian integer, `0` or `1`. **[modern]**
- A valueless BOOLEAN carries no value bytes at all (element count `0`). **[2006]**

### 4.2 Alignment / padding

All payloads are padded with zero bytes up to the next 4-byte boundary. Names and opaque byte payloads whose length is not a multiple of 4 are zero-filled to the boundary; the pad bytes are part of the pair's encoded size. **[2006]**

### 4.3 Strings

A string is a 4-byte big-endian length **N** (the character count, excluding any NUL terminator), followed by **N** bytes, followed by zero padding to the next 4-byte boundary. **[2006]**

### 4.4 Opaque byte array

A BYTE_ARRAY (opaque) is `nelem` raw bytes followed by zero padding to a 4-byte boundary. The element count (section 3.2 field 5) carries the byte length; there is no second inline length. **[2006]**

### 4.5 Numeric arrays

An array of a numeric type is a 4-byte big-endian element count followed by that many elements, each encoded as its widened unit from section 4.1 (4 bytes each for <=32-bit element types; 8 bytes each for 64-bit element types), in order. **[2006]**

### 4.6 String array

A STRING_ARRAY is `nelem` strings, each encoded as in section 4.3 (length + bytes + pad), concatenated in order. **[2006]**

---

## 5. Nested lists

- **NVLIST value**: the value is a complete embedded nvlist - version + flags header (section 3.1), its pairs, and its own zero terminator - encoded inline in place of the value. It does not repeat the 4-byte bootstrap header. **[2006]**
- **NVLIST_ARRAY value**: `nelem` embedded nvlists as above, concatenated in order, each with its own header and terminator. **[2006]**

Nesting is recursive: the label's `vdev_tree` (01 section 3.3) is an NVLIST whose `children` (01 section 3.3) is an NVLIST_ARRAY, each element itself an NVLIST following these same rules. **[2006]**

---

## 6. Type codes

The type field (section 3.2 field 4) is a 4-byte big-endian integer. The codes are assigned in a fixed order; a reader matches by numeric value. **[2006]** for the original ordered set (codes 1-20, the classic Solaris types the 2006 spec builds on); **[modern]** for the codes appended later (21 and up).

| Code | Type | Value encoding | Provenance |
|-----:|------|----------------|------------|
| 1 | BOOLEAN (valueless) | none (element count 0) | **[2006]** |
| 2 | BYTE | 1 byte in a 4-byte unit | **[2006]** |
| 3 | INT16 | 4-byte unit | **[2006]** |
| 4 | UINT16 | 4-byte unit | **[2006]** |
| 5 | INT32 | 4-byte unit | **[2006]** |
| 6 | UINT32 | 4-byte unit | **[2006]** |
| 7 | INT64 | 8-byte hyper | **[2006]** |
| 8 | UINT64 | 8-byte hyper | **[2006]** |
| 9 | STRING | length + bytes + pad (4.3) | **[2006]** |
| 10 | BYTE_ARRAY | opaque bytes + pad (4.4) | **[2006]** |
| 11 | INT16_ARRAY | count + 4-byte units (4.5) | **[2006]** |
| 12 | UINT16_ARRAY | count + 4-byte units (4.5) | **[2006]** |
| 13 | INT32_ARRAY | count + 4-byte units (4.5) | **[2006]** |
| 14 | UINT32_ARRAY | count + 4-byte units (4.5) | **[2006]** |
| 15 | INT64_ARRAY | count + 8-byte hypers (4.5) | **[2006]** |
| 16 | UINT64_ARRAY | count + 8-byte hypers (4.5) | **[2006]** |
| 17 | STRING_ARRAY | count strings (4.6) | **[2006]** |
| 18 | HRTIME | 8-byte hyper | **[2006]** |
| 19 | NVLIST | embedded list (section 5) | **[2006]** |
| 20 | NVLIST_ARRAY | count embedded lists (section 5) | **[2006]** |
| 21 | BOOLEAN_VALUE | 4-byte 0/1 | **[modern]** |
| 22 | INT8 | 4-byte unit | **[modern]** |
| 23 | UINT8 | 4-byte unit | **[modern]** |
| 24 | BOOLEAN_ARRAY | count + 4-byte units | **[modern]** |
| 25 | INT8_ARRAY | count + 4-byte units | **[modern]** |
| 26 | UINT8_ARRAY | count + 4-byte units | **[modern]** |

A floating-point (DOUBLE) type exists only in userland builds and does not appear in on-disk pool data. **[modern]**

### 6.1 Types actually used in a label config

The label config list (01 section 3) uses a small subset: UINT64 (8) for the numeric keys, STRING (9) for names/paths, NVLIST (19) for `vdev_tree`/`features_for_read`/nested members, NVLIST_ARRAY (20) for `children`/`spares`/`l2cache`, UINT64_ARRAY (16) for members such as `hole_array`, and BOOLEAN (1) / BOOLEAN_VALUE (21) for feature-set membership. An implementer that handles these correctly can read every stock label; the remaining codes must still be parsed (skipped via each pair's encoded size) so an unknown pair does not desync the stream. **[modern]**

---

## 7. Reader recipe

1. Read 4 bootstrap bytes. Require byte 0 == `1` (XDR) for on-disk config; if it is `0` (native), byte 1 must equal the reader's host order or the blob is not portable (section 2).
2. Decode two 4-byte integers: version (expect `0`) and flags.
3. Loop:
	a. Read encoded size and decoded size (two 4-byte integers). If decoded size is `0`, stop - end of list.
	b. Read the name (XDR string), the type code, and the element count.
	c. If element count is `0` and type is BOOLEAN, the pair has no value; otherwise decode the value per its type (sections 4-5).
	d. A reader may instead advance by the pair's encoded size to skip a pair whose type it does not need; the encoded size always lands on the next pair. **[2006]**
4. For NVLIST / NVLIST_ARRAY values, recurse into step 2 (embedded lists have a version+flags header and terminator but no bootstrap header).

---

## 8. Quick reference

| Item | Value / rule | Provenance |
|------|--------------|------------|
| Bootstrap header | 4 raw bytes: encoding, endian, 0, 0 | **[2006]** |
| On-disk encoding | XDR (encoding byte `1`) | **[2006]** |
| Endian byte on XDR decode | ignored; XDR body is big-endian, portable | **[modern]** |
| List header | version (4B, `0`) + flags (4B) | **[2006]** |
| Per-pair overhead | encoded size (4B) + decoded size (4B) + name + type (4B) + nelem (4B) | **[2006]** framing / **[modern]** exact bytes |
| Name | XDR string: len(4B) + bytes + pad to 4 | **[2006]** |
| 32-bit-and-under scalars | one 4-byte big-endian unit | **[2006]** |
| 64-bit scalars / HRTIME | 8-byte big-endian hyper | **[2006]** |
| String | len(4B) + bytes + zero pad to 4 | **[2006]** |
| Terminator | two 4-byte zeros (decoded size 0) | **[2006]** |
| Embedded nvlist | own header + terminator, no bootstrap header | **[2006]** |
| Type codes 1-20 | classic ordered set | **[2006]** |
| Type codes 21-26 | appended later | **[modern]** |
