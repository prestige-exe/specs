# TUNDRA – 24-bit Color Text-Mode Image Format

by Joint Effort. Original public description posted with the TundraDraw
editor circa 2002, but in widespread use as the 24-bit "true colour"
ANSI/ASCII art format from the late 1990s onward as artists
experimented with extending the palette beyond 16 colours.

TUNDRA stores a text-mode character grid where each cell carries a
24-bit RGB foreground colour and a 24-bit RGB background colour
instead of the 4-bit attribute byte of BIN / ADF / IDF / XBin. It is
the format of choice for art that uses colour gradients and shading
beyond what the VGA 16-colour palette can express.

## File layout

```
+--------------------+
| Header (9 bytes)   |
+--------------------+
| Cell stream        |  variable length, terminated by EOF (or SAUCE)
+--------------------+
| SAUCE (optional)   |
+--------------------+
```

## Header (9 bytes)

| Offset | Size | Field | Description |
| --- | --- | --- | --- |
| 0 | 1 | Version | Currently 24 (0x18) |
| 1 | 8 | ID | ASCII "TUNDRA24" |

The version byte indicates the bit depth (always 24 in released
versions; a future 8-bit or 16-bit palette variant would change it).
The "TUNDRA24" magic identifies the file unambiguously.

## Cell stream

Cells are stored row by row, left to right. The decoder maintains an
implicit cursor at (0,0) and advances one cell per character drawn.

Each cell record begins with a *type* byte that selects one of four
encodings, optionally followed by an explicit position, optionally
followed by foreground/background RGB, and finally a CP437 character
byte.

The type byte uses four bit flags:

| Bit | Value | Meaning when set |
| --- | --- | --- |
| 0 | 0x01 | Position follows (4 bytes: row16 then column16, big-endian) |
| 1 | 0x02 | Foreground RGB follows (4 bytes: 0x00, R, G, B) |
| 2 | 0x04 | Background RGB follows (4 bytes: 0x00, R, G, B) |
| 3..7 | – | Reserved, must be zero |

After all the optional fields specified by the type byte, the cell's
character byte follows (one byte, CP437).

Decoding loop:

```pseudo
row = 0; col = 0
fg = (255,255,255); bg = (0,0,0)
while reader has bytes:
    t = read_u8()
    if t == 1: row = read_u16_be(); col = read_u16_be()
    if t == 2: skip(1); fg = (read_u8(), read_u8(), read_u8())
    if t == 4: skip(1); bg = (read_u8(), read_u8(), read_u8())
    if t == 6: ...  # both fg and bg follow
    if t == 8: ...  # raw character with no position, no colour change
    ch = read_u8()
    draw(row, col, ch, fg, bg)
    col += 1
```

The first byte after the colour blocks is the literal character. The
prefixed zero in each RGB block (`0x00, R, G, B`) is structural – it
makes each RGB field exactly four bytes, which simplifies file
parsing on platforms that prefer aligned reads.

When type byte bit 0 (Position) is set, the next four bytes carry an
explicit (row, column) as two big-endian 16-bit values, overriding
the cursor.

When neither colour flag is set, the cell uses the most recent fg/bg
the decoder has seen. The initial values are an implementation choice;
white-on-black is conventional.

## Reserved characters in the stream

The four type-byte values 1, 2, 4, and 6 (and any combination of
those flag bits including position) must not occur as character bytes
without the corresponding type prefix; the parser would mistake them
for type bytes. The encoder works around this by always emitting a
type byte before each cell – type 0 is "no flags, just a character",
but a character that happens to equal 1/2/4/6/etc. is preceded by an
attribute change or position change that pads out the type byte to a
non-conflicting value.

In practice, TUNDRA writers emit a type byte of 8 (all flags clear,
high bit set as a sentinel) for plain character cells in some
implementations; refer to TundraDraw and Ansilove for the canonical
encoder behaviour.

## Width and height

TUNDRA does not encode width or height in its header – they are
derived from SAUCE if present, or from the cursor positions reached
during decoding. Without SAUCE, viewers typically assume 80 columns.

## SAUCE

If present, DataType = 1 (Character), FileType = 8 (TUNDRA). TInfo1
and TInfo2 carry width and rows; TInfoS carries the font name.

## Why 24-bit?

By the early 2000s, monitors and graphics adapters had outgrown the
16-colour text-mode palette by an order of magnitude, and ANSI artists
wanted gradient shading without the 4-colour-cycle dithering tricks
required by VGA. TUNDRA is the result – the same grid-based character
art, with no palette constraint at all.

The cost is file size: TUNDRA files are several times larger than
their equivalent XBin form for any non-trivial image, and the format
has no compression layer. Renderers that target both formats
typically translate TUNDRA's 24-bit colours to the nearest VGA index
when the output target is a true text-mode console.

## File identification

- File extension: `.TND`.
- Magic: byte 0 = 0x18 (version 24), bytes 1..8 = ASCII "TUNDRA24".
- Minimum file size: 9 (header) + at least one cell (2 bytes).
