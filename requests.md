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
| 2 | 2026-07-19 | Label/uberblock self-checksum: digest-to-word packing of the embedded SHA-256 (needed before claiming interop) | open
