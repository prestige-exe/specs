# TheDraw Font File (.TDF)

by Carsten "Roy/SAC" Cumbrowski,
"TheDraw Fonts File (.TDF) Specifications", April 2014.
Reverse-engineered from TheDraw and the TDFONTS.EXE editor; this is
the only public writedown of the format.

TheDraw was the dominant DOS ANSI/ASCII editor of the early 90s. The
.TDF file is its font format: a collection of bitmap-style glyphs
that the editor stamps into a drawing for headers, logos, and group
greets. One .TDF holds up to 34 fonts. The editor enforces that
limit; the on-disk format does not.

There are three font types:

- **Block** — solid glyphs built from CP437 characters as drawn.
- **Color** — Block with a foreground/background attribute per cell.
- **Outline** — drawn with placeholder letters that the renderer
  remaps to a fixed set of box-drawing characters at display time.

## File layout

```
+-------------------------------------------+
| File header (20 bytes)                    |
+-------------------------------------------+
| Font 1 header (bytes 20..232)             |
+-------------------------------------------+
| Font 1 character data (BlockSize bytes,   |
| null-terminated if another font follows)  |
+-------------------------------------------+
| Font 2 header                             |
+-------------------------------------------+
| Font 2 character data                     |
+-------------------------------------------+
| ...                                       |
+-------------------------------------------+
```

The last font in a collection is not null-terminated. Every earlier
font is.

## File header (20 bytes)

| Offset | Size | Field | Value |
| --- | --- | --- | --- |
| 0 | 1 | Magic | `0x13` |
| 1 | 18 | Signature | `TheDraw FONTS file` |
| 19 | 1 | EOF | `0x1A` |

The `0x1A` is the DOS EOF marker. `TYPE thefont.tdf` halts at that
byte instead of dumping binary to the screen.

## Font header

Offsets here are file offsets for the first font. For later fonts,
add the running file position.

| Offset | Size | Field | Notes |
| --- | --- | --- | --- |
| 20 | 4 | Marker | `55 AA 00 FF`. Start of a font definition |
| 24 | 1 | NameLen | Length of the font name, 1..12 |
| 25 | 12 | Name | Font name. Read only `NameLen` bytes; the remainder is undefined |
| 37 | 4 | Reserved | Typically zero |
| 41 | 1 | FontType | `00`=Outline, `01`=Block, `02`=Color |
| 42 | 1 | Spacing | Letter spacing, stored as `0x01..0x29` for values 0..40 |
| 43 | 2 | BlockSize | Size of this font's character data, including the trailing null if another font follows. Little-endian word |
| 45 | 188 | CharOffsets | 94 little-endian words, one per glyph for ASCII 33 (`!`) through 126 (`~`). Each value is the byte offset of that glyph's data from the start of the character data block. `FFFF` means the glyph is undefined |
| 233 | BlockSize | Character data | See below |

The character offset table can point multiple glyphs at the same
position. The TDFONTS.EXE editor's *Copy character* command does
this to share storage between identical glyphs, which is also the
escape hatch for the 64K offset ceiling described below.

## Computing later font positions

```
font_1_header  = 20
font_1_data    = 233
font_2_header  = 232 + font_1_blocksize + 1
font_2_data    = font_2_header + 213
font_n_header  = font_(n-1)_data + font_(n-1)_blocksize
font_n_data    = font_n_header + 213
```

Each font header occupies 213 bytes (marker through the end of the
character offset table). The `+1` between fonts is the null
terminator that lives at the end of each non-final font's character
data.

## Character data

Each glyph begins with two bytes:

| Offset | Size | Field | Range |
| --- | --- | --- | --- |
| 0 | 1 | MaxWidth | 1..30 (`0x1E`) |
| 1 | 1 | MaxHeight | 1..12 (`0x0C`). See caveat |

**MaxHeight caveat.** Lines that end in `&` (the descender marker)
are not counted toward MaxHeight. A 6-line glyph with descenders on
rows 4 and 5 reports MaxHeight=4. Use MaxHeight to advance the
cursor on a line break. Do not use it to bound the read. Read until
a null terminator.

After the size bytes comes a stream of cells, with rows separated by
`0x0D` (carriage return) and the glyph terminated by `0x00`.

For **Block** and **Outline** fonts: one byte per cell, the CP437
character to render (or, for Outline, the placeholder letter — see
below).

For **Color** fonts: two bytes per cell. First byte is the CP437
character. Second byte is the standard PC text-mode attribute: high
nibble = background (0..7), low nibble = foreground (0..F). A color
byte of `0x00` is legal and is *not* a terminator — termination
applies only to the character byte.

When a line ends before MaxWidth is reached, the remaining cells in
that row are transparent.

## Outline font placeholders

Outline fonts do not store CP437 box-drawing characters directly.
They store ASCII letters that the renderer maps to box characters
at display time, so the same outline glyph can be re-themed without
rewriting every byte.

| Letter | CP437 | Glyph | Role |
| --- | --- | --- | --- |
| A | `0xCD` | `═` | Horizontal double |
| B | `0xC4` | `─` | Horizontal single |
| C | `0xB3` | `│` | Vertical single |
| D | `0xBA` | `║` | Vertical double |
| E | `0xD5` | `╒` | Upper-left outer corner / Up-to-Right inner corner |
| F | `0xBB` | `╗` | Upper-right outer corner / Right-to-Down inner corner |
| G | `0xD6` | `╓` | Up-to-Right outer corner / Upper-left inner corner |
| H | `0xBF` | `┐` | Right-to-Down outer corner / Upper-right inner corner |
| I | `0xC8` | `╚` | Lower-left inner corner |
| J | `0xBE` | `╛` | Lower-right inner corner |
| K | `0xC0` | `└` | Lower-left outer corner |
| L | `0xBD` | `╜` | Lower-right outer corner |
| M | `0xB5` | `╡` | Right tee |
| N | `0xC7` | `╟` | Left tee |
| O | `0xF7` | `≈` | Hard space. Non-breaking space inside a glyph |
| @ | `0x40` | `@` | Leading-space filler |
| & | `0x26` | `&` | Descender marker. Line is rendered but not counted in MaxHeight |

The TheDraw convention for outline borders:

- Double lines are the outermost edge of a column.
- Double lines are the topmost edge of a beam.

The editor enforces this. The file format does not. A hand-written
.TDF that violates the rule will load and render, but the corner
joins will be wrong.

The descender marker matters because TheDraw aligns glyphs by line
count. Without descender marking, a baseline run like `Aa Qq Pp` will
shift vertically wherever a tailed glyph appears. Symbols that
typically need a descender: `$ , ; _ Q g j p q y`.

## Limits and quirks

- **34 fonts maximum per file.** The editor enforces this; the
  format technically allows more.
- **65,534-byte character data ceiling per font.** Character offsets
  are 16-bit, with `FFFF` reserved for "undefined". A large color
  font (2 bytes per cell, up to 30 wide × 12 tall + CR + null per
  glyph, 94 glyphs) can hit this ceiling well before all 94 glyphs
  are populated. Sharing offsets between identical glyphs is the
  workaround.
- **Glyph range is 33..126**, the printable ASCII subset. Space (32)
  and DEL (127) are not stored.
- **Spacing stored offset-by-one.** The byte value `0x01` represents
  zero spacing; `0x29` represents 40.
- **Single empty font file is 233 bytes** — the 20-byte file header
  plus the 213-byte font header, with no character data.

## References

- Roy/SAC, "TheDraw Fonts File (.TDF) Specifications", 23 April 2014.
  roysac.com/blog/2014/04/thedraw-fonts-file-tdf-specifications/
- TheDraw in-program documentation and the TDFONTS.EXE editor.
