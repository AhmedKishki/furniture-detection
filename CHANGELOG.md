# Changelog

Every change to furniture-detection gets a version and an entry here, newest first. Versions follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html): documentation and fixes that keep the behaviour bump the patch number, and new features or changed behaviour bump the minor number while the crate is at 0.x. The entries follow [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [0.2.0] - 2026-11-10

### Changed

- A copy from another page now pays its share of the text it copies. The text has to be written once, so the pages that can copy it split its cost: a copy's length is its own cost plus the line's length by its tokens divided by the number of users, the line's own page included. A user is a page whose copy is shorter than the tokens. Before, two pages that printed the same line each counted as a copy of the other and neither paid for the text, so a table header repeated on two pages, a note that cites the same work twice, or a contents entry and the heading it lists were removed as furniture. A running head or folio on many pages pays next to nothing and is still found, and a stamp that is long enough to cost much by its tokens is still found on two pages.
- The search for copies stops at the length of the token description alone, because it now also counts users.
- The measured effect on the 550 sampled pages of the evaluation is in [evals/furniture-detection.md](evals/furniture-detection.md): the lines judged to be text that the measure removed fall from 72 to 35, and three furniture lines are lost.

### Limits

- A running head or folio that only a few pages can use now stays in the text, such as the folios of the front matter and of chapter openings.

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

[0.2.0]: https://github.com/AhmedKishki/furniture-detection/compare/v0.1.1...v0.2.0
[0.1.1]: https://github.com/AhmedKishki/furniture-detection/compare/v0.1.0...v0.1.1
[0.1.0]: https://github.com/AhmedKishki/furniture-detection/releases/tag/v0.1.0
