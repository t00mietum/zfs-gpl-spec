<!-- markdownlint-disable MD007 -- Unordered list indentation -->
<!-- markdownlint-disable MD010 -- No hard tabs -->
<!-- markdownlint-disable MD033 -- No inline html -->
<!-- markdownlint-disable MD055 -- Table pipe style [Expected: leading_and_trailing; Actual: leading_only; Missing trailing pipe] -->
<!-- markdownlint-disable MD041 -- First line in a file should be a top-level heading -->
<div align="center">

[![License: GPL v2+](https://img.shields.io/badge/License-GPLv2%2B-blue.svg)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.html)
![Lifecycle: Pre-alpha](https://img.shields.io/badge/Lifecycle-Pre--alpha-red)

</div>

<!-- TOC ignore:true -->
# ZFS-GPL Spec

Gatekeeper-approved, fact-only functional specifications for the ZFS on-disk format and observable behavior. These specs are the sole authoritative channel feeding the [`zfs-gpl`](https://github.com/t00mietum/zfs-gpl) implementation.

<!-- TOC ignore:true -->
## Table of contents

<!-- TOC -->

- [What this repo is](#what-this-repo-is)
- [How it is consumed](#how-it-is-consumed)
- [Layout](#layout)
- [Provenance tags](#provenance-tags)
- [Copyright and license](#copyright-and-license)

<!-- /TOC -->

## What this repo is

This repository holds functional specifications written as facts: layout, offsets, magic numbers, field meanings, endianness, checksum and compression identities, feature semantics, behavioral requirements, and test cases stated as input and expected output. Each spec is reviewed by a gatekeeper before it is marked authoritative.

Nothing here is copied from OpenZFS source. No source lines (verbatim or paraphrased), comments, or internal identifier/function names are reproduced, and no spec mirrors the source's code structure or sequence where that was an authorial choice rather than dictated by the on-disk format. On-disk key strings and documented magic values and constants are interface facts and are stated as such.

## How it is consumed

The `zfs-gpl` implementation repo consumes these specs pinned to specific commit hashes of this repository. An implementer works from the approved fact statements alone and never sees or references OpenZFS source. See [`provenance.md`](provenance.md) for what was studied and how the facts were derived, and [`gatekeeper-log.md`](gatekeeper-log.md) for the per-spec review record.

## Layout

- `specs/format/` - on-disk format specifications.
- `provenance.md` - source studied, derivation method, per-fact provenance tagging.
- `gatekeeper-log.md` - dated review log; one entry per spec approval.
- `project/` - spec backlog and design notes.

## Provenance tags

Within a spec, individual facts carry tags:

- `[2006]` - corroborated by the public 2006 Sun "ZFS On-Disk Specification".
- `[modern]` - a post-2006 addition or refinement required to interoperate with pools written by recent OpenZFS.

## Copyright and license

> Copyright © 2026 t00mietum (ID: f⍒Ê🝅ĜᛎỹqFẅ▿⍢Ŷ‡ʬẼᛏ🜣)<br>
> Licensed under [GNU GPL v2 Or Later License](https://spdx.org/licenses/GPL-2.0-or-later.html). No warranty. See [`license.md`](license.md).
