# XBin – eXtended Binary Text File

by Olivier "Tasmaniac" Reubens / ACiD
"XBIN File Format Specification", original distribution as XBIN.DOC
inside the ACiD release tools, August 1996.

XBin is the "binary text" file format for high-end DOS ANSI art. It
solves three problems that limited the older BIN format:

1. BIN files are fixed at 160-byte rows (80 columns × 2 bytes per
   character cell). XBin supports any screen width.
2. BIN files cannot record a custom font. XBin can carry a complete
   character-generator bitmap.
3. BIN files cannot record a custom palette. XBin can carry a 16-colour
   6-bit-per-channel VGA palette.

The result is a single self-contained file that reproduces a piece of
art pixel-for-pixel including its custom font and palette – exactly
what you need for a final "release" archive of ANSI/ASCII art.

## File layout

```
+--------------------+
| Header (11 bytes)  |
+--------------------+
| Palette (optional, 48 bytes)
+--------------------+
| Font (optional, NumChars * FontSize bytes)
+--------------------+
| Image data (Width * Height * 2 bytes, optionally compressed)
+--------------------+
| SAUCE record (optional)
+--------------------+
```

## Header (11 bytes)

| Offset | Size | Field | Description |
| --- | --- | --- | --- |
| 0 | 4 | ID | "XBIN" followed by 0x1A (EOF) – five bytes that double as a magic number and DOS EOF marker |
| 4 | 1 | EofChar | (above; counted as the 5th byte of the magic) |
| 5 | 2 | Width | Image width in characters, little-endian |
| 7 | 2 | Height | Image height in characters, little-endian |
| 9 | 1 | FontSize | Height of each font character in scanlines (1..32). 0 if no font block. |
| 10 | 1 | Flags | Bit-flag byte (see below) |

The ID + EOF arrangement is the SAUCE / BBS convention: a DOS `TYPE`
command stops at the 0x1A, so the header alone is enough to mark the
file as XBin without rendering binary garbage to the screen.

## Flag byte

| Bit | Mnemonic | Meaning |
| --- | --- | --- |
| 0 | Palette | A 48-byte custom palette block follows the header |
| 1 | Font | A custom font block follows (palette, if any, comes first) |
| 2 | Compress | Image data is RLE-compressed |
| 3 | NonBlink | iCE colors: blink bit re-purposed as bright background |
| 4 | 512Chars | Font has 512 characters instead of 256 (uses the VGA alt-charset bit) |
| 5..7 | – | Reserved, must be zero |

The 512-character mode hijacks the blink/intensity attribute bit to
select between two 256-character font banks – a VGA hardware trick
documented by IBM but rarely used outside of XBin art. When 512Chars
is set, NonBlink is implicitly true; you cannot have both blink and
512-character mode at the same time.

## Palette block

48 bytes if the Palette flag is set: 16 entries × 3 bytes (R, G, B),
where each channel is a 6-bit VGA palette value (0..63, not 0..255).

The palette order matches the IBM PC EGA/VGA text-mode palette
register order:

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

If no palette block is present, the default VGA text-mode palette is
assumed.

## Font block

If the Font flag is set, a font block follows the (optional) palette
block. Layout:

```
NumChars × FontSize bytes
```

where:

- `NumChars` is 256, or 512 if the 512Chars flag is set.
- `FontSize` is the header's FontSize field (height of each glyph in
  scanlines). Common values are 8 (CGA-style square), 14 (EGA), and
  16 (VGA).
- Each glyph is 8 pixels wide and `FontSize` pixels tall. Each
  scanline is one byte; the high bit of each byte is the leftmost
  pixel.

Glyph N starts at offset N × FontSize within the font block.

If no font block is present, the renderer uses its default CP437 font.

## Image data

Width × Height character cells. Each cell is two bytes:

```
<character byte> <attribute byte>
```

The attribute byte uses the standard IBM PC text-mode format:

```
bit  7  6  5  4  3  2  1  0
     B  bR bG bB I  fR fG fB
```

(see [[avatar]] for the same byte layout). Cells are stored row by
row, left to right within a row.

## RLE compression

If the Compress flag is set, the image data is compressed as a
sequence of runs. Each run starts with a count byte and a 2-bit
compression-type indicator packed into the high bits:

```
bit  7  6  5  4  3  2  1  0
     TT C  C  C  C  C  C  C
```

| TT | Run type | Bytes per run |
| --- | --- | --- |
| 00 | None | 2 × (count+1) – literal char/attr pairs |
| 01 | Char | 1 + 2 × count – one char, count attrs |
| 10 | Attr | 2 × (count+1) char bytes, then 1 attr byte… (see below) |
| 11 | Both | char, attr, count repetitions |

For all four types, the actual repeat count is `(count_field + 1)`,
so the count_field 0 means one occurrence and 63 means 64. The
maximum run length is therefore 64.

Encoding details for each type:

- **00 None**: `<count> <c1> <a1> <c2> <a2> ... <c_n> <a_n>` – emit
  each char/attr pair once.
- **01 Char**: `<count> <char> <a1> <a2> ... <a_n>` – emit `char` with
  each of n attributes.
- **10 Attr**: `<count> <attr> <c1> <c2> ... <c_n>` – emit each of n
  characters with the same attribute. (Some implementations swap the
  order; refer to the reference XBin decoder in ACiDDraw / PabloDraw
  for the canonical layout.)
- **11 Both**: `<count> <char> <attr>` – emit the same char/attr pair
  n times.

Compression is optional; a writer may emit all runs as type 00 and
still produce a valid file. Practical compressors choose runs
greedily, switching types at every change in char or attr.

## Where XBin fits

| Format | Width | Custom font | Custom palette | Per-cell attribute |
| --- | --- | --- | --- | --- |
| ANSI (.ANS) | streaming | no | no | sequential |
| BIN (.BIN) | 80 only (or any if length implied) | no | no | yes |
| XBin (.XB) | any | yes | yes | yes |
| TUNDRA (.TND) | any | no | 24-bit RGB per cell | yes |
| ADF (Artworx) | 80 only | yes | yes | yes |
| IDF (iCEDraw) | any | yes | yes | yes |

For final-form archive distribution where reproducibility matters,
XBin and TUNDRA are the two formats artists target. For 24-bit colour
art the answer is TUNDRA; for everything else, XBin.

## SAUCE

XBin files commonly have a SAUCE record. The relevant SAUCE values
are DataType = 6 (XBin), FileType = 0. TInfo1/2 carry width and
height; if those fields are 0 the renderer uses the XBin header.

## Why "0x1A in the magic"?

Including the DOS EOF byte in the four-byte ID lets the file pass the
"is this a text file?" test for DOS terminals and BBS download
descriptions: a download description text file with an XBin appended
to it still types cleanly to the screen, because everything from the
0x1A onward is suppressed by `TYPE`. Tasmaniac applied the same idea
to SAUCE itself; see [[sauce-00.5]].
