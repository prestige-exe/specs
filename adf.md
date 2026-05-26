# ADF – Artworx Data Format

Origin: ACiDDraw / Artworx editor, late 1980s. The original Artworx
documentation never received a wide public release; the format is
documented here from the reverse-engineering work done by the SyncTERM,
PabloDraw, and Ansilove projects.

ADF is one of the early "ANSI editor save formats" – a single file
that holds a text-mode image together with its custom font and
palette. It predates XBin and is structurally similar but simpler:
fixed 80-column width, fixed font height, no compression.

## File layout

```
+--------------------+
| Version byte (1)   |
+--------------------+
| Palette (192 bytes)|
+--------------------+
| Font (4096 bytes)  |
+--------------------+
| Image data         |  (80 cols * rows * 2 bytes)
+--------------------+
| SAUCE (optional)   |
+--------------------+
```

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 1 | Version: must be 0x01 |
| 1 | 192 | Palette: 64 entries × 3 bytes (R, G, B), 6 bits per channel |
| 193 | 4096 | Font: 256 glyphs × 16 scanlines × 1 byte |
| 4289 | rest | Image data: rows of 80 cells × 2 bytes (char, attr) |

## Version byte

A single byte at offset 0, must be `0x01`. There is no published v2.

## Palette

192 bytes immediately after the version byte: 64 RGB triples in VGA
6-bit-per-channel format (each channel 0..63). The first 16 entries
are the foreground/background palette in standard CGA/EGA/VGA text
mode order:

| Index | Default colour |
| --- | --- |
| 0 | Black |
| 1 | Blue |
| 2 | Green |
| 3 | Cyan |
| 4 | Red |
| 5 | Magenta |
| 6 | Brown |
| 7 | Light grey |
| 8 | Dark grey |
| 9 | Light blue |
| 10 | Light green |
| 11 | Light cyan |
| 12 | Light red |
| 13 | Light magenta |
| 14 | Yellow |
| 15 | White |

Entries 16..63 are unused by the text-mode renderer but stored anyway
to keep the file format aligned with the VGA hardware palette layout.

## Font

A flat 4096-byte block: 256 characters × 16 scanlines × 8 pixels wide.
Each scanline is one byte; the high bit is the leftmost pixel. The
font replaces the system CP437 font for the duration of rendering;
character codes still index into the same 256-entry table, but the
glyph bitmaps are whatever the artist drew.

## Image data

From offset 4289 to end-of-file (less SAUCE, if any), the file is a
sequence of character/attribute pairs:

- 80 cells per row.
- 2 bytes per cell (`char`, then `attr`).
- Attribute byte has the standard IBM text-mode layout (see [[bin]]).
- Number of rows = `(image_bytes) / (80 * 2)`.

## Limitations

- Width is fixed at 80 columns. There is no way to encode a wider or
  narrower screen.
- Font height is fixed at 16 scanlines. CGA / EGA-height fonts (8 or
  14) cannot be represented.
- No compression. Even mostly-blank canvases take the full storage.

These limitations are exactly the ones IDF ([[idf]]) and XBin
([[xbin]]) were created to lift.

## SAUCE for ADF

If present, DataType = 1 (Character), FileType = 5 (ADF). TInfo1 /
TInfo2 may carry width and height, but since ADF is fixed at 80
columns, only TInfo2 (rows) carries new information.

## Identification

ADF files have no magic – the first byte is just the version `0x01`,
which is far from unique. In practice, identification relies on the
file extension `.ADF` and on the structural test "file size ≥ 4289 and
`(file_size − 4289) mod 160 == 0`".
