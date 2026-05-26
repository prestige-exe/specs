# Hudson Message Base Format

by Adam Hudson
First shipped with QuickBBS 1.0 (1987) and subsequently adopted by
RemoteAccess (RA), SuperBBS, EzyCom, and the rest of the QuickBBS-
family software. Documented in the QuickBBS and RemoteAccess source
distributions and in the structures published with the RA SDK.

The Hudson Message Base (HMB) was the dominant local message store
on FidoNet-aware BBS systems on MS-DOS through roughly 1993. It is
a fixed-record, six-file layout optimised for the file-handle and
memory limits of DOS. Its two famous ceilings, a maximum of 200
conferences and a 32 KB per-message text limit, were the principal
drivers that pushed sysops toward Squish and later JAM.

## Files

A Hudson base consists of six files in a single directory, all
sharing fixed prefixes. Conference numbers are folded into the
records rather than into per-conference files.

| File | Purpose |
| --- | --- |
| `MSGHDR.BBS` | Fixed-size message headers, one per message |
| `MSGTXT.BBS` | Message body text, allocated in 256-byte blocks |
| `MSGTOIDX.BBS` | Recipient index, one byte-pair per message |
| `MSGIDX.BBS` | Message-number to conference index |
| `MSGINFO.BBS` | Global state: high/low message number per conference |
| `LASTREAD.BBS` | Per-user lastread pointers for every conference |

All multi-byte fields are little-endian, matching Intel real-mode
conventions. Strings are space-padded, not NUL-terminated, in
Turbo Pascal `string[N]` form where the first byte is the length.

## MSGHDR.BBS

`MSGHDR.BBS` is a flat array of fixed-size records. Each record is
approximately 190 bytes and describes one message. The record
number plus one equals the message's global message number; record
zero corresponds to message 1.

| Offset | Size | Field | Description |
| --- | --- | --- | --- |
| 0 | 4 | MsgNum | Global message number |
| 4 | 1 | ToLen | Length byte for `To` |
| 5 | 35 | To | Recipient name, space-padded |
| 40 | 1 | FromLen | Length byte for `From` |
| 41 | 35 | From | Sender name, space-padded |
| 76 | 1 | SubjLen | Length byte for `Subject` |
| 77 | 71 | Subject | Subject, space-padded |
| 148 | 1 | DateLen | Length byte for `Date` |
| 149 | 19 | Date | FTSC date string (`DD MMM YY  HH:MM:SS`) |
| 168 | 2 | TimesRead | Read counter |
| 170 | 2 | DestNode | Destination FidoNet node |
| 172 | 2 | OrigNode | Origin node |
| 174 | 2 | Cost | Message cost in cents |
| 176 | 2 | OrigNet | Origin net |
| 178 | 2 | DestNet | Destination net |
| 180 | 2 | DestZone | Destination zone |
| 182 | 2 | OrigZone | Origin zone |
| 184 | 2 | DestPoint | Destination point |
| 186 | 2 | OrigPoint | Origin point |
| 188 | 4 | Replyto | Message number this is a reply to |
| 192 | 4 | Attr | Attribute flags |
| 196 | 4 | Up | Next reply in the same thread |
| 200 | 2 | StartRec | First 256-byte block in `MSGTXT.BBS` |
| 202 | 2 | NumRecs | Number of blocks used in `MSGTXT.BBS` |
| 204 | 2 | DestRead | Reserved / scanner state |
| 206 | 1 | Board | Conference number (1..200) |
| 207 | 4 | PostTime | Unix-style posting timestamp |

Field offsets vary slightly between releases (QuickBBS 2.64, RA 1.11,
RA 2.50). The canonical layout in the RemoteAccess `STRUCTRA.PAS`
unit is the de-facto standard. The two key invariants across all
revisions are the 1-byte `Board` discriminator capping conferences
at 200 and the 16-bit `StartRec` / `NumRecs` pair capping body
text at 65535 blocks of 256 bytes (in practice, sysops hit the
~32 KB practical limit well before the theoretical 16 MB).

## MSGTXT.BBS

`MSGTXT.BBS` is a heap of 256-byte blocks. Each message's text
starts at block `StartRec` (zero-based) and occupies `NumRecs`
consecutive blocks. The text itself is plain CP437 with CR (0x0D)
as the line terminator. Kludge lines (`^A` prefixed) sit at the
top of the text block in echomail and netmail messages; tearline,
origin and SEEN-BY / PATH live at the bottom. The text body is
truncated to fit within `NumRecs * 256` bytes, with no
continuation mechanism. This is the source of the often-quoted
"32K Hudson limit". The practical limit varies by reader but
nearly all QuickBBS-family editors enforce 16 KB or 32 KB.

Deleted messages leave their blocks marked unused; the free-space
map is reconstructed at pack time by walking `MSGHDR.BBS`.

## MSGTOIDX.BBS

`MSGTOIDX.BBS` is a parallel array to `MSGHDR.BBS` used for fast
"mail check" scans without reading every header. Each record is
typically 36 or 37 bytes and contains:

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 35 | To name, upper-cased, space-padded |
| 35 | 1 | Conference (`Board`) |
| 36 | 1 | Status byte (read / received / deleted) |

A user-login mail-check loop reads `MSGTOIDX.BBS` sequentially
rather than seeking through full headers. Some QuickBBS variants
elide the status byte and the record is 36 bytes; RA 2.x and EzyCom
use 37.

## MSGIDX.BBS

`MSGIDX.BBS` is the message-number to conference cross-reference,
2 or 3 bytes per record. Record N corresponds to message N+1 and
holds the conference number that message belongs to (or zero for
deleted). It exists so a scan such as "show me all messages in
conference 7 between numbers 1000 and 2000" can be performed
without touching `MSGHDR.BBS` or `MSGTOIDX.BBS`.

Typical layout:

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 4 | MsgNum |
| 4 | 1 | Board |

Older QuickBBS revisions packed this as a 2-byte-per-record array
indexed implicitly by message number; the 5-byte form was
standardised by RemoteAccess.

## MSGINFO.BBS

A single record holding global counters. The structure is small
and varies between QuickBBS 2.x and RA 1.x/2.x. Common fields:

| Offset | Size | Field | Description |
| --- | --- | --- | --- |
| 0 | 4 | LowMsg | Lowest active message number |
| 4 | 4 | HighMsg | Highest message number assigned |
| 8 | 4 | TotalActiveMsgs | Active (non-deleted) count |
| 12 | 400 | MsgsOnBoard | `array[1..200] of word` of message counts per conference |
| 412 | 800 | HiOnBoard | `array[1..200] of longint` of high msg per conference |

The 200-entry arrays are the hard cap on conference count. Some
later RA revisions stretched the structure to 250 by changing the
declared array bounds; this is incompatible with stock QuickBBS
and pre-2.50 RA tools.

## LASTREAD.BBS

A two-dimensional array sized `[1..NumUsers, 1..200] of longint`.
The record for user index U covers 200 longints (one per
conference). A reader seeks to `(U - 1) * 800` and reads
800 bytes. The user index is the record number into `USERS.BBS`,
not the user name, so a renamed or repacked user file must trigger
a matching rewrite of `LASTREAD.BBS`.

This layout is exceptionally efficient for the typical operation
("what is user 47's lastread in conference 12?") at the cost of
wasting 800 bytes per user for sparse conference subscriptions.

## Attribute flags

The 32-bit `Attr` field uses the standard FidoNet message
attribute bitmask, identical in semantics (though not always in
bit position) to FTS-0001 and JAM:

| Bit | Meaning |
| --- | --- |
| 0x00000001 | Private |
| 0x00000002 | Crash |
| 0x00000004 | Received |
| 0x00000008 | Sent |
| 0x00000010 | File attached |
| 0x00000020 | In transit |
| 0x00000040 | Orphan |
| 0x00000080 | Kill / sent |
| 0x00000100 | Local origin |
| 0x00000200 | Hold |
| 0x00000800 | File request |
| 0x00001000 | Return receipt request |
| 0x00002000 | Is return receipt |
| 0x00004000 | Audit request |
| 0x00008000 | File-update request |
| 0x00010000 | Scanned by tosser |
| 0x00020000 | Echomail |
| 0x40000000 | Deleted |

The Deleted flag is set in place; the record is not removed from
`MSGHDR.BBS` until an offline pack runs.

## Packing and concurrency

There is no in-place reuse of `MSGHDR.BBS` slots or `MSGTXT.BBS`
blocks. Deletions accumulate until a sysop runs the pack utility
(`QPACK`, `RAMSG`, `MSGPACK`, etc.), which rewrites every file
with the deleted records removed and the surviving blocks
compacted. Pack must be run with the BBS offline; concurrent
access during a pack will corrupt the base.

For online access Hudson relies on DOS SHARE.EXE and per-record
locking via the DOS file-locking interrupts (`INT 21h` AH=5C).
The Pascal runtime in QuickBBS / RA wraps this in a `LockRec` /
`UnlockRec` helper. Multi-node systems rely on a network-aware
SHARE replacement (LANtastic, Novell, DESQview/NETBIOS) or a
single-tasking arrangement.

## Limits

| Limit | Value | Source |
| --- | --- | --- |
| Conferences | 200 | 1-byte `Board` field, `array[1..200]` in `MSGINFO.BBS` |
| Message number | 2^31 | 32-bit `MsgNum` |
| Active messages | practical ~16k–65k | Performance of linear `MSGTOIDX.BBS` scan |
| Body length | ~32 KB practical, 16 MB theoretical | 16-bit `NumRecs * 256` |
| `To` / `From` name | 35 chars | Pascal `string[35]` |
| Subject | 71 chars | Pascal `string[71]` |

The conference cap is the limit most often hit in practice. By
1993 large FidoNet hubs were carrying 300+ echoes and had no
option but to migrate to Squish or JAM. The 32 KB body limit
became an issue earlier: PGP-signed messages, source-code echoes,
and ARCmail summaries routinely exceeded it.

## Implementations

- **QuickBBS 2.64 / 2.76** by Adam Hudson. The original.
- **RemoteAccess 1.11 / 2.02 / 2.50** by Andrew Milner. The
  most widely deployed Hudson host through the mid-1990s.
- **SuperBBS** by Aki Antman and Risto Virkkala. Reads and writes
  the same base.
- **EzyCom** by Peter Davies. Compatible variant.
- **FastEcho**, **GEcho**, **SquishMail**, **ImailMt** etc. all
  toss directly into a Hudson base on QuickBBS-family systems.

## References

- `STRUCTRA.PAS`, RemoteAccess BBS source distribution, Andrew
  Milner, Wantree Development, 1992–1996.
- `STRUCTS.PAS`, QuickBBS source, Adam Hudson, 1987–1990.
- `RAFAQ.TXT`, RemoteAccess Frequently Asked Questions, sections
  on message-base internals.
- FTSC FTS-0001: A Basic FidoNet Technical Standard, for the
  attribute-flag definitions referenced by HMB.
