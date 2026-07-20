<!-- omit in toc -->
# Contributing

This repository is one half of a clean-room wall. It holds only gatekeeper-approved, fact-only specifications. What may enter here is tightly constrained, because the `zfs-gpl` implementation's GPL relicense depends on these specs containing facts and no CDDL expression.

<!-- omit in toc -->
## Table of Contents

- [The two roles](#the-two-roles)
- [What a spec may contain](#what-a-spec-may-contain)
- [What a spec may never contain](#what-a-spec-may-never-contain)
- [The pipeline](#the-pipeline)
- [Provenance tagging](#provenance-tagging)
- [Code of Conduct](#code-of-conduct)


## The two roles

The wall is two mutually exclusive roles. Nobody does both:

- **Spec side** may study OpenZFS (or other CDDL) source and writes only functional specifications into this repo. It never writes implementation code.
- **Implementation side** works in the `zfs-gpl` repo from these approved specs alone and never opens OpenZFS source.

If you have written implementation code for `zfs-gpl`, you may not author specs here, and vice versa.

## What a spec may contain

Facts about the on-disk format and observable behavior: layout, offsets, sizes, magic values, endianness, checksum construction, enumerated values, on-disk nvlist key strings, feature semantics, and test cases stated as input and expected output. On-disk key strings and documented magic constants are interoperability facts and are stated exactly where the format requires a byte-for-byte match.

## What a spec may never contain

- Source lines from OpenZFS, verbatim or paraphrased.
- Comments or internal identifier/function/macro names from the source.
- Structure or sequence that mirrors the source where that ordering was an authorial choice rather than dictated by the on-disk format.

## The pipeline

1. A dirty-side pass studies the source at a single pinned commit and drafts a spec as facts (status `DRAFT`).
2. A gatekeeper reviews the draft against the wall rules, strips any leaked expression, confirms the facts, and records the review in [`gatekeeper-log.md`](gatekeeper-log.md).
3. On approval the header flips `DRAFT -> APPROVED`. Only approved specs are authoritative.
4. The implementation repo bumps its submodule pin to the approved commit and implements from it.

See [`provenance.md`](provenance.md) for the pinned source, derivation method, and per-fact tagging.

## Provenance tagging

Every fact carries a tag so a reader can separate the durable published core from modern refinements:

- `[2006]` - corroborated by the public 2006 Sun "ZFS On-Disk Specification".
- `[modern]` - source-only; a post-2006 addition required to interoperate with recent OpenZFS.

## Code of Conduct

This project and everyone participating in it is governed by the [Code of Conduct](code_of_conduct.md).
