# furniture-detection

Finds the furniture on a PDF's pages: the running heads, page numbers and stamps that repeat from page to page. Taking it out of extracted text keeps sentences whole across page breaks and keeps book titles and page numbers out of search results.

You give it each page's text lines with their positions, from any PDF reader, and it returns the lines that are furniture.

## Results

On 550 pages from 22 books, with every disagreement judged by hand:

| | Furniture found | Correct flags | Time per page |
|---|---|---|---|
| furniture-detection | 98% | 92% | 5 ms |
| docling's layout model | 95% | 99% | 970 ms |
| Recurrence rule | 44% | 97% | |

The recurrence rule takes a line for furniture when it repeats, digits aside, in the top or bottom 12% of at least half the pages, which is the kind of rule PDF converters such as [pdf-inspector](https://crates.io/crates/pdf-inspector) and [pdf-markdown](https://crates.io/crates/pdf-markdown) use. [The evaluation](evals/furniture-detection.md) gives the details.

## How it works

Furniture is what other pages already tell you: the running head on page 41 repeats the one on page 39, and the page number on page 41 is page 39's plus two. The crate writes each line down three ways and counts the bits each takes:

1. word by word, from how often its words and word pairs occur in the document;
2. as a copy of the line above, which is how a numbered list describes itself;
3. as a copy of the line at the same place on another page, with any number moved on by the pages in between.

A line is furniture when the copy from another page is shortest. Every cost is fitted to the document at hand, so the crate adapts to each layout by itself. [docs/furniture-detection.md](docs/furniture-detection.md) explains the method from the basics.

## Install

Add it to `Cargo.toml`; it needs Rust 1.92 or later.

```toml
[dependencies]
furniture-detection = "0.1"
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

`furniture(&pages)` takes the whole document and returns the `(page, index)` of each furniture line.

- Number the pages 1, 2, 3, … in reading order, by position rather than by printed page number, and include each page without text as a page with no lines, so that the numbering runs without gaps.
- Give each line its text and its box in points, with the origin at the top of the page. PDF puts the origin at the bottom, so a box from `y0` up to `y1` becomes `top = height - y1` and `bottom = height - y0`.

## License

MIT. See [LICENSE](LICENSE).
