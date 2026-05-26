# ANSI Escape Sequences as Used in BBS ANSI Art

Underlying standard: ECMA-48 / ANSI X3.64 / ISO/IEC 6429,
"Control Functions for Coded Character Sets". The full ECMA-48
specification is enormous; what BBSes and ANSI artists actually used is
a small subset implemented by MS-DOS's ANSI.SYS device driver (Microsoft
Corporation, shipped with PC DOS / MS-DOS 2.0 in 1983).

This document covers that subset. For the full standard, see ECMA-48
5th edition (1991).

## Anatomy of a sequence

An ANSI control sequence has the form:

```
ESC [ <params> <intermediate> <final>
```

| Byte | Range | Description |
| --- | --- | --- |
| ESC | 0x1B | Introducer |
| `[` | 0x5B | CSI – Control Sequence Introducer (in 7-bit form) |
| params | 0x30–0x3F | Zero or more parameters, decimal numbers separated by `;` |
| intermediate | 0x20–0x2F | Rare in BBS art; ANSI.SYS does not use |
| final | 0x40–0x7E | Single byte selecting the command |

A single ESC byte at 0x1B is the CP437 left-arrow glyph (←). When ANSI
art is loaded into a non-ANSI-aware viewer, sequences appear as
`←[1;31m...` and the like.

In an 8-bit terminal CSI may also be 0x9B (single byte) instead of
`ESC [`; ANSI.SYS does not accept the 8-bit form, and neither do
practically any BBS clients, so all art uses the two-byte form.

## Final bytes used in BBS ANSI

Of the dozens of ECMA-48 commands, only a handful appear in BBS art:

| Final | Mnemonic | Name | Used for |
| --- | --- | --- | --- |
| `A` | CUU | Cursor Up | Animation |
| `B` | CUD | Cursor Down | Animation |
| `C` | CUF | Cursor Forward | Skip / animation |
| `D` | CUB | Cursor Back | Animation |
| `E` | CNL | Cursor Next Line | Rare |
| `F` | CPL | Cursor Previous Line | Rare |
| `G` | CHA | Cursor Horizontal Absolute | Animation |
| `H` | CUP | Cursor Position | Set cursor to row;col |
| `J` | ED | Erase in Display | Clear screen |
| `K` | EL | Erase in Line | Clear to end-of-line |
| `S` | SU | Scroll Up | Rare |
| `T` | SD | Scroll Down | Rare |
| `f` | HVP | Horizontal & Vertical Position | Same as CUP |
| `m` | SGR | Select Graphic Rendition | Set colour and attributes |
| `s` | SCP | Save Cursor Position | ANSI.SYS extension |
| `u` | RCP | Restore Cursor Position | ANSI.SYS extension |
| `n` | DSR | Device Status Report | Used for "where am I" queries |
| `h` | SM | Set Mode | ANSI.SYS extension for modes |
| `l` | RM | Reset Mode | ANSI.SYS extension for modes |

## Cursor movement

Default parameter for all movement commands is 1.

```
ESC [ <n> A      Up n rows (stops at row 1)
ESC [ <n> B      Down n rows (stops at last row)
ESC [ <n> C      Right n columns (stops at last column)
ESC [ <n> D      Left n columns (stops at column 1)
ESC [ <r> ; <c> H    Move to row r, column c (1-based)
ESC [ <r> ; <c> f    Same as above
ESC [ s          Save cursor position (ANSI.SYS extension)
ESC [ u          Restore saved cursor position
```

Cursor positioning is 1-based: `ESC [ 1 ; 1 H` is the home position.
Coordinates outside the screen are clamped to the screen edge.

## Erasure

```
ESC [ J     Erase from cursor to end of screen
ESC [ 0 J   Same as above
ESC [ 1 J   Erase from beginning of screen to cursor
ESC [ 2 J   Erase entire screen (cursor stays put)

ESC [ K     Erase from cursor to end of line
ESC [ 0 K   Same
ESC [ 1 K   Erase from beginning of line to cursor
ESC [ 2 K   Erase entire line
```

`ESC [ 2 J` followed by `ESC [ 1 ; 1 H` is the canonical "clear and
home" used by every BBS at login. In the strictest ANSI.SYS
implementation `ESC [ 2 J` also homes the cursor – many BBS scripts
relied on this and broke on more standards-compliant terminals.

## Colours and attributes – SGR

`ESC [ <attr> ; <attr> ; ... m`

Each attribute is a small integer. Multiple may be combined with `;`.
`ESC [ m` or `ESC [ 0 m` resets everything to default.

| Code | Attribute |
| --- | --- |
| 0 | Reset all attributes |
| 1 | Bold / bright foreground |
| 2 | Faint (rarely supported) |
| 3 | Italic (rarely supported on BBS) |
| 4 | Underline (monochrome terminals only on DOS) |
| 5 | Slow blink |
| 6 | Rapid blink (rarely supported) |
| 7 | Reverse video |
| 8 | Conceal / invisible |
| 22 | Bold off |
| 24 | Underline off |
| 25 | Blink off |
| 27 | Reverse off |
| 28 | Conceal off |

Foreground colour:

| Code | Colour |
| --- | --- |
| 30 | Black |
| 31 | Red |
| 32 | Green |
| 33 | Yellow / brown |
| 34 | Blue |
| 35 | Magenta |
| 36 | Cyan |
| 37 | White (light grey) |
| 39 | Default foreground |

Background colour (only 8 colours – no "bright" backgrounds on a CGA
text card without the `iCE colors` mode hack, see below):

| Code | Colour |
| --- | --- |
| 40 | Black |
| 41 | Red |
| 42 | Green |
| 43 | Yellow / brown |
| 44 | Blue |
| 45 | Magenta |
| 46 | Cyan |
| 47 | White (light grey) |
| 49 | Default background |

The "bright" foreground colours are produced by combining attribute 1
(bold) with the foreground colour. Code 31 alone is dim red; `1;31` is
bright red. Some renderers also treat attribute 5 (blink) as
"bright background" – see iCE colors.

### Composite colour table for BBS ANSI

The IBM PC text-mode palette has 16 fixed colours. ANSI art targets:

| Foreground | Code | Background | Code |
| --- | --- | --- | --- |
| Black | 0;30 | Black | 40 |
| Red | 0;31 | Red | 41 |
| Green | 0;32 | Green | 42 |
| Yellow / brown | 0;33 | Yellow / brown | 43 |
| Blue | 0;34 | Blue | 44 |
| Magenta | 0;35 | Magenta | 45 |
| Cyan | 0;36 | Cyan | 46 |
| Light grey | 0;37 | Light grey | 47 |
| Dark grey | 1;30 | – | – |
| Light red | 1;31 | – | – |
| Light green | 1;32 | – | – |
| Yellow | 1;33 | – | – |
| Light blue | 1;34 | – | – |
| Light magenta | 1;35 | – | – |
| Light cyan | 1;36 | – | – |
| White | 1;37 | – | – |

Background colours codes 40–47 only address the dim half of the
palette. The high bit of the attribute byte in VGA text mode is the
blink bit by default, which is why "bright" backgrounds need a special
mode.

### iCE colors

ACiD coined the term "iCE colors" for VGA text mode with the blink bit
re-purposed as a high-intensity background bit (BIOS function INT 10h,
AX=1003h, BL=00h disables blink). When this mode is active, an SGR
sequence of `ESC [ 5 ; 4<n> m` selects a bright background – effectively
extending the background palette from 8 to 16 colours.

iCE colors files are not portable to terminals that have not been
switched into this mode. SAUCE records this in `TFlags` bit 0 of
character-type files; see [[sauce-00.5]].

## Other modes (ANSI.SYS extensions)

```
ESC [ = <n> h    Set screen mode
ESC [ = <n> l    Reset screen mode
ESC [ ? 7 h      Enable line wrap
ESC [ ? 7 l      Disable line wrap (some clones)
```

ANSI.SYS screen modes:

| n | Mode |
| --- | --- |
| 0 | 40 × 25 monochrome text |
| 1 | 40 × 25 colour text |
| 2 | 80 × 25 monochrome text |
| 3 | 80 × 25 colour text (default for BBS art) |
| 4 | 320 × 200 4-colour graphics |
| 5 | 320 × 200 monochrome graphics |
| 6 | 640 × 200 monochrome graphics |
| 7 | Wrap at end of line (on/off via h/l) |
| 13 | 320 × 200 colour graphics |
| 14 | 640 × 200 colour graphics |
| 15 | 640 × 350 monochrome graphics |
| 16 | 640 × 350 colour graphics |
| 17 | 640 × 480 monochrome graphics |
| 18 | 640 × 480 16-colour graphics |
| 19 | 320 × 200 256-colour graphics |

ANSI art is overwhelmingly mode 3 (80×25 colour text), occasionally
mode 4 / extended 80×50 with the VGA 8×8 font.

## Keyboard remapping

ANSI.SYS also accepts keyboard re-mapping sequences:

```
ESC [ <ascii> ; <ascii> p
```

This was used by some BBSes to trap user keystrokes and re-purpose
them. Notoriously it was also a malware vector ("ANSI bombs") on
public BBSes prior to MS-DOS 5 — a malicious .ANS file could remap a
user's `DIR` key to `del *.* /y`. Modern terminals do not implement
keyboard remapping and ANSI bombs are a historical curiosity.

## DSR – Device Status Report

```
ESC [ 6 n     Report cursor position
              Terminal replies:  ESC [ <r> ; <c> R
```

Used by some "ANSI detection" routines on login: send ESC [ 6 n and
look for the R response within a short window. If nothing comes back,
the caller's terminal probably is not ANSI.

## Character set

The character set used by ANSI art on a DOS BBS is CP437 (see
[[cp437]]). Text bytes outside the escape sequences are CP437 code
points, and the box-drawing glyphs (0xB0–0xDF) are what most artwork is
built from.

## ANSI art file conventions

- File extension is `.ANS`.
- Default screen width is 80, default height grows as the file does.
- A SAUCE record (see [[sauce-00.5]]) at the end of the file may
  declare width, height, iCE colors, and a non-default font/codepage.
- An EOF byte (0x1A) immediately before the SAUCE record terminates the
  visible content – DOS `TYPE` will stop here, so a sauce'd ANSI file
  displays cleanly on a DOS prompt as well as on a SAUCE-aware viewer.
- ANSImation is just ANSI with deliberate cursor positioning and
  pauses; the encoding is identical, but renderers play the file
  character-by-character at a controlled rate.

## Common BBS sequence cheatsheet

```
ESC [ 2 J ESC [ 1 ; 1 H     Clear screen, cursor home
ESC [ 0 m                   Reset attributes
ESC [ 1 ; 31 m              Bright red on default background
ESC [ 0 ; 33 ; 44 m         Brown on blue
ESC [ 1 ; 37 ; 44 m         White on blue (the classic "menu" colour)
ESC [ 5 ; 41 m              Blinking (or bright-background, in iCE) red
ESC [ s ... ESC [ u         Save / restore cursor position
ESC [ 12 ; 35 H             Move to row 12, column 35
ESC [ K                     Clear to end-of-line
```
