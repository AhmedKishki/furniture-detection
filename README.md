# furniture-detection

A printed page carries text that belongs to the page rather than to the book's argument: running heads and feet, page numbers (folios), a publisher's or repository's stamp at the foot of every page. Typesetters call it furniture. Left in extracted text, it splits sentences that run across a page break and fills search results with book titles and page numbers.

This crate takes the positioned text lines of a PDF's pages and returns the lines that are furniture. It does not read PDFs: the lines come from a PDF reader of your choice.

## What counts as furniture

Furniture is the part of a page that other pages already tell you. A running head on page 41 repeats the running head on page 39, and the folio on page 41 is the folio on page 39 plus two. A sentence of the text is predicted by the words around it on its own page, and no other page holds it.

The crate turns this into a measure. Every line is described in three ways, each with a length in bits: by its own tokens, as a copy of the line above it in its column, and as a copy of the nearest line on another page. A line is furniture when the copy from another page is its shortest description. The parameters of the descriptions are fitted to the document being read, and there is no threshold, pattern list, trained model or rule about where on the page furniture sits. A stamp in the side margin and a running head that changes with each chapter are found the same way as a folio at the foot of the page.

## Adding it

The crate is not on crates.io. Add it as a git dependency:

```toml
[dependencies]
furniture-detection = { git = "https://github.com/AhmedKishki/furniture-detection" }
```

It needs Rust 1.92 or later and depends only on `unicode-normalization`.

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

The same example is the crate's documentation example, so `cargo test` compiles and runs it.

## Input and output

```rust
pub fn furniture(pages: &[Page]) -> HashSet<(u32, usize)>

pub struct Page { pub page: u32, pub lines: Vec<Line> }
pub struct Line { pub text: String, pub left: f32, pub top: f32, pub right: f32, pub bottom: f32 }
```

A `Line` is one line of text with its box in points. The origin is at the top of the page and y grows downward, so `top < bottom`, and a line with a smaller y sits above one with a larger y. PDF's own coordinates put the origin at the bottom of the page; a box from `y0` up to `y1` in those becomes `top = height - y1` and `bottom = height - y0`.

The pages must come in reading order with ascending page numbers. A page's number is its place in the document, not its printed folio, because a folio is expected to move on by the difference between two pages' numbers. Pass every page, with empty `lines` for a page without text, so that the numbers run without gaps: the search for copies tries pages nearest first by their position in the slice and stops early on the assumption that the page offset grows with that position, so a gap can make it stop before a page that is nearer by number.

`furniture` compares pages, so it takes the whole document at once. It returns the `(page, index)` of each furniture line: the `page` of its `Page` and its index in that page's `lines`. A line whose text is empty after normalisation is never furniture.

## Limits

- A one-page document has no furniture. The measure needs several pages that share furniture.
- A copy must hold the same tokens, numbers aside. A letter-spaced running head that the PDF reader splits differently on each page (`O NTO LOG Y`, `ON TO LOG Y`) has no copy and stays in the text.
- Text that recurs at the same place on nearby pages is described as a copy and taken for furniture: short note, citation and index lines with a common pattern, code repeated across listings, a title page printed twice. In the evaluation, note, citation and index lines are 46 of its 86 errors.
- Furniture whose text changes on every page without a number that moves with the page has no copy and stays. A folio printed somewhere else on a few pages, such as chapter openings, can only copy from a distant page and may stay.

## Cost

On 22 books (7,235 pages), whole PDF extraction with this measure (reading the PDF, building lines and paragraphs, and finding furniture) took 5.0 ms per page with a mean peak memory of 92 MB on an 8-core laptop CPU. The detector is one part of that work, so it costs no more on its own.

## Documentation

- [docs/furniture-detection.md](docs/furniture-detection.md) explains the measure: the three descriptions and their codes, the decision by expectation–maximization, two worked examples, the limits and the references.
- [evals/furniture-detection.md](evals/furniture-detection.md) compares it with docling's layout model on 550 pages of 22 books, with its harness and frozen inputs. It finds 97.3% of the lines docling labels as furniture, and after every disagreement was judged by hand it makes 86 errors to docling's 48.

## License

MIT. See [LICENSE](LICENSE).
