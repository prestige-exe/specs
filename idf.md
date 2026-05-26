# IDF – iCEDraw File Format

Origin: iCEDraw editor by iCE (Insane Creators Enterprise), mid-1990s.
The format is documented here from the published iCEDraw source and
from the reverse-engineering work done by PabloDraw, SyncTERM, and
Ansilove.

IDF is iCE's answer to the limitations of BIN and ADF: arbitrary
width, RLE-compressed image data, custom font, custom palette. It is
contemporary with XBin and largely equivalent in capability; XBin won
broader adoption because Tasmaniac documented it publicly and ACiDDraw
shipped with strong export support.

## File layout

```
+--------------------+
| Header (12 bytes)  |
+--------------------+
| Image data (RLE)   |
+--------------------+
| Font (4096 bytes)  |
+--------------------+
| Palette (48 bytes) |
+--------------------+
| SAUCE (optional)   |
+--------------------+
```

## Header (12 bytes)

| Offset | Size | Field | Description |
| --- | --- | --- | --- |
| 0 | 4 | ID | "\x041.4" – literally 0x04, then "1.4" (ASCII '1', '.', '4') |
| 4 | 2 | X1 | Left column of image (almost always 0) |
| 6 | 2 | Y1 | Top row of image (almost always 0) |
| 8 | 2 | X2 | Right column of image (inclusive) |
| 10 | 2 | Y2 | Bottom row of image (inclusive) |

All multi-byte fields are little-endian. The image width is `X2 − X1 +
1` columns; the image height is `Y2 − Y1 + 1` rows.

The four-byte ID begins with the EOT control character (0x04) and is
followed by the ASCII string "1.4". This is the format's only magic
number.

## Image data (RLE)

Following the header, the image is stored as a stream of 16-bit
little-endian words. Each word is one cell:

```
high byte: attribute   low byte: character
```

If a word's low byte equals 0x01 and high byte equals 0x00 it is *not*
a literal cell – it is the RLE marker introducer. After the marker
introducer come:

```
word repeat_count   word cell_value
```

Both also little-endian; `cell_value` is then emitted `repeat_count`
times.

Wait – the canonical iCEDraw decoder treats a *full word* of 0x0001 as
the introducer. Actually, the introducer is when the word read equals
0x0001 (in little-endian: byte 0x01 then byte 0x00) at the start of a
record. The next word is the run length; the word after is the cell to
repeat.

For implementers, refer to the PabloDraw `IDFFile.cs` source, which is
the canonical open-source reference. The decoding loop:

```pseudo
i = 0
while reader has bytes:
    w = read_le16()
    if w == 0x0001:
        count = read_le16()
        cell  = read_le16()
        for k in 1..count:
            image[i++] = cell
    else:
        image[i++] = w
```

The image array is `width × height` cells, indexed left-to-right then
top-to-bottom. A character byte 0x01 with a zero attribute byte cannot
be encoded as a literal (the decoder would interpret it as the RLE
introducer); to emit such a cell, encode it as a run of one.

## Font

4096 bytes immediately after the image data: 256 characters × 16
scanlines × 1 byte each. Layout identical to ADF ([[adf]]) and XBin
([[xbin]]).

## Palette

48 bytes at the end (before any SAUCE): 16 entries × 3 bytes (R, G, B)
in VGA 6-bit-per-channel format. Index order matches the standard
CGA/EGA/VGA text mode palette (see [[adf]]).

Note this is 16 entries, not the 64 entries that ADF stores. IDF only
keeps the colours the text-mode palette can address.

## Width and height

Computed from the header:

```
width  = X2 − X1 + 1
height = Y2 − Y1 + 1
```

X1 and Y1 are typically 0 in practice; the rectangle fields were
intended to support sub-image storage but the released iCEDraw never
exposed that workflow.

## SAUCE

If present, DataType = 1 (Character), FileType = 4 (IDF). TInfo1 and
TInfo2 carry width and rows; the SAUCE values should match the
header, but a renderer should prefer the header on mismatch.

## Identification

- File extension: `.IDF`.
- Magic: bytes 0..3 must be `04 31 2E 34` (EOT, '1', '.', '4').
- Minimum size: 12 (header) + 4096 (font) + 48 (palette) = 4156 bytes
  before any image data.
