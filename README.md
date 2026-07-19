# zfs-gpl-spec

Gatekeeper-approved, fact-only functional specifications for the ZFS on-disk format and observable behavior. These specs are the sole authoritative channel feeding the `zfs-gpl` implementation.

## What this repo is

This repository holds functional specifications written as facts: layout, offsets, magic numbers, field meanings, endianness, checksum and compression identities, feature semantics, behavioral requirements, and test cases stated as input and expected output. Each spec is reviewed by a gatekeeper before it is marked authoritative.

Nothing here is copied from OpenZFS source. No source lines (verbatim or paraphrased), comments, or internal identifier/function names are reproduced, and no spec mirrors the source's code structure or sequence where that was an authorial choice rather than dictated by the on-disk format. On-disk key strings and documented magic values and constants are interface facts and are stated as such.

## How it is consumed

The `zfs-gpl` implementation repo consumes these specs pinned to specific commit hashes of this repository. An implementer works from the approved fact statements alone and never sees or references OpenZFS source. See `PROVENANCE.md` for what was studied and how the facts were derived, and `gatekeeper-log.md` for the per-spec review record.

## Layout

- `specs/format/` - on-disk format specifications.
- `PROVENANCE.md` - source studied, derivation method, per-fact provenance tagging.
- `gatekeeper-log.md` - dated review log; one entry per spec approval.

## Provenance tags

Within a spec, individual facts carry tags:

- `[2006]` - corroborated by the public 2006 Sun "ZFS On-Disk Specification".
- `[modern]` - a post-2006 addition or refinement required to interoperate with pools written by recent OpenZFS.
