# Using furniture-detection

<!-- Signatures sit in `rs` blocks, which rustdoc does not test. Every `rust` block is a doctest: src/lib.rs includes this file, so `cargo test` compiles and runs it. -->

furniture-detection finds the furniture of a PDF: the running heads, page numbers and stamps that repeat from page to page. It does not read PDFs itself, so you extract each page's lines of text and their positions with a PDF reader, pass all the pages to [`furniture`](#furniture), and get back the lines that are furniture. Dropping those lines before you join the rest into paragraphs keeps sentences whole across page breaks and keeps running heads and page numbers out of search results. [furniture-detection.md](furniture-detection.md) explains how the crate decides what is furniture.

## Quick start

Add the crate to `Cargo.toml`; it needs Rust 1.92 or later.

```toml
[dependencies]
furniture-detection = "0.2"
```

This program builds a book of four pages, each with a running head, one line of text and a page number, and finds the head and the page number of every page.

```rust
use std::collections::HashSet;

use furniture_detection::{Line, Page, furniture};

// Each line's box is in points, measured from the top-left corner of the page.
let text = [
    "Value is not a thing but a relation between people,",
    "hidden under the form of a relation between things.",
    "Labour time measures it only where markets have",
    "made the work of each producer comparable to all.",
];
let line = |text: String, left, top, right| Line { text, left, top, right, bottom: top + 10.0 };
let pages: Vec<Page> = (1..=4)
    .map(|n| Page {
        page: n,
        lines: vec![
            line("On the Nature of Entropy".to_string(), 150.0, 30.0, 290.0),
            line(text[n as usize - 1].to_string(), 72.0, 60.0, 360.0),
            line((n + 40).to_string(), 210.0, 600.0, 222.0),
        ],
    })
    .collect();

let found = furniture(&pages);

// The running head (index 0) and the page number (index 2) of every page.
let expected: HashSet<(u32, usize)> = (1..=4).flat_map(|n| [(n, 0), (n, 2)]).collect();
assert_eq!(found, expected);
```

`furniture` returns a set of `(page, index)` pairs: a page's number and the index of a line in that page's `lines`. The line of text on each page is not in the set, because no other page repeats it, while each head repeats the head of the page before and each page number is the one before it plus one.

## Reference

The public API is the function `furniture` and the two types it reads, `Page` and `Line`.

### `furniture`

```rs
pub fn furniture(pages: &[Page]) -> HashSet<(u32, usize)>
```

Finds the furniture lines of a document, the lines that other pages predict. Each line is described in three ways, by its own words, as a copy of the line above it on its page, or as a copy of the nearest line on another page, and it is furniture when the copy from another page is the shortest description in bits. A copy may move a number on by the difference between the two pages' numbers, which is how page numbers are found. A copy also pays its share of what the copied text costs, split among the pages that copy it, so a line that only a few pages repeat, such as a table header on two pages, stays in the text. The copy of the line above lets a numbered list describe itself, so a list that recurs on nearby pages can be described within its page instead of as furniture.

Every weight in these descriptions is fitted to `pages`, so the function has no settings. The weights, and the counts of words and numbers that the descriptions use, come from every page, so a line's verdict depends on the other pages. Pass the whole document in one call, because part of it can give a different result and a single page has no furniture at all.

**Parameters.** `pages` holds every page of the document, in reading order and numbered by position, with a page without lines for each page without text. [`Page`](#page) gives the conventions.

**Returns.** A `HashSet` with one `(page, index)` pair for each furniture line. `page` is the line's [`Page::page`](#pagepage), and `index` is the line's position in that page's [`Page::lines`](#pagelines), counting every line passed, lines with empty text included. The set has no order; [Mapping the result back to lines](#mapping-the-result-back-to-lines) lists the furniture in reading order. A line whose text is empty after normalizing ([`Line::text`](#linetext)) is never furniture. The set is empty when no line is furniture, as for an empty slice or a single page.

**Errors and panics.** `furniture` returns no error and validates none of its input. It can panic only on integer overflow, in a build with overflow checks, as debug builds have by default:

- when a line holds a run of digits whose value is so close to `i64::MAX` (9,223,372,036,854,775,807) that adding the difference between two page numbers, or 1 down a list, overflows `i64`;
- when the document's lines hold more than `u32::MAX` (4,294,967,295) tokens in all, a token being a word, a number, a punctuation mark or the end of a line.

A release build wraps the same arithmetic instead of panicking.

**Example.**

```rust
use furniture_detection::{Line, Page, furniture};

// A page alone has no other page to be copied from, so nothing on it is
// furniture, and the same holds for no pages at all.
let head = Line {
    text: "On the Nature of Entropy".to_string(),
    left: 150.0,
    top: 30.0,
    right: 290.0,
    bottom: 40.0,
};
let page = Page { page: 1, lines: vec![head] };
assert!(furniture(&[page]).is_empty());
assert!(furniture(&[]).is_empty());
```

### `Page`

```rs
#[derive(Debug, Clone, PartialEq)]
pub struct Page {
    pub page: u32,
    pub lines: Vec<Line>,
}
```

One page of the document: its number and its lines of text. Pass `furniture` one `Page` for every page of the PDF, in reading order with ascending page numbers. The order matters because `furniture` looks for copies on the nearest pages first, by their position in the slice, and stops at the first page too far away by page number to give a shorter description than the best found so far. The difference in page numbers grows with the distance in the slice only when the pages come in order and are numbered by position; otherwise the search can stop before a page that is near by number but far in the slice.

**Fields.** [`page`](#pagepage), the page's position in the document, and [`lines`](#pagelines), its lines of text.

**Errors and panics.** Building a `Page` cannot fail, and `furniture` accepts any page number and any number of lines. The cases in which it panics are listed under [`furniture`](#furniture).

**Example.**

```rust
use furniture_detection::{Line, Page};

// The second page, with a running head at the top and a page number at the foot.
let page = Page {
    page: 2,
    lines: vec![
        Line {
            text: "On the Nature of Entropy".to_string(),
            left: 150.0,
            top: 30.0,
            right: 290.0,
            bottom: 40.0,
        },
        Line {
            text: "42".to_string(),
            left: 210.0,
            top: 600.0,
            right: 222.0,
            bottom: 610.0,
        },
    ],
};
```

#### `Page::page`

```rs
pub page: u32
```

The page's position in the document: 1 for the first page of the PDF, 2 for the second, and so on, with no number skipped. A copy from another page may move a number on by the difference between the two pages' numbers, which matches a printed page number because both grow by one from page to page. The printed page number cannot take their place, although it numbers the same pages, because it is missing on unnumbered pages and restarts at 1 after front matter numbered in roman numerals.

The result names each furniture line by this number, so every page needs its own. The detection itself uses only the differences between page numbers, so counting from 0 works as well.

**Errors and panics.** None: `furniture` accepts any `u32`. Repeated numbers do not make it panic, but its result then cannot tell those pages apart.

**Example.**

```rust
use furniture_detection::{Line, Page};

// The fourteenth page of the PDF, whose printed page number is "xii".
let folio = Line {
    text: "xii".to_string(),
    left: 296.0,
    top: 740.0,
    right: 308.0,
    bottom: 750.0,
};
let page = Page { page: 14, lines: vec![folio] };
```

#### `Page::lines`

```rs
pub lines: Vec<Line>
```

The page's lines of text, one [`Line`](#line) for each line as the PDF reader groups them. A copy must hold the same words and punctuation as the line it copies, numbers aside, so the reader must group a running head into a line the same way on two pages for one to copy the other.

`furniture` names a furniture line by its index in this vector, counting every line, lines with empty text included. A page without text, such as a blank page or a scanned page with no text layer, has an empty vector and still belongs in the slice, as [Passing pages without text](#passing-pages-without-text) explains.

**Errors and panics.** None: `furniture` accepts an empty vector and lines with empty text.

**Example.**

```rust
use furniture_detection::Page;

// A blank third page keeps its place and its number, with no lines.
let blank = Page { page: 3, lines: Vec::new() };
```

### `Line`

```rs
#[derive(Debug, Clone, PartialEq)]
pub struct Line {
    pub text: String,
    pub left: f32,
    pub top: f32,
    pub right: f32,
    pub bottom: f32,
}
```

One line of text and its box on the page. The box is in points, 1/72 of an inch, measured from the top-left corner of the page, with x growing to the right and y growing downward: `left` is less than `right`, `top` is less than `bottom`, and a line with a smaller y sits higher on the page.

`furniture` places a line by the left edge, the horizontal centre and the right edge of its box, each taken at the box's vertical centre. The distance between lines on two pages is taken at whichever of the three points lie closest, left edge to left edge, centre to centre or right edge to right edge, so a page number set flush left, centred or flush right stays at distance 0 from the one on the next page even when its width grows from 99 to 100. The line above a line, which a numbered list copies, is the nearest higher line on the same page whose horizontal extent overlaps its own.

Positions are compared only with one another, so any unit works if every page uses the same unit and the same origin. Boxes measured upward from the bottom of the page, as PDF measures them, must be flipped first, as [Converting a PDF reader's bottom-origin boxes](#converting-a-pdf-readers-bottom-origin-boxes) shows.

**Fields.** [`text`](#linetext), the line's text, and [`left`](#lineleft), [`top`](#linetop), [`right`](#lineright) and [`bottom`](#linebottom), the edges of its box.

**Errors and panics.** Building a `Line` cannot fail, and `furniture` checks none of its fields. No coordinate makes `furniture` panic, not even NaN or infinity; the one kind of text that can is described under [`Line::text`](#linetext).

**Example.**

```rust
use furniture_detection::Line;

// A running head 140 points wide, 30 to 40 points below the top of the page.
let head = Line {
    text: "On the Nature of Entropy".to_string(),
    left: 150.0,
    top: 30.0,
    right: 290.0,
    bottom: 40.0,
};
```

#### `Line::text`

```rs
pub text: String
```

The line's text as the PDF reader extracted it. `furniture` first normalizes it: it expands the Latin ligatures U+FB00 to U+FB06 into their letters, drops soft hyphens, zero-width spaces, byte-order marks and control characters other than whitespace, composes the text to Unicode NFC and collapses each run of whitespace to one space. It then splits the text into words, numbers and punctuation marks, ignoring case and spaces, so the text needs no cleaning beforehand. A line whose text is empty after normalizing is never furniture.

A number is a run of ASCII digits or a roman numeral written the standard way. A copy from another page may keep each number or move it on by the difference between the pages' numbers at little cost, so a page number, or a chapter number in a running head, does not stop a line from being a copy.

**Errors and panics.** In a build with overflow checks, a run of digits so close to `i64::MAX` that adding a difference between page numbers overflows `i64` makes `furniture` panic, as [`furniture`](#furniture) explains.

**Example.**

```rust
use furniture_detection::{Line, Page, furniture};

// The running head as a reader might extract it: in capitals on page 2, with a
// doubled space and a soft hyphen on page 3.
let heads = [
    "On the Nature of Entropy",
    "ON THE NATURE OF ENTROPY",
    "On  the Nature of En\u{00AD}tropy",
    "On the Nature of Entropy",
];
let text = [
    "Value is not a thing but a relation between people,",
    "hidden under the form of a relation between things.",
    "Labour time measures it only where markets have",
    "made the work of each producer comparable to all.",
];
let line = |text: String, left, top, right| Line { text, left, top, right, bottom: top + 10.0 };
let pages: Vec<Page> = (1..=4)
    .map(|n| Page {
        page: n,
        lines: vec![
            line(heads[n as usize - 1].to_string(), 150.0, 30.0, 290.0),
            line(text[n as usize - 1].to_string(), 72.0, 60.0, 360.0),
            line((n + 40).to_string(), 210.0, 600.0, 222.0),
        ],
    })
    .collect();

// Every head is furniture.
let found = furniture(&pages);
assert!((1..=4).all(|n| found.contains(&(n, 0))));
```

#### `Line::left`

```rs
pub left: f32
```

The x of the left edge of the line's box, in points from the left edge of the page. `furniture` matches lines set flush left on different pages by it, and with `right` it decides whether two lines overlap across the page.

**Errors and panics.** None.

**Example.**

```rust
use furniture_detection::Line;

// A page number set flush left: its left edge stays at the 72-point margin
// while its right edge moves as the number grows.
let folio = Line {
    text: "100".to_string(),
    left: 72.0,
    top: 740.0,
    right: 90.0,
    bottom: 750.0,
};
```

#### `Line::top`

```rs
pub top: f32
```

The y of the top edge of the line's box, in points down from the top edge of the page. With `bottom` it gives the line's vertical centre, the only vertical position that `furniture` uses.

**Errors and panics.** None.

**Example.**

```rust
use furniture_detection::Line;

// The running head sits above the page number, so its top is the smaller.
let head = Line {
    text: "On the Nature of Entropy".to_string(),
    left: 150.0,
    top: 30.0,
    right: 290.0,
    bottom: 40.0,
};
let folio = Line {
    text: "41".to_string(),
    left: 210.0,
    top: 740.0,
    right: 222.0,
    bottom: 750.0,
};
assert!(head.top < folio.top);
```

#### `Line::right`

```rs
pub right: f32
```

The x of the right edge of the line's box, in points from the left edge of the page. `furniture` matches lines set flush right on different pages by it, and with `left` it decides whether two lines overlap across the page.

**Errors and panics.** None.

**Example.**

```rust
use furniture_detection::Line;

// A page number set flush right: its right edge stays at 540 points while its
// left edge moves as the number grows.
let folio = Line {
    text: "100".to_string(),
    left: 522.0,
    top: 740.0,
    right: 540.0,
    bottom: 750.0,
};
```

#### `Line::bottom`

```rs
pub bottom: f32
```

The y of the bottom edge of the line's box, in points down from the top edge of the page, so it is no less than `top`. `furniture` uses only the centre between `top` and `bottom`, so a reader that gives only a line's baseline can pass the baseline as both.

**Errors and panics.** None.

**Example.**

```rust
use furniture_detection::Line;

// A reader gave only the baseline of this page number, 750 points down.
let baseline = 750.0;
let folio = Line {
    text: "41".to_string(),
    left: 210.0,
    top: baseline,
    right: 222.0,
    bottom: baseline,
};
```

## Recipes

### Converting a PDF reader's bottom-origin boxes

PDF measures positions from the bottom-left corner of the page with y growing upward, so a reader that passes its coordinates through gives boxes that way. `furniture` takes the line with the smaller y as the higher one when it looks for the line above, so such boxes must be flipped. A box from y0 at its foot up to y1 at its head has `top = height - y1` and `bottom = height - y0`, where `height` is the page's height, or the y of its top edge when the page's box does not start at 0. Use each page's own height, since the pages of one PDF can differ in size. The x coordinates stay as they are.

```rust
use furniture_detection::Line;

/// Converts a box from (x0, y0) at its lower left to (x1, y1) at its upper
/// right, measured from the bottom of a page `height` points tall.
fn from_bottom_origin(text: &str, [x0, y0, x1, y1]: [f32; 4], height: f32) -> Line {
    Line {
        text: text.to_string(),
        left: x0,
        top: height - y1,
        right: x1,
        bottom: height - y0,
    }
}

// A running head 30 points below the top of a US Letter page, 792 points tall.
let head = from_bottom_origin("On the Nature of Entropy", [150.0, 752.0, 290.0, 762.0], 792.0);
assert_eq!((head.top, head.bottom), (30.0, 40.0));
```

### Passing pages without text

A blank page, a full-page figure or a scanned page without a text layer gives no lines, but it still needs its `Page`, with empty `lines`, so that the page numbers remain positions in the document. A copy from another page expects a printed page number to move on by the difference between the two pages' numbers, and the search for copies expects that difference to grow with the distance between the pages in the slice. Dropping a page without text breaks one of these expectations however the pages after it are numbered: numbered again, the page after the gap is numbered one more than the page before it while its printed number is two more; left with their own numbers, the pages after it leave a gap that the search does not expect.

Numbering the pages by their position as you build them keeps both, because a page without text takes its number like any other:

```rust
use std::collections::HashSet;

use furniture_detection::{Line, Page, furniture};

// Each page's line of text; the third page is blank.
let text = [
    Some("Value is not a thing but a relation between people,"),
    Some("hidden under the form of a relation between things."),
    None,
    Some("Labour time measures it only where markets have"),
    Some("made the work of each producer comparable to all."),
];
let line = |text: String, left, top, right| Line { text, left, top, right, bottom: top + 10.0 };

// Each page's lines as a reader gives them, the blank page with none.
let read: Vec<Vec<Line>> = text
    .iter()
    .zip(41u32..)
    .map(|(text, folio)| match text {
        Some(text) => vec![
            line("On the Nature of Entropy".to_string(), 150.0, 30.0, 290.0),
            line(text.to_string(), 72.0, 60.0, 360.0),
            line(folio.to_string(), 210.0, 600.0, 222.0),
        ],
        None => Vec::new(),
    })
    .collect();

// Number the pages 1, 2, 3, ... by position, the blank page included.
let pages: Vec<Page> = read
    .into_iter()
    .zip(1..)
    .map(|(lines, page)| Page { page, lines })
    .collect();

let found = furniture(&pages);
let expected: HashSet<(u32, usize)> = [1, 2, 4, 5].into_iter().flat_map(|n| [(n, 0), (n, 2)]).collect();
assert_eq!(found, expected);
```

### Mapping the result back to lines

The set names each furniture line by its page number and its index, in no order. When the pages are numbered from 1 by position, `pages[page as usize - 1].lines[index]` is the line. To list the furniture in reading order, walk the pages and look each line up in the set:

```rust
use furniture_detection::{Line, Page, furniture};

/// The furniture lines of `pages` in reading order, each with its page number.
fn furniture_lines(pages: &[Page]) -> Vec<(u32, &Line)> {
    let found = furniture(pages);
    let mut lines = Vec::new();
    for page in pages {
        for (index, line) in page.lines.iter().enumerate() {
            if found.contains(&(page.page, index)) {
                lines.push((page.page, line));
            }
        }
    }
    lines
}

// The four pages of the quick start.
let text = [
    "Value is not a thing but a relation between people,",
    "hidden under the form of a relation between things.",
    "Labour time measures it only where markets have",
    "made the work of each producer comparable to all.",
];
let line = |text: String, left, top, right| Line { text, left, top, right, bottom: top + 10.0 };
let pages: Vec<Page> = (1..=4)
    .map(|n| Page {
        page: n,
        lines: vec![
            line("On the Nature of Entropy".to_string(), 150.0, 30.0, 290.0),
            line(text[n as usize - 1].to_string(), 72.0, 60.0, 360.0),
            line((n + 40).to_string(), 210.0, 600.0, 222.0),
        ],
    })
    .collect();

let listed: Vec<(u32, &str)> = furniture_lines(&pages)
    .into_iter()
    .map(|(page, line)| (page, line.text.as_str()))
    .collect();
assert_eq!(
    listed[..4],
    [(1, "On the Nature of Entropy"), (1, "41"), (2, "On the Nature of Entropy"), (2, "42")]
);
```

### Separating furniture from body text

To keep the furniture instead of discarding it, for instance to cite a passage by the page number printed on its page, split each page's lines in two:

```rust
use furniture_detection::{Line, Page, furniture};

/// Each page's lines split into its body text and its furniture, both in the
/// order of the page's lines.
fn split(pages: &[Page]) -> Vec<(Vec<&Line>, Vec<&Line>)> {
    let found = furniture(pages);
    pages
        .iter()
        .map(|page| {
            let (mut body, mut furniture_lines) = (Vec::new(), Vec::new());
            for (index, line) in page.lines.iter().enumerate() {
                if found.contains(&(page.page, index)) {
                    furniture_lines.push(line);
                } else {
                    body.push(line);
                }
            }
            (body, furniture_lines)
        })
        .collect()
}

// The four pages of the quick start.
let text = [
    "Value is not a thing but a relation between people,",
    "hidden under the form of a relation between things.",
    "Labour time measures it only where markets have",
    "made the work of each producer comparable to all.",
];
let line = |text: String, left, top, right| Line { text, left, top, right, bottom: top + 10.0 };
let pages: Vec<Page> = (1..=4)
    .map(|n| Page {
        page: n,
        lines: vec![
            line("On the Nature of Entropy".to_string(), 150.0, 30.0, 290.0),
            line(text[n as usize - 1].to_string(), 72.0, 60.0, 360.0),
            line((n + 40).to_string(), 210.0, 600.0, 222.0),
        ],
    })
    .collect();

let (body, furniture_lines) = &split(&pages)[1];
assert_eq!(body[0].text, "hidden under the form of a relation between things.");
assert_eq!(furniture_lines[1].text, "42");
```

### Removing furniture before joining lines into paragraphs

Whatever rule joins lines into paragraphs, run it on the body lines alone, so that a paragraph that runs over a page break joins its last line on one page to its first line on the next. With the furniture left in, the page number and the running head land inside a sentence that runs over the break: the four pages of the quick start would read `… a relation between people, 41 On the Nature of Entropy hidden under the form …`. The rule in this example is the simplest one, joining every body line with a space:

```rust
use furniture_detection::{Line, Page, furniture};

/// The body text of `pages`, furniture removed, its lines joined by spaces.
fn body_text(pages: &[Page]) -> String {
    let found = furniture(pages);
    let mut lines = Vec::new();
    for page in pages {
        for (index, line) in page.lines.iter().enumerate() {
            if !found.contains(&(page.page, index)) {
                lines.push(line.text.as_str());
            }
        }
    }
    lines.join(" ")
}

// The four pages of the quick start.
let text = [
    "Value is not a thing but a relation between people,",
    "hidden under the form of a relation between things.",
    "Labour time measures it only where markets have",
    "made the work of each producer comparable to all.",
];
let line = |text: String, left, top, right| Line { text, left, top, right, bottom: top + 10.0 };
let pages: Vec<Page> = (1..=4)
    .map(|n| Page {
        page: n,
        lines: vec![
            line("On the Nature of Entropy".to_string(), 150.0, 30.0, 290.0),
            line(text[n as usize - 1].to_string(), 72.0, 60.0, 360.0),
            line((n + 40).to_string(), 210.0, 600.0, 222.0),
        ],
    })
    .collect();

assert_eq!(
    body_text(&pages),
    "Value is not a thing but a relation between people, \
     hidden under the form of a relation between things. \
     Labour time measures it only where markets have \
     made the work of each producer comparable to all."
);
```
