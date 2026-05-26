# BIN – Binary Text Image File

Origin: the IBM PC text-mode video buffer at segment B800:0000 (CGA /
EGA / VGA colour text mode). The "BIN" file format is nothing more
than a raw dump of that buffer. The convention was popularised in the
early 1990s ANSI art scene as a way to ship pixel-accurate text-mode
artwork without the cost of decoding ANSI escape sequences.

## File layout

```
+-----------+-----------+-----------+ ... +-----------+
| ch0  at0  | ch1  at1  | ch2  at2  |     | chN  atN  |
+-----------+-----------+-----------+ ... +-----------+
```

The file is simply an interleaved array of character/attribute pairs:

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 1 | Character byte of cell 0 (CP437) |
| 1 | 1 | Attribute byte of cell 0 |
| 2 | 1 | Character byte of cell 1 |
| 3 | 1 | Attribute byte of cell 1 |
| ... | ... | ... |

No header, no magic, no terminator (except an optional SAUCE record).

## Character and attribute bytes

The character byte is a single CP437 code point (see [[cp437]]). The
attribute byte uses the IBM PC text-mode format:

```
bit  7  6  5  4  3  2  1  0
     B  bR bG bB I  fR fG fB
```

| Bits | Meaning |
| --- | --- |
| 0..2 | Foreground colour (R/G/B index) |
| 3 | Foreground intensity (bright) |
| 4..6 | Background colour |
| 7 | Blink (or background intensity, see iCE colors) |

Colour bits use the same palette as the SGR colour table in
[[ansi-escape]] and the AVATAR attribute byte in [[avatar]].

## Width and height

A BIN file does not encode its dimensions. The width is conventionally
80 columns; the height is `file_size / (80 * 2)`. Some BIN files are
made for screen widths other than 80 (40, 132, 160) and rely on either
SAUCE metadata or out-of-band knowledge to render correctly.

If a SAUCE record is present (see [[sauce-00.5]]) and DataType = 5
(BinaryText), `TInfo1` carries half the width in characters (so a
80-column file stores 40 in TInfo1). To recover width: `width = TInfo1
* 2`. The number of rows is then `(FileSize_without_SAUCE / 2) /
width`.

This rather unusual encoding of width preserves backward compatibility
with viewers that always assumed 80 columns; new viewers can use
TInfo1 to handle wider screens (the format does not impose a maximum
width).

## Compression

BIN files are uncompressed. A 160 × 100 cell BIN is 32,000 bytes
regardless of content. For large pieces this is expensive – the
successor XBin format ([[xbin]]) optionally compresses the same data
with RLE and adds a header, custom font, and palette.

## Loading into video memory

The original use case was the literal `BLOAD` / segment copy:

```asm
push    es
mov     ax, 0B800h        ; CGA/VGA colour text page
mov     es, ax
mov     di, 0
mov     ds, segment_of_bin
mov     si, offset_of_bin
mov     cx, 80 * 25       ; 2000 cells of 2 bytes each
rep     movsw
pop     es
```

A BIN file that exactly matches the screen geometry can be loaded with
a single REP MOVSW.

## SAUCE for BIN

For a SAUCE'd BIN:

| Field | Value |
| --- | --- |
| DataType | 5 (BinaryText) |
| FileType | 0 |
| TInfo1 | Width / 2 (number of half-screens wide) |
| TInfo2 | Number of rows (0 = unknown, derive from file size) |
| TInfoS | Font name (default "IBM VGA") |
| ANSiFlags (TFlags) | iCE colors / aspect / letter spacing |

The font name is what tells a modern viewer to render the file with a
particular fixed-width DOS font. Without SAUCE, the default is IBM VGA
(8 × 16 cells).

## Relationship to other formats

- **ANSI** ([[ansi-escape]]) is a procedural / streaming representation
  of the same image: a series of cursor-positioning and colour
  commands followed by the characters.
- **BIN** is the snapshot. Convert ANSI → BIN by running the ANSI
  through an interpreter and dumping the resulting screen.
- **XBin** ([[xbin]]) is BIN plus a header, optional palette, optional
  font, and optional RLE compression.
- **TUNDRA** uses the same character grid but stores per-cell 24-bit
  foreground and background instead of a 4-bit attribute byte.

BIN's main strength is simplicity – the loader is a few lines of
assembly. Its weakness is the lack of any in-band metadata (width,
height, font, palette), which is why every modern usage either pairs
it with SAUCE or uses XBin instead.
