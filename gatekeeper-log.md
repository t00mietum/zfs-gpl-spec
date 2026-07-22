# Gatekeeper log

Dated record of gatekeeper reviews. One entry per spec approval. The gatekeeper reviews each draft against the wall rules (facts only; no source lines, comments, internal identifiers, or source-mirroring structure), confirms facts are stated correctly, strips any leaked expression, and records the outcome here.

---

## 2026-07-19 - `specs/format/01-vdev-label-uberblock.md`

Reviewed: VDEV label and uberblock on-disk format - label size and count, on-device label positions, reserved boot region, internal label regions, config nvlist encoding and key strings, uberblock array geometry, uberblock field layout, self-checksum construction, endianness detection, and active-uberblock selection.

Spot-checks against the pinned source (`9b7642df...`) confirmed as facts: label size 256 KiB and the four-label front/rear placement; label region sizes (8 KiB / 8 KiB / 112 KiB / 128 KiB); reserved boot region 3.5 MiB at device offset 512 KiB; uberblock slot shift clamped to 10..13 (1 KiB..8 KiB); uberblock field offsets (0/8/16/24/32/40+128/168/176/184/192/200/208, fixed portion 216 bytes); magic values (uberblock, MMP, checksum trailer); trailer size 40 bytes (8 magic + 32 checksum); minimum block shift value 9.

Stripped: one internal source macro name in the RAID-Z reflow section. Replaced with the numeric fact (shift of 9; stored value times 512 recovers the byte offset). On-disk nvlist key strings, documented magic values, and constants were retained as interface facts.

Outcome: APPROVED. Header status changed DRAFT -> APPROVED.

---

## 2026-07-20 - `specs/format/02-label-checksum-sha256-packing.md`

Reviewed: the SHA-256 digest-to-word packing for the label/uberblock self-checksum trailer (request #2) - the mapping from the 256-bit SHA-256 output to the four 64-bit checksum words, the big-endian pairing orientation, host-endianness independence of the word values vs. writer-native on-disk byte order, the reader comparison rule, and packing test vectors.

Spot-checks against the pinned source (`9b7642df...`) confirmed as facts: the label/uberblock checksum is SHA-256; the 256-bit result is stored as four 64-bit words; each word is the big-endian 64-bit pairing of two consecutive FIPS 180 hash words, high word first (`word[j] = (H(2j)<<32)|H(2j+1)`); the word values are forced to one big-endian orientation so they are identical for little- and big-endian writers (there is no separate byte-swap variant of this checksum's value); the words are then stored in writer-native order and corrected on read via the trailer magic; all four words are compared with no truncation.

Leak scan: clean. No source identifiers, function/type/macro names, comments, or source-mirroring structure. FIPS 180 notation (H0..H7, D[0..31]) and neutral `word[j]` terms only. The two worked examples use the published standard SHA-256 test vectors (empty input and "abc"), independently verified, not ZFS-specific data.

Outcome: APPROVED. Header status set DRAFT -> APPROVED.

---

## 2026-07-21 - `specs/format/03-config-nvlist-xdr-encoding.md`

Reviewed: the byte-exact XDR wire encoding of the config nvlist (request #3) - the 4-byte bootstrap header (encoding + endian + two reserved), the version+flags list header, the per-pair framing (encoded size, decoded size, XDR-string name, type code, element count, value), XDR primitive rules (4-byte big-endian widening, 8-byte hypers, string length+pad, opaque arrays, numeric/string arrays), nested NVLIST / NVLIST_ARRAY embedding, the full type-code table, and the two-zero terminator. Refines spec 01 section 3.1; the key catalog stays in spec 01 and is not duplicated.

Spot-checks against the pinned source (`9b7642df...`) confirmed as facts: on-disk config uses the XDR encoding (encoding byte 1); the XDR body is inherently big-endian and the endian tag byte is not consulted when decoding XDR (only the native encoding checks it); list header is version (0) + flags; each pair carries a 4-byte encoded size and 4-byte decoded size ahead of name/type/nelem/value; strings are length-prefixed and padded to 4; 64-bit values are 8-byte hypers; embedded lists repeat the version+flags header and terminator but not the bootstrap header; the list ends at a pair whose sizes are zero; type codes are a fixed ordered enumeration (classic set 1-20, appended set 21+).

Leak scan: clean. Scanned for source identifiers (type/struct/function/macro names, field names, the source's descriptive comment wording) - none present. Type names used (UINT64, STRING, NVLIST, etc.) are the neutral capability names shared with public XDR/nvlist documentation, not the source `DATA_TYPE_*` symbols; field descriptions are independently worded, not copied from the source layout comment. Referenced key strings (`vdev_tree`, `children`, `features_for_read`) are on-disk interface facts already approved in spec 01.

Outcome: APPROVED. Header status set DRAFT -> APPROVED.

---

## 2026-07-21 - `specs/format/04-block-pointer-dva.md`

Reviewed: the 128-byte block pointer and its DVAs (request #4) - the sixteen-word/bit map, the three DVAs (vdev id, allocated size, gang bit, vdev-relative offset in 512-byte sectors, the 4 MiB data-start skip and `(offset << 9) + 0x400000` device-offset formula), the property word bit fields (LSIZE/PSIZE with +1 sector bias, compression, embedded flag, checksum identity, DMU type, indirection level, crypt/dedup/byteorder), the birth-txg words (physical/logical split), fill count and 256-bit checksum, the byteorder-bit byteswap rule, the hole/gang/embedded variants (including the 112-byte embedded payload word set and the byte-unit embedded size encoding), and the checksum (7.1) and compression (7.2) identity code enumerations.

Spot-checks against the pinned source (`9b7642df...`) confirmed as facts: 128-byte pointer, 3 DVAs, 512-byte sector unit; DVA word 0 = ASIZE (bits 0..23, `raw<<9`, no bias), reserved bits 24..31, vdev id at bit 32 (24-bit active width, high 8 zero); DVA word 1 = 63-bit offset + gang bit 63; data area begins 4 MiB in (2 front labels + 3.5 MiB boot, matching spec 01); property word field offsets (LSIZE 0..15, PSIZE 16..31, comp 32..38, embedded 39, checksum 40..47, type 48..55, level 56..60, crypt 61, dedup 62, byteorder 63); LSIZE/PSIZE `(raw+1)<<9` with 32 MiB ceiling; byteorder polarity 0=BE/1=LE with swap-iff-differs; physical birth word 0 meaning "same as logical"; fill count and four checksum words; embedded payload = 14 words (all except property and logical-birth) = 112 bytes with byte-unit sizes and etype at bits 40..47 (0 DATA, 2 REDACTED); checksum identity codes 0..14 and compression identity codes 0..16 in their on-disk order.

Leak scan: clean. Scanned for source identifiers (struct/field names, `BP_*`/`DVA_*`/`BPE_*` macros, `BF64_*`, `ZIO_CHECKSUM_*`/`ZIO_COMPRESS_*` symbols, `SPA_*` constants, gang/label struct type names) - none present. Field and size abbreviations used (DVA, ASIZE, LSIZE, PSIZE, vdev, gang, fill count) are the neutral on-disk terms from the public 2006 spec; algorithm names (fletcher-2/4, SHA-256/512, Skein, Edon-R, BLAKE3, LZJB, gzip, ZLE, LZ4, zstd) are public algorithm names, not source symbols; the numeric identity codes are on-disk interface facts. Cross-references to specs 01 and 02 point at already-approved material and do not restate it.

Outcome: APPROVED. Header status set DRAFT -> APPROVED.
