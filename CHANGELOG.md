# Changelog

Every change to furniture-detection gets a version and an entry here, newest first. Versions follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html): documentation and fixes that keep the behaviour bump the patch number, and new features or changed behaviour bump the minor number while the crate is at 0.x. The entries follow [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [0.1.1] - 2026-11-05

### Added

- `docs/usage.md`, a usage reference: every public function, type and field with its conventions, return value, panics and an example, and recipes for converting bottom-origin boxes, passing pages without text and removing furniture before joining lines. Every example runs as a doctest.
- Documentation for every public item, and a `missing_docs` warning for any new one.

### Changed

- The documentation of `furniture` now states that, in a build with overflow checks, a line holding a run of digits close to `i64::MAX` makes it panic on integer overflow; a release build wraps instead.

## [0.1.0] - 2026-11-05

### Added

- `furniture`, which finds the running heads, page numbers and stamps of a document's pages by comparing three descriptions of every line by their length in bits, and the `Page` and `Line` types it takes.
- `docs/furniture-detection.md`, which explains the measure from the basics, and an evaluation against docling on 550 pages from 22 books.

[0.1.1]: https://github.com/AhmedKishki/furniture-detection/compare/v0.1.0...v0.1.1
[0.1.0]: https://github.com/AhmedKishki/furniture-detection/releases/tag/v0.1.0
