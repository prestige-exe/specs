# NAPLPS – North American Presentation Level Protocol Syntax

Standardized as ANSI X3.110-1983 in the United States and CSA T500-1983
in Canada. Developed jointly by AT&T and the Canadian Department of
Communications from the Telidon videotex system. Adopted by Prodigy
(launched 1988), Bell Canada Alex, Knight-Ridder Viewtron, Time-Mirror
Gateway, and the AT&T Sceptre terminal. ISO later folded the same
escape mechanism into ISO/IEC 2022 "Code Extension Techniques" and
issued the harmonized graphics syntax as ISO/IEC 9281.

NAPLPS is a presentation-level protocol. It encodes vector graphics,
geometric primitives, color selection, and dynamically defined
characters as a stream of 7-bit (or optionally 8-bit) ASCII bytes,
which means a videotex frame can travel through any text-transparent
asynchronous link including a 1200 bps Bell 212A modem. The receiver
parses the byte stream and draws into a Picture Description Buffer
which is then rasterized to the display.

## Code extension and the seven code-set model

NAPLPS uses the ISO 2022 code extension framework. The 7-bit code
space is divided into a C0 control set, a GL graphic set occupying
columns 2 through 7 (0x20 through 0x7F), and locking shifts to bring
alternate sets into GL or GR. In 8-bit operation a C1 set occupies
columns 8 and 9 (0x80 through 0x9F) and GR occupies columns A through
F (0xA0 through 0xFF).

Four graphic sets G0, G1, G2, G3 can be designated into the GL or GR
positions. The default designations on power-up are:

| Set | Default content |
| --- | --- |
| G0 | ASCII (Primary Character Set) |
| G1 | PDI – Picture Description Instructions |
| G2 | Supplementary Character Set (accented letters, symbols) |
| G3 | Mosaic / DRCS (Dynamically Redefinable Character Set) |

Two C-sets are available:

| Set | Default content |
| --- | --- |
| C0 | Primary control set (cursor control, NUL, ESC, etc.) |
| C1 | Supplementary control set (delimiters for protocol features) |

Locking and single shifts move sets between GL and GR:

| Function | Byte | Action |
| --- | --- | --- |
| LS0 (SI) | 0x0F | Lock G0 into GL |
| LS1 (SO) | 0x0E | Lock G1 into GL |
| LS2 | ESC 6E | Lock G2 into GL |
| LS3 | ESC 6F | Lock G3 into GL |
| LS1R | ESC 7E | Lock G1 into GR |
| LS2R | ESC 7D | Lock G2 into GR |
| LS3R | ESC 7C | Lock G3 into GR |
| SS2 | ESC 4E | Single-shift G2 for one character |
| SS3 | ESC 4F | Single-shift G3 for one character |

Designation uses the ISO 2022 sequence `ESC <I> <F>` where `<I>`
indicates the target set (94-char, 96-char, multi-byte) and `<F>` is
the final byte registered to the desired set. The PDI set has the
registered final byte assigned by ISO; ASCII is assigned the
conventional `B` final.

## PDI – Picture Description Instructions

When G1 holds the PDI set and G1 is in GL, the bytes 0x20 through 0x7F
encode geometric drawing opcodes plus their operand bytes. PDI opcodes
occupy columns 2 and 3 (0x20 through 0x3F). The opcodes defined by
X3.110 are:

| Hex | Mnemonic | Meaning |
| --- | --- | --- |
| 0x20 | RESET | Reset graphics state to defaults |
| 0x21 | DOMAIN | Set logical coordinate domain, point size, line texture |
| 0x22 | TEXT | Render character string with current text attributes |
| 0x23 | TEXTURE | Set fill texture, line texture, highlight |
| 0x24 | POINT SET ABS | Move logical pen to absolute point, plot pixel |
| 0x25 | POINT SET REL | Move pen by relative delta, plot pixel |
| 0x26 | POINT ABS | Plot single point at absolute coordinate |
| 0x27 | POINT REL | Plot single point at relative coordinate |
| 0x28 | LINE ABS | Draw line from current point to absolute point |
| 0x29 | LINE REL | Draw line by relative delta |
| 0x2A | SET & LINE ABS | Move and draw to absolute point |
| 0x2B | SET & LINE REL | Move and draw by relative delta |
| 0x2C | ARC OUTLINED | Outline arc, three points define curve |
| 0x2D | ARC FILLED | Filled arc / pie segment |
| 0x2E | ARC OUTLINED FILLED | Outlined and filled arc |
| 0x2F | ARC OUTLINED FILLED HIGHLIGHTED | Outline + fill + highlight |
| 0x30 | RECTANGLE OUTLINED | Axis-aligned rectangle outline |
| 0x31 | RECTANGLE FILLED | Filled rectangle |
| 0x32 | RECTANGLE OUTLINED FILLED | Both outline and fill |
| 0x33 | RECTANGLE HIGHLIGHTED | Outline + highlight border |
| 0x34 | POLYGON OUTLINED | Closed polygon outline |
| 0x35 | POLYGON FILLED | Filled polygon |
| 0x36 | POLYGON OUTLINED FILLED | Outline and fill |
| 0x37 | POLYGON HIGHLIGHTED | Outline + highlight |
| 0x38 | INCREMENTAL POINT | Bitmap point cloud, delta-encoded |
| 0x39 | INCREMENTAL LINE | Bitmap polyline, delta-encoded |
| 0x3A | INCREMENTAL POLYGON FILLED | Bitmap polygon, delta-encoded |
| 0x3B | INCREMENTAL POLYGON HIGHLIGHTED | Above + border highlight |
| 0x3C | SET COLOR | Select drawing color (operand follows) |
| 0x3D | WAIT | Pause for operand-supplied time |
| 0x3E | SELECT COLOR | Select color from look-up table |
| 0x3F | BLINK | Define blink phase and rate |

Each opcode is followed by zero or more operand bytes. Operand byte
count is implicit: the parser reads operands until it encounters
another byte in columns 2 or 3, which is the next opcode.

## Numeric operand encoding

Operand bytes occupy columns 4 through 7 (0x40 through 0x7F). Bit 5 of
each operand byte indicates whether more operand bytes follow for the
same coordinate (multi-precision) or whether this byte ends the
current value. The remaining bits pack X and Y in interleaved fashion
for two-dimensional coordinates, sign in the high bit of the value
after de-interleaving.

In short-form single-byte operand:

```
bit  7  6  5  4  3  2  1  0
     1  S  m  Xs Ys X2 Y2 X1 Y1
```

(Schematic; the exact bit packing per byte and the multi-byte
extension are tabled in clause 5.4 of X3.110.) A coordinate is a
sign-magnitude fraction interpreted relative to the current logical
DOMAIN. By raising the operand byte count via the "more" bit, the
encoder gains precision: typical NAPLPS frames use two or three
operand bytes per coordinate pair, yielding 6 to 10 bits per axis.

The logical coordinate system is normalized. The DOMAIN opcode sets
the unit square; coordinates are mapped into device pixels by the
decoder.

## Color selection

NAPLPS supports two color modes. SET COLOR (0x3C) takes an immediate
RGB triple (or YBR triple in alternate mode) directly in the operand
bytes. SELECT COLOR (0x3E) takes an index into a programmable look-up
table previously loaded via the same SET COLOR opcode in look-up mode.
Look-up table sizes of 8, 16, and 32 entries are defined. Channel
precision is selectable between 1 bit, 2 bits, 3 bits, or 4 bits per
channel by service-level negotiation.

A common Prodigy frame used a 16-entry palette with 3 bits per
channel, giving 512 selectable colors out of which 16 are loaded.

## DRCS – Dynamically Redefinable Character Sets

DRCS lets a frame ship its own glyphs. A DRCS character is a small
bitmap, typically 6 by 10 cells, encoded as PDI INCREMENTAL POINT data
into a slot within the G3 set. Once defined, a code point in G3
renders that bitmap. DRCS is how NAPLPS frames carry custom fonts,
icons, and small sprites without leaving the 7-bit byte stream.

## Mosaic graphics

The mosaic set is a fallback for receivers that do not implement PDI.
Mosaic codes in G3 render fixed 2-by-3 block characters in the
foreground color, similar in spirit to the Teletext G1 mosaic set. A
frame can fall back to mosaic display on a Telidon Level 1 terminal.

## Frame composition

A NAPLPS frame begins with a header that designates code sets and
selects the screen dimensions, followed by a sequence of operations
that mix:

- Cursor control C0 codes (CR, LF, BS, FF) for text positioning.
- ASCII or supplementary text in G0/G2.
- PDI graphics opcodes through G1.
- DRCS definitions through G3.
- C1 protocol delimiters that mark service boundaries.

There is no global file framing. Frames stream end to end. End-of-frame
is signaled by an application-level delimiter, on Prodigy the proprietary
TBOL packet wrapper, on Telidon a CCITT T.101 service envelope.

## Level structure

ANSI X3.110 defines presentation level capabilities. Implementations
quote conformance to:

- Level 1: alpha-mosaic, 7-bit text, simple block graphics. Compatible
  with European CEPT Level 1 receivers.
- Level 2: alpha-geometric, PDI primitives, DRCS.
- Level 3: alpha-photographic, run-length-encoded bitmap regions.
- Level 4: alpha-animated, time-sequenced operations.

Prodigy targeted a constrained Level 2 profile to keep frame size
small over 1200 bps dial-up.

## Relation to CEPT and the unified syntax

The European CEPT T/CD 6-1 videotex profile (Antiope, Bildschirmtext,
Minitel) uses the same ISO 2022 escape framework but a different
graphic vocabulary built on mosaic and parallel attributes. CCITT
Recommendation T.101 (1988) defined three interworking data syntaxes:

- Data Syntax I: Japanese CAPTAIN, JIS X 0208 with mosaic.
- Data Syntax II: European CEPT alpha-mosaic.
- Data Syntax III: NAPLPS alpha-geometric / X3.110.

T.101 specifies gateway behavior between the three syntaxes. The
practical interchange between NAPLPS and CEPT was limited by the
geometric versus mosaic divide; CEPT receivers could not render PDI.

## File extensions and conventions

NAPLPS frames captured to disk on DOS systems carry `.NAP` or `.PDI`
extensions. Header bytes are not standardized at the file level. A
runtime presenter (such as the AT&T Sceptre terminal firmware, or
later DOS players including NAPLPS-DOS and the Prodigy Reception
System) drives the parser directly from the byte stream.

## References

- ANSI X3.110-1983, "Videotex / Teletext Presentation Level Protocol
  Syntax (North American PLPS)", American National Standards Institute,
  1983.
- CSA T500-1983, Canadian Standards Association, 1983.
- CCITT Recommendation T.101, "International Interworking for Videotex
  Services", 1988.
- ISO/IEC 9281-1:1990, "Picture Coding Methods Part 1: Identification".
- ISO/IEC 2022:1994, "Information Technology – Character Code
  Structure and Extension Techniques".
- Fleming, J., "The NAPLPS Standard", Byte Magazine, February 1985.
- Bell-Northern Research, "Telidon Reference Manual", 1981.
