# AVATAR / AVT-0+ – Advanced Video Attribute Terminal Assembler and Recreator

by George A. Stanislav
Original: AVATAR Level 0, FidoNet Technical Standards Committee
document FSC-0025, 26 December 1988
Extension: AVT/0+ ("Avatar Level 0 Plus")

AVATAR was designed for FidoNet BBS systems as a more efficient
alternative to ANSI. ANSI sequences for cursor positioning and colour
are large – three to seven bytes per command. AVATAR encodes the same
operations in two or three bytes by using control characters in the
0x16–0x19 range. On a 2400 bps modem and a screen full of colour
changes, the savings are visible.

## Vocabulary

| Code | Hex | Mnemonic | Meaning |
| --- | --- | --- | --- |
| ^V ^A | 16 01 | Set attribute | Next byte is a video attribute |
| ^V ^B | 16 02 | Blink on | Toggle blink bit |
| ^V ^C | 16 03 | Up | Cursor up one row |
| ^V ^D | 16 04 | Down | Cursor down one row |
| ^V ^E | 16 05 | Right | Cursor right one column |
| ^V ^F | 16 06 | Left | Cursor left one column |
| ^V ^G | 16 07 | Clear EOL | Clear from cursor to end of line |
| ^V ^H r c | 16 08 r c | Move | Move cursor to row r, column c |
| ^V ^I | 16 09 | (Reserved) | |
| ^V ^J | 16 0A | (Reserved) | |
| ^V ^K | 16 0B | (Reserved) | |
| ^V ^L | 16 0C | Clear screen | Clear screen, home cursor, attribute = current |
| ^V ^Y c n | 16 19 c n | RLE | Repeat character c, n times (n = 1..255) |
| ^L | 0C | Form feed | Clear screen |
| ^X | 18 | Cancel | Reserved (FSC-0037 extension) |
| ^Y | 19 | Reserved | |

Other byte values pass through unchanged.

The ^V (0x16) escape leads almost every command. The trailing byte is
the actual command code. Most commands take no parameters; a few take
one or two parameter bytes.

## Attribute byte (^V ^A)

The attribute byte is a single byte interpreted exactly like a VGA
text-mode attribute:

```
bit  7  6  5  4  3  2  1  0
     B  bR bG bB I  fR fG fB
```

| Bits | Meaning |
| --- | --- |
| 0..2 | Foreground colour (R/G/B) |
| 3 | Foreground intensity |
| 4..6 | Background colour |
| 7 | Blink (or background intensity in iCE-mode terminals) |

Foreground/background colour bits map directly to the IBM PC palette:

| Value | Colour |
| --- | --- |
| 0 | Black |
| 1 | Blue |
| 2 | Green |
| 3 | Cyan |
| 4 | Red |
| 5 | Magenta |
| 6 | Brown |
| 7 | Light grey |

With the intensity bit set, those eight become dark grey, light blue,
light green, light cyan, light red, light magenta, yellow, and white.

This is the same layout as a CGA/EGA/VGA text-mode attribute byte, so
^V ^A followed by the literal attribute byte sets the colours
identically to writing the same byte into video memory.

## Cursor positioning

```
^V ^H <row> <col>
```

Row and column are 1-based, single bytes. (1,1) is upper left. Values
above 25 / 80 are valid for screens taller or wider than the IBM PC
default.

The four single-step cursor movements (^V ^C, ^D, ^E, ^F) wrap or stop
at the screen edge depending on the terminal; FSC-0025 does not
specify the edge behaviour.

## Run-length encoding (^V ^Y)

```
^V ^Y <char> <count>
```

Writes `<char>` `<count>` times. Count is encoded as a single byte, so
1..255. RLE is the main reason AVATAR fits a screen-full of art into
fewer bytes than the same screen in ANSI – long horizontal runs of the
same character (block art borders, fill characters) collapse to three
bytes per run regardless of length.

## AVT/0+ extensions

AVT/0+ (alternately "Avatar Level 0 Plus") added a small set of higher
codes. The two most common are:

| Code | Meaning |
| --- | --- |
| ^V ^X | (cancel pending command) |
| ^V ^P p q r ...  | Pattern fill: q rows of `p` characters with attribute r |
| ^V ^N | Goto X,Y from one-byte X (column) and one-byte Y (row) |
| ^Y c n | Direct RLE (no ^V) – `c` for `n` repetitions |

The ^Y c n short form is the one most widely implemented; ^V ^Y is the
fully Level 0 conformant form. Both produce the same RLE expansion.

## Comparison with ANSI

For the colour change "bright white on blue":

- ANSI: `ESC [ 1 ; 37 ; 44 m` = 11 bytes
- AVATAR: `^V ^A 1F` = 3 bytes

For "draw a 60-character line of `═`":

- ANSI: 60 bytes of `═` (0xCD)
- AVATAR: `^V ^Y CD 3C` = 4 bytes

AVATAR was particularly popular on RemoteAccess and QuickBBS systems
during the 1990–1993 period. After 14400/V.32bis modems became common
the bandwidth argument weakened and ANSI re-established dominance,
especially since most modern terminals and emulators implement ANSI
but not AVATAR.

## File format

A pure AVATAR file is a stream of CP437 bytes interspersed with the
control sequences above. No header, no terminator other than EOF. A
trailing SAUCE record (see [[sauce-00.5]]) is permitted; DataType =
Character, FileType = 3 (AVATAR).

## SAUCE TFlags for AVATAR

For DataType Character / FileType AVATAR, the ANSiFlags byte applies
just as for ANSI:

| Bit | Meaning |
| --- | --- |
| 0 (B) | Non-blink (iCE colors) |
| 1..2 (LS) | Letter spacing: 0=legacy, 1=8-pixel, 2=9-pixel |
| 3..4 (AR) | Aspect ratio: 0=legacy, 1=square pixels, 2=stretched |

See SAUCE for full TInfoS font-name convention.

## Implementation note

The minimal AVATAR decoder is a state machine with two states – "data"
and "escape". In the data state, byte ^V (0x16) transitions to escape;
any other byte is written to the screen. In the escape state, the
command byte selects the action (with 0 or more parameter bytes read
from the stream before returning to data).
