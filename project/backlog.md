<!-- markdownlint-disable MD007 -- Unordered list indentation -->
<!-- markdownlint-disable MD010 -- No hard tabs -->
<!-- markdownlint-disable MD033 -- No inline html -->
<!-- markdownlint-disable MD055 -- Table pipe style [Expected: leading_and_trailing; Actual: leading_only; Missing trailing pipe] -->
<!-- markdownlint-disable MD041 -- This is a spec-repo backlog -->
# Requirements

Spec-coverage backlog. Each item is a spec to draft, gatekeep, and approve. Specs are written in the order the [`zfs-gpl`](https://github.com/t00mietum/zfs-gpl) implementation needs them. Companion: [`design.md`](design.md).

## Conventions

| Icon | Status
| :--: | :--
| 🔘   | Not started
| 🛠️   | Started, and/or partially complete
| ✋   | Defer
| ✅   | Complete
| 🚫   | Canceled

## Backlog

### Durable core (2006-corroborated)

- ✅ VDEV label and uberblock (`specs/format/01-vdev-label-uberblock.md`, approved `7f86d1b`)
- ✅ Label/uberblock self-checksum: digest-to-word packing detail needed for byte-exact interop (`specs/format/02-label-checksum-sha256-packing.md`)
- ✅ Config nvlist: XDR wire encoding (`specs/format/03-config-nvlist-xdr-encoding.md`; key catalog already in spec 01 §3)
- ✅ Block pointers and DVAs: layout, checksum and compression identities, endianness (`specs/format/04-block-pointer-dva.md`, approved `req4`)
- ✅ DMU: dnode and object-set layout (`specs/format/05-dmu-dnode-objset.md`, approved `req5`)
- ✅ ZAP: micro and fat (`specs/format/06-zap.md`)
- ✅ DSL: dataset and snapshot on-disk structures (`specs/format/07-dsl-dataset-snapshot.md`)
- 🔘 ZPL: directory and file layout, attributes

### Modern features (post-2006, source-only)

- 🔘 Feature flags: detection and gating
- 🔘 Compression identities: lz4, zstd, gzip
- 🔘 Checksum identities beyond the core: edonr, blake3, skein
- 🔘 Native encryption format
- 🔘 dRAID and raidz geometry
- 🔘 Log spacemaps, large dnodes, large/embedded blocks

### Done

- ✅ VDEV label and uberblock spec (first spec through the pipeline; one leaked macro name stripped in review)
- ✅ Label/uberblock SHA-256 digest-to-word packing spec (request #2; leak scan clean)
- ✅ Config nvlist XDR wire-encoding spec (request #3; leak scan clean - neutral type names only, no source identifiers)
- ✅ Block pointer / DVA layout spec (request #4; leak scan clean - neutral field terms + public algorithm names + numeric identity codes, no source symbols)
- ✅ DMU dnode / object-set layout spec (request #5; leak scan clean - neutral on-disk role names + numeric type/flag codes, no source struct/field/macro symbols)
- ✅ ZAP micro/fat spec (request #6; leak scan cleaned in review - identifier-shaped field labels neutralized; three draft fact errors corrected before approval: NUL-exclusion in the hash, 32-bit CD width, 48-bit wide-hash truncation)
- ✅ DSL directory/dataset/snapshot spec (request #7; leak scan clean - neutral field labels, on-disk key strings retained as interface facts; two draft errors corrected before approval: deadlist bucket rule is strictly-less-than, clone sets hold head *dataset* objects despite a stale source comment saying dir objects)

### Future and/or deferred

### Canceled
