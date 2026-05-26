# Wildcat! Drop Files – CALLINFO.BBS and USERS.SYS

Origin: Mustang Software, Inc. (MSI), Bakersfield, California.
Wildcat! 1.0 shipped in 1986. The product evolved through
Wildcat! 3.x (the last DOS line) and Wildcat! 4.x (DOS, then
Windows). Mustang sold the product to Santronics Software on
November 19, 1998, after which it continued as the Wildcat!
Interactive Net Server (WINS / WINServer) on Windows.

Wildcat! was the dominant commercial BBS package on DOS for much
of the late 80s and early 90s, competing primarily with PCBoard
and TBBS. Its native drop files are `CALLINFO.BBS` (an ASCII
line-based format, used through Wildcat 3.x) and `USERS.SYS` (a
binary record format, used by Wildcat-aware doors that needed
fuller user-record access). Starting with Wildcat 4, the package
also wrote DOOR.SYS for compatibility with the broader DOS-door
ecosystem (see [[door-sys]]).

The two formats are distinct from the RemoteAccess `USERS.SYS`
of the same name; both formats are widely referenced, and a door
must know which BBS wrote the file before interpreting it.

## CALLINFO.BBS

Format: plain ASCII (CP437), one field per line, line terminator
CR LF. Lines are identified by position, not by label. Used by
Wildcat! 1.x, 2.x, and 3.x as the native drop file. Wildcat 3.0
introduced an extended variant; Wildcat 4 deprecated the file in
favour of DOOR.SYS but continued writing it for backward
compatibility.

The standard line layout, as written by Wildcat 3.x:

| Line | Field | Example | Description |
| --- | --- | --- | --- |
| 1 | Caller's full name | `JOHN SMITH` | Real name, uppercase by convention |
| 2 | Calling from | `ANYTOWN, CA` | City and state |
| 3 | Baud rate (text) | `2400 BAUD` | Connection speed as printable text |
| 4 | Communications port | `COM1` | Serial port, `LOCAL` for sysop console |
| 5 | Security level | `30` | Numeric access level |
| 6 | Total times on | `42` | Lifetime login count |
| 7 | Last date called | `01/01/95` | MM/DD/YY |
| 8 | Seconds remaining | `3600` | Time left this session |
| 9 | Minutes remaining | `60` | Time left in minutes |
| 10 | Graphics mode | `COLOR` | `COLOR` if ANSi, `MONO` if ASCII |
| 11 | Screen length | `25` | Lines per screen |
| 12 | Expert mode | `NOVICE` | `EXPERT` or `NOVICE` |
| 13 | Conferences registered | `1 2 3` | Space-separated list |
| 14 | Conference exited from | `0` | Current conference number |
| 15 | Expiration date | `12/31/99` | MM/DD/YY |
| 16 | User record number | `42` | Position in the user file |
| 17 | Default protocol | `Z` | Z = ZModem, Y = YModem, X = XModem |
| 18 | Total uploads | `5` | Lifetime files uploaded |
| 19 | Total downloads | `87` | Lifetime files downloaded |
| 20 | Daily download "K" | `0` | KB downloaded today |
| 21 | Daily download max | `1024` | KB allowed per day |
| 22 | Date of birth | `01/01/70` | MM/DD/YY |

Older Wildcat 1.x and 2.x wrote a shorter form, with the early
lines (name, location, baud, port, level, time) reliably present
and later lines absent or differently ordered. Doors targeting
ancient Wildcats parse defensively: read what is present, do not
require fields past line 12.

The baud rate on line 3 is printable text rather than a bare
number, with `BAUD` as a suffix. Doors must parse the leading
integer; `LOCAL` indicates the sysop console with no modem in the
loop.

Conferences registered on line 13 is a space-separated list of
conference numbers the caller has subscribed to, distinct from the
DOOR.SYS comma-separated convention.

## USERS.SYS

Format: binary, fixed-length record. Used by Wildcat-aware doors
that needed access to the full user record rather than the subset
in CALLINFO.BBS. The file holds a single record describing the
current caller; it is rewritten on each door launch.

The record is laid out as a Pascal-style fixed-length structure.
The conceptual fields, in approximate declaration order, are:

| Group | Fields |
| --- | --- |
| Identity | Caller's full name, alias / handle (where supported), record number |
| Address | City, state, ZIP, country, phone numbers |
| Security | Security level, expiration date, sysop flag, locked flag |
| Statistics | Total calls, total uploads, total downloads, "K" uploaded, "K" downloaded, posts |
| Session | Time remaining (this session), time used today, daily download "K" used, daily download max |
| Preferences | Graphics mode, expert mode, screen length, default protocol, page length |
| Dates | First-called date, last-called date, birthdate, last-new-files date |
| Conferences | Bitmap or list of conferences subscribed to, last conference visited |

Each name and address field is a fixed-length character array
(Pascal-style: length byte followed by character bytes, padded to
the declared length, or in some Wildcat versions a C-style
null-terminated string padded with spaces). Numeric fields are
little-endian integers of various widths. Dates are stored as
binary day counts from a fixed epoch or as packed MM/DD/YY bytes
depending on Wildcat version.

The exact byte layout differs between Wildcat 3.x and Wildcat 4.x;
a door reading `USERS.SYS` needs to know which Wildcat wrote it.
Wildcat 4 added fields for Internet email address, FidoNet node
numbers, and language preference that are absent from the 3.x
layout. The total record length grew from roughly 190 bytes in
late Wildcat 3.x to over 250 bytes in Wildcat 4.x.

The canonical reference for the layout is the Wildcat! Sysop's
Manual appendix on door development for each version, and the
Wildcat! Developer Kit (`WCDK`) headers that MSI published for
authors writing Wildcat-native doors.

## Other Wildcat drop files

Wildcat! 3.x and 4.x also write or accept:

- `INFO.BBS`: an extended Wildcat-specific ASCII drop file with
  more fields than `CALLINFO.BBS`. Used by some Wildcat-aware
  doors as a richer alternative to CALLINFO.BBS without the binary
  complexity of USERS.SYS.
- `DOOR.SYS`: from Wildcat 4 onwards, written for compatibility
  with the wider DOS-door ecosystem. See [[door-sys]].

The sysop selects which drop files to write per-door through the
Wildcat door configuration. Production Wildcat boards typically
wrote DOOR.SYS plus one of `CALLINFO.BBS` or `USERS.SYS`, picking
the latter pair to match what each installed door expected.

## Notes for door authors

- The Wildcat baud rate field on `CALLINFO.BBS` line 3 is the line
  rate, not a locked DTE rate. A door that wants the actual DTE
  rate needs to either lock its own serial port or read DOOR.SYS
  instead.
- `COM` line is `LOCAL` for the sysop console on Wildcat, not
  `COM0` as in DOOR.SYS.
- Wildcat dates are 2-digit year. Doors that arithmetic on them
  need a Y2K rule; the conventional pivot is "00 through 79 = 20xx,
  80 through 99 = 19xx".
- `USERS.SYS` is a single-record file describing the current caller
  only, not the full user database. The full database lives
  separately and is not exposed to doors.
- Wildcat 4 USERS.SYS includes string fields that are
  null-terminated within their fixed-length slot; do not assume
  every byte of the slot is meaningful.

## Wildcat versus the universal drop files

| Format | Origin | Type | Notes |
| --- | --- | --- | --- |
| `CALLINFO.BBS` | Wildcat! 1.x onwards | ASCII line-based | Native, narrow field set |
| `USERS.SYS` (Wildcat) | Wildcat! 3.x onwards | Binary record | Native, wide field set |
| `INFO.BBS` | Wildcat! 3.x | ASCII line-based | Extended ASCII, less widely used |
| `DOOR.SYS` | GAP, Wildcat! 4 onwards | ASCII line-based | Universal, see [[door-sys]] |
| `DORINFO1.DEF` | QuickBBS / RemoteAccess | ASCII line-based | Not native, sometimes configured |

A door that targets Wildcat specifically reads `USERS.SYS` (binary)
and falls back to `CALLINFO.BBS` (ASCII). A door that wants
maximum compatibility across BBS packages reads `DOOR.SYS` and
falls back to `DORINFO1.DEF`, and may not interact with the native
Wildcat formats at all.

## References

- Wildcat! Sysop's Manual (MSI), versions 3.x and 4.x, appendix on
  door development.
- Wildcat! Developer Kit (WCDK) headers, MSI.
- Santronics Software Wildcat! 4 BBS Suite documentation,
  www.santronics.com/products/wildcatdos/suite.php.
- Mustang Software Wildcat! 4.11 and 5 archives on Internet Archive.
- fsxNet drop file reference, www.fsxnet.nz/resources/doors/dropfiles.
- See also [[door-sys]] for the universal GAP-format drop file and
  [[dorinfo]] for the QuickBBS / RemoteAccess drop file.
