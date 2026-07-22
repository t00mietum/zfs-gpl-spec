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

---

## 2026-07-22 - `specs/format/05-dmu-dnode-objset.md`

Reviewed: the DMU dnode and object set (request #5) - the 512-byte dnode slot and 16 KiB/32-slot dnode block, the 64-byte dnode core (per-field byte offsets: type/indblkshift/levels/nblkptr/bonustype/checksum/compress/flags/datablksec/bonuslen/extra-slots/pad/maxblkid/used/reserved), the flag byte bits, the block-pointer/bonus/spill tail region and the spill-pointer position, large multi-slot dnodes and interior slots, the object-set header (meta-dnode at 0, ZIL header at 512, type at 704, flags at 712, the two 32-byte MACs, the 1024/2048/4096 sizes and their size-based feature detection, the user/group/project-used accounting dnodes), object numbering via `N << 9` in the meta-dnode array with object 0 = meta-dnode and negative special accounting objects, the ZIL header field layout, object-set type and flag codes, the DMU object-type enumeration (classic table-indexed 0..53 plus the bit-encoded high-bit form with metadata/encrypted/byteswap sub-fields), the byteswap-class set, and the block-pointer indirection walk.

Spot-checks against the pinned source (`9b7642df...`) confirmed as facts: dnode core 64 bytes, slot 512 bytes, 32 dnodes per 16 KiB block; core field offsets 0/1/2/3/4/5/6/7/8/10/12/13/16/24/32 with sizes as tabled; flag bits 0 (used-in-bytes) / 2 (spill present); tail = nblkptr*128 pointers then bonus then optional trailing 128-byte spill; classic max 3 pointers / 320-byte bonus; large dnode = `(extra_slots+1)*512` up to 16384; object-set header offsets 0/512/704/712/720/752 and accounting dnodes at 1024/1536/2048 with V1/V2/V3 sizes 1024/2048/4096; ZIL header 192 bytes with the embedded log block pointer at header offset 16; object-set type codes 0..3; object-set flag bits 0..2; object number N at meta-dnode offset `N<<9`, object 0 = meta-dnode, special objects -1/-2/-3; DMU object-type list values 0..53 in the tabled order and the 0x80/0x40/0x20/0x1f bit-encoded form; ten byteswap classes in order.

Leak scan: clean. Scanned for source identifiers (struct/field names such as the `dn_*`/`os_*`/`zh_*` members, `dnode_phys`/`objset_phys`/`zil_header` type names, `DMU_OT_*`/`DMU_OST_*`/`DMU_BSWAP_*` enumerators, `DNODE_*`/`OBJSET_*`/`DN_*` macros) - none present. Field descriptions are independently worded; the object-type and byteswap-class names used (object directory, dnode, object set, ZAP, znode, ACL, etc.) are the neutral on-disk role names, and the type/flag values are on-disk interface facts. Cross-references to specs 01 and 04 point at already-approved material and do not restate it.

Outcome: APPROVED. Header status set DRAFT -> APPROVED.

---

## 2026-07-22 - `specs/format/06-zap.md`

Reviewed: the ZAP name-value store (request #6) - the three block-type discriminator words, the salted CRC-64 hash (ECMA-182 reflected polynomial, table generation, key-byte order, 28/48-bit truncation), the collision differentiator and its lowest-unused assignment, the micro ZAP (64-byte header with salt/normalization-flags, 64-byte entries, free-slot rule, full-scan lookup), the fat ZAP header block (field offsets 0..96, embedded pointer table in the block's second half, external table geometry, doubling state), leaf blocks (48-byte header, 2-byte leaf-hash-table entries, 24-byte chunks: entry/array/free with type codes 252/251/253), name/value encoding (values always big-endian, string keys stored with NUL, 21-byte array-chunk payloads, entries never spanning leaves), the lookup recipe, micro-to-fat upgrade conditions and preserved state, and byte-swap rules.

Spot-checks against the pinned source (`9b7642df...`) confirmed as facts: block-type words `0x8000000000000003` / `...01` / `...00`; header magic `0x2F52AB2AB`; leaf magic `0x2AB1EAF`; CRC-64 poly `0xC96C5795D7870F42` (reflected, table-driven, salt-seeded); hash truncation to the top 28 bits (48 with the wide-hash flag, which also caps CD at 65535); micro header/entry offsets (salt 8, normalization flags 16; entry value 0 / cd 8 / name 14, 50-byte name field, 49-char limit); fat header offsets 0/8/16/24/32/40/48/56/64/72/80/88/96 with the embedded table at the block midpoint and shift = bs-4; external table 2^(bs-3) entries per block; leaf header offsets (prefix 16, magic 24, counts 28/30, prefix-len 32, freelist 34, flags 36) and the CD-sorted flag bit 0; leaf hash table at 48 with 2^(bs-5) entries indexed by the bits below the prefix; chunk region formula ((2^bs - 2^(bs-4))/24 - 2) and 24-byte chunks; entry-chunk field offsets with the 32-bit CD at 12 (originally 16-bit + pad) and 64-bit hash at 16; array-chunk 21-byte payload with next at 22; 0xFFFF end-of-chain; big-endian value/integer-key bytes never swapped; upgrade conditions and salt/normalization/CD preservation.

Corrected during review (draft errors caught by re-verification before approval): (1) string-key hashing excludes the terminating NUL while the stored name and its length include it - the draft initially claimed the NUL was hashed; (2) the leaf entry CD is a 32-bit field at offset 12, not 16-bit + 2 pad (that is the original layout, widened in place); (3) the wide-hash flag keeps the top 48 bits, not all 64.

Leak scan: cleaned, then clean. The draft used several identifier-shaped labels mirroring source member names (salt/table/flag field spellings, int-length/count abbreviations); all were replaced with neutral English field labels. Final scan for source identifiers (`mz_*`/`zap_*`/`zt_*`/`lh_*`/`le_*`/`la_*`/`lf_*` members, `mzap_phys`/`zap_phys`/`zap_leaf_*` type names, `ZBT_*`/`ZAP_CHUNK_*`/`ZAP_*` macros and their abbreviations) - none present. Magic values, block-type words, chunk-type codes, the CRC polynomial, and flag bit positions are on-disk interface facts; algorithm naming (CRC-64, ECMA-182) is public. Cross-references to specs 04 and 05 point at already-approved material.

Outcome: APPROVED. Header status set DRAFT -> APPROVED.
