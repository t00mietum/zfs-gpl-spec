# Specification requests

The logged, one-way question channel from the implementation side. This file is the only inbound path across the clean-room wall; the outbound path is the approved specs themselves. Every request and its disposition lives permanently in this repo's history, which is the point: the question channel is evidence, not a risk.

## Rules

- A request states a format area or a functional question: "specify the fat ZAP hash and collision behavior", "what selects the active uberblock when txgs tie". Questions and functional needs only.
- A request never contains implementation code, proposed structure, or identifier names beyond on-disk interface facts already in approved specs.
- Answers never appear here. An answer is an approved spec (new or revised), referenced by the request's disposition line. Facts flow only through the gatekept pipeline.
- Dispositions: `open`, `spec'd -> <spec file> @ <commit>`, `declined - <reason>`.

## Log

| # | Date | Request | Disposition
| :-- | :-- | :-- | :--
| 1 | 2026-07-18 | Vdev label geometry, uberblock format, endianness detection, active-uberblock selection | spec'd -> `specs/format/01-vdev-label-uberblock.md` @ `7f86d1b`
| 2 | 2026-07-19 | Label/uberblock self-checksum: digest-to-word packing of the embedded SHA-256 (needed before claiming interop) | spec'd -> `specs/format/02-label-checksum-sha256-packing.md` @ `cf0592b`
| 3 | 2026-07-21 | Config nvlist: byte-exact XDR wire encoding (bootstrap header, pair framing, primitive/array/nested encodings, type codes, terminator) - needed to unpack the label config | spec'd -> `specs/format/03-config-nvlist-xdr-encoding.md` @ `6743872`
| 4 | 2026-07-21 | Block pointer and DVA on-disk layout: the 128-byte bp word/bit map, DVA encoding (vdev/offset/asize/gang, sector-shift and vdev-relative offset), the property word bit fields (LSIZE/PSIZE/comp/checksum/type/level/embedded/dedup/byteorder), birth/fill/checksum words, byteorder-bit byteswap rule, and the hole/gang/embedded variants; plus the checksum and compression identity code enumerations | spec'd -> `specs/format/04-block-pointer-dva.md` @ `ec34d1a`
| 5 | 2026-07-22 | DMU dnode and object-set layout: the 64-byte dnode core (field offsets, block geometry, flags), the block-pointer/bonus/spill tail region and large-dnode slots, the object-set header (meta-dnode, ZIL header, type/flags, MACs, accounting dnodes, 1024/2048/4096 sizes), object numbering via the meta-dnode array, and the DMU object-type enumeration selected by the dnode/bp type byte | spec'd -> `specs/format/05-dmu-dnode-objset.md` @ `8d04e97`
| 6 | 2026-07-22 | ZAP object format: micro and fat block layouts and their discrimination, the hash (salt, algorithm, truncation) and collision differentiator, the pointer table (embedded and external), leaf structure (header, hash table, entry/array/free chunks), name/value encoding and byte order, and micro-to-fat upgrade conditions - needed to read the MOS object directory and every name-keyed object | spec'd -> `specs/format/06-zap.md` @ `8543205`
| 7 | 2026-07-22 | DSL dataset and snapshot on-disk structures: the DSL directory and dataset records (field offsets, sizes, flags), namespace wiring from the MOS object directory through child maps to heads and snapshots, snapshot chain pointers and head/snapshot discrimination, clone/origin linkage, property ZAP encoding and inheritance, and the deadlist/block-pointer-object formats - needed to enumerate datasets and reach a filesystem's object set | spec'd -> `specs/format/07-dsl-dataset-snapshot.md` @ `63e31f0`
