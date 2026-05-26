# World System Teletext

Originally developed by the BBC and the IBA in the United Kingdom from
1972 (CEEFAX from the BBC, ORACLE from the IBA), standardized as the
"World System Teletext" specification jointly published by the BBC,
IBA, and BREMA in 1976, revised 1981. Adopted internationally as CCIR
Recommendation 653 (System B) and later reissued as ETSI ETS 300 706
(1997) and ETSI EN 300 706 (2003), "Enhanced Teletext specification".

Teletext is a one-way broadcast data service carried in the vertical
blanking interval (VBI) of a 625-line analogue PAL television signal.
A page is 40 columns by 24 rows of character cells, transmitted at
6.9375 Mbit/s as NRZ data with a clock run-in and framing code. The
character set, graphics characters, and inline control codes are
designed to fit the entire visible state of one row into 40 bytes.

## Page model

Pages are addressed by magazine and page number. The magazine is the
high nibble (1 through 8, encoded with odd parity); the page is two
hex digits MM PPP rendered to viewers as a three-digit decimal-looking
address (100 through 899). Within a page, subpages 0001 through 3F7F
hexadecimal are rotating "carousel" variants of the same page number
broadcast in sequence.

A magazine is transmitted as a serial or parallel stream. In serial
transmission, packets for one magazine are interleaved one magazine at
a time and a new magazine header implicitly terminates the previous.
In parallel transmission (the modern default), magazines are
independent and may interleave packet-by-packet.

## Packet structure

Each VBI line carries one Teletext packet of 45 bytes:

| Field | Bytes | Description |
| --- | --- | --- |
| Clock run-in | 2 | 0x55 0x55 for receiver bit synchronization |
| Framing code | 1 | 0x27 marks packet start |
| Magazine and packet address | 2 | Hamming 8/4 encoded, 3-bit magazine + 5-bit packet number |
| Data | 40 | Packet payload, 7-bit + odd parity per byte |

Packet 0 is the page header (page number, subcode, control bits, page
name). Packets 1 through 24 are the 24 display rows. Packets 26
through 29 are enhancement data (Level 2.5 and above). Packets 30 and
31 are independent data services (IDL).

## Character cell display

Each of the 40 by 24 cells holds one of:

- A G0 alphanumeric character (Latin, national variant).
- A G1 mosaic graphics character (block graphics).
- A control character that sets attributes for the cells to its right
  on the same row.

The cell is 12 pixels wide by 10 pixels tall in the reference raster
(480 wide by 240 tall display area inside a 720 by 576 PAL frame).
Foreground and background colors are drawn at full saturation from a
fixed 8-color palette.

## G0 character set and national options

G0 occupies the 7-bit range 0x20 through 0x7F. The base set is similar
to US-ASCII with substitutions in 13 code points to carry national
characters. The substitutions are switched by three "national option
selection" bits in packet 0 plus an optional packet X/28/0 selection.
National options defined include:

| Region | Codes substituted |
| --- | --- |
| English | £ at 0x23, ½ at 0x7B, etc. |
| French | àéèêçùî at 0x40, 0x5B–0x5F, 0x7B–0x7E |
| German | §ÄÖÜäöüß at 0x40, 0x5B–0x5F, 0x7B–0x7F |
| Swedish/Finnish | ÉÄÖÅÜéäöåü |
| Italian | éçùò |
| Portuguese/Spanish | çñ¡¿ |
| Czech/Slovak | čťž |
| Polish | ąśżźńłŁ |
| Turkish | ğŞİşı |

The full table of 13 national variants is given in clause 15 of EN 300
706.

## G1 mosaic graphics

When the row is in graphics mode, bytes 0x20 through 0x3F and 0x60
through 0x7F are interpreted as 2-by-3 mosaic block characters. Each
of the six bits in the byte (minus bit 5, which separates the two
ranges, and bit 7, which is parity) selects one of six rectangular
sub-cells:

```
+----+----+
| b0 | b1 |
+----+----+
| b2 | b3 |
+----+----+
| b4 | b6 |
+----+----+
```

This yields 64 contiguous mosaic shapes per byte block. A row can be
in "contiguous" mosaic mode (the default after a graphics color
control) or "separated" mosaic mode where each sub-cell is drawn with
a one-pixel gap. Codes 0x40 through 0x5F in graphics mode revert to
G0 letters (so capital letters can be mixed into a graphics row
without leaving graphics mode).

## Control characters (column 0x00–0x1F)

Bytes 0x00 through 0x1F in a row act as "spacing attributes". Each
appears in the cell as a space and changes the rendering of the cells
to its right until end-of-row or until another control code. EN 300
706 Table 3 defines:

| Code | Name |
| --- | --- |
| 0x00 | Alpha black (rarely used at Level 1) |
| 0x01 | Alpha red |
| 0x02 | Alpha green |
| 0x03 | Alpha yellow |
| 0x04 | Alpha blue |
| 0x05 | Alpha magenta |
| 0x06 | Alpha cyan |
| 0x07 | Alpha white |
| 0x08 | Flash |
| 0x09 | Steady |
| 0x0A | End box |
| 0x0B | Start box |
| 0x0C | Normal height |
| 0x0D | Double height |
| 0x0E | Double width |
| 0x0F | Double size |
| 0x10 | Mosaic black |
| 0x11 | Mosaic red |
| 0x12 | Mosaic green |
| 0x13 | Mosaic yellow |
| 0x14 | Mosaic blue |
| 0x15 | Mosaic magenta |
| 0x16 | Mosaic cyan |
| 0x17 | Mosaic white |
| 0x18 | Conceal |
| 0x19 | Contiguous mosaics |
| 0x1A | Separated mosaics |
| 0x1B | ESC (Level 2.5 enhancement escape) |
| 0x1C | Black background |
| 0x1D | New background (set background to current foreground) |
| 0x1E | Hold mosaics |
| 0x1F | Release mosaics |

"Hold mosaics" keeps the last mosaic character displayed in the cell
where a subsequent graphics-color control appears, so the row does not
show a gap when changing colors mid-graphic.

Row state resets at the start of each row: alpha mode, white
foreground, black background, steady, normal height, contiguous
mosaics.

## Level structure

The specification defines presentation levels added over time:

- Level 1: Basic Teletext, the 1976 set: 8 colors, alpha and mosaic
  G0/G1, spacing attributes, double height. Every receiver supports
  Level 1.
- Level 1.5: Extended G0 character sets for non-Latin scripts (Greek,
  Cyrillic, Arabic, Hebrew). Selected via packet X/28/0.
- Level 2.5: Up to 32 simultaneously displayable colors out of a
  4096-color palette, DRCS for user-defined glyphs, side panels,
  proportional spacing reservation, non-spacing attributes carried in
  packets X/26.
- Level 3.5: True proportional spacing, full DRCS, additional flash
  modes, redefinable color map.

Most domestic CEEFAX and ORACLE receivers in the 1980s implemented
Level 1 only. Level 2.5 became common in late 1990s and 2000s digital
TV decoders.

## DRCS – Dynamically Redefinable Character Sets

At Level 2.5 and above, packets X/26 and Y/28 can deliver up to 48
custom 12-by-10 pixel glyphs into a runtime G0 or G1 slot. Each glyph
is encoded as 20 bytes of pixel data, two bits per pixel for color
indices. DRCS is used for station logos, weather symbols, and
non-Latin scripts not covered by the extended G0 sets.

## TOP – Table of Pages

Defined in EN 300 706 Annex A.1. TOP transmits a directory of all
pages in the magazine indexed by topic, with block, group, and page
classifications. Page 1F0 (and the linked AIT, BTT, MPT pages)
contains the table. TOP allows a receiver to display a "page index"
and to navigate to the next or previous page within a topic without
the viewer typing a page number.

## FastText (Flof) – Full Level One Features

The CEEFAX FastText extension uses packet 27/0 to deliver four colored
link targets per page, displayed at the bottom of the screen as red,
green, yellow, and cyan labeled buttons. The viewer presses the
correspondingly colored remote-control key to jump. Packet 27/0
contains the four target page addresses plus a link-control byte.
FastText is independent of TOP and the two are sometimes both present.

## Subtitling

Page 888 (UK) and equivalent page numbers in other countries carry
subtitle pages. A subtitle row uses the "start box" / "end box"
controls (0x0B / 0x0A) to mark which characters should be displayed
over picture. The receiver shows only the boxed regions, leaving the
rest transparent. Subtitles update by transmitting a fresh copy of
page 888 with new boxed text whenever the dialogue changes.

## VBI line allocation

In PAL countries, Teletext is transmitted on VBI lines 6 through 22
and 318 through 335 (with constraints from the broadcaster). Each
line carries exactly one packet. A full page (24 rows + header)
requires 25 packets and so 25 VBI lines, which at 50 fields per
second yields a worst-case page acquisition time around 0.5 seconds
for a page transmitted every field, longer for pages in a large
magazine carousel.

## Hamming coding

Critical fields (magazine and packet address, page number, control
bits) are Hamming 8/4 encoded: each 4 data bits become 8 transmitted
bits with single-error correction. This protects page selection from
the VBI's relatively low signal-to-noise environment. The 40-byte
data payload uses 7 data bits plus odd parity per byte instead, which
detects but does not correct single-bit errors.

## File capture and emulation

Modern Teletext archives are stored as `.t42` (raw 42-byte packet
streams, 40 data bytes plus the two magazine/packet bytes) or
`.vbi` (full VBI samples). The `vhs-teletext` decoder, the BBC
"Domesday" subtitle archive, and ETSI's reference decoder consume
these formats. Pages can also be carried in MPEG-2 transport streams
using DVB Teletext (ETSI EN 300 472) as private data PIDs.

## References

- ETSI EN 300 706 v1.2.1 (2003-04), "Enhanced Teletext specification".
- ETSI ETS 300 706 (1997), prior edition of the same.
- CCIR Recommendation 653-2, "Teletext systems", ITU-R, 1990.
- BBC, IBA, BREMA, "Broadcast Teletext Specification", September 1976,
  revised September 1981.
- ETSI EN 300 472, "Specification for conveying ITU-R System B
  Teletext in DVB bitstreams".
