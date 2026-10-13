<!-- markdownlint-disable MD007 -- Unordered list indentation -->
<!-- markdownlint-disable MD010 -- No hard tabs -->
<!-- markdownlint-disable MD041 -- First line in a file should be a top-level heading -->

<!-- TOC ignore:true -->
# Project backlog

<!-- TOC ignore:true -->
## Table of contents

<!-- TOC -->

- [Introduction](#introduction)
- [Issues](#issues)
- [Old format](#old-format)
	- [Bugs](#bugs)
	- [Features and enhancements](#features-and-enhancements)
	- [Done](#done)
		- [Done - Bugs](#done---bugs)
		- [Done - Features and enhancements](#done---features-and-enhancements)
	- [Deferred](#deferred)
	- [Canceled](#canceled)
- [Template](#template)

<!-- /TOC -->

## Introduction

Going forward, new issues in the new template at the bottom of this file, will go in the '## New format' section only. No more status emojis. Refer to '## Reference' for sort order. Issues in the old format (with status emojis) won't be refactored, but will continue to be worked until moved to closed, canceled, or deferred sections, and emojis updated. (Eventually this will all be moved to nano-git-db anyway. This new template is an intermediate effort to make issues going forward more structured and importable.)

Spec-coverage backlog. Each item is a spec to draft, gatekeep, and approve. Specs are written in the order the [`zfs-gpl`](https://github.com/t00mietum/zfs-gpl) implementation needs them. Companion: [`design.md`](design.md).

## Issues

## Old format

### Bugs

### Features and enhancements

- Modern features (post-2006, source-only)
	- 🔘 Feature flags: detection and gating
	- 🔘 Compression identities: lz4, zstd, gzip
	- 🔘 Checksum identities beyond the core: edonr, blake3, skein
	- 🔘 Native encryption format
	- 🔘 dRAID and raidz geometry
	- 🔘 Log spacemaps, large dnodes, large/embedded blocks

### Done

#### Done - Bugs

#### Done - Features and enhancements

- Durable core (2006-corroborated)
	- ✅ VDEV label and uberblock (`specs/format/01-vdev-label-uberblock.md`, approved `7f86d1b`)
	- ✅ Label/uberblock self-checksum: digest-to-word packing detail needed for byte-exact interop (`specs/format/02-label-checksum-sha256-packing.md`)
	- ✅ Config nvlist: XDR wire encoding (`specs/format/03-config-nvlist-xdr-encoding.md`; key catalog already in spec 01 §3)
	- ✅ Block pointers and DVAs: layout, checksum and compression identities, endianness (`specs/format/04-block-pointer-dva.md`, approved `req4`)
	- ✅ DMU: dnode and object-set layout (`specs/format/05-dmu-dnode-objset.md`, approved `req5`)
	- ✅ ZAP: micro and fat (`specs/format/06-zap.md`)
	- ✅ DSL: dataset and snapshot on-disk structures (`specs/format/07-dsl-dataset-snapshot.md`)
	- ✅ ZPL: directory and file layout, attributes (`specs/format/08-zpl-znode-sa-dirent.md`)

- ✅ VDEV label and uberblock spec (first spec through the pipeline; one leaked macro name stripped in review)

- ✅ Label/uberblock SHA-256 digest-to-word packing spec (request #2; leak scan clean)

- ✅ Config nvlist XDR wire-encoding spec (request #3; leak scan clean - neutral type names only, no source identifiers)

- ✅ Block pointer / DVA layout spec (request #4; leak scan clean - neutral field terms + public algorithm names + numeric identity codes, no source symbols)

- ✅ DMU dnode / object-set layout spec (request #5; leak scan clean - neutral on-disk role names + numeric type/flag codes, no source struct/field/macro symbols)

- ✅ ZAP micro/fat spec (request #6; leak scan cleaned in review - identifier-shaped field labels neutralized; three draft fact errors corrected before approval: NUL-exclusion in the hash, 32-bit CD width, 48-bit wide-hash truncation)

- ✅ ZPL znode/SA/dirent spec (request #8; leak scan clean - neutral field labels, on-disk key strings (master-node keys, SA registry names) retained as interface facts; two draft errors corrected before approval: the sixteen fixed SA attributes do get written to the registry (their numbers are just pre-assigned), and quota-ZAP keys are the hex of the full 64-bit FUID, not a composite string). ACE semantics, FUID table schema, and ZIL records deferred to later specs.

- ✅ DSL directory/dataset/snapshot spec (request #7; leak scan clean - neutral field labels, on-disk key strings retained as interface facts; two draft errors corrected before approval: deadlist bucket rule is strictly-less-than, clone sets hold head *dataset* objects despite a stale source comment saying dir objects)

### Deferred

### Canceled

## Template

### Old format

- 🔘 Not started

- 🛠️ Started, and/or partially complete

- 🔬 Testing not started or finished

- ✋ Defer

- ✅ Complete

- 🚫 Canceled

### New format

- Notes:

	- Only use rows that you actually need or expect will be filled in. Always fill in the title, ID, Type, Status, Opened and Created by.

	- The ID is the local time to the hundredth of a second. Opened is when it was written down, which may differ. (Use a keyboard macro and possibly something like project 'zuid' to generate.)

	- Status values meaning: Testing means the fix is in and checks are running or still to run. Waiting on signoff means automated testing passed. Moot means something else changed that made it irrelevant. Canceled means it still applies but was decided against. Waiting for testing means the fix is in and waits on a long CI run or an outside test host. Can't reproduce means a real attempt to reproduce it failed.

	- As issues are worked, and statuses change, place them in correct sorting order within the list:
		- First by status: Waiting for answers, Waiting on signoff, Testing, Waiting for testing, Can't reproduce, Stalled, Started, Queued, Done, Deferred, Canceled, Moot
		- Then by severity|priority: Critical, High, Avg, Low
		- Then by type: Bugs, [not bugs together]

	- Rows marked [Bug] are for bugs only, and rows marked [Feature] for features and enhancements. Children are not nested. They sit at the top level and point back with Parent ID.

Template:

- Title
	- ID: YYYYmmDDHHMMSSNN
	- Type: [Bug|Feature|Enhancement|Task]
	- Status: [Queued|Waiting for answers|Waiting on signoff|Waiting for testing|Started|Testing|Stalled|Can't reproduce|Moot|Canceled|Deferred|Done]
	- Needs local test suite run?:
	- Needs external testing:
	- Priority|Severity [Bug]: [Critical|High|Avg|Low]
	- Opened:
	- Opened by:
	- Assigned to:
	- Parent ID:
	- Prereq IDs:
	- Related IDs:
	- Target OS:
	- Test environment:
	- Version and build:
	- Requirements  [Feature]:
		- Hierarchical bulleted list.
	- Steps to reproduce [Bug]:
		- …
	- Incorrect behavior [Bug]:
	- Expected behavior [Bug]:
	- Reproduced [Bug]: [No, or when, where and how]
	- Possible cause [Bug]:
	- Actual cause [Bug]:
		- …
	- Estimated effort: [High|Avg|Low]
	- Actual effort: [High|Avg|Low]
	- Progress log:
		- …
	- Decisions:
		- …
	- Actual fix [Bug]:
	- Branch:
	- Commit:
	- Test case: [Reason not applicable, or CI test case #]
	- Acceptance signoff:
	- Superseded by ID:
	- Closed:
