<!-- markdownlint-disable MD007 -- Unordered list indentation -->
<!-- markdownlint-disable MD010 -- No hard tabs -->
<!-- markdownlint-disable MD033 -- No inline html -->
<!-- markdownlint-disable MD055 -- Table pipe style [Expected: leading_and_trailing; Actual: leading_only; Missing trailing pipe] -->
<!-- markdownlint-disable MD041 -- This is a spec-repo design doc -->
# Design

How this spec repo is organized and the rules it runs under. The spec-coverage task list lives in [`backlog.md`](backlog.md). Provenance and review records are in [`../provenance.md`](../provenance.md) and [`../gatekeeper-log.md`](../gatekeeper-log.md).

## Assumptions

- This repo is the spec side of a clean-room wall. It contains only gatekeeper-approved, fact-only specifications.
- The published 2006 Sun "ZFS On-Disk Specification" corroborates the durable core; post-2006 facts are `[modern]` and come only through the pipeline.

## Project structure

### Folder structure

- `specs/format/` - on-disk format specifications, numbered in coverage order.
- `provenance.md`, `gatekeeper-log.md` - derivation method and per-spec review record.
- `project/` - this design doc and the spec backlog.

### Data flow

Dirty-side draft (pinned source) -> gatekeeper review -> approved spec -> consumed by the `zfs-gpl` implementation at a pinned commit.

## Direction decisions

- Facts, never expression. No source lines, comments, internal identifiers, or source-mirroring structure. On-disk key strings and documented magic constants are interface facts and are stated exactly.
- Per-fact provenance tags (`[2006]` / `[modern]`) so a reader can separate the durable published core from modern refinements.
- One pinned source commit of record; recorded in `provenance.md`.

## Plan

Specs are written in the order the implementation needs them, starting from the durable core (labels, uberblock, block pointers, DMU/DSL/ZAP/ZPL) and expanding to modern features as the read and write paths require them. See [`backlog.md`](backlog.md).

## Architecture

### Testing

Specs state conformance facts as input and expected output, so the implementation can derive independent test vectors without seeing the source.
