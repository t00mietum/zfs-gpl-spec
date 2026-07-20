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
