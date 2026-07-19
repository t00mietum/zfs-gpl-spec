# ZFS on-disk format: VDEV label and uberblock

**Provenance**

- OpenZFS commit studied for behavioral corroboration: `9b7642df` (reference only).
- Status: **APPROVED** 2026-07-19 (gatekeeper review complete). Authoritative.
- Each fact below is tagged with a provenance marker:
	- **[2006]** - corroborated by the public 2006 Sun "ZFS On-Disk Specification" (the durable core, independently published).
	- **[modern]** - source-only; a post-2006 addition or refinement observed in current OpenZFS. Implementers should treat these as required for interoperating with pools written by recent OpenZFS, but they are not part of the original published spec.

This document describes the format an implementer must match to read the VDEV label and locate/validate the active uberblock. It states requirements, not any particular program's control flow.

---

## 1. Scope and reading model

To open a pool for reading, an implementer must:

1. Locate and read the VDEV label(s) on each leaf device.
2. Parse the label's name-value (nvlist) area to learn pool identity and the vdev tree.
3. Scan the uberblock array in the label(s), validate each candidate, and select the active uberblock (the entry point to the rest of the pool).

All multi-byte integers in these structures are stored in the **native byte order of the host that wrote them**; readers detect and correct byte order from magic values (see sections 5 and 7). All fixed structures described here are self-checksummed (see section 6).

---

## 2. The VDEV label

### 2.1 Size and count

- Each VDEV label is exactly **256 KiB** (262144 bytes). **[2006]**
- Every leaf (physical) device carries **four** identical-format labels, named L0, L1, L2, L3. **[2006]** They are written redundantly; the four are not required to be identical in content at any instant (an in-progress update may leave copies at different transaction groups), which is precisely why there are four.

### 2.2 Positions on the device

Given a device of usable size `psize` bytes (a multiple of 256 KiB; any trailing partial-label remainder is not used):

- **L0** at byte offset `0`.
- **L1** at byte offset `256 KiB` (`0x40000`).
- **L2** at byte offset `psize - 2*256KiB` (the second-to-last 256 KiB).
- **L3** at byte offset `psize - 256KiB` (the last 256 KiB).

So L0/L1 sit at the very front of the device and L2/L3 at the very end. **[2006]**

General placement rule for label index `l` (0..3), for a position `off` within a label:

```
device_offset(l, off) = off + l*256KiB                       for l in {0,1}   (front pair)
device_offset(l, off) = off + l*256KiB + (psize - 4*256KiB)   for l in {2,3}   (rear pair)
```

The rear-pair term places L2 and L3 in the last two label-sized slots of the device. **[modern]** (mechanical restatement of the front/rear split, which is **[2006]**.)

### 2.3 Reserved boot region between the front and rear label pairs

Immediately after the two front labels there is a boot region reserved on the device: **[2006]**

- Boot region device offset: `2 * 256KiB` = `512 KiB` (`0x80000`), i.e. right after L0+L1.
- Boot region size: **3.5 MiB** (`7 << 19` = 3670016 bytes). **[modern]** (The existence of a reserved boot area between the front labels and the pool data is **[2006]**; the exact 3.5 MiB size is the current value.)

Consequences for a reader computing where pool data may live:

- Reserved space at the **start** of a leaf device = two labels + boot region = `2*256KiB + 3.5MiB`.
- Reserved space at the **end** of a leaf device = two labels = `2*256KiB`.

Any byte offset that falls inside the front reserved region or the rear two-label region belongs to label/boot metadata, not to allocatable pool data. **[2006]**

### 2.4 Internal regions of one 256 KiB label

Each label is divided into four consecutive regions, in this order: **[2006]**

| Order | Region | Size | Offset within label | Role |
|-------|--------|------|---------------------|------|
| 1 | Blank / VTOC padding | 8 KiB (`0x2000`) | `0x00000` | Left blank so a legacy partition/VTOC disk label can coexist; ignored by ZFS. **[2006]** |
| 2 | Boot header / boot-env block | 8 KiB (`0x2000`) | `0x02000` | Reserved 8 KiB. **[2006]** In current OpenZFS this holds a boot-environment block (see 2.6); the 2006 spec named it the "boot block header" and left it unused. **[modern]** for the boot-env repurposing. |
| 3 | Name-value pair area (nvlist) | 112 KiB (`0x1C000`) | `0x04000` | XDR-packed nvlist describing pool + vdev config (section 3). **[2006]** |
| 4 | Uberblock array | 128 KiB (`0x20000`) | `0x20000` | Ring of uberblock slots (section 4). **[2006]** |

Total: 8 + 8 + 112 + 128 = 256 KiB. **[2006]**

### 2.5 Name-value pair area layout

The 112 KiB region is one structure: **[2006]**

- Bytes `0 .. (112KiB - 40)` : the XDR-packed nvlist, zero-padded after the packed data to fill the region.
- Last **40 bytes** of the region: an embedded checksum trailer (section 6). The nvlist itself therefore has `112KiB - 40` = 114648 usable bytes.

### 2.6 Boot-env block layout (region 2) **[modern]**

The 8 KiB boot-env region, when used, is laid out as:

- Offset 0, 8 bytes: a version tag. Known values: `0` = raw ASCII payload (e.g. a GRUB environment blob), `1` = the payload is itself an XDR-packed nvlist.
- Then the payload bytes, filling up to the trailer.
- Last **40 bytes**: an embedded checksum trailer (section 6), covering this 8 KiB block.

A reader that only needs pool data may ignore this region. Its existence and 8 KiB size are **[2006]**; the version-tagged content format is **[modern]**.

---

## 3. The label config nvlist

### 3.1 Encoding

The nvlist in region 3 is serialized with ZFS's XDR nvlist encoding (the "XDR" nvlist format, not the "native" host-endian nvlist format). A reader must decode XDR-encoded name/value pairs. **[2006]**

### 3.2 Key strings (the interface)

The following are the exact on-disk key strings a reader must match. These strings are the interoperability contract; reproduce them byte-for-byte. Values are unsigned 64-bit integers unless noted.

**Pool-wide, top-level keys** (present in every leaf device's label): **[2006]** unless marked.

| Key string | Type | Meaning |
|------------|------|---------|
| `version` | uint64 | ZFS on-disk pool version number. For feature-flag pools this is a sentinel large value (see 3.4). **[2006]** |
| `name` | string | Pool name. **[2006]** |
| `state` | uint64 | Pool state (section 3.5). **[2006]** |
| `txg` | uint64 | Transaction group in which this label was written. **[2006]** |
| `pool_guid` | uint64 | 64-bit unique identifier of the pool. **[2006]** |
| `top_guid` | uint64 | GUID of the top-level vdev that contains this leaf. **[2006]** |
| `guid` | uint64 | GUID of this leaf vdev. **[2006]** |
| `vdev_tree` | nvlist | Nested nvlist describing the vdev subtree rooted at this device's top-level vdev (section 3.3). **[2006]** |
| `hostid` | uint64 | Numeric host id of the last writer. **[2006]** |
| `hostname` | string | Hostname of the last writer. **[2006]** |
| `features_for_read` | nvlist | Set of feature names that must be understood merely to read pool metadata (MOS). Each member is a feature name key (e.g. `com.delphix:...`) with a boolean/int value. A reader that does not implement a listed read feature must not proceed. **[modern]** |
| `comment` | string | Optional free-text pool comment. **[modern]** |
| `compatibility` | string | Optional compatibility-set name(s). **[modern]** |
| `create_txg` | uint64 | txg in which this vdev/pool object was created. **[modern]** |
| `errata` | uint64 | Errata identifier the pool is subject to (not on disk in older pools). **[modern]** |

**Auxiliary top-level members** (present on the top of the tree when applicable):

| Key string | Type | Meaning |
|------------|------|---------|
| `vdev_children` | uint64 | Number of top-level vdevs in the whole pool. **[modern]** |
| `spares` | nvlist array | Hot-spare vdev configs. **[2006]** |
| `l2cache` | nvlist array | L2ARC cache vdev configs. **[modern]** |
| `hole_array` / `is_hole` | uint64 array / uint64 | Records "hole" (removed/absent) top-level vdev slots. **[modern]** |
| `split` (`splitcfg`) | nvlist | Config used when splitting a mirrored pool; members `orig_guid`, `split_guid`, `guid_list`. **[modern]** |

### 3.3 vdev_tree nvlist keys

The `vdev_tree` nvlist (and each nested child) uses these keys:

| Key string | Type | Meaning |
|------------|------|---------|
| `type` | string | Vdev type token (section 3.6). **[2006]** |
| `id` | uint64 | Index of this vdev among its siblings. **[2006]** |
| `guid` | uint64 | This vdev's GUID. **[2006]** |
| `path` | string | Device path (leaf vdevs). **[2006]** |
| `devid` | string | Stable device id (leaf vdevs). **[2006]** |
| `phys_path` | string | Physical device path. **[modern]** |
| `vdev_enc_sysfs_path` | string | Enclosure sysfs path. **[modern]** |
| `fru` | string | Physical FRU location. **[modern]** |
| `whole_disk` | uint64 | 1 if ZFS owns the entire disk. **[2006]** |
| `metaslab_array` | uint64 | Object number of the metaslab array (space map index) for this top-level vdev. **[2006]** |
| `metaslab_shift` | uint64 | log2 of metaslab size. **[2006]** |
| `ashift` | uint64 | log2 of this vdev's minimum allocatable/sector size (allocation shift). **[2006]** |
| `asize` | uint64 | Allocatable size of this top-level vdev in bytes. **[2006]** |
| `is_log` | uint64 | 1 if this is a separate intent-log (SLOG) vdev. **[2006]** |
| `nparity` | uint64 | RAID-Z parity count (1/2/3), present for raidz/draid. **[2006]** |
| `children` | nvlist array | Child vdev nvlists (interior vdevs: mirror, raidz, etc.). **[2006]** |
| `DTL` | uint64 | Object number of this leaf's dirty-time-log space map. **[2006]** |
| `create_txg` | uint64 | txg the vdev was added. **[modern]** |
| `offline` / `faulted` / `degraded` / `removed` | uint64 | Persistent per-vdev fault states, stored separately because a vdev can hold several at once. **[modern]** |
| `resilver_txg` / `rebuild_txg` | uint64 | Persistent resilver/rebuild progress markers. **[modern]** |
| `not_present` | uint64 | Marked absent at last import. **[modern]** |
| `is_spare` | uint64 | 1 if a spare. **[2006]** |
| `nonallocating` (`non_allocating`) | uint64 | Top-level vdev currently not accepting new allocations. **[modern]** |
| `removing` | uint64 | Device removal in progress. **[modern]** |
| draid geometry keys | uint64 | dRAID layout parameters accompany a `type` of `draid`/`dspare`. **[modern]** |

Namespaced (vendor-prefixed) keys carried in the tree, all **[modern]**:

- `com.delphix:vdev_zap_top`, `com.delphix:vdev_zap_leaf`, `com.klarasystems:vdev_zap_root` - object numbers of per-vdev ZAP objects.
- `com.delphix:has_per_vdev_zaps` - boolean marker that per-vdev ZAPs exist.
- `com.delphix:indirect_object`, `com.delphix:indirect_births`, `com.delphix:prev_indirect_vdev` - indirect (post-removal) vdev mapping references.

Interpretation rule for readers: the presence of `nparity` plus `type == "raidz"`/`"draid"` selects RAID-Z/dRAID reconstruction; `ashift` governs on-disk offset alignment; `asize` bounds the allocatable region; `metaslab_array`/`metaslab_shift` locate the space maps. A reader that only walks metadata via the uberblock's root pointer still needs `type`, `ashift`, `guid`, and the tree shape (`children`) to map DVA (vdev,offset) addresses to physical device offsets. **[2006]**

### 3.4 The `version` value and feature flags

- A plain integer `version` (1..28 historically) selects a legacy on-disk version. **[2006]**
- A pool using feature flags stores `version` = the sentinel `5000`, and advertises capabilities through feature-name keys (notably inside `features_for_read`) rather than a monotonic number. A reader must recognize the sentinel and then gate on the feature set. **[modern]**

### 3.5 Pool `state` values

Enumerated integer values: **[2006]**

- `0` = active (in use).
- `1` = exported.
- `2` = destroyed.
- `3` = reserved as spare. **[modern]**
- `4` = L2ARC cache device. **[modern]**

### 3.6 `type` token strings

Exact strings for the `type` key: **[2006]** unless marked -
`root`, `mirror`, `replacing`, `raidz`, `disk`, `file`, `missing`, `spare`, `log`, `l2cache`, and **[modern]**: `draid`, `dspare`, `hole`, `indirect`.

---

## 4. The uberblock array (label region 4)

- Region size: **128 KiB** (`0x20000`), located at offset `0x20000` within each label. **[2006]**
- The region is a ring of fixed-size **uberblock slots**. Each transaction group's uberblock is written to the next slot in rotation, so the array holds a history of recent uberblocks. **[2006]**
- **Slot size:**
	- 2006 baseline: **1 KiB** per slot, giving **128** slots. **[2006]**
	- Modern refinement: slot size = `2^shift` where `shift = clamp(top-level vdev ashift, 10, 13)`. So slot size ranges from 1 KiB (ashift <= 10) up to 8 KiB (ashift >= 13), and slot count = `128KiB / slot_size` (128 down to 16). A reader must derive the slot size from the top-level vdev's `ashift` to index the array correctly. **[modern]**
- Slot `n` (0-based) begins at label offset `0x20000 + n * slot_size`. **[modern]** derivation; the array's location is **[2006]**.
- When multi-host protection (MMP) is enabled, the **last** slot of each label's ring is reserved for MMP heartbeat writes. **[modern]**

An uberblock occupies the front of its slot; the remainder of the slot is padding, except the final 40 bytes which are the embedded checksum trailer covering the whole slot (section 6). **[2006]** for the self-checksummed-uberblock concept; the trailer-at-end-of-slot placement is **[modern]** in its exact geometry.

---

## 5. Uberblock on-disk layout

The uberblock is a packed sequence of 64-bit fields. All offsets are relative to the start of the uberblock (the start of its slot). The first two fields must never move: a reader identifies and version-checks an uberblock from them before anything else. **[2006]**

| Offset | Size | Field role | Provenance |
|--------|------|-----------|------------|
| 0 | 8 | **Magic** = `0x00bab10c`. Identifies an uberblock and encodes byte order (section 7). | **[2006]** |
| 8 | 8 | **Version** - on-disk SPA version this uberblock conforms to (mirrors the config `version`; sentinel `5000` for feature-flag pools). | **[2006]** |
| 16 | 8 | **txg** - transaction group number of this uberblock (the sync that wrote it). | **[2006]** |
| 24 | 8 | **guid_sum** - arithmetic sum (mod 2^64) of all vdev GUIDs in the pool. Used as a consistency check that every expected device is present. | **[2006]** |
| 32 | 8 | **timestamp** - UTC wall-clock seconds at the time this uberblock was written. | **[2006]** |
| 40 | 128 | **Root block pointer (rootbp)** - a 128-byte block pointer addressing the Meta-Object-Set (MOS) objset. This is the entry point to all pool metadata. Its internal layout is specified separately (block-pointer spec). | **[2006]** |
| 168 | 8 | **software_version** - highest SPA version supported by the software that wrote this txg (informational; distinct from the on-disk `version` above). | **[modern]** |
| 176 | 8 | **mmp_magic** = `0xa11cea11`. When present and valid, signals MMP metadata follows. May be absent (zero) in uberblocks written by very old software, but is always written by current software. | **[modern]** |
| 184 | 8 | **mmp_delay** - nanoseconds since the last MMP write. Zero (with a valid mmp_magic) means MMP is off. | **[modern]** |
| 192 | 8 | **mmp_config** - packed MMP interval / fail-intervals / sequence / valid-bits (section 6.4). | **[modern]** |
| 200 | 8 | **checkpoint_txg** - nonzero iff this uberblock is a pool checkpoint; if nonzero, holds the txg the uberblock had when it was checkpointed. Zero = not a checkpoint. | **[modern]** |
| 208 | 8 | **raidz_reflow_info** - packed RAID-Z expansion reflow state+offset (section 5.1). | **[modern]** |

Fixed portion size: **216 bytes**. The rest of the slot up to `slot_size - 40` is padding; the final 40 bytes are the checksum trailer. **[modern]** for the exact field tail; the six 2006 fields (magic, version, txg, guid_sum, timestamp, rootbp) are **[2006]**.

Reader note: because later fields were appended over time and never reordered, a reader can parse older uberblocks by reading only the leading fields it recognizes; absent trailing fields read as their surrounding padding/zero. A conforming reader must not assume any field past `rootbp` is meaningful unless the pool/software version and the relevant magic (e.g. `mmp_magic`) indicate it.

### 5.1 raidz_reflow_info packing **[modern]**

A single 64-bit word:

- Bits 0..54: reflow byte offset, stored shifted right by 9 (the minimum block shift). The stored value is `offset >> 9`; multiply by 512 to recover the byte offset.
- Bits 55..63 (9 bits): reflow scratch state, an enumerated value (0 = scratch not in use; other values denote valid / invalid-synced / invalid-synced-on-import / invalid-synced-reflow states).

A reader that does not support RAID-Z expansion may ignore this word unless it must interpret an in-progress expansion.

---

## 6. Self-checksum scheme for label structures and uberblocks

Every fixed label structure - the nvlist area (region 3), the boot-env block (region 2), and each uberblock slot (region 4) - carries an **embedded self-checksum trailer** in its final 40 bytes. This is the "label" checksum scheme. **[2006]**

### 6.1 Trailer layout (40 bytes, at the end of the checksummed area)

| Offset in trailer | Size | Field | Meaning |
|-------------------|------|-------|---------|
| 0 | 8 | trailer magic = `0x0210da7ab10c7a11` | Validates the trailer and encodes byte order (section 7). **[2006]** |
| 8 | 32 | checksum (four 64-bit words) | 256-bit checksum value. **[2006]** |

### 6.2 Checksum algorithm

The checksum function is **SHA-256**, producing the 256-bit value stored in the four words. **[2006]**

### 6.3 The "self" (offset-anchored) checksum construction

The label checksum is anchored to the block's own device location so a stale copy read from the wrong place cannot validate. To **verify** a checksummed area (nvlist region, boot-env block, or uberblock slot):

1. Note `off` = the absolute byte offset on the device where this area begins.
2. Read the trailer magic at `area_size - 40`; determine byte order (section 7).
3. Build a 32-byte "verifier" value = the four 64-bit words `{ off, 0, 0, 0 }` (with `off` byte-swapped to match if the area is byte-swapped).
4. Temporarily place this verifier into the trailer's 32-byte checksum field (in a working copy), leaving the trailer magic in place.
5. Compute SHA-256 over the **entire area** (all bytes from offset 0 through `area_size`, i.e. including the trailer magic and the injected verifier).
6. Compare the result against the checksum value that was actually stored in the trailer (byte-swapped for comparison if the area is byte-swapped). Equal = valid.

To **write**, the producer performs the mirror operation: sets the trailer magic, injects the same `{off,0,0,0}` verifier, hashes the whole area, then stores the resulting hash into the trailer's checksum field. **[2006]** for the offset-anchored SHA-256-over-the-block construction; the exact 4-word verifier encoding `{off,0,0,0}` is confirmed **[modern]** but consistent with the 2006 description.

Rationale a reader can rely on: the checksum is not stored over the pool's transaction pointer; it depends only on the block contents plus its physical offset. This is what lets a reader validate any uberblock slot without first knowing which txg it holds.

### 6.4 mmp_config bit packing **[modern]**

The 64-bit `mmp_config` word (uberblock offset 192) packs:

- Bits 0..7 - valid-bit mask: `0x01` = write-interval present, `0x02` = sequence number present, `0x04` = fail-intervals present, `0xf8` reserved.
- Bits 8..31 (24 bits) - write interval in milliseconds.
- Bits 32..47 (16 bits) - sequence number (sub-second write ordering).
- Bits 48..63 (16 bits) - fail intervals.

A field is only meaningful if both `mmp_magic` is valid and the corresponding valid bit is set. A read-only importer uses these to detect that another host may have the pool active.

---

## 7. Endianness detection

ZFS writes these structures in the host's native byte order and detects order on read from the known magic constants. **[2006]**

- **Uberblock:** read the 8-byte magic at offset 0.
	- If it equals `0x00bab10c` (native read), the uberblock's 64-bit fields are in the reader's byte order; no swap.
	- If it equals the byte-swapped form `0x0cb1ba0000000000`, every 64-bit field in the uberblock must be byte-swapped before use.
	- If it matches neither, the slot does not hold a valid uberblock. **[2006]**
- **Checksum trailer / label areas:** read the 8-byte trailer magic (`0x0210da7ab10c7a11`). If it reads byte-swapped, the stored checksum words and the injected offset verifier must be byte-swapped so the SHA-256 comparison is done in a consistent order (section 6.3). **[2006]**
- **nvlist area:** the XDR nvlist encoding is self-describing and endian-defined by the XDR format itself, so the packed nvlist is portable regardless of writer endianness. **[2006]**

---

## 8. Selecting the active uberblock

To find the pool's entry point, a reader scans the uberblock array across the available label copies and picks the "best" valid uberblock. **[2006]**

### 8.1 Validity gate

An uberblock slot is a **candidate** only if all of these hold:

1. The device read of the slot succeeded.
2. Magic at offset 0 is `0x00bab10c` (after applying byte-order detection from section 7).
3. The slot's embedded SHA-256 self-checksum verifies against the slot's own device offset (section 6.3).

Slots failing any of these are ignored. **[2006]**

### 8.2 Ranking - highest valid txg wins

Among all candidates, the active uberblock is the one with the **highest `txg`**. **[2006]** This is the most recently committed transaction group whose uberblock is intact.

### 8.3 Tie-breakers **[modern]**

When two candidates share the same `txg` (possible after a power loss with a briefly-offline mirror replica), break the tie in order:

1. Higher `timestamp` wins.
2. If still tied and both carry valid MMP magic with a valid sequence field, higher MMP sequence number wins.
3. Otherwise either is acceptable.

### 8.4 Notes for a read-only implementer

- Scanning all four labels (and all slots) increases the chance of finding a good, most-recent uberblock even if some labels are damaged. A single valid highest-txg uberblock from any label is sufficient to proceed.
- The config nvlist to pair with the chosen uberblock should be taken from a label on the same device where that uberblock was found, so the vdev tree and the uberblock agree. **[modern]**
- Optional bounded rewind: a reader may deliberately cap the accepted txg (accept the highest valid txg not exceeding a chosen limit) to open an older-but-consistent state. This is a policy choice, not a format requirement. **[modern]**
- A checkpointed uberblock (nonzero `checkpoint_txg`, section 5) represents a saved earlier state; ordinary read-open uses the normal highest-txg selection and does not require checkpoint handling. **[modern]**

---

## 9. Constants quick reference

| Name | Value | Provenance |
|------|-------|------------|
| Label size | 256 KiB | **[2006]** |
| Labels per device | 4 (L0,L1 front; L2,L3 rear) | **[2006]** |
| Blank/VTOC pad | 8 KiB @ label off 0 | **[2006]** |
| Boot header / boot-env | 8 KiB @ label off 0x2000 | **[2006]** (boot-env content **[modern]**) |
| nvlist area | 112 KiB @ label off 0x4000 | **[2006]** |
| Uberblock ring | 128 KiB @ label off 0x20000 | **[2006]** |
| Uberblock slot size | 1 KiB (2006); 2^clamp(ashift,10,13), max 8 KiB (modern) | **[2006]** / **[modern]** |
| Reserved boot region (between front labels and data) | 3.5 MiB @ device off 0x80000 | **[2006]** (3.5 MiB size **[modern]**) |
| Uberblock magic | `0x00bab10c` | **[2006]** |
| Byte-swapped uberblock magic | `0x0cb1ba0000000000` | **[2006]** |
| MMP magic | `0xa11cea11` | **[modern]** |
| Checksum trailer magic | `0x0210da7ab10c7a11` | **[2006]** |
| Checksum algorithm (label/uberblock) | SHA-256, offset-anchored self-checksum | **[2006]** |
| Checksum trailer size | 40 bytes (8 magic + 32 checksum) | **[2006]** |
| Root block pointer size | 128 bytes | **[2006]** |
| nvlist encoding | XDR nvlist format | **[2006]** |
