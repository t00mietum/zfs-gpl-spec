# ZFS on-disk format: label/uberblock self-checksum - SHA-256 digest-to-word packing

**Provenance**

- OpenZFS commit studied for behavioral corroboration: `9b7642df` (reference only).
- Status: **APPROVED** 2026-07-20 (gatekeeper review complete). Authoritative.
- Each fact below is tagged with a provenance marker:
	- **[2006]** - corroborated by the public 2006 Sun "ZFS On-Disk Specification" (the durable core, independently published).
	- **[modern]** - source-only; a post-2006 addition or refinement observed in current OpenZFS. Required to interoperate with pools written by recent OpenZFS, but not part of the original published spec.

This document refines section 6 of `01-vdev-label-uberblock.md`. It specifies exactly how the 256-bit SHA-256 digest that self-checksums a label structure (nvlist area, boot-env block, or uberblock slot) is packed into the four 64-bit words of the embedded checksum trailer, so an implementer can produce and verify a byte-exact match. It answers request #2: the digest-to-word packing needed before claiming interop.

The surrounding constructs - the 40-byte trailer, the trailer magic, the offset-anchored `{off,0,0,0}` verifier injection, and the SHA-256-over-the-whole-area computation - are specified in `01-vdev-label-uberblock.md` section 6 and are not restated here. This document covers only the mapping from the SHA-256 output to the four stored words, and the byte order of those words on disk.

---

## 1. The checksum field as four 64-bit words

The trailer's 32-byte checksum field (trailer offset 8, immediately after the 8-byte trailer magic) is treated as **four consecutive 64-bit words**, indexed 0..3 in increasing address order. The 256-bit checksum value occupies these four words. **[2006]**

The label/uberblock self-checksum algorithm is **SHA-256**. **[2006]** SHA-256 produces a 256-bit result conventionally expressed, per FIPS 180, as eight 32-bit hash words H0..H7, and as a 32-byte big-endian digest string in which H0 is emitted first (most significant) and H7 last. This document uses D[0..31] for that standard 32-byte digest string and H0..H7 for the eight 32-bit words, where D[4k..4k+3] is the big-endian encoding of H(k).

---

## 2. Digest-to-word mapping (the core fact)

The four stored 64-bit words carry the digest in **big-endian pairing**: each stored word is the big-endian 64-bit integer formed from two consecutive 32-bit hash words, high word first. **[2006]** for SHA-256 into the four words; **[modern]** for the confirmed exact pairing/orientation below.

Expressed as integer values (independent of the reader's host byte order):

```
word[0] = (H0 << 32) | H1
word[1] = (H2 << 32) | H3
word[2] = (H4 << 32) | H5
word[3] = (H6 << 32) | H7
```

Equivalently, in byte terms: word[j] (as a 64-bit integer value) equals the big-endian interpretation of the eight digest bytes D[8j .. 8j+7]. The eight digest bytes of a word therefore run high-order first: D[8j] is the most significant byte of word[j]'s value, D[8j+7] the least.

Note the pairing granularity: the digest's eight 32-bit words are grouped into four 64-bit words **two at a time** with the earlier 32-bit word in the high half. A reader must not swap the two halves of a 64-bit word, and must not reorder the eight 32-bit words.

---

## 3. Host-endianness independence of the value

The four word **values** in section 2 are the same regardless of the byte order of the host that wrote the structure. **[modern]** The producer forces the big-endian orientation of the digest into the word values so that the on-disk checksum *value* is identical across writer architectures; there is no separate little-endian variant of the label/uberblock checksum value. This is the property that lets a reader compute the expected value the same way for every pool. **[modern]**

Interaction with the section-7 endianness detection of `01-vdev-label-uberblock.md`:

- The four words are stored to the device in the **native byte order of the writing host**, exactly like every other fixed field in the label (the trailer magic records that order). **[2006]**
- Therefore the value packing of section 2 and the on-disk byte order are two separate layers. Section 2 fixes the word *values*; the writer's native order then determines the *bytes* those values occupy on disk.
- A reader detects byte order from the trailer magic (`01` section 7). If the trailer magic reads byte-swapped, the reader byte-swaps each of the four 64-bit checksum words (and the injected offset verifier) back to native before comparing. After that correction the reader holds the section-2 word values on any host. **[2006]** for native-order storage + magic-based correction; **[modern]** for the confirmation that the corrected values match section 2 on both architectures.

Concretely, the four checksum bytes on disk are:

- **Big-endian writer:** the checksum field is the raw 32-byte SHA-256 digest D[0..31] in order (D[0] at the lowest checksum-field address). **[modern]**
- **Little-endian writer:** the checksum field is D grouped in 8-byte runs with each run reversed - bytes D[7],D[6],...,D[0], then D[15],...,D[8], then D[23],...,D[16], then D[31],...,D[24]. **[modern]**

Both encode the same four word values of section 2; the trailer magic tells the reader which case it is looking at.

---

## 4. Reader verification recipe (packing layer)

Given a checksummed area already located per `01` section 6, to check the stored checksum against a freshly computed SHA-256:

1. Compute the area's SHA-256 (with the `{off,0,0,0}` verifier injected, per `01` section 6.3). Obtain the standard 32-byte digest D[0..31] / hash words H0..H7.
2. Form the four expected 64-bit word values by the section-2 mapping: `expected[j] = (H(2j) << 32) | H(2j+1)`.
3. Read the four stored 64-bit words from the trailer's checksum field. If the trailer magic (`01` section 7) indicates byte-swapped order, byte-swap each of the four words.
4. The area is valid (at the checksum layer) iff `stored[j] == expected[j]` for all j in 0..3. **[2006]**

There is no truncation or masking: all four words participate in the comparison for the label/uberblock (SHA-256) checksum. **[2006]**

---

## 5. Worked example (packing test vectors)

These vectors isolate the digest-to-word packing of section 2. They use the standard FIPS 180 SHA-256 of a raw input and do **not** include the offset-verifier construction of `01` section 6.3; they exist so an implementer can confirm the mapping alone before wiring in the full label verify. The digests are the published SHA-256 standard test vectors, not ZFS-specific facts.

### 5.1 Empty input

`SHA-256("")` = `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`

Packed into the four words by section 2:

```
word[0] = 0xe3b0c44298fc1c14
word[1] = 0x9afbf4c8996fb924
word[2] = 0x27ae41e4649b934c
word[3] = 0xa495991b7852b855
```

On-disk checksum-field bytes for these values:

- Big-endian writer: `e3 b0 c4 42 98 fc 1c 14  9a fb f4 c8 99 6f b9 24  27 ae 41 e4 64 9b 93 4c  a4 95 99 1b 78 52 b8 55` (the raw digest in order).
- Little-endian writer: `14 1c fc 98 42 c4 b0 e3  24 b9 6f 99 c8 f4 fb 9a  4c 93 9b 64 e4 41 ae 27  55 b8 52 78 1b 99 95 a4` (each 8-byte word reversed).

### 5.2 Input "abc"

`SHA-256("abc")` = `ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad`

Packed into the four words by section 2:

```
word[0] = 0xba7816bf8f01cfea
word[1] = 0x414140de5dae2223
word[2] = 0xb00361a396177a9c
word[3] = 0xb410ff61f20015ad
```

A conforming implementation that packs an arbitrary 32-byte SHA-256 digest into the four words per section 2 must reproduce these word values.

---

## 6. Constants and rules quick reference

| Item | Value / rule | Provenance |
|------|--------------|------------|
| Checksum algorithm | SHA-256 (256-bit) | **[2006]** |
| Stored form | four 64-bit words, checksum-field offset 8..39 of the 40-byte trailer | **[2006]** |
| Word value packing | `word[j] = (H(2j) << 32) \| H(2j+1)`, i.e. big-endian 64-bit of digest bytes `D[8j..8j+7]` | **[2006]** core / **[modern]** exact orientation |
| Pairing granularity | eight 32-bit hash words grouped two-at-a-time, earlier word in the high 32 bits; halves never swapped | **[modern]** |
| Value vs. architecture | word *values* identical for LE and BE writers (big-endian orientation forced); no separate LE checksum value | **[modern]** |
| On-disk byte order | words stored in writer-native order; reader corrects via trailer magic (`01` section 7) | **[2006]** |
| Comparison | all four words compared; no truncation/masking | **[2006]** |
