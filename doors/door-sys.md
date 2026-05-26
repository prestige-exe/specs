# DOOR.SYS – Universal BBS Door Drop File

Origin: GAP Communications, c. 1989, for use with GAP BBS. Adopted
by every major BBS package (RemoteAccess, Wildcat!, Renegade,
Spitfire, Synchronet, ...) as the de-facto universal drop file
format.

When a caller "drops to a door" – launches a third-party door game
or utility – the BBS writes a small text file describing the current
session, then runs the door, which reads the file to learn who is
calling, what speed they're at, how much time they have left, and so
on. `DOOR.SYS` is the most widely supported of these drop files; a
door that reads `DOOR.SYS` will run under almost any DOS BBS.

## Format

Plain ASCII (CP437), one field per line, line terminator CR LF.
Lines are read in fixed positional order – the field is identified
by line number, not by a label.

Trailing whitespace on each line is conventionally stripped. Empty
fields are present as a blank line, never absent.

## Line definitions

| Line | Field | Example | Description |
| --- | --- | --- | --- |
| 1 | Comm port | `COM1` | The serial port the caller is on. `COM0` indicates local login (no modem). |
| 2 | Baud rate | `38400` | DTE rate (computer ↔ modem), in bps |
| 3 | Parity bits | `8` | Always `8` in practice |
| 4 | Node number | `1` | Multi-line BBS node |
| 5 | Locked DTE rate | `38400` | Same as line 2 when DTE is locked |
| 6 | Screen display | `Y` | `Y` if caller wants graphics, `N` if ASCII-only |
| 7 | Printer toggle | `N` | `Y` if log printing is on |
| 8 | Page bell | `N` | `Y` if sysop page bell is enabled |
| 9 | Caller alarm | `Y` | `Y` if any kind of alarm is on |
| 10 | User's full name | `JOHN SMITH` | Real name, uppercase by convention |
| 11 | Calling from | `ANYTOWN, CA` | City and state |
| 12 | Home phone | `714-555-1234` | Caller's voice phone |
| 13 | Work phone | `714-555-5678` | Caller's data phone (often blank) |
| 14 | Password | `PASSWORD` | Caller's password (security risk – many BBSes write `*****`) |
| 15 | Security level | `30` | Numeric access level |
| 16 | Total times on | `42` | Caller's lifetime login count |
| 17 | Last date called | `12/31/95` | MM/DD/YY |
| 18 | Seconds remaining | `3600` | Seconds left this session |
| 19 | Minutes remaining | `60` | Same as line 18 in minutes |
| 20 | Graphics mode | `GR` | `GR` = ANSI, `NG` = none, `7E` = 7-bit only, `RIP` = RIP graphics |
| 21 | Screen length | `25` | Lines per screen |
| 22 | User mode | `N` | `Y` if "expert" mode |
| 23 | Conferences/forums registered | `1,2,3,4,5,6,7,8,9` | Comma-separated list |
| 24 | Conference exited from | `0` | Conference number on door entry |
| 25 | Expiration date | `12/31/99` | MM/DD/YY of account expiry |
| 26 | User record number | `42` | Record # in USERS.BBS |
| 27 | Default protocol | `Z` | One letter: Z=ZModem, Y=YModem, X=XModem, K=Kermit, … |
| 28 | Total uploads | `5` | Lifetime file uploads |
| 29 | Total downloads | `87` | Lifetime file downloads |
| 30 | Daily download "K" | `0` | KB downloaded today |
| 31 | Daily download max | `2048` | KB allowed per day |
| 32 | Caller's birthdate | `01/01/70` | MM/DD/YY |
| 33 | Path to user files | `C:\BBS\GEN\` | Trailing backslash |
| 34 | Path to door files | `C:\BBS\DOORS\` | Trailing backslash |
| 35 | Sysop's full name | `JANE DOE` | First-last, uppercase |
| 36 | Alias | `THEUSER` | Caller's handle / alias |
| 37 | Event time | `00:00` | Next event time, HH:MM (24-hour) |
| 38 | Error-correcting | `Y` | `Y` if connection is error-corrected (MNP/V.42) |
| 39 | BBS in ASCII or ANSI | `Y` | `Y` if BBS forces ANSI for this caller |
| 40 | Use record locking | `N` | `Y` if SHARE.EXE locking is requested |
| 41 | BBS default colour | `7` | Default IBM text-mode attribute |
| 42 | Time credits in minutes | `0` | Bonus minutes earned |
| 43 | Last new files scan | `12/31/95` | MM/DD/YY |
| 44 | Time of last call | `23:59` | HH:MM |
| 45 | Max files per day | `0` | 0 = unlimited |
| 46 | Files downloaded today | `0` | – |
| 47 | Total "K" uploaded | `0` | Lifetime |
| 48 | Total "K" downloaded | `0` | Lifetime |
| 49 | User comment | `Has been informed` | Free-form |
| 50 | Total doors opened | `0` | Lifetime |
| 51 | Total messages left | `0` | Lifetime |

Some BBSes extend `DOOR.SYS` beyond line 51 with package-specific
fields. Doors that read past line 51 risk incompatibility; line 51
is the conventional safe stopping point.

## Example DOOR.SYS

```
COM1
38400
8
1
38400
Y
N
N
Y
JOHN SMITH
ANYTOWN, CA
714-555-1234
714-555-5678
PASSWORD
30
42
12/31/95
3600
60
GR
25
N
1,2,3,4,5
0
12/31/99
42
Z
5
87
0
2048
01/01/70
C:\BBS\GEN\
C:\BBS\DOORS\
JANE DOE
THEUSER
00:00
Y
Y
N
7
0
12/31/95
23:59
0
0
0
0
Has been informed
0
0
```

## Notes for door authors

- **Always check `Comm port = COM0`** – this means the user is logged
  in locally at the sysop console. There is no modem; do not write
  AT commands or expect modem responses.
- **The baud rate is the DTE rate, not the line rate** – a 14400 bps
  modem connected to a PC at a locked 38400 reports 38400 here.
- **Path strings include trailing backslashes** – do not add another
  when concatenating.
- **Dates are 2-digit year** – which means a door that does arithmetic
  on them needed a Y2K rule. Most doors picked "00–79 = 20xx,
  80–99 = 19xx". This is one of the legitimate Y2K bugs in the BBS
  world; doors that did not pick a rule misordered messages around
  the turn of the millennium.
- **The "GR" line 20 value** is the only reliable indicator of
  graphics capability. Line 6 ("Y/N graphics") is an older field
  retained for backward compatibility but does not distinguish ANSI
  from RIP from monochrome.

## DOOR.SYS vs other drop files

| Drop file | Origin | Format | Length | Notes |
| --- | --- | --- | --- | --- |
| `DOOR.SYS` | GAP | Line-based ASCII | 51 lines | Universal |
| `DORINFO1.DEF` / `DORINFOx.DEF` | RemoteAccess / QuickBBS | Line-based ASCII | 17 lines | See [[dorinfo]] |
| `PCBOARD.SYS` | PCBoard | Fixed binary record | 128 bytes | See [[pcboard-sys]] |
| `CHAIN.TXT` | WWIV | Line-based ASCII | 16 lines | WWIV-specific |
| `EXITINFO.BBS` | RemoteAccess | Binary record | ~200 bytes | RA-specific |
| `INFO.BBS` | Wildcat! | Line-based ASCII | 36 lines | Wildcat-specific |
| `CALLINFO.BBS` | WildCat! 2.x | Line-based ASCII | 22 lines | Older |
| `SFDOORS.DAT` | Spitfire | Line-based ASCII | 25 lines | Spitfire-specific |
| `USERS.SYS` | RemoteAccess | Binary | variable | Full user record |

A door that wants maximum compatibility writes its own drop-file
reader for at least `DOOR.SYS` and `DORINFO1.DEF`; commercial doors
typically support a dozen or more formats.
