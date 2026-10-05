# CrossText 1: Crossword Puzzle Format and Game Specification

**Version:** 1.0  
**File extension:** `.crosstext` (recommended), `.ct` (optional)  
**Format declaration:** `@crossword 1`  
**Document date:** 2026-10-05

This document completely defines CrossText version 1, a UTF-8 text format for blocked, rectangular crossword puzzles, and a baseline interactive crossword player. A developer needs no other CrossText document, schema, numbering convention, or example to implement it. The Markdown file is the specification; the code blocks labeled `crosstext` contain puzzle files.

CrossText stores puzzle geometry, clues, and optional solutions and presentation annotations. The grid determines entries and clue numbers. Entry coordinates and lengths, when supplied, are assertions against that geometry. They never override it. Ordinary cells hold one ASCII letter; rebus cells can hold several ASCII letters in one square. Clue and metadata text can use Unicode.

## Contents

1. [Scope and normative language](#1-scope-and-normative-language)
2. [Document structure and lexical rules](#2-document-structure-and-lexical-rules)
3. [Grammar](#3-grammar)
4. [Headers and metadata](#4-headers-and-metadata)
5. [Grid and solution sections](#5-grid-and-solution-sections)
6. [Entries and numbering](#6-entries-and-numbering)
7. [Clues](#7-clues)
8. [Answer assertions and answer text](#8-answer-assertions-and-answer-text)
9. [Cell annotations and givens](#9-cell-annotations-and-givens)
10. [Themes](#10-themes)
11. [Validation](#11-validation)
12. [Normalized data model](#12-normalized-data-model)
13. [Parser and writer requirements](#13-parser-and-writer-requirements)
14. [Rendering and clue navigation](#14-rendering-and-clue-navigation)
15. [Input behavior](#15-input-behavior)
16. [Checking, revealing, and completion](#16-checking-revealing-and-completion)
17. [Runtime state and persistence](#17-runtime-state-and-persistence)
18. [Errors, warnings, and limits](#18-errors-warnings-and-limits)
19. [Complete examples](#19-complete-examples)
20. [Conformance and acceptance tests](#20-conformance-and-acceptance-tests)
21. [Implementation checklist](#21-implementation-checklist)

## 1. Scope and normative language

**MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are requirement words. MUST requirements define conformance. SHOULD requirements can have justified exceptions. MAY features are optional.

CrossText 1 supports:

- Rectangular grids with open cells and black squares.
- Across and Down entries, numbered automatically.
- Mixed entry lengths, including rectangular and large grids.
- Complete, partial, or absent answer knowledge.
- Optional author assertions containing coordinates, directions, and lengths.
- Circles, rebus cells, immutable given values, themes, and clue enumerations.
- Unicode text in clues, metadata, and theme notes.
- Deterministic navigation, editing, checking, and completion behavior.

It does not define barred crosswords, diagonal entries, changing grids, multiple grids in one file, multi-cell characters, diagramless puzzles, alternate accepted answers, cryptographic answer protection, executable content, networking, scoring, or collaborative synchronization. Those require a future version or an application layer. Punctuation and spaces in a phrase answer are omitted from cell values. A clue enumeration can preserve its human-readable word lengths.

Two geometry profiles exist:

- **`traditional`**, the default: every open cell belongs to both an Across and a Down entry; every entry has at least three cells; all open cells form one orthogonally connected component.
- **`general`**: every open cell belongs to at least one entry; every entry has at least two cells; all open cells still form one orthogonally connected component. Two-letter entries and unchecked cells are allowed.

Both profiles use the same numbering and gameplay rules. A `general` puzzle must not be described as meeting the `traditional` profile merely because it looks like a conventional crossword. Symmetry is separately declared and is not required by either profile.

The words in a puzzle are an editorial matter. A validator does not consult a dictionary, judge clue quality, forbid abbreviations or duplicate answers, or require a particular block percentage.

## 2. Document structure and lexical rules

### 2.1 Encoding and lines

A file MUST be valid UTF-8. One UTF-8 byte-order mark is permitted only at the beginning and is ignored. No other byte-order mark receives special treatment. An embedded U+FEFF outside a string is not whitespace and is invalid.

Line endings MAY be LF or CRLF, including a mixture. A lone CR is invalid. A final line ending is optional. Implementations MUST normalize supported line endings before parsing lines. A physical line cannot continue onto the next line. Newlines within text are represented by string escapes.

Only ASCII space U+0020 and horizontal tab U+0009 count as horizontal whitespace. Nonbreaking spaces and other Unicode whitespace do not separate tokens. Blank lines and comment-only lines are ignored everywhere, including inside grid sections. Their presence does not count toward grid height.

### 2.2 Comments

A semicolon `;` starts a comment wherever it occurs outside a quoted string. The comment continues to the end of the physical line. Comment stripping MUST track quoted strings and escapes, not simply split on every semicolon.

```text
; A full-line comment
@title "Coffee; then a crossword" ; An end-of-line comment
[grid]
...#... ; # is a black square, not a comment marker
```

The characters `#`, `//`, and `--` do not introduce comments. There are no block comments. A semicolon inside a decoded string is ordinary text.

### 2.3 Quoted strings and escaping

All free text and all answer-sequence fields MUST be double-quoted strings. Strings follow the JSON string syntax specified here:

- Literal Unicode scalar values are allowed except `"`, `\`, and U+0000–U+001F.
- The only escapes are `\"`, `\\`, `\/`, `\b`, `\f`, `\n`, `\r`, `\t`, and `\u` followed by exactly four hexadecimal digits.
- A `\u` escape for a high UTF-16 surrogate MUST be immediately followed by a `\u` escape for its matching low surrogate. Their pair decodes to one Unicode scalar value. Unpaired surrogates are invalid.
- Unknown escapes such as `\q`, single quotes as delimiters, and literal multiline strings are invalid.

Examples:

```text
A1 "The word \"yes\""
A2 "A clue with a semicolon; it stays in the clue"
@notes "Line one\nLine two"
@title "Caf\u00E9 Crossword"
```

Decoded strings are plain text. A player MUST NOT execute HTML, Markdown, scripts, or URLs embedded in them. A player MAY display escaped line breaks as actual line breaks. Other decoded control characters MUST be rendered inertly, for example as visible symbols or spaces; they MUST NOT control a terminal or the host interface. Text is not implicitly trimmed or Unicode-normalized. Required nonempty text must contain at least one character other than Unicode whitespace or a control character.

For file solutions, ASCII uppercase letters are required. Runtime input has separate normalization rules in section 15. Metadata and clues retain their original case.

### 2.4 Headers and sections

The first nonblank, noncomment line MUST be `@crossword 1`, optionally with surrounding horizontal whitespace and a trailing comment. All header lines precede all sections. A header appearing after a section is an error.

A section header occupies its entire logical line after comment stripping, for example `[grid]`. Surrounding horizontal whitespace is allowed; whitespace inside the brackets is not. Built-in names are lowercase and case-sensitive.

Recognized sections are:

| Section | Required? | Purpose |
| --- | --- | --- |
| `[grid]` | Yes | Geometry only |
| `[clues]` | Yes | Exactly one clue for every derived entry |
| `[solution]` | No | A complete solution grid |
| `[answers]` | No | Entry assertions, optionally with answer sequences |
| `[cells]` | No | Circles and rebus capabilities |
| `[givens]` | No | Initial immutable cell values |
| `[themes]` | No | Theme membership and notes |

Each section MAY appear at most once. Sections MAY appear in any order. References are resolved after reading the whole document. Optional record sections MAY be empty. `[grid]` and `[solution]` MUST have the declared number of data rows; `[clues]` cannot be empty in a valid puzzle because a valid puzzle has at least one entry.

`[puzzle]`, `[across]`, and `[down]` are not aliases in version 1. Across and Down clues share `[clues]`. Letter-filled `[grid]` sections from informal earlier sketches are not valid CrossText 1.

### 2.5 Numbers, identifiers, and properties

Unsigned integers use ASCII decimal digits without a sign or leading zeros, except the integer zero itself when a field permits zero. All indices, dimensions, lengths, revisions, and clue numbers in this format are positive, so `0` is invalid in those fields.

Coordinates are `row,column`, with no internal whitespace. Coordinates are **one-based**: `1,1` is the top-left square. Width is the number of columns; height is the number of rows.

Entry identifiers have the form `A` or `D` followed by a positive number without leading zeros: `A1`, `D1`, `A12`. These are case-sensitive. Theme identifiers are an ASCII letter followed by zero to 31 ASCII letters, digits, `_`, or `-`. Theme identifiers and entry identifiers have separate namespaces.

Record fields are separated by one or more spaces or tabs, except for the internally compact grid and answer syntax. A property is `name=value`, without whitespace around `=`. Properties can appear in any order. Duplicate properties and unknown built-in properties are errors. Values containing spaces must use quoted strings where the property grammar allows them.

## 3. Grammar

The following EBNF defines the syntax after decoding UTF-8, normalizing line endings, removing an optional initial BOM, and stripping comments outside strings. Leading and trailing horizontal whitespace on each line is ignored. Blank lines are ignored. Semantic restrictions elsewhere in this document also apply.

`{ X }` means zero or more repetitions; `[ X ]` means optional; `|` means alternatives. Quoted EBNF literals are exact source characters. `STRING` is the quoted-string syntax from section 2.3; EBNF's own quotation marks are not file delimiters unless the literal indicates one.

```ebnf
DOCUMENT         = VERSION_LINE, { HEADER_LINE }, { SECTION } ;
VERSION_LINE     = "@crossword", WS, "1" ;
HEADER_LINE      = SIZE_HEADER | PROFILE_HEADER | SYMMETRY_HEADER
                 | TEXT_HEADER | REVISION_HEADER | EXT_HEADER ;
SIZE_HEADER      = "@size", WS, POSINT, "x", POSINT ;
PROFILE_HEADER   = "@profile", WS, ("traditional" | "general") ;
SYMMETRY_HEADER  = "@symmetry", WS, ("none" | "rotational-180") ;
REVISION_HEADER  = "@revision", WS, POSINT ;
TEXT_HEADER      = "@", TEXT_KEY, WS, STRING ;
TEXT_KEY         = "title" | "author" | "description" | "copyright"
                 | "license" | "source" | "language" | "date"
                 | "difficulty" | "notes" | "id" ;
EXT_HEADER       = "@", EXT_NAME, WS, STRING ;

SECTION          = GRID_SECTION | SOLUTION_SECTION | CLUES_SECTION
                 | ANSWERS_SECTION | CELLS_SECTION | GIVENS_SECTION
                 | THEMES_SECTION | EXT_SECTION ;
GRID_SECTION     = "[grid]", { GRID_ROW } ;
SOLUTION_SECTION = "[solution]", { SOLUTION_ROW } ;
CLUES_SECTION    = "[clues]", { CLUE_RECORD } ;
ANSWERS_SECTION  = "[answers]", { ANSWER_RECORD } ;
CELLS_SECTION    = "[cells]", { CELL_RECORD } ;
GIVENS_SECTION   = "[givens]", { GIVEN_RECORD } ;
THEMES_SECTION   = "[themes]", { THEME_RECORD } ;
EXT_SECTION      = "[", EXT_NAME, "]", { EXT_LINE } ;

GRID_ROW         = GRID_CELL, { [ WS ], GRID_CELL } ;
GRID_CELL        = "." | "#" ;
SOLUTION_ROW     = SOLUTION_CELL, { [ WS ], SOLUTION_CELL } ;
SOLUTION_CELL    = "#" | LETTER | REBUS_TOKEN ;
REBUS_TOKEN      = "{", LETTER, LETTER, { LETTER }, "}" ;
ANSWER_SEQUENCE  = ANSWER_CELL, { ANSWER_CELL } ;
ANSWER_CELL      = LETTER | REBUS_TOKEN ;

CLUE_RECORD      = ENTRY_ID, WS, STRING,
                   [ WS, "enumeration=", STRING ] ;
ANSWER_RECORD    = ENTRY_ID, WS, COORD, WS, DIRECTION, WS, POSINT,
                   [ WS, STRING ] ;
DIRECTION        = "ACROSS" | "DOWN" ;
CELL_RECORD      = COORD, WS, CELL_PROPERTY, { WS, CELL_PROPERTY } ;
CELL_PROPERTY    = "circle=", BOOL | "rebus=", BOOL ;
GIVEN_RECORD     = COORD, WS, STRING ;
THEME_RECORD     = THEME_ID, WS, STRING, WS, THEME_PROPERTY,
                   { WS, THEME_PROPERTY } ;
THEME_PROPERTY   = "entries=", ENTRY_LIST | "note=", STRING
                 | "reveal=", ("always" | "completion") ;
ENTRY_LIST       = ENTRY_ID, { ",", ENTRY_ID } ;

COORD            = POSINT, ",", POSINT ;
ENTRY_ID         = ("A" | "D"), POSINT ;
THEME_ID         = ASCII_ALPHA, { ASCII_ALPHA | DIGIT | "_" | "-" } ;
EXT_NAME         = "x-", LOWER_ALPHA,
                   { LOWER_ALPHA | DIGIT | "-" } ;
BOOL             = "true" | "false" ;
POSINT           = NONZERO_DIGIT, { DIGIT } ;
LETTER           = "A" | "B" | ... | "Z" ;
ASCII_ALPHA      = "A" | ... | "Z" | "a" | ... | "z" ;
LOWER_ALPHA      = "a" | ... | "z" ;
DIGIT            = "0" | ... | "9" ;
NONZERO_DIGIT    = "1" | ... | "9" ;
WS               = (" " | TAB), { " " | TAB } ;
```

Each header, section header, grid row, and record above occupies one physical line. Sections terminate at the next section header or EOF. `EXT_LINE` is an opaque logical line within an extension section; a section-header-shaped line starts a new section and cannot be extension content. The grammar alone does not enforce record counts, permitted bounds, or required headers/sections.

The optional final `STRING` in an answer record is decoded first, then parsed using `ANSWER_SEQUENCE`. There are no spaces, tabs, blocks, or empty cells in that decoded sequence. A given's decoded string is a single cell value, **not** an answer sequence: write `"HE"`, not `"{HE}"`.

## 4. Headers and metadata

`@crossword` and `@size` are required. Every header key occurs at most once. There are no repeatable author or notes headers; put multiple authors or paragraphs into one string if needed.

| Header | Value and default | Meaning |
| --- | --- | --- |
| `@crossword` | Bare integer `1`; required | Major format version |
| `@size` | Bare `WIDTHxHEIGHT`; required | Exact geometry; lowercase `x` |
| `@profile` | `traditional` or `general`; default `traditional` | Geometry validation profile |
| `@symmetry` | `none` or `rotational-180`; default `none` | Enforced block-pattern symmetry |
| `@title` | Nonempty string; display default `Untitled Crossword` | Puzzle title |
| `@author` | String; default empty | Attribution |
| `@description` | String; default empty | Introductory text |
| `@copyright` | String; default empty | Copyright statement |
| `@license` | String; default empty | License text or identifier |
| `@source` | String; default empty | Attribution or origin, possibly a URL |
| `@language` | Nonempty string; default `en` | Informational language tag; `en`, `en-KE`, etc. are recommended |
| `@date` | String `YYYY-MM-DD`; no default | Gregorian publication date |
| `@difficulty` | String; default empty | Informational label, e.g. `Medium` |
| `@notes` | String; default empty | General solver-visible notes |
| `@id` | Nonempty string; absent by default | Publisher-assigned identity |
| `@revision` | Positive integer; default `1` | Publisher-assigned content revision |

The date MUST be an actual calendar date with a four-digit year from `0001` to `9999`, including normal Gregorian leap-year rules. `@language` is informational; readers need not use a language-tag registry. It does not expand the A–Z cell alphabet. An author may therefore supply non-English clue text, but answers still use ASCII letters in version 1.

Metadata does not change input rules, numbering, minimum lengths, or checking. Do not infer a profile from language or a title. Source URLs are data, not instructions to fetch anything.

### 4.1 Extensions

Metadata keys and section names matching `x-...` MAY be used for **nonessential, nonsemantic** application information, such as an editor's private notes. The suffix begins with a lowercase letter and contains only lowercase letters, digits, and hyphens; the total name length cannot exceed 64 characters.

```text
@x-editor "Crossword Workshop"
[x-editor-notes]
This record is opaque to other readers.
```

A reader MUST accept well-formed extension headers and sections, retain their names and data in its parsed result, and ignore them for core puzzle behavior. Unknown unprefixed headers, sections, and properties are errors. Extensions cannot redefine cells, add playable entry directions, override answers, or require custom behavior. If the puzzle depends on such behavior, it is not a conforming version 1 file.

## 5. Grid and solution sections

### 5.1 Geometry: `[grid]`

The grid contains exactly `HEIGHT` data rows, each with exactly `WIDTH` cell tokens:

- `.`: an open square.
- `#`: a black square.

Spaces or tabs between cell tokens are optional and do not create cells. Compact and spaced rows have identical meaning:

```text
#...#
# . . . #
```

`[grid]` MUST NOT contain letters, question marks, dashes, commas, or rebus braces. Givens belong in `[givens]`. Answers belong in `[solution]` or `[answers]`. A file with no solution information is a valid playable puzzle.

### 5.2 Complete answers: `[solution]`

If present, a solution grid has exactly the same dimensions and block positions as `[grid]`:

- `#`: the corresponding black square.
- `A`–`Z`: the answer value for one ordinary or rebus-capable cell.
- `{LETTERS}`: one cell whose answer has 2–32 uppercase ASCII letters.

Whitespace may separate tokens, but cannot occur within a braced token. Adjacent braced and single-letter tokens are unambiguous:

```text
{HE}AR
{HE} A R
```

Both lines contain **three cells**, with values `HE`, `A`, and `R`. Braces are delimiters and are not part of the expected value. A braced value requires `rebus=true` for that coordinate in `[cells]`. A marked rebus cell MAY have a one-letter solution, written without braces. `{A}`, `{}`, lowercase letters, nested braces, and `{ICE CREAM}` are invalid.

Every open square in a solution grid MUST have a value. `.` is not allowed in `[solution]`; partial answer knowledge uses `[answers]`. A solution grid is never initially copied into the player's entries. Only `[givens]` supplies initial letters.

### 5.3 Answer knowledge

Define the expected-value map `expected[row,column]`, initially unknown for every open square. A solution grid, if present, populates every open square. Answer sequences in `[answers]` add expected values at their derived coordinates. All overlapping sources MUST agree cell by cell, including complete rebus strings.

A file has **complete answer knowledge** if every open square has an expected value after combining these sources. It has **partial answer knowledge** if some but not all have one, and **no answer knowledge** if none have one. Givens do not create expected values. They may be checked against known expected values, but a given by itself is not an independently supplied solution.

An application MUST expose checking only to the extent supported by this map. It MUST NOT treat unknown cells as correct or treat an arbitrary filled grid as solved.

## 6. Entries and numbering

### 6.1 Entry discovery

An Across run is a maximal uninterrupted horizontal sequence of open cells. A Down run is a maximal uninterrupted vertical sequence of open cells. A run of **at least two cells** is an entry. A one-cell run is not an entry in either profile.

An open square starts an Across entry when:

1. It is in column 1 or the square immediately to its left is black; and
2. The square immediately to its right exists and is open.

It starts a Down entry when:

1. It is in row 1 or the square immediately above it is black; and
2. The square immediately below it exists and is open.

A run extends until the next black square or the grid edge. A rebus square is one cell for geometry, entry length, and numbering. Do not stop or renumber at circled, given, or rebus squares.

Discovery always uses the two-cell rule first. Profile checks then reject two-cell entries under `traditional`. A parser must not silently remove two-cell entries to make a traditional grid appear valid.

### 6.2 Numbering

Visit cells in row-major order: top row left to right, then the next row. Start a counter at 1. If a square starts either direction, give it the next number and increment once. If it starts both directions, they share that number. An open square starting neither direction has no number.

For each Across entry starting at number `n`, its identifier is `An`; for each Down entry it is `Dn`. Numbers are shared across both directions, not independently generated. Gaps in either clue list are normal.

```text
number = 0
for row = 1 through height:
    for column = 1 through width:
        if cell is black: continue
        acrossStart = open to right AND (at left edge OR black to left)
        downStart   = open below    AND (at top edge  OR black above)
        if acrossStart OR downStart:
            number = number + 1
            cell.number = number
            if acrossStart: create Anumber with its maximal horizontal run
            if downStart:   create Dnumber with its maximal vertical run
```

Out-of-bounds neighbors are not open. The resulting entries are immutable structural records. Their lengths are cell counts; they are not counts of letters, bytes, or clue enumeration symbols.

### 6.3 Example of derived numbers

For this geometry:

```text
#...#
.....
.....
.....
#...#
```

The numbered squares are:

```text
#  1  2  3  #
4  .  .  .  5
6  .  .  .  .
7  .  .  .  .
#  8  .  .  #
```

Across: `A1` at `1,2` length 3; `A4` at `2,1` length 5; `A6` at `3,1` length 5; `A7` at `4,1` length 5; `A8` at `5,2` length 3.

Down: `D1` at `1,2` length 5; `D2` at `1,3` length 5; `D3` at `1,4` length 5; `D4` at `2,1` length 3; `D5` at `2,5` length 3.

## 7. Clues

Each record in `[clues]` has an entry identifier, a nonempty clue string, and an optional display enumeration:

```text
A1 "Curved path"
A4 "Evidence that establishes a claim" enumeration="(5)"
D1 "Symbol that points a direction"
```

Every derived entry MUST have exactly one clue. A clue MUST refer to a derived entry of the same direction and number. A nonexistent identifier, duplicate clue, or missing clue is an error. Records may be in any order; presentation sorts by numeric clue number within each direction.

`enumeration` is a nonempty string used only for display, such as `"(3, 4)"`, `"(2-3)"`, or `"(4 letters in 3 cells)"`. It MUST NOT determine geometry or split a solution. There is no automatic mathematical validation of enumerations: phrase conventions and rebuses differ. If absent, the player SHOULD show the entry's cell count, explicitly labeled as cells where ambiguity is possible.

Clue text may contain human-readable cross-references such as `"See 8-Across"`. Version 1 does not parse or enforce these references. All navigation comes from actual entry identifiers in the app's clue list, not guessed identifiers in prose.

## 8. Answer assertions and answer text

`[answers]` is optional. Each record asserts an existing entry's start coordinate, direction, and cell length, and MAY supply its answer sequence:

```text
A4 2,1 ACROSS 5 "PROOF"
D1 1,2 DOWN 5 "ARROW"
A6 3,1 ACROSS 5
```

The complete syntax is:

```text
ENTRY_ID ROW,COLUMN DIRECTION CELL_LENGTH ["ANSWER_SEQUENCE"]
```

A record without an answer string is a structural assertion only. It does not create answer knowledge. The section may assert any subset of entries, but an entry may appear at most once. No entry is created by an assertion.

For every record, all of the following MUST hold:

1. The identifier exists in the derived entry map.
2. `A` corresponds to `ACROSS`, or `D` to `DOWN`.
3. The coordinate is exactly the entry's derived start.
4. The asserted length equals the derived cell count.
5. If an answer sequence is supplied, it decodes to exactly that many cells.
6. Every decoded value is uppercase ASCII letters and obeys the target cell's rebus rules.
7. Every value agrees with any value supplied by another answer or `[solution]` at that cell.

An answer sequence is compact: ordinary letters each represent a cell; a braced token represents one rebus cell. It contains no block markers or whitespace, even within the string. Examples:

| Answer sequence | Cell values | Cell count | Letter count |
| --- | --- | --- | --- |
| `"PROOF"` | `P`, `R`, `O`, `O`, `F` | 5 | 5 |
| `"{HE}AR"` | `HE`, `A`, `R` | 3 | 4 |
| `"ICE{CREAM}"` | `I`, `C`, `E`, `CREAM` | 4 | 8 |

An answer such as `"NEW YORK"` is invalid. A seven-cell answer is `"NEWYORK"`, with `enumeration="(3, 4)"` on its clue if desired. Do not invent brace boundaries from the flattened answer text.

Answers need not be supplied for both directions. All Across answers alone supply a full expected-value map in a traditional puzzle. Partial assertions with answer strings are permitted and do not make an incomplete `[solution]` permissible.

## 9. Cell annotations and givens

### 9.1 `[cells]`

A cell record applies boolean presentation or input properties to one open square:

```text
1,2 circle=true
3,4 rebus=true
4,3 circle=true rebus=true
```

The defaults are `circle=false` and `rebus=false`. A record MUST include at least one property. Each coordinate may occur at most once, and only `circle` and `rebus` are supported properties. Explicit `false` is permitted.

A coordinate outside the grid or on a black square is an error. A circle is a visual annotation; it does not change the answer. A rebus property permits a cell value of 1–32 ASCII letters. A nonrebus cell permits exactly one letter. Empty runtime cells are allowed in either case.

Rebus marking is explicit even when a solution or a given exposes a multi-letter token. Missing `rebus=true` is an error; readers MUST NOT silently add it. This permits the same input interface for files with and without solutions.

### 9.2 `[givens]`

A given record supplies one initial, immutable cell value:

```text
1,1 "HE"
2,2 "R"
```

The coordinate MUST be an open square and may appear at most once. The decoded value contains only uppercase `A`–`Z`. It has exactly one letter on an ordinary square or 1–32 letters on a marked rebus square. It contains no braces, spaces, or punctuation.

A given MUST match any known expected value at that coordinate. Unknown expected values remain unknown. Givens are visible immediately, participate in entry fill status, and cannot be overwritten, erased, pasted over, revealed over, or undone into a different value. They are restored by a full reset.

The player's initial state is otherwise blank. Circles and rebus markings alone do not prefill letters.

## 10. Themes

`[themes]` groups entries and provides plain-text notes. It has no effect on numbering or correctness.

```text
T1 "Shared sound" entries=A1,D1 note="Both answers contain a two-letter cell." reveal=completion
```

Each record contains a unique theme identifier, a nonempty title string, and these properties:

| Property | Requirement | Meaning |
| --- | --- | --- |
| `entries` | Required, once | Nonempty comma-separated list of existing entry identifiers |
| `note` | Optional, once; default empty | Explanatory text |
| `reveal` | Optional, once; default `always` | When the note becomes visible |

Spaces are not allowed inside the entry list. Duplicate members in one theme are errors. An entry may belong to several themes. All references must exist. Themes render in source order, and members appear in the supplied order.

Theme titles and membership are always visible. `reveal=completion` hides only the note until **fill completion**: every open square has a nonempty value. It does not require verified correctness. Once unlocked during the current attempt, that note remains available until a full reset, even if a square is later cleared. This rule works for solution-free puzzles too. A title or a clue can itself reveal theme information; authors should choose their text accordingly.

A player MUST make every theme and its permitted note accessible, for example in a theme panel. It MAY additionally highlight members or allow selecting a member entry from that panel. Themes do not imply cell circles; circles must be declared in `[cells]`.

## 11. Validation

A valid file satisfies all syntax, bounds, and semantic requirements. Validation MUST complete before the player opens an interactive puzzle. An editor MAY display an invalid draft, but MUST label it invalid and must not present it as a validated game.

### 11.1 Validation order

A practical validator uses the following order:

1. **Transport:** byte limits, UTF-8, BOM, line endings, and line limits.
2. **Syntax:** strings, header declarations, section names, record shapes, property names, duplicate declarations, and version support.
3. **Geometry:** dimensions, row counts, row widths, token legality, at least one open square, and connectivity.
4. **Derivation:** maximal runs, entry starts, shared numbering, coordinate-to-entry mappings.
5. **Profile and symmetry:** entry lengths, checked-cell requirements, declared symmetry.
6. **References:** clue coverage, assertions, cell/given coordinates, and theme members.
7. **Answers:** token lengths, rebus capabilities, crossing agreement, solution geometry, and given agreement.

A validator MAY collect multiple independent errors. It MUST NOT manufacture derived entries from an invalid geometry or cascade thousands of missing-clue errors after failing to parse a grid. Once geometry cannot be determined reliably, stop dependent validation and report the original errors.

### 11.2 Geometry requirements

- Width and height are positive integers within the version 1 bounds.
- Grid rows and widths exactly match `@size`.
- The grid contains at least one open square and at least one derived entry.
- All open squares are orthogonally connected. Diagonal contact does not connect squares.
- Every open square belongs to at least one derived entry.
- Under `traditional`, every open square belongs to an Across and a Down entry and all entries have at least three cells.
- Under `general`, derived entries may have two cells and a square may belong to only one direction.
- If `@symmetry rotational-180` is declared, the block/open state of `(r,c)` equals that of `(height+1-r,width+1-c)` for every coordinate.

Symmetry checks only black-square geometry. Letters, numbers, circles, rebus declarations, givens, clues, and theme membership need not be symmetric. `@symmetry none` imposes no symmetry rule. Readers MUST NOT add blocks, trim outer rows, or transpose dimensions to repair a grid.

### 11.3 Record and answer requirements

- Exactly one clue exists for every derived entry, with no extra identifiers.
- Each answer assertion identifies a derived entry and agrees with its coordinate, direction, and cell length.
- Each solution or answer value obeys its target square's single-letter or rebus capability.
- All known expected values agree at crossings. Agreement means identical full strings, not equal first letters or equal flattened words.
- A solution grid, if supplied, is complete and shares the exact block pattern.
- Every given is legal and agrees with any known expected value.
- All theme references exist; all record IDs, coordinates, and property names are unique where specified.
- Mandatory text and required properties are present and nonempty.

Absence of solutions, optional metadata, themes, circles, or givens is not an error. Partial answer knowledge is not an error. An app MUST NOT weaken profile requirements simply because an author omitted `[answers]`.

### 11.4 Suggested editorial warnings

A validator MAY warn about repeated flattened answers, suspicious clue enumerations, uncommon lengths, missing attribution, or answers that appear in visible notes. Such warnings do not invalidate an otherwise conforming file. No dictionary lookup is needed for conformance.

## 12. Normalized data model

After successful parsing and validation, implementations SHOULD produce an equivalent model to the following. This is an abstract data model, not a required programming language or a second interchange format.

```text
Puzzle:
  formatVersion: 1
  width, height: positive integers
  profile: traditional | general
  symmetry: none | rotational-180
  metadata: decoded values with documented defaults
  extensions: retained extension headers and section bodies
  cells: height × width array
  entriesById: map from A<number>/D<number> to Entry
  acrossOrder: entry IDs sorted by numeric number
  downOrder: entry IDs sorted by numeric number
  navigationOrder: acrossOrder followed by downOrder
  themes: source-ordered list
  answerCoverage: none | partial | complete

Cell:
  row, column: one-based integers
  blocked: boolean
  number: positive integer or absent
  acrossId, downId: entry identifier or absent
  circle: boolean, default false
  rebus: boolean, default false
  given: ASCII token or absent
  expected: ASCII token or unknown

Entry:
  id, number, direction
  start: row,column
  cells: ordered coordinate list, left-to-right or top-to-bottom
  cellLength: length(cells)
  clue: decoded nonempty plain text
  enumeration: decoded plain text or absent
  themeIds: identifiers in theme source order

Theme:
  id, title, memberEntryIds, note
  reveal: always | completion
```

Black squares have no number, entry memberships, given, or expected value, and no circle or rebus capability. The normalized defaults for their boolean annotation fields can be `false`.

Maintain **one runtime value per open cell**. Do not store independent Across and Down strings and attempt to reconcile them later. Each entry's visible answer is read from its ordered cell coordinates; editing an intersection immediately updates both entries.

Keep the immutable puzzle model separate from mutable play state. In particular, an expected value is not a player's value, and a supplied answer is not an initial entry.

## 13. Parser and writer requirements

### 13.1 Parsing algorithm

A conforming reader MUST support every built-in section, both geometry profiles, rebuses, circles, givens, themes, and all permitted solution-knowledge states within the stated limits. It cannot silently ignore a supported semantic feature.

A suitable algorithm is:

```text
read bytes and enforce file-size bound
strictly decode UTF-8; remove optional leading BOM
normalize CRLF to LF; reject lone CR
split into physical lines; retain source locations

for each line:
    scan for a semicolon outside quoted strings
    obtain the logical content before that comment
    trim leading/trailing ASCII spaces and tabs
    if empty: continue
    if before all sections and line starts with @:
        parse one header, rejecting duplicates and trailing tokens
    else if line is a section header:
        select that section, rejecting duplicates and unknown names
    else:
        parse one row or record for the selected section
        if no section exists, report unexpected content

require @crossword 1 as the first substantive line
require @size, [grid], and [clues]
validate dimensions and grid; derive entries and numbers
validate profile and symmetry
resolve all clue, cell, given, answer, and theme references
build expected-value map; reject conflicts
validate givens against known expected values
return a normalized puzzle or structured errors
```

This sketch does not permit checking the version only after unsafe parsing: an implementation SHOULD detect an unsupported version as soon as it reads the declaration. Parsing must not depend on the order of sections. Buffer records and resolve forward references later.

A solution tokenizer MUST recognize a braced token as one cell. A string decoder MUST decode escapes before applying answer/given semantic checks. Core record parsers MUST consume the whole logical line; extra words or trailing properties are errors. Headers are line records too.

Extension bodies are opaque and need not satisfy the core string or record grammar. Comment scanning in them still respects quote/escape boundaries on each physical line, but an unmatched quote in opaque extension text is not itself a core-string error. A section header terminates an extension body. A reader SHOULD retain original extension-body lines, including comments, so a writer can preserve them.

### 13.2 Writer behavior

A conforming writer emits valid version 1 files. A canonical writer SHOULD:

- Use UTF-8 without BOM and LF line endings, with a final LF.
- Write `@crossword 1`, then `@size`, then profile/symmetry and metadata.
- Write sections in this order: grid, solution, answers, clues, cells, givens, themes, extensions, omitting absent optional sections.
- Use compact grid and solution rows without intercell whitespace.
- Emit uppercase solution tokens and braces only for multi-letter cell values.
- Sort clues and answer assertions by increasing number, Across before Down when numbers tie.
- Sort cell and given records by row then column.
- Preserve theme order and member order.
- Quote strings correctly, escaping control characters, quotes, and backslashes.
- Preserve extension data without interpreting it.

Comments, spacing, original escape spelling, and metadata order need not survive canonical serialization. A writer claiming **lossless source editing** MUST additionally preserve those source details for untouched records. Canonical round trips need only preserve the normalized semantic model, including expected-value coverage and optional annotations.

When exporting a solution-free copy, remove `[solution]` and remove the optional answer string from **every** `[answers]` record, or omit `[answers]` entirely. Givens remain by design. Clue enumerations, metadata, and theme notes may intentionally contain answer information and require an editorial decision. Merely hiding a solution panel does not remove answers from a downloadable file.

### 13.3 Unsupported versions

A declaration such as `@crossword 2` MUST produce an unsupported-version error. Do not guess that a later file is backward compatible. A missing declaration is a missing-header error, not an inferred version. File extension and MIME type never override the declaration.

## 14. Rendering and clue navigation

### 14.1 Initial rendering

A conforming baseline player MUST provide:

- A grid in the declared row/column orientation, with square cells.
- Black squares that cannot receive letters or be selected for entry.
- Small clue numbers at the upper-left of numbered open squares.
- Centered player values, initially blank except for visibly distinguished givens.
- Visible circles and a discoverable indicator/control for rebus-capable squares.
- Across and Down clue lists sorted by their numeric numbers.
- A visible active square, active direction, active entry, and active clue.
- A way to read title, attribution, optional metadata/notes, and theme information.
- Accessible controls for checking, revealing, resetting, and rebus editing when applicable.

A multi-letter rebus value is displayed inside its one square, with smaller text or wrapping as needed. A player MUST provide access to the full value without truncation, such as through its cell editor or accessible label. It MUST NOT enlarge the grid, add extra squares, or split the token across cells.

An ordinary filled square displays one letter. Empty rebus squares should have a neutral marker that reveals capability, not the hidden expected token. Expected answers must not appear in the visible grid or its accessible labels before the relevant reveal operation.

Colors and exact typography are application choices. Active square, active entry, circles, givens, and feedback MUST remain distinguishable without relying on color alone. The inactive crossing clue MAY also be marked.

### 14.2 Initial selection and ordered entries

Define `navigationOrder` as all Across entries in increasing numeric order, followed by all Down entries in increasing numeric order. If either direction has no entries, its list is empty.

On opening or fully resetting a puzzle, select the first entry in `navigationOrder`. Select its first editable square, or its start square if every square is a given. This sets the active direction and clue. A valid puzzle always has at least one entry.

When a clue is activated, select its entry and its first editable square, or its start if all are given. Do not automatically jump to the first blank square; filled editable squares remain selectable.

### 14.3 Selecting grid squares

On the first click/tap on an open square:

1. Keep the existing active direction if that square belongs to an entry in that direction.
2. Otherwise choose Across if available, otherwise Down.
3. Select that entry and the clicked square, even if it is a given.

Clicking/tapping the already active square toggles direction when it has both memberships. With only one membership, it keeps that entry. The player's highlight and visible active clue update together. Clicking a black square has no effect on puzzle selection.

### 14.4 Moving among entries

- **Tab:** activate the next entry in `navigationOrder`, wrapping from the last to the first.
- **Shift+Tab:** activate the previous entry, wrapping from the first to the last.
- **Enter** or **Space** while focused on the grid: toggle direction at the active square, if both memberships exist; otherwise do nothing.
- Activating an Across/Down clue or theme member selects that entry according to section 14.2.

These shortcuts apply only while the grid has input focus. Typing in a rebus editor or another control does not trigger grid shortcuts. Buttons or equivalent controls MUST offer the same navigation operations on touch devices.

A grid that intercepts Tab MUST also provide a documented Escape action to leave grid interaction and focus a labeled control outside the grid. Ordinary host-interface focus navigation resumes outside the grid. This avoids a keyboard trap.

### 14.5 Arrow keys

Left/Right move geometrically within the current row; Up/Down within the current column. In the indicated direction, scan to the nearest open square, skipping black squares. Do not wrap across edges. If there is no such square, selection is unchanged.

On arrival, Left/Right choose the destination's Across entry when it has one; Up/Down choose its Down entry when it has one. If the preferred direction does not exist, retain the previous direction if available, otherwise use the destination's only membership. No values change.

This geometric navigation can move to another entry across a block. Automatic advancement after typing is different: it stays within the current entry.

### 14.6 Accessibility and small screens

All controls MUST be keyboard operable and have meaningful accessible names. An open square's accessible description MUST make its row, column, clue number if any, value/blank state, circle/rebus/given status, and active direction available without exposing expected answers. The current clue must be readable while editing the grid.

A small-screen layout MAY show one clue list at a time, but both directions and every clue remain reachable. Resizing or zooming must not change coordinates or lose runtime values. A player MAY offer other shortcut layouts or navigation preferences, but MUST document their mapping to the same operations; the behavior specified above is the default baseline.

## 15. Input behavior

### 15.1 Runtime values and normalization

A player's open-cell value is either empty or an uppercase ASCII token. Empty is not represented by a literal dot in runtime state.

For runtime input only:

- ASCII `a`–`z` normalize to `A`–`Z`.
- ASCII `A`–`Z` are retained.
- Ordinary single-letter entry rejects all other characters.
- Do not silently transliterate accented letters or remove punctuation from answer values.

For input methods with composition, wait for a committed text value before validating or changing the grid. Partial composition events MUST NOT repeatedly advance or overwrite cells.

An ordinary cell's nonempty value has exactly one letter. A marked rebus cell's nonempty value has 1–32 letters. Given values are immutable under every operation.

### 15.2 Ordinary typing and advancement

With an editable active square, typing one valid ASCII letter replaces its whole current value with that letter, including when the active square is rebus-capable. It clears current checking feedback for that square and updates both crossing entries.

After a successful replacement, move to the next editable square **later in the active entry's ordered cell list**, skipping givens but not skipping filled squares. At the end, stay on the square just changed. Do not wrap, switch clues, or advance to another entry automatically.

Typing while a given is active changes nothing and provides non-disruptive feedback that it is fixed. Invalid input changes neither selection nor values. To enter a multi-letter token, use the rebus editor or the structured paste syntax below.

### 15.3 Rebus editor

A player MUST provide an explicit, labeled rebus editor for a marked rebus square, reachable from keyboard and touch. Its exact shortcut is an application choice and must be discoverable.

- It edits only the active square.
- It accepts 1–32 ASCII letters and normalizes lowercase.
- Committing an empty value clears the square.
- Committing a nonempty value replaces the entire token and uses the ordinary advancement rule.
- Clearing via an empty commit does not advance.
- Canceling preserves value and selection.
- It rejects invalid characters or excessive length without a partial commit.
- It cannot edit a given square or turn an unmarked square into a rebus square.

The expected answer is not a suggested default. The editor starts with the current player token, if any.

### 15.4 Backspace and Delete

**Delete** clears the active square if it is editable, without moving. At a given square it does nothing.

**Backspace** works as follows:

1. If the active square is editable and nonempty, clear it and stay there.
2. Otherwise, find the preceding editable square in the active entry, skipping givens. If one exists, select it and clear its entire value, even if that preceding square was already empty.
3. If none exists, do nothing.

A rebus token is cleared as a whole by grid Backspace/Delete. Character-by-character editing occurs only inside the rebus editor. Clearing invalidates current checking feedback for affected squares and updates both crossing entries.

### 15.5 Paste and multi-character committed text

A conforming player MUST provide a paste operation within the active entry. A committed multi-character input event outside the rebus editor is handled as the same operation.

Normalize ASCII lowercase to uppercase. Ignore ASCII spaces, tabs, CR, and LF **outside** braced tokens. The remaining input is a sequence of single letters and braced tokens with 2–32 letters. Examples:

```text
cat       -> C, A, T
C A T     -> C, A, T
{he}ar    -> HE, A, R
```

Any other character, unmatched brace, whitespace inside a braced token, empty token, or one-letter braced token makes the entire operation invalid. A wholly whitespace input is a no-op.

Paste begins at the active square, which must be editable, and targets that square followed by later editable squares in the active entry, skipping givens. It overwrites existing values. Each token goes into one target square; it does not spill from a rebus square into neighbors.

Before committing, validate all tokens and targets. If there are more tokens than available squares, or a multi-letter token targets an unmarked square, reject the entire paste. No partial write, no clipping, and no wrap are permitted. An invalid paste gives a useful message and preserves values and selection.

On success, paste is one undoable transaction. Clear checking feedback for all changed squares. Select the next editable square after the last target if one exists; otherwise select the last target. Do not change the active entry.

### 15.6 Clearing, undo, and reset

A player MUST expose Clear entry and Clear puzzle operations. They clear editable values and their current checking feedback; givens remain. They do not change selection, assistance history, or already-unlocked theme notes.

A player MUST support Undo for value-changing transactions, including typing, paste, rebus commit, clear, and reveal. Undo restores the prior player values and current checking feedback for affected cells, and their prior selection. Givens never become editable. Redo is recommended but optional.

A full **Reset attempt** returns to initial givens/blank values, initial selection, no current feedback, and no assistance or theme-unlock history. It starts a new undo history. Because this discards the current attempt, the UI SHOULD offer confirmation or an immediately usable recovery action.

These runtime controls do not modify the source file. An app may separately provide puzzle-authoring tools.

## 16. Checking, revealing, and completion

### 16.1 Cell comparison

For a check operation, compare the whole runtime token with the expected token, without flattening rebus boundaries or consulting clue prose.

| Situation | Cell check result |
| --- | --- |
| Player value empty | `blank` |
| Nonempty value, expected unknown | `unknown` |
| Nonempty value equals known expected value | `correct` |
| Nonempty value differs from known expected value | `incorrect` |

A blank remains blank even when its expected value is unknown. Unknown is never a synonym for correct. A rebus square containing `H` when its expected value is `HE` is incorrect, not partially correct. A circle does not affect comparison.

### 16.2 Check scopes and summaries

A player MUST offer Check cell, Check entry, and Check puzzle to the extent that known values exist in the selected scope. A wholly unknown scope can show a disabled operation with an explanation, or an explicit “No solution available” result. It must not claim success.

A check scope includes givens as well as editable squares. Return counts of `blank`, `unknown`, `correct`, and `incorrect`. Its aggregate result is:

1. `incorrect` if any square is incorrect;
2. Otherwise `incomplete` if any square is blank;
3. Otherwise `unverified` if any square is unknown;
4. Otherwise `correct`.

For partial answer knowledge, the app MUST communicate the extent checked, for example “12 correct, 2 incorrect, 3 blank, 5 without a solution.” Merely filling all cells cannot change unknown comparisons into known ones.

Correctness feedback appears after an explicit check operation. Live checking MAY be a separate, clearly labeled opt-in setting. The baseline does not flag wrong letters while typing. A check can mark incorrect values, but does not erase or replace them. Editing a square invalidates its displayed feedback immediately, including the feedback as seen from the crossing entry.

### 16.3 Reveal scopes

A player MUST offer Reveal cell, Reveal entry, and Reveal puzzle when the relevant scope has known answers. A reveal transaction:

- Copies known expected values into editable squares in the chosen scope.
- Leaves unknown-value squares and givens unchanged.
- Clears old checking feedback on changed squares.
- Records which editable squares received answer assistance.
- Reports how many squares could not be revealed because their expected values were unknown.

A reveal can target a correctly filled editable square; that square still counts as assisted because its answer was explicitly exposed. If no editable square in the scope has a known answer, explain that no answer can be revealed. No hidden network lookup is part of CrossText.

Reveal is undoable as a value change. Assistance history is monotonic within an attempt: undoing a reveal does not erase the fact that assistance was used. The app SHOULD distinguish a completed puzzle solved with assistance from one without reveal assistance; no scoring formula is prescribed.

### 16.4 Fill completion versus verified completion

Maintain two independent predicates:

```text
fillComplete = every open square has a nonempty player value
verifiedComplete = fillComplete
                   AND every open square has a known expected value
                   AND every player value equals that expected value
```

After each transaction, recompute these predicates. Givens count as filled. Rebuses count as filled when they contain any legal nonempty token, even if that token is wrong.

- If complete expected values are available and `verifiedComplete` is true, the app MAY automatically announce “Solved,” with assistance information.
- If `fillComplete` is true but expected values are partial or absent, use “Grid filled; correctness unverified.” Do not announce “Solved.”
- If complete answers are known but the filled grid is wrong, do not announce success. The baseline does not identify wrong squares until a check is requested.
- Completion must be recalculated if the player later edits or clears the grid. Any previously earned completion event may remain in attempt history, but must not misrepresent the current state.

Automatically testing the complete grid for the success event does not require showing cell-by-cell live checking. A solution-free puzzle is still fully playable; its completion can only be fill completion under this specification.

### 16.5 Ordinary and rebus examples

Expected `A` with player `a` becomes `A` at input time and checks correct. A file containing lowercase `a` as an expected value is invalid and is not normalized.

Expected cells `HE`, `A`, `R` require runtime values `HE`, `A`, `R`. Values `H`, `E`, `AR` are both wrongly segmented and, unless their target squares are marked accordingly, illegal runtime input. Equal flattened text is insufficient.

## 17. Runtime state and persistence

CrossText describes an authored puzzle, not an attempt save file. A player SHOULD keep progress separately, locally or through its own storage service. It MUST NOT rewrite `[solution]` with player guesses.

A minimal runtime state contains:

```text
values[row,column]          // empty or legal ASCII token
activeCell                 // open coordinate
activeEntryId              // an entry containing activeCell
currentCheckFeedback       // per-cell results for the value last checked
revealedCellsEver           // assistance history within this attempt
unlockedThemeNotes         // completion-gated notes unlocked this attempt
undoHistory                 // value/feedback/selection transactions
```

`activeDirection` can be derived from `activeEntryId`. Optional pencil marks, timing, statistics, and preferences belong to the app, not the file. If pencil entry is offered, it must obey the same value and crossing constraints and must not alter expected answers.

Persistence SHOULD associate a save with a cryptographic digest of the source file content, optionally together with `@id` and `@revision`. Using title alone is unsafe because different puzzles may share a title. Using only `@id` without a revision or content check can restore progress onto a changed grid.

When restoring a save, verify the source association, coordinates, entry selection, legal values, rebus capabilities, and immutable givens. Invalid or incompatible saved state MUST NOT corrupt the puzzle model; report the issue and offer a fresh attempt. Resume assistance and unlocked-note history if progress resumes. Recompute completion rather than trusting a saved “solved” flag.

A save file is an application-defined artifact and cannot claim to be a CrossText puzzle simply by adding undocumented sections. An author who wants a prefilled puzzle must intentionally create legal `[givens]` records.

## 18. Errors, warnings, and limits

### 18.1 Structured diagnostics

Every error SHOULD have a stable code, human-readable message, source line and column, and relevant entry identifier or coordinate. Line and column positions are one-based; columns count Unicode scalar values in the decoded physical source line, including original indentation. BOM removal does not occupy a column. For EOF errors, use the location immediately after the last source character. For malformed UTF-8, give a byte offset when a decoded column cannot be determined.

When a derived relationship fails, point to the asserting record if possible. A crossing conflict SHOULD include both contributing source locations. Diagnostics must explain the inconsistency without quietly choosing one source over another.

Recommended codes:

| Code | Meaning |
| --- | --- |
| `E_ENCODING` | Invalid UTF-8, unsupported line ending, or illegal BOM position |
| `E_LIMIT` | A mandatory version 1 limit is exceeded |
| `E_VERSION` | Unsupported `@crossword` version |
| `E_SYNTAX` | Malformed record, string, escape, property, or trailing content |
| `E_HEADER_MISSING` | Required header missing or version declaration not first |
| `E_HEADER_UNKNOWN` | Unknown unprefixed header |
| `E_DUPLICATE` | Duplicate header, section, record ID/coordinate, or property |
| `E_SECTION_MISSING` | Required section missing |
| `E_SECTION_UNKNOWN` | Unknown unprefixed section |
| `E_GRID_SIZE` | Wrong row count or cell count |
| `E_GRID_TOKEN` | Invalid geometry token |
| `E_GRID_EMPTY` | No open squares or no entries |
| `E_DISCONNECTED` | Open squares do not form one component |
| `E_ORPHAN_CELL` | Open square belongs to no entry |
| `E_UNCHECKED_CELL` | Traditional square lacks one direction |
| `E_ENTRY_LENGTH` | Entry violates profile's minimum cell count |
| `E_SYMMETRY` | Declared block symmetry does not hold |
| `E_CLUE_MISSING` | Derived entry lacks a clue |
| `E_ENTRY_UNKNOWN` | Record refers to a nonexistent entry |
| `E_ASSERTION` | Entry coordinate, direction, or length assertion differs |
| `E_COORDINATE` | Coordinate out of bounds or refers to a black square |
| `E_SOLUTION_SIZE` | Solution dimensions or block positions differ |
| `E_SOLUTION_TOKEN` | Illegal answer/given token or answer cell count |
| `E_REBUS_REQUIRED` | Multi-letter token on a square not marked for rebus |
| `E_ANSWER_CONFLICT` | Different expected values asserted at one square |
| `E_GIVEN_CONFLICT` | Given disagrees with known expected value |
| `E_THEME_REFERENCE` | Invalid or duplicate theme member |
| `E_METADATA` | Invalid date or required nonempty metadata |

Equivalent code names are allowed unless an implementation advertises this diagnostic-code set. Error conditions themselves are mandatory. Warnings SHOULD use separate codes or severity values.

### 18.2 Failure and recovery

A reader MUST reject invalid core data and MUST NOT “repair” it by padding rows, truncating answers, renumbering clues to match prose, picking one conflicting solution, ignoring a misspelled property, or lowercasing/uppercasing file answers. A duplicate record must not silently overwrite an earlier record.

An editor MAY offer explicit repair actions, with a preview and fresh validation afterward. Loading failures should leave existing progress intact. A game MUST NOT enter an apparently playable state with unresolved fatal errors.

For error accumulation, recover at the next physical line or recognizable section header. Never let a malformed string swallow the rest of the document. Invalid grid geometry prevents dependent numbering validation; independent metadata errors can still be reported. Bound both processing and diagnostics; a validator MAY stop after 100 errors and add a notice that more were suppressed.

### 18.3 Mandatory version 1 limits

These limits make acceptance predictable and bound resource use. A conforming v1 reader/player MUST handle every valid file within them. A file exceeding them is invalid v1, even if a particular implementation could process it.

| Quantity | Limit |
| --- | --- |
| Raw file size, including comments and BOM | 1,048,576 bytes |
| Width and height | Each 1–99 |
| Total grid positions | At most 9,801, implied by dimensions |
| Physical line length, excluding line-ending bytes | 65,536 UTF-8 bytes |
| Decoded individual string | 4,096 Unicode scalar values |
| Single cell's nonempty answer/input value | 1–32 ASCII letters; ordinary cells exactly 1 |
| Theme records | At most 128 |
| Extension header/section name | At most 64 ASCII characters |
| Theme identifier | At most 32 ASCII characters |
| `@revision` | 1–2,147,483,647 |
| Derived clue number | At most 9,801, implied by numbered cells |
| Asserted entry length | 1–99 syntactically in bounds, then exactly the derived length |

The two-cell discovery rule and profile validation still apply to small dimensions. For example, `1x1` cannot be a valid puzzle because it has no entry; `4x1` can be valid under `general`.

Parsers MUST compare numeric bounds without overflowing their integer representation. Extremely long decimal strings are not safe merely because the grammar recognizes digits. Check lengths/bounds before conversion or use an overflow-safe conversion.

No file instructs the reader to download assets, execute extensions, or open source URLs. Parsing, validation, and baseline play require no network access.

## 19. Complete examples

Each code block in this section is a **whole, independently valid CrossText file**. Save the contents of a block as UTF-8, without the Markdown fences. Every clue number and answer crossing below is consistent with its geometry. The standard example deliberately has mixed three- and five-cell entries; the format imposes no uniform word length.

### 19.1 Traditional puzzle with a complete solution and assertions

```crosstext
@crossword 1
@size 5x5
@profile traditional
@symmetry rotational-180
@title "Point and Counterpoint"
@author "CrossText specification"
@description "A small crossword with mixed entry lengths."
@language "en"
@date "2026-10-05"
@difficulty "Easy"
@id "crosstext-example-point-and-counterpoint"
@revision 1
@notes "Answers are supplied for checking, but the play grid begins blank."

[grid]
#...#
.....
.....
.....
#...#

[solution]
#ARC#
PROOF
ARGUE
ROUND
#WET#

[answers]
A1 1,2 ACROSS 3 "ARC"
D1 1,2 DOWN 5 "ARROW"
D2 1,3 DOWN 5 "ROGUE"
D3 1,4 DOWN 5 "COUNT"
A4 2,1 ACROSS 5 "PROOF"
D4 2,1 DOWN 3 "PAR"
D5 2,5 DOWN 3 "FED"
A6 3,1 ACROSS 5 "ARGUE"
A7 4,1 ACROSS 5 "ROUND"
A8 5,2 ACROSS 3 "WET"

[clues]
A1 "Curved path"
A4 "Evidence that establishes a claim"
A6 "Debate a point"
A7 "Shaped like a circle"
A8 "Covered in water"
D1 "Pointer shot from a bow"
D2 "Person who breaks the rules"
D3 "Determine how many"
D4 "Standard score on a golf hole"
D5 "Gave food to"
```

Expected structure:

| Entry | Start | Cells | Answer |
| --- | --- | --- | --- |
| A1 | 1,2 | 3 | ARC |
| A4 | 2,1 | 5 | PROOF |
| A6 | 3,1 | 5 | ARGUE |
| A7 | 4,1 | 5 | ROUND |
| A8 | 5,2 | 3 | WET |
| D1 | 1,2 | 5 | ARROW |
| D2 | 1,3 | 5 | ROGUE |
| D3 | 1,4 | 5 | COUNT |
| D4 | 2,1 | 3 | PAR |
| D5 | 2,5 | 3 | FED |

There are 21 open squares, 4 black squares, 8 numbered squares, and 10 entries. Across lengths are `3,5,5,5,3`; Down lengths are `5,5,5,3,3`. Answer coverage is complete. The solution and the redundant answer records agree at every square. Initial selection is `A1` at `1,2`; no letters initially appear.

Either `[solution]` alone or the supplied complete set of answer strings would provide all expected values. Supplying both demonstrates strict consistency checks, not a requirement to duplicate data.

### 19.2 The same geometry without any supplied answers

```crosstext
@crossword 1
@size 5x5
@symmetry rotational-180
@title "Point and Counterpoint — Solver Copy"
@author "CrossText specification"
@notes "This file supports filling and navigation. It has no solution data."

[grid]
#...#
.....
.....
.....
#...#

[clues]
A1 "Curved path"
A4 "Evidence that establishes a claim"
A6 "Debate a point"
A7 "Shaped like a circle"
A8 "Covered in water"
D1 "Pointer shot from a bow"
D2 "Person who breaks the rules"
D3 "Determine how many"
D4 "Standard score on a golf hole"
D5 "Gave food to"
```

This has exactly the same entries and numbering as example 19.1. Its omitted profile defaults to `traditional`. Answer coverage is none. A legal filled grid produces fill completion and unlocks any completion-gated theme notes, but it cannot produce verified completion. Check and Reveal explain that no solutions are available.

### 19.3 Rebus, circles, givens, a theme, and answer-only solutions

```crosstext
@crossword 1
@size 3x3
@title "A Sound in One Square"
@author "CrossText specification"
@description "One square can hold two letters. The middle R is given."
@notes "Read the circled squares from top to bottom after filling the grid."
@x-editor "Reference fixture"

[grid]
...
...
...

[cells]
1,1 circle=true rebus=true
2,2 circle=true
3,3 circle=true

[givens]
2,2 "R"

[answers]
A1 1,1 ACROSS 3 "{HE}AR"
A4 2,1 ACROSS 3 "ARE"
A5 3,1 ACROSS 3 "RED"
D1 1,1 DOWN 3
D2 1,2 DOWN 3
D3 1,3 DOWN 3

[clues]
A1 "Perceive a sound" enumeration="(4 letters in 3 cells)"
A4 "Exist, with a plural subject" enumeration="(3)"
A5 "Color of a stop sign" enumeration="(3)"
D1 "Use your ears" enumeration="(4 letters in 3 cells)"
D2 "Verb after 'they'" enumeration="(3)"
D3 "Opposite of green on a traffic signal" enumeration="(3)"

[themes]
T1 "A diagonal message" entries=A1,A4,A5,D1,D2,D3 note="The circled values are HE, R, D: together they spell HERD." reveal=completion

[x-editor-notes]
The theme explanation is plain text; no special extraction code is required.
```

Expected solution, shown here for explanation rather than as another input file:

```text
{HE} A R
 A   R E
 R   E D
```

The first row is three cells, not four. `A1` and `D1` are both `HEAR`, while `A4`, `D2` are `ARE`, and `A5`, `D3` are `RED`. Start numbers are 1 at `1,1`, 2 at `1,2`, 3 at `1,3`, 4 at `2,1`, and 5 at `3,1`.

All nine expected values come from the Across answer records, so answer coverage is complete despite the absence of `[solution]` and Down answer strings. Only `2,2` is initially filled, with immutable `R`. The cell `1,1` is editable, marked rebus-capable, and initially empty. Three cells are circled. The theme note is hidden until fill completion.

An equivalent complete solution section would be:

```text
[solution]
{HE}AR
ARE
RED
```

Adding that section to the full example is valid and must not change its expected map. Removing its answer strings and keeping no solution section produces the same playable rebus geometry without answer knowledge; the given remains visible.

### 19.4 A partial solution with one known Across answer

```crosstext
@crossword 1
@size 5x5
@title "Partial Answer Knowledge"
@description "Only 4-Across is supplied for checking."

[grid]
#...#
.....
.....
.....
#...#

[answers]
A4 2,1 ACROSS 5 "PROOF"

[clues]
A1 "Curved path"
A4 "Evidence that establishes a claim"
A6 "Debate a point"
A7 "Shaped like a circle"
A8 "Covered in water"
D1 "Pointer shot from a bow"
D2 "Person who breaks the rules"
D3 "Determine how many"
D4 "Standard score on a golf hole"
D5 "Gave food to"
```

Exactly five of 21 expected values are known, at row 2, columns 1–5. This also supplies one known value in each Down entry that crosses that row. Checking a fully filled grid matching example 19.1 returns 5 correct and 16 unknown; its aggregate result is `unverified`. Revealing the puzzle fills only `PROOF` in row 2 and reports 16 unavailable answers. The file remains valid.

### 19.5 Rectangular geometry and a direction with no entries

```crosstext
@crossword 1
@size 4x1
@profile general
@title "One Row"

[grid]
....

[solution]
BOOK

[clues]
A1 "Bound reading material" enumeration="(4)"
```

Width is 4, height is 1. There is one four-cell Across entry, `A1` at `1,1`, and no Down entries. The puzzle is connected and valid under `general`. The Down clue list is empty. Direction toggling is a no-op and Tab returns to `A1`. The same file under `traditional` is invalid because each open square lacks a Down entry.

## 20. Conformance and acceptance tests

### 20.1 Conformance categories

An implementation must state which category it implements:

| Category | Mandatory behavior |
| --- | --- |
| **CrossText 1 Reader/Validator** | Parses all v1 syntax, resolves references, derives numbering, enforces every semantic rule and limit, produces a normalized result or diagnostics, and retains extensions |
| **CrossText 1 Writer** | Produces valid v1 files and preserves intended normalized semantics, including answer coverage, when serializing supported models |
| **CrossText 1 Player** | Includes Reader/Validator conformance and implements the rendering, navigation, input, checking, reveal, completion, and undo requirements in this document |
| **CrossText 1 Lossless Editor** | Includes Reader/Validator and Writer conformance and additionally preserves untouched source formatting/comments/escape spelling |

A syntax-only parser may call itself a parser, but cannot claim Reader/Validator conformance. An app that ignores rebus annotations, numbers directions separately, treats unknown answers as correct, or silently repairs conflicts cannot claim Player conformance.

Required behaviors are conditional on content where appropriate: a puzzle with no known answers has no functioning Reveal operation; a puzzle with no rebus squares has no need to show a rebus editor. These conditions do not permit an implementation to omit support when such content is present.

### 20.2 Valid-file tests

The five complete examples are normative acceptance fixtures. In addition to accepting each one, verify:

| Test | Expected result |
| --- | --- |
| Example 19.1, all redundant answers included | 21 expected values; 10 entries; numbers and answers match its table |
| Example 19.1, omit `[answers]` | Identical expected map and gameplay |
| Example 19.1, omit `[solution]` | Identical expected map from answer strings |
| Example 19.2 | No expected values; fill completion never implies verified completion |
| Example 19.3 | Nine cells, three circles, one rebus, one given, six entries; full expected map |
| Example 19.3, move `[clues]` before `[grid]` | Same normalized model; forward references work |
| Example 19.3, add its equivalent `[solution]` | Same expected map, no conflict |
| Example 19.4 | Exactly five known expected values; partial checking/reveal |
| Example 19.5 | Width 4, height 1; one Across entry; empty Down list |
| Any example with CRLF or mixed LF/CRLF | Same normalized model |
| Any example with an initial UTF-8 BOM or no final newline | Same normalized model |
| Insert blank/comment-only lines between grid rows | Same dimensions and geometry |
| Add spaces/tabs between grid tokens | Same normalized geometry |
| Clue text `"A semicolon; inside a clue"` with a trailing `; comment` | Semicolon inside retained; outside comment ignored |
| Metadata `@title "Caf\u00E9"` | Decodes to `Café` |
| Metadata `@title "\uD83D\uDE00"` | Decodes to one Unicode scalar, U+1F600 |
| Canonically serialize then parse any example | Equivalent normalized model, including extensions and expected coverage |

Changes to clue text in lexical tests must preserve required entry IDs and other records. An example is a complete fixture; a table row describes only the stated mutation.

### 20.3 Invalid-file tests

Starting from the indicated valid example, each single mutation MUST fail validation. Recommended diagnostic codes are shown; additional independent errors are allowed.

| Base and mutation | Expected reason/code |
| --- | --- |
| 19.1: change version to `@crossword 2` | Unsupported version, `E_VERSION` |
| 19.1: remove `@size` | Missing required header, `E_HEADER_MISSING` |
| 19.1: repeat `@title` | Duplicate header, `E_DUPLICATE` |
| 19.1: put `@author` after `[grid]` | Header after section, `E_SYNTAX` |
| 19.1: rename `[grid]` to `[puzzle]` | Unknown section, `E_SECTION_UNKNOWN` |
| 19.1: make a geometry row `#ARC#` | Letters in geometry, `E_GRID_TOKEN` |
| 19.1: delete one cell from a geometry row | Wrong width, `E_GRID_SIZE` |
| 19.1: remove `D2` clue | Missing clue, `E_CLUE_MISSING` |
| 19.1: duplicate `A1` clue | Duplicate clue, `E_DUPLICATE` |
| 19.1: add clue `A2 "No such entry"` | Unknown entry, `E_ENTRY_UNKNOWN` |
| 19.1: assert `A4 2,2 ACROSS 5 "PROOF"` | Wrong start, `E_ASSERTION` |
| 19.1: assert `A4 2,1 DOWN 5 "PROOF"` | Wrong direction, `E_ASSERTION` |
| 19.1: assert `A4 2,1 ACROSS 4 "PROOF"` | Wrong cell length, `E_ASSERTION` |
| 19.1: change D1's answer to `"BROWN"` | Crossing conflict, `E_ANSWER_CONFLICT` |
| 19.1: change solution row 1 to `AARC#` | Black/open mismatch, `E_SOLUTION_SIZE` |
| 19.1: replace a solution letter with `.` | Incomplete solution token, `E_SOLUTION_TOKEN` |
| 19.1: use lowercase `"proof"` in an answer | Invalid file-answer alphabet, `E_SOLUTION_TOKEN` |
| 19.1: append `extra` to a clue line | Unconsumed trailing token, `E_SYNTAX` |
| 19.1: add `[cells]` with `1,1 circle=true` | Annotation on a block, `E_COORDINATE` |
| 19.1: add a given `2,1 "B"` | Given disagrees with P, `E_GIVEN_CONFLICT` |
| 19.3: remove `rebus=true` from `1,1` | Multi-letter expected token unsupported there, `E_REBUS_REQUIRED` |
| 19.3: change A1 answer to `"HEAR"` | Four tokens for a three-cell entry, `E_SOLUTION_TOKEN` |
| 19.3: give D1 the answer `"{HA}AR"` | HE versus HA at `1,1`, `E_ANSWER_CONFLICT` |
| 19.3: change given `2,2` to `"{R}"` | Given is not a direct token, `E_SOLUTION_TOKEN` |
| 19.3: use `circle=true circle=false` in one record | Duplicate property, `E_DUPLICATE` |
| 19.3: use `circel=true` | Unknown property, `E_SYNTAX` |
| 19.3: theme members `A1,A1` or `A99` | Duplicate/nonexistent member, `E_THEME_REFERENCE` |
| 19.5: remove `@profile general` | Default traditional has unchecked squares, `E_UNCHECKED_CELL` |
| Any: use a lone CR line ending | `E_ENCODING` |
| Any: use a string with `\q` or unpaired `\uD800` | Invalid escape, `E_SYNTAX` |
| Any: add `@date "2026-02-29"` where no date exists | Non-leap-year invalid date, `E_METADATA` |
| Any: use dimension 100 or revision 2147483648 | `E_LIMIT` |

Two additional standalone malformed puzzles exercise geometry validation:

```text
; Disconnected general grid; the top and bottom entries cannot touch.
@crossword 1
@size 3x3
@profile general
[grid]
...
###
...
[clues]
A1 "Top row"
A2 "Bottom row"
```

This fails with `E_DISCONNECTED`. Its apparent clue numbering does not make its disconnected shape valid.

```text
; Two-cell entries are discovered, then rejected by the traditional profile.
@crossword 1
@size 2x2
[grid]
..
..
[clues]
A1 "Top row"
A3 "Bottom row"
D1 "Left column"
D2 "Right column"
```

This fails with `E_ENTRY_LENGTH`. Changing only the profile to `general` makes this geometry and clue coverage valid, even without answers.

### 20.4 Gameplay tests

Tests start with a new attempt unless stated otherwise. All assertions refer to the normalized shared-cell state, not separately stored word strings.

| Test | Expected result |
| --- | --- |
| Open 19.1 | Active A1 at `1,2`; 21 blank values; no solution letters shown |
| Type `a`, then `r`, then `c` | A1 contains ARC; selection ends at `1,4`; D1/D2/D3 show the shared letters |
| From that state, Backspace once | Clear `1,4`; remain there |
| Backspace again while `1,4` is blank | Move to `1,3` and clear its R |
| Reset, then Tab | A4 selected at `2,1` |
| Reset, then Shift+Tab | D5 selected at `2,5` |
| Click `1,2`, then click it again | The second click toggles between A1 and D1 |
| At `2,1`, press Right | Select `2,2` in A4; no values change |
| At `1,4`, press Left | Select `1,3` in A1 |
| At `1,4`, press Right | No open square to right; selection unchanged |
| At `1,2`, press Down | Select `2,2` in D1 |
| Paste `arc` into a blank A1 at `1,2` | ARC entered atomically; active square `1,4` |
| Paste `arcs` into A1 at `1,2` | Reject overflow; no values or selection change |
| Check a blank A1 in 19.1 | 3 blank, aggregate incomplete; no letters revealed |
| Fill A1 with ABD, then Check entry | 1 correct, 2 incorrect, aggregate incorrect |
| Change one previously checked square | Only that square's displayed feedback invalidated; crossings update |
| Reveal A1, then Undo | Prior values/feedback/selection restored; reveal-assistance history retained |
| Enter the exact complete grid of 19.1 | Fill complete and verified complete |
| Enter the same grid in 19.2 | Fill complete, unverified; never “Solved” |
| Fill all of 19.4 as in 19.1, then Check puzzle | 5 correct, 16 unknown, aggregate unverified |
| Reveal puzzle in blank 19.4 | Only row 2 becomes PROOF; 16 unavailable squares reported |
| Open 19.3 | Only `2,2` has R; `1,1` is an empty editable rebus square |
| Paste `{he}ar` at `1,1` in A1 of 19.3 | Three cells become HE, A, R; Check entry reports correct |
| Paste `hear` at the same position | Four tokens overflow three targets; no partial change |
| Paste `{he}ar` into A1 of 19.1 | Reject multi-letter token at unmarked `1,2` |
| Select given `2,2` in 19.3 and type/delete | R remains unchanged |
| Use rebus editor to commit H at `1,1`, then check that cell | Incorrect against expected HE |
| Fill every square of 19.3 with any legal values | Theme note unlocks even if some values are wrong |
| Clear a square after theme unlock | Fill completion becomes false; unlocked note stays available |
| Reset 19.3 | Only given R remains; theme note locks; assistance history clears |
| Open 19.5, toggle direction, then Tab | Still A1; there is no Down entry |

For canonical entry-order navigation, the order in example 19.1 is:

```text
A1, A4, A6, A7, A8, D1, D2, D3, D4, D5
```

For source-ordered themes, the circular-diagonal interpretation in example 19.3 is supplied by its text, not a hidden extraction algorithm. A conforming player need not infer HERD; it displays the circles and later displays the supplied explanation.

### 20.5 Invariants suitable for automated tests

After every accepted load and runtime transaction:

1. Every open coordinate has exactly one runtime value, shared by all memberships.
2. No black coordinate has a runtime value or entry membership.
3. Every entry's cells are contiguous, maximal, ordered, and agree with its start and length.
4. Every numbered square is an entry start, and every entry start has the correct shared row-major number.
5. Every entry has exactly one clue.
6. Nonempty runtime values obey cell capabilities, and givens are unchanged.
7. Selection refers to an open square contained in its active entry.
8. Expected values never change during play.
9. Unknown expected values never count as correct or revealed.
10. A file with partial or absent answer knowledge can never have `verifiedComplete=true`.
11. No mutation produces independent, conflicting Across and Down values at a crossing.
12. Failed input/paste transactions leave the preceding state intact.
13. Parse → canonical write → parse preserves the normalized semantic model.

## 21. Implementation checklist

A developer can build the baseline app in the following sequence:

1. Implement strict UTF-8/line handling, a quoted-string-aware comment scanner, and core token parsers with source positions.
2. Read headers and all section records, enforcing syntax, duplicates, and bounds.
3. Tokenize geometry and derive maximal entries and shared row-major numbers.
4. Enforce connectivity, profile rules, and declared symmetry.
5. Attach exactly one clue per entry; resolve assertions, annotations, givens, and themes.
6. Build an expected-value map from solution grids and answer strings, rejecting all conflicts.
7. Initialize shared runtime values from givens only; select the first navigation entry.
8. Render cells, numbers, rebus indicators, circles, clues, metadata, and themes.
9. Implement atomic input, paste, rebus editing, direction changes, arrows, and entry navigation.
10. Add undo, clear/reset, checking, reveal, and the two completion predicates.
11. Add accessible keyboard/touch controls and progress persistence if desired.
12. Run all valid fixtures, invalid mutations, gameplay tests, and invariants above before claiming conformance.

The file's geometry is authoritative, optional answers determine what can be verified, and the shared cell state is authoritative during play. Those three rules keep parsing, rendering, navigation, and checking consistent across implementations.
