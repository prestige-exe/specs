# ANSI.SYS – The DOS Console Driver

Microsoft Corporation. Shipped with PC-DOS 2.0 and MS-DOS 2.0 (1983)
and every subsequent retail DOS through MS-DOS 6.22 and Windows
9x's `COMMAND.COM`. Documented in the "Microsoft MS-DOS Programmer's
Reference" (versions 3.3 through 6) and described in detail in Ralf
Brown's Interrupt List under INT 29h and the CON device.

ANSI.SYS is an installable character-device driver that hooks the
console output path. When loaded, output written to standard output
or standard error (via INT 21h functions 02h, 06h, 09h, 40h on file
handle 1 or 2) passes through ANSI.SYS, which scans for ESC `[`
sequences and translates them into BIOS INT 10h video calls. Without
ANSI.SYS the same bytes pass straight to DOS's built-in console
driver, which prints the ESC byte as the CP437 left-arrow glyph and
ignores nothing.

This document describes ANSI.SYS specifically. For the underlying
ECMA-48 standard see [[ansi-escape]].

## Loading

ANSI.SYS is loaded from `CONFIG.SYS` as a device driver:

```
DEVICE=C:\DOS\ANSI.SYS
```

Once `HIMEM.SYS` is loaded (DOS 5 and later), it can be loaded into
upper memory:

```
DEVICEHIGH=C:\DOS\ANSI.SYS
```

ANSI.SYS occupies roughly 4 KB of conventional memory (DOS 5
versions) and is not unloadable without rebooting.

## Command-line options (DOS 4 and later)

The driver accepts options on the `DEVICE=` line:

| Option | Effect |
| --- | --- |
| /X | Allows independent remapping of extended keys (gray keys distinct from main-keyboard equivalents). |
| /K | Disables extended-key processing entirely; the keyboard behaves like an 84-key AT keyboard. Also disables keyboard reassignment in some MS-DOS 6 releases. |
| /R | Adjusts line wrap and slows scroll for screen readers (DOS 6.2 and later). |
| /S | Disables keyboard redefinition (some third-party clones; not all official ANSI.SYS releases honor this). |

PC-DOS 7 added `/L` which preserves the screen length set by `MODE
CON LINES=` across mode changes.

The `/K` switch is the one most relevant to BBS security: with `/K`,
the `ESC [ ... p` keyboard reassignment sequence is ignored, which
defeats ANSI bomb files. Most BBS users never set `/K` because they
also never set anything else, and the option was undocumented in
early DOS releases.

## Supported escape sequences

ANSI.SYS implements a subset of ECMA-48. The sequences it recognizes
are:

| Sequence | Mnemonic | Function |
| --- | --- | --- |
| `ESC [ n A` | CUU | Cursor up n rows |
| `ESC [ n B` | CUD | Cursor down n rows |
| `ESC [ n C` | CUF | Cursor forward n columns |
| `ESC [ n D` | CUB | Cursor backward n columns |
| `ESC [ r ; c H` | CUP | Cursor to row r, column c (1-based) |
| `ESC [ r ; c f` | HVP | Same as CUP |
| `ESC [ s` | SCP | Save cursor position |
| `ESC [ u` | RCP | Restore cursor position |
| `ESC [ 2 J` | ED | Erase display, home cursor |
| `ESC [ K` | EL | Erase from cursor to end of line |
| `ESC [ ...params m` | SGR | Select graphic rendition |
| `ESC [ 6 n` | DSR | Device status report; replies with `ESC [ r;c R` |
| `ESC [ = n h` | SM | Set screen mode |
| `ESC [ = n l` | RM | Reset screen mode |
| `ESC [ = 7 h` / `l` | SM/RM | Enable or disable line wrap at column 80 |
| `ESC [ src ; dst p` | (proprietary) | Keyboard reassignment |

Sequences not in this list are silently consumed. `ED` with
parameter 0 or 1 (erase from cursor / erase to cursor) is not
implemented by ANSI.SYS even though the ECMA-48 standard defines
them; only `ESC [ 2 J` works, and it also homes the cursor (a
deviation from ECMA-48). Likewise `EL` only erases from cursor to
end of line; the `0`, `1`, and `2` parameter variants are not
supported.

## SGR – supported attributes

The recognized SGR parameters are:

| Parameter | Effect |
| --- | --- |
| 0 | All attributes off (white on black, no bold, no blink) |
| 1 | Bold / high-intensity foreground |
| 4 | Underline (MDA only; on CGA/EGA/VGA color text mode this is a no-op) |
| 5 | Blink |
| 7 | Reverse video |
| 8 | Concealed (foreground = background) |
| 30–37 | Foreground color |
| 40–47 | Background color |

ANSI.SYS does not implement attribute codes 2 (faint), 3 (italic), 6
(rapid blink), or any of the reset variants 22, 24, 25, 27, 28. It
also does not implement 38 / 48 (256-color or true-color extensions),
which were added to xterm decades later and are absent from every
DOS-era driver. To clear bold on ANSI.SYS the artist must emit
`ESC [ 0 m` and then re-set the desired foreground and background.

The 40–47 background range only addresses the dim half of the
palette. The high bit of the EGA/VGA text attribute byte is the
blink-enable bit in default text mode; bright backgrounds require
BIOS INT 10h AX=1003h to remap that bit, which ANSI.SYS does not do.

## Screen modes

`ESC [ = n h` selects a screen mode via INT 10h function 00h:

| n | Mode (BIOS) |
| --- | --- |
| 0 | 40 x 25 monochrome text (BIOS 00h) |
| 1 | 40 x 25 color text (BIOS 01h) |
| 2 | 80 x 25 monochrome text (BIOS 02h) |
| 3 | 80 x 25 color text (BIOS 03h) |
| 4 | 320 x 200 4-color CGA (BIOS 04h) |
| 5 | 320 x 200 monochrome CGA (BIOS 05h) |
| 6 | 640 x 200 monochrome CGA (BIOS 06h) |
| 7 | Line wrap toggle (parameter to `h` / `l`) |
| 13 | 320 x 200 16-color EGA (BIOS 0Dh) |
| 14 | 640 x 200 16-color EGA (BIOS 0Eh) |
| 15 | 640 x 350 monochrome EGA (BIOS 0Fh) |
| 16 | 640 x 350 16-color EGA (BIOS 10h) |
| 17 | 640 x 480 monochrome VGA (BIOS 11h) |
| 18 | 640 x 480 16-color VGA (BIOS 12h) |
| 19 | 320 x 200 256-color VGA (BIOS 13h) |

Mode 3 is the default text mode and is what every ANSI art file
expects. ANSI.SYS does not render escape sequences meaningfully in
the graphics modes (4-6, 13-19): cursor positioning works, but `m`
SGR does not map to graphics-mode pixels.

## Keyboard reassignment

The proprietary sequence

```
ESC [ <src> ; <dst1> ; <dst2> ; ... ; p
```

reassigns the key whose original code is `<src>` to produce the
sequence `<dst1> <dst2> ...` when pressed. Sources and destinations
are either decimal byte values or quoted ASCII strings. Extended
(scan-coded) keys are addressed as `0; <scan>`.

```
ESC [ "DIR";13 ; 65;65;65 p
```

reassigns capital A to type "DIR" followed by Enter. Practical use:

```
ESC [ 0;59 ; "type readme.txt";13 p
```

remaps F1 (extended scan code 59) to type "type readme.txt" and
Enter.

The same mechanism is the basis of "ANSI bombs": a `.ANS` file
posted to a BBS could embed an `ESC [ ... p` sequence that, when
displayed, reassigned a common key (Enter, the space bar, the letter
D) to type a destructive command. The classic payload remapped to
`"DEL *.* /Y" 13`. BBS sysops who loaded ANSI.SYS with the `/K`
option to disable reassignment were immune; most did not, and
ANSI-bomb scanning was a job for third-party utilities like ANSICHK,
SCANANSI, and the better terminal programs (Telix, Qmodem,
TheDraw's viewer) that filtered the `p` sequence on input.

By MS-DOS 7 (Windows 95) and reliably in Windows NT, the keyboard
reassignment was disabled by default in the NT console subsystem,
ending the ANSI bomb era.

## Line wrap

Cursor movement past column 80 wraps to the next row by default in
mode 3. The behavior at the exact boundary differs between DOS
versions: PC-DOS 2.x ANSI.SYS wraps eagerly (writing column 80
advances the cursor to column 1 of the next row immediately), while
MS-DOS 5 and later defer the wrap until the next character is
written (the cursor sits at column 81, off-screen, until the next
write). BBS ANSI art that draws to the last column of the last row
exhibits the difference as either a spurious scroll or a "stuck"
last cell.

`ESC [ = 7 h` enables wrap; `ESC [ = 7 l` disables it (subsequent
writes past column 80 overwrite the rightmost cell). Disabled wrap
is rare in BBS art.

## Cursor position

Cursor positions are 1-based. `ESC [ 1 ; 1 H` is the top-left
corner. Positions outside the active screen mode are clamped to the
nearest edge. `ESC [ s` / `ESC [ u` save and restore one stack
level only; nesting save / restore is not supported (the second
`ESC [ s` overwrites the first).

## Differences across DOS versions

- PC-DOS 2.0 / MS-DOS 2.0 (1983): initial release. No options. Some
  sequence parsing bugs around malformed parameter lists.
- DOS 3.3: stabilized, no major changes.
- DOS 4.0: added the keyboard reassignment with extended-key
  awareness; `/X` and `/K` options introduced.
- DOS 5.0: significantly larger driver, supports loading high,
  cleaner mode handling on VGA.
- DOS 6 / 6.22: added `/R` for screen readers.
- PC-DOS 7: added `/L` and additional palette controls.

The size grew from roughly 1.6 KB in DOS 2.0 to over 9 KB in DOS
6.22.

## Alternative drivers

Several third-party replacements were written, mostly to address
ANSI.SYS's slowness:

- NANSI.SYS by Daniel Kegel, 1986. A small, fast ANSI driver that
  writes directly to video memory rather than calling INT 10h. Roughly
  three to five times faster than ANSI.SYS at scrolling. Implements
  the same sequence subset.
- FANSI-CONSOLE by Hersey Micro Consulting. A commercial driver with
  scrollback buffer, screen save / restore, and additional sequences.
- FCONSOLE / ZANSI / ANSIPLUS. Various shareware drivers in the late
  1980s and early 1990s.
- DOS/V's built-in ANSI handling in Japanese DOS, which integrated
  JIS character support with the standard sequence subset.

ANSI.COM (a TSR variant) and the COMMAND.COM-internal ANSI of some
DR-DOS releases also existed but were less common on BBS-era
machines than ANSI.SYS proper.

## Detection

A BBS or program can detect ANSI.SYS presence by emitting
`ESC [ 6 n` (Device Status Report) and reading from `STDIN` with a
short timeout. If the response `ESC [ r;c R` arrives within roughly
one second, ANSI.SYS (or a compatible driver) is loaded. The same
test is used by BBS clients to detect terminal-side ANSI support on
the remote caller.

The DOS multiplex interrupt INT 2Fh function 1A00h reports ANSI.SYS
presence by returning AL=FFh in installed drivers from MS-DOS 4 and
later. This is the documented programmatic detection path.

## References

- Microsoft Corporation, "Microsoft MS-DOS Programmer's Reference",
  version 5.0 (1991) and version 6.0 (1993).
- IBM Corporation, "IBM Personal Computer Disk Operating System
  Technical Reference", versions 3.3 and 4.0.
- Brown, R., "Ralf Brown's Interrupt List", releases 50 through 61,
  entries for INT 2Fh / AH=1Ah and the CON device.
- Duncan, R., "Advanced MS-DOS Programming", 2nd edition, Microsoft
  Press, 1988, chapter on installable device drivers.
- Kegel, D., "NANSI.SYS source release", USENET, 1986.
- ECMA-48, "Control Functions for Coded Character Sets", 5th
  edition, 1991, for the underlying standard ANSI.SYS partially
  implements.
