# furniture-detection

Finds the furniture on a PDF's pages: running heads, page numbers and stamps. Left in extracted text, furniture splits sentences at page breaks and fills search results with book titles and page numbers. The crate takes each page's positioned text lines, from any PDF reader, and returns the lines that are furniture.

## Why

Most tools, among them the Rust PDF converters [pdf-inspector](https://crates.io/crates/pdf-inspector) and [pdf-markdown](https://crates.io/crates/pdf-markdown), treat a line as furniture when it repeats, digits aside, near the top or bottom of the page. On 550 pages from 22 books, such a rule (repeating on at least half the pages, within the top or bottom 12%) found 44% of the furniture: running heads that change with each chapter repeat on too few pages, and stamps in the margin sit outside the bands. [Docling](https://github.com/docling-project/docling)'s layout model found 95%. This crate found 98%, without a model, and whole extraction took 5 ms a page where docling took 970 ms.

It pays with false alarms. 92% of what it flags is furniture, against 99% for docling, because short notes, citations and index lines that repeat their pattern a page or two apart look like furniture to it. [The evaluation](evals/furniture-detection.md) has the details.

## How it works

Furniture is what other pages already tell you: the running head on page 41 repeats the one on page 39, and the page number on page 41 is page 39's plus two. The crate therefore writes each line down three ways and counts the bits each takes:

1. word by word, from how often its words and word pairs occur in the document;
2. as a copy of the line above, which is how a numbered list describes itself;
3. as a copy of the line at the same place on another page, with any number moved on by the pages in between.

A line is furniture when the copy from another page is shortest. The costs are fitted to each document, so there is no threshold or position rule to tune. [docs/furniture-detection.md](docs/furniture-detection.md) explains the method from the basics.

## Install

The crate is not on crates.io yet, so add it from git. It needs Rust 1.92 or later.

```toml
[dependencies]
furniture-detection = { git = "https://github.com/AhmedKishki/furniture-detection" }
```

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

`furniture(&pages)` returns the `(page, index)` of each furniture line.

- Pass every page of the document in reading order, numbered 1, 2, 3, … by position rather than by printed page number. A page without text goes in with no lines, because the search for copies assumes the numbers have no gaps.
- Give each line its text and its box in points, with the origin at the top of the page. PDF puts the origin at the bottom, so a box from `y0` up to `y1` becomes `top = height - y1` and `bottom = height - y0`.

## Limits

Furniture stays in the text when no nearby page holds a cheap copy of it: in a one-page document, in a running head that the PDF reader splits differently on each page (`O NTO LOG Y`, `ON TO LOG Y`), in text that changes on every page without a number that counts up, and in a page number printed elsewhere on only a few pages, such as chapter openings.

## License

MIT. See [LICENSE](LICENSE).
