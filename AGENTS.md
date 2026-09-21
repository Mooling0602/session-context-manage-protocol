# AGENTS.md — session-context-manage-protocol

Guidelines for AI agents working in this repository. Any parent workspace spec
applies in full; this file only adds project-specific rules.

## Project

SCMP (Session Context Manage Protocol) specification repository. Deliverables
are Chinese-draft specification documents (`docs/`) and machine-readable JSON
Schemas (`schema/`). There is no runtime code here — adapters live in their
own repositories.

## Language

- **Commit messages MUST be in English.** (Established convention; the user
  will ask for a force-push amend if violated on `vibe`.)
- Specification documents are written in Simplified Chinese during the `0.x`
  draft era. An English normative version is planned for v1.0 (see
  `versions.md` and the roadmap in `README.md`).

## Git

- `main` is the protected integration branch; `vibe` is the freestyle branch.

## Documentation conventions

- RFC 2119 style keywords (MUST / MUST NOT / SHOULD / SHOULD NOT / MAY) are
  used in their Chinese forms（必须/不得/应当/不应当/可以）with the English
  terms kept in the header note of each document.
- Documents are numbered by layer: `00` overview, `10` L1 tools, `20` L2 data
  model, `30`/`40` L3 bindings, `35` host adaptation, `90` conformance.
- Any semantic or structural change to the spec MUST be accompanied by a
  matching entry in the `versions.md` changelog (bump draft version first,
  then log).
- JSON Schemas under `schema/` must carry an `$id` pointing at the raw file on
  this repository's default branch and a `const`/`enum`-constrained
  `protocolVersion` matching the current draft.
- Design decisions worth recording long-term go into `00-overview.md`
  (non-goals §3 / open questions §8) rather than living only in commit
  history.

## Verification

- No build or test suite for the spec itself. Before committing, re-read the
  edited section for internal links (`[text](file.md)`) and table formatting;
  keep line width reasonable for terminal reading.
- Docs site (MkDocs Material): after any structural docs change (nav, links,
  new/renamed files), run `mkdocs build --strict` (deps:
  `pip install -r requirements-docs.txt`). Links from `docs/` to files outside
  it (e.g. `schema/`) must use absolute GitHub URLs, not `../` relative paths.
