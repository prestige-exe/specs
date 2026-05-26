# DORINFOx.DEF – RemoteAccess / QuickBBS Door Drop File

Origin: QuickBBS by Adam Hudson (1987), inherited by RemoteAccess
(Andrew Milner, 1989). Documented in the RemoteAccess Sysop Manual
appendix on door development.

`DORINFO1.DEF` is the second most common drop-file format after
`DOOR.SYS`. The `1` is the node number; multi-line BBSes write
`DORINFO1.DEF` for node 1, `DORINFO2.DEF` for node 2, etc.

The format predates `DOOR.SYS` by a year or two. It is more compact
(17 lines vs 51) and exposes less information, but it is the format
QuickBBS / RemoteAccess natively writes, so any door that wants to
run on those systems must read it.

## Format

Plain ASCII (CP437), one field per line, CR LF terminated. Lines are
identified by their position.

## Line definitions

| Line | Field | Example | Description |
| --- | --- | --- | --- |
| 1 | BBS name | `MY BBS` | – |
| 2 | Sysop first name | `JANE` | – |
| 3 | Sysop last name | `DOE` | – |
| 4 | Comm port | `COM1` | `COM0` for local login |
| 5 | Baud-parity-stopbits | `38400 BAUD,N,8,1` | DTE rate plus framing |
| 6 | Network type | `0` | Network number, 0 if no networking |
| 7 | User first name | `JOHN` | – |
| 8 | User last name | `SMITH` | – |
| 9 | Calling from | `ANYTOWN, CA` | – |
| 10 | ANSI flag | `1` | 0 = ASCII, 1 = ANSI, 2 = AVATAR |
| 11 | Security level | `30` | Numeric access level |
| 12 | Time remaining | `60` | Minutes left |
| 13 | Fossil driver | `0` | 0 = direct port I/O, 1 = use fossil driver |

Some RemoteAccess versions extend the file with up to four more
lines that carry the user's record number, language number, etc.;
these are not portable and a conservative door should not rely on
them.

The crucial fields for any door are:

- **Line 4 (comm port)**: `COM0` means local session, no modem.
- **Line 5 (baud rate)**: the BBS prefix the number with `BAUD,N,8,1`
  or similar – parse the leading digits.
- **Line 10 (ANSI)**: how to format output.
- **Line 12 (time remaining)**: in *minutes*, not seconds.

## Example DORINFO1.DEF

```
MY BBS
JANE
DOE
COM1
38400 BAUD,N,8,1
0
JOHN
SMITH
ANYTOWN, CA
1
30
60
0
```

## QuickBBS-original quirks

The original 1987 QuickBBS wrote 12 lines, not 13; the fossil driver
flag was added by RemoteAccess. Doors that target the absolute
oldest BBSes should treat line 13 as optional.

The user's name in QuickBBS / RA is split into first and last on two
lines (lines 7 and 8); concatenating them with a single space
produces the full name. `DOOR.SYS` puts the full name on a single
line, which is the more durable choice.

## Network type (line 6)

The "network number" field is interpreted by Maximus, the multi-line
extension to QuickBBS, and by RemoteAccess in its multi-line
configurations. 0 means a single-line BBS or local; non-zero values
identify which network this node belongs to. Most doors ignore it.

## DORINFO and node number

The file name encodes the node:

```
DORINFO1.DEF       Node 1
DORINFO2.DEF       Node 2
DORINFO16.DEF      Node 16
```

A door looking for "the current DORINFO" should consult its
command-line arguments or its `BBS` environment variable to learn
which node it is on; some BBSes also write a generic `DORINFO.DEF`
(without a number) for single-node configurations.

## When to use DORINFO vs DOOR.SYS

- A door that runs on RemoteAccess, QuickBBS, or one of their clones
  (FrontDoor's BBS, EleBBS, ProBoard) can rely on `DORINFOx.DEF`.
- A door that wants to run anywhere uses `DOOR.SYS` (see
  [[door-sys]]) and falls back to `DORINFOx.DEF` if `DOOR.SYS` is
  absent.
- For full user-record access (security flags, last-on date,
  preferences) `DORINFO` is insufficient; consult `EXITINFO.BBS` on
  RemoteAccess, `USERS.SYS` on RA, or `USERS.BBS` on QuickBBS.

## Concurrent writes

When a multi-line BBS launches several doors at once, each node
writes its own `DORINFOx.DEF`. A door must read the file named for
its node, not the generic one. The convention is that the BBS passes
the node number as a command-line argument (`/N1`, `-n 1`, `1` as
first arg, etc.); robust doors accept any of these forms and
construct the filename accordingly.
