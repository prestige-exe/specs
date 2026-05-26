# CHAIN.TXT – WWIV Door Drop File

Origin: WWIV BBS, Wayne Bell, 1987 onward. Documented in the WWIV
SDK and reference manuals from WWIV 4.0 through the modern
WWIV-5/WWIV-6 codebase.

`CHAIN.TXT` is WWIV's drop file for "chains" – the WWIV term for
external programs (doors) launched from the BBS. WWIV does not write
`DOOR.SYS` or `DORINFO1.DEF` natively; a WWIV door reads `CHAIN.TXT`
or relies on a wrapper utility to convert.

## Format

Plain ASCII (CP437), one field per line, CR LF terminated, fields
identified by position. Lines 1–16 are the original 1987 set; later
WWIV versions extend through line 24+ with newer fields.

## Line definitions

| Line | Field | Example | Description |
| --- | --- | --- | --- |
| 1 | User number | `42` | – |
| 2 | Real name | `John Smith` | – |
| 3 | Handle | `TheUser` | Display name / alias |
| 4 | Calling from | `Anytown, CA` | – |
| 5 | Age | `25` | – |
| 6 | Gender | `M` | `M` or `F` |
| 7 | Gold | `100` | Credits / score |
| 8 | Last on date | `12/31/95` | – |
| 9 | Times on system | `42` | Lifetime call count |
| 10 | Screen width | `80` | Columns |
| 11 | Screen length | `25` | Rows |
| 12 | Expert mode | `1` | 0 = no, 1 = yes |
| 13 | Sysop level | `255` | Security level (0–255) |
| 14 | ANSI flag | `1` | 0 = none, 1 = ANSI |
| 15 | DOS flag | `0` | 0 = remote, 1 = local |
| 16 | Time left | `60.0` | Minutes remaining (decimal) |
| 17 | Baud rate | `38400` | DTE rate |
| 18 | Comm port | `1` | 1, 2, 3, 4; 0 = local |
| 19 | Sysop's name | `Wayne Bell` | – |
| 20 | Modem's locked DTE | `38400` | – |
| 21 | Time of login | `08:30:00` | HH:MM:SS |
| 22 | "Net" type | `0` | Networking type |
| 23 | User's net node | `0` | WWIVnet node, if any |
| 24 | User's net number | `0` | WWIVnet number |

## Notes

- **Comm port 0** indicates local sysop login (no modem).
- **Time left is decimal**, not HH:MM – 60.5 is 60 minutes 30
  seconds.
- **Sysop level 255** is the standard WWIV sysop level; the rest of
  the range (0–254) is user-defined.
- WWIV uses 1-based port numbers (`1` = COM1) whereas some other
  drop files use `COM1` as a string.

## CALLINFO.BBS and DOORINFO

WWIV can be configured to also write `CALLINFO.BBS` (a Wildcat-style
drop file) for compatibility with non-WWIV doors. The
`//CHAIN.TXT` or `//DOORS` sysop command also generates `DOOR.SYS`
on modern WWIV versions, allowing universal doors to run on a WWIV
BBS without modification.

## WWIVnet awareness

Lines 22–24 describe the user's identity on WWIVnet, the FidoNet-
style messaging network specific to WWIV BBSes. Doors that
implement WWIVnet messaging (the few that did) use these to address
messages back to the user via the network.

## Why a separate format?

Wayne Bell wrote WWIV as a complete BBS from scratch, with its own
message base, file base, and door interface. By the time `DOOR.SYS`
became the de facto standard around 1990, WWIV had thousands of
sysops and several hundred doors built specifically against
`CHAIN.TXT`. The WWIV community kept the format rather than
disrupting the existing door ecosystem; instead, the BBS gained the
ability to *also* write `DOOR.SYS` for any door that wanted it.

## Where to find documentation

Authoritative reference: WWIV source distribution, file
`bbs/conio.cpp` / `bbs/external.cpp` (chain.txt writer), and the
WWIV Sysop Manual chapter on chains.
