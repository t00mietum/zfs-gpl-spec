# Provenance

## Source studied

Facts in these specifications were derived by studying the OpenZFS source tree at a single pinned commit:

- Repository: `openzfs/zfs`
- Commit: `9b7642df931a765509403318553f2910daf71305`

The OpenZFS source is CDDL-licensed and is never published, redistributed, or reproduced here. It is a reference used only to establish and confirm facts about the on-disk format and observable behavior.

## Method

Each fact is derived, not copied:

- The on-disk format (layout, offsets, sizes, magic values, endianness, checksum construction, enumerated values, nvlist key strings) was studied at the pinned commit and restated as requirements in original wording.
- Durable-core facts are corroborated against the public 2006 Sun "ZFS On-Disk Specification", which was independently published and describes the same on-disk structures.
- No source code (verbatim or paraphrased), comments, or internal identifier/function names are reproduced. Descriptions state format requirements; they do not follow the source's code structure or sequence where that ordering was an authorial choice.
- On-disk key strings and documented magic values and constants are interoperability facts and are reproduced exactly where the format requires a byte-for-byte match.

## Per-fact provenance tagging

Every fact in a spec carries a provenance tag so a reader can distinguish the durable published core from modern refinements:

- `[2006]` - corroborated by the public 2006 Sun "ZFS On-Disk Specification".
- `[modern]` - source-only; a post-2006 addition or refinement observed in current OpenZFS and required to interoperate with pools written by recent OpenZFS. Not part of the original published spec.

## Authorship

The specifications are written by Claude (Anthropic's LLM-based agent), operating as the spec side of the clean-room process - a role isolated from the implementation side, which never sees OpenZFS source. The human maintainer directs the work and contributes documentation, not spec content. From 2026-07-19 forward, Claude commits are authored as `Claude <claude@bubblesnet.com>` with trailers recording the exact model, effort setting, and isolated instance (`Model:` / `Effort:` / `Instance:`), plus `Co-Authored-By: Claude <noreply@anthropic.com>` so the work is visibly credited to Claude's GitHub identity. Earlier commits were authored under the maintainer's project identity; history is not rewritten.

## Review

Draft specs are gatekept before they become authoritative. The gatekeeper strips any leaked expression, confirms facts are stated correctly, and records the review in `gatekeeper-log.md`. Only approved specs are authoritative.
