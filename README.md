# furniture-detection

Finds the running heads, page numbers and stamps on the pages of a PDF, so that you can leave them out of the extracted text.

Typesetters call this text furniture. It belongs to the page rather than to the book's argument, and left in the text it splits sentences that run across a page break and fills search results with book titles and page numbers. The crate takes the positioned text lines of a document's pages and tells you which lines are furniture. It does not read PDFs itself, so the lines come from a PDF reader of your choice.

## How it works

Most tools find furniture with a rule: a line that repeats, digits aside, near the top or bottom of most pages. Such a rule misses a running head that changes with each chapter and a stamp printed in the side margin, so this crate decides with a measure instead.

Furniture is the text that other pages already tell you. The running head on page 41 repeats the one on page 39, and the page number on page 41 is the one on page 39 plus two, while a sentence of the body appears on no other page.

The crate therefore asks, for every line, which is the cheapest way to write it down:

1. word by word, from how often its words and word pairs occur in the document;
2. as a copy of the line above it, which is how a numbered list describes itself;
3. as a copy of the line at the same place on another page, with any number moved on by the pages in between.

Each way costs a number of bits, and a line whose cheapest description is a copy from another page is furniture. The costs are fitted to each document, so the crate needs no threshold, pattern list, trained model or rule about where on the page furniture sits: a stamp in the side margin and a running head that changes with each chapter are found the same way as a page number at the foot.

[docs/furniture-detection.md](docs/furniture-detection.md) explains the method from the basics, with worked examples.

## Install

The crate is not on crates.io yet, so add it from git:

```toml
[dependencies]
furniture-detection = { git = "https://github.com/AhmedKishki/furniture-detection" }
```

It needs Rust 1.92 or later.

## Usage

```rust
use std::collections::HashSet;

use furniture_detection::{Line, Page, furniture};

// A line's box in points, the origin at the top of the page.
fn line(text: &str, left: f32, top: f32, right: f32) -> Line {
    let text = text.to_string();
    Line { text, left, top, right, bottom: top + 10.0 }
}

let body = [
    "Value is not a thing but a relation between people,",
    "hidden under the form of a relation between things.",
    "Labour time measures it only where markets have",
    "made the work of each producer comparable to all.",
    "Machines enter this measure as dead labour that",
    "passes its value on to the product bit by bit.",
    "Waste appears where the product finds no buyer",
    "and its labour is spent without being counted.",
];
let pages: Vec<Page> = (1..=4u32)
    .map(|n| {
        let k = 2 * (n as usize - 1);
        let lines = vec![
            line("On the Nature of Entropy", 150.0, 30.0, 290.0),
            line(body[k], 72.0, 60.0, 360.0),
            line(body[k + 1], 72.0, 74.0, 360.0),
            line(&(n + 40).to_string(), 210.0, 600.0, 222.0),
        ];
        Page { page: n, lines }
    })
    .collect();

// The running head (line 0) and the folio (line 3) of every page.
let expected: HashSet<(u32, usize)> = (1..=4).flat_map(|n| [(n, 0), (n, 3)]).collect();
assert_eq!(furniture(&pages), expected);
```

## Input and output

`furniture(&pages)` returns the `(page, index)` of every furniture line: its page's number and its index in that page's `lines`.

- Give it the whole document at once, because it works by comparing pages.
- Pass every page in reading order, numbered by its place in the document (1, 2, 3, …) rather than by its printed page number. A page without text goes in with no lines, because the search for copies assumes that the numbers have no gaps.
- Give each line its text and its box in points, with the origin at the top of the page and y growing downward. PDF puts the origin at the bottom, so a box from `y0` up to `y1` there becomes `top = height - y1` and `bottom = height - y0`.

## What to expect

On 550 pages from 22 books, the crate found 97.3% of the items that docling's layout model labels as page headers and footers, along with repository stamps that docling missed. It also takes more real text for furniture than docling does, mostly short notes, citations and index lines that repeat their pattern at the same place a page or two apart: after every disagreement was judged by hand, it made 86 errors to docling's 48. Whole PDF extraction with the crate took 5 ms a page, where docling took 970 ms.

Furniture stays in the text when no nearby page holds a cheap copy of it:

- in a one-page document, which has no other page;
- in a running head that the PDF reader splits differently on each page, such as `O NTO LOG Y` and `ON TO LOG Y`;
- when the text changes on every page without a number that counts up;
- in a page number printed somewhere else on only a few pages, such as chapter openings, whose nearest copy is many pages away.

[evals/furniture-detection.md](evals/furniture-detection.md) holds the full evaluation, with its harness and frozen inputs.

## Other tools

Other detectors compare lines across pages and keep those that repeat near the top or bottom, either between neighbouring pages ([Lin 2003](https://doi.org/10.1117/12.472833)) or over most of the document. Rust PDF converters such as [pdf-inspector](https://crates.io/crates/pdf-inspector) and [pdf-markdown](https://crates.io/crates/pdf-markdown) do this as one step of their conversion. Layout models such as [docling](https://github.com/docling-project/docling) label page headers and footers with a trained model instead; the evaluation above compares the crate with docling.

This crate differs from both kinds. It decides by comparing description lengths, with no position bands, repetition thresholds or trained model, and it is a library of its own that takes its lines from any PDF reader.

## License

MIT. See [LICENSE](LICENSE).
