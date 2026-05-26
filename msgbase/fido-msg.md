# FidoNet *.MSG Message Format

defined alongside FTS-0001 by Randy Bush and the FidoNet Technical
Standards Committee (FTSC). Introduced with Fido 11 in 1984 and
standardised in FTS-0001 §4. Still in use as the netmail store on
most modern FidoNet mailers and points.

`*.MSG` is the simplest of the FidoNet message stores: one file
per message in a single directory. Each file consists of a
fixed 190-byte header followed by the message text, NUL-terminated.
Files are named with the message number followed by `.MSG`
(`1.MSG`, `2.MSG`, ..., `N.MSG`). It survives because it is
trivially correct, requires no shared state across processes, and
maps cleanly to per-message file locking on every operating system
that has files.

## Files and naming

A `*.MSG` area is a directory. The directory holds:

| Pattern | Purpose |
| --- | --- |
| `N.MSG` | One message, N being the integer message number |
| `1.MSG` | The first message in this area, by convention |

There is no per-directory header file, no index file, and no
allocation table. The set of active message numbers is determined
by directory scan. Holes are permitted: deleting message 17 simply
removes `17.MSG`; the next inbound message is assigned the next
unused integer.

The directory typically corresponds to a single FidoNet conference
or to the netmail area. Echomail in this format is uncommon (the
overhead of one inode per message scales badly with traffic), so
`*.MSG` survives mainly as the netmail store for points and small
nodes, and as the inbound staging area before a tosser converts to
JAM, Squish, or Hudson.

## Header layout (190 bytes)

The header is exactly 190 bytes, fixed-format, little-endian.
Strings are NUL-terminated within fixed-width fields and padded
with binary zeros, not spaces.

| Offset | Size | Field | Description |
| --- | --- | --- | --- |
| 0 | 36 | FromUserName | Sender name, NUL-terminated, CP437 |
| 36 | 36 | ToUserName | Recipient name, NUL-terminated |
| 72 | 72 | Subject | Subject, NUL-terminated |
| 144 | 20 | DateTime | FTSC date string (`DD MMM YY  HH:MM:SS\0`) |
| 164 | 2 | TimesRead | Read counter |
| 166 | 2 | DestNode | Destination node |
| 168 | 2 | OrigNode | Origin node |
| 170 | 2 | Cost | Cost in cost units |
| 172 | 2 | OrigNet | Origin net |
| 174 | 2 | DestNet | Destination net |
| 176 | 2 | DestZone | Destination zone (FTS-0001 type 2) |
| 178 | 2 | OrigZone | Origin zone (type 2) |
| 180 | 2 | DestPoint | Destination point |
| 182 | 2 | OrigPoint | Origin point |
| 184 | 2 | ReplyTo | Message this is a reply to |
| 186 | 2 | Attr | Attribute bitmask |
| 188 | 2 | Next | Reserved / next reply linkage |

The zone and point fields at offsets 176-185 are the FTS-0001
type 2 extension. The original 1984 layout had these eight bytes
as `Dest Reserved` and `Orig Reserved`; type 2 (defined in
FSC-0048 and folded back into FTS-0001) repurposed them so that
true 4D addressing could be carried in the header rather than only
in kludge lines. Implementations must inspect both the header zone
fields and any `^A INTL` kludge line in the body to recover the
full address, since older mailers leave the header zone at zero.

## Attribute flags

`Attr` at offset 186 is a 16-bit bitmask:

| Bit | Mnemonic | Meaning |
| --- | --- | --- |
| 0x0001 | Private | Private mail |
| 0x0002 | Crash | Send at top priority |
| 0x0004 | Received | Recipient has read |
| 0x0008 | Sent | Sent OK |
| 0x0010 | FileAttached | One or more files attached |
| 0x0020 | InTransit | Currently in transit |
| 0x0040 | Orphan | No legal route found |
| 0x0080 | KillSent | Delete after successful send |
| 0x0100 | Local | Posted locally |
| 0x0200 | HoldForPickup | Hold for the recipient to poll |
| 0x0400 | Reserved | Reserved |
| 0x0800 | FileRequest | Subject contains a file request |
| 0x1000 | ReturnReceiptRequest | Confirmation requested |
| 0x2000 | IsReturnReceipt | Body is a return receipt |
| 0x4000 | AuditRequest | Audit trail requested |
| 0x8000 | FileUpdateReq | File-update request |

Flags above 0x8000 (echomail, scanned, locked, deleted) exist in
the 32-bit attribute extensions used by JAM and Squish; the
on-disk `*.MSG` attribute field is only 16 bits and cannot carry
them. Tossers translate as needed when moving messages in or out
of a `*.MSG` area.

## Message body

The body begins at offset 190 and runs to the first NUL byte (or
the end of the file, whichever is earlier). The body is CP437
text. Lines are terminated with CR (0x0D); LF is not used and
should be stripped on read. The body is the FTSC "message text"
and carries three categories of content in well-defined positions:

1. **Kludge lines** at the top.
2. **The visible message**, possibly preceded by a `>` quote block.
3. **Tearline, origin, and SEEN-BY / PATH** at the bottom (echomail
   only).

Kludge lines begin with `^A` (Ctrl-A, 0x01) and are terminated by
CR. They are not displayed by message readers. The most common
kludges, all defined by FTSC documents:

| Kludge | Source | Purpose |
| --- | --- | --- |
| `^AMSGID: <orig-addr> <unique>` | FTS-0009 | Globally unique ID |
| `^AREPLY: <msgid>` | FTS-0009 | Reference to parent message |
| `^AINTL <dest> <orig>` | FSC-0001 / FTS-4001 | True 4D addresses |
| `^AFMPT <point>` | FSC-0001 | Origin point number |
| `^ATOPT <point>` | FSC-0001 | Destination point number |
| `^APID:` | FTS-0046 | Producing program identifier |
| `^ATID:` | FTS-0046 | Tosser identifier |
| `^ACHRS:` | FTS-5003 | Character set declaration |
| `^AFLAGS` | FSC-0053 | Per-message routing flags |
| `^ATZUTC:` | FSC-0093 | UTC offset in minutes |

The tearline is a single line beginning with `--- ` (three dashes
and a space) and identifying the originating editor:

```
--- GoldED+/W32 1.1.5-b20180707
```

The origin line follows and identifies the originating system and
its 4D address:

```
 * Origin: Prestige BBS (1:103/705)
```

The leading space, asterisk, and ` Origin: ` prefix are
mandatory. Anything between `Origin:` and the parenthesised
address is the system name. The origin address is the canonical
address that echomail uses for duplicate detection in combination
with `MSGID`.

## SEEN-BY and PATH (echomail)

Echomail messages carry two trailing kludge-like control blocks
that are written without the `^A` prefix:

```
SEEN-BY: 103/705 705/1 707/0 1
SEEN-BY: 270/101 270/102
^APATH: 103/705 707/0 270/101
```

`SEEN-BY` is a node-list of every system that has already received
the message. A tosser must not forward an echomail message to a
node already on its `SEEN-BY` list. Multiple `SEEN-BY` lines
accumulate; addresses are stored as net/node pairs sorted by net
then node, with the net suppressed when it equals the preceding
entry's net.

`PATH` is the actual route the message took. Only the
2D address is stored, prefixed with `^A` so it counts as a kludge
and is hidden from the reader. Each tosser appends its own address
to the end of `PATH` before forwarding.

The semantics are specified in FTS-0004 (later superseded by
FTS-4000 series). The dupe-detection algorithm is: compute the
echomail uniqueness as the tuple `(origin-address, MSGID)`. If
that tuple has been seen within the configured retention window,
drop the message; otherwise add the local address to `SEEN-BY`
and `PATH`, and forward to every link not in `SEEN-BY`.

## Locking

The natural concurrency model is one file per message. A reader
opens the file read-only; a writer (the tosser, or the user
marking a message read) opens it for exclusive write. POSIX
`fcntl` advisory locks and DOS SHARE work without coordination
across the directory. A mailer scanning the directory while a
tosser is writing must be prepared to skip files that vanish
mid-scan.

There is no atomic counter for "next message number". Tossers
either lock the directory itself (where the OS supports it) or
use a sidecar `.LCK` file. Some implementations write to a
temporary name and `rename()` into place.

## Strengths and weaknesses

`*.MSG` is verbose on disk: a one-line message uses an inode and
a minimum filesystem block (often 4 KB) for under 200 bytes of
text. On the original Fido 11 systems with 360 KB floppies this
was tolerable only because traffic was small. By 1990 the format
was already too expensive for echomail, which is why every modern
high-volume base is JAM, Squish, SMB, or Hudson.

Its lasting strength is its lack of shared state. There is no
file header to corrupt, no allocation table to lose, and no index
to rebuild. A `*.MSG` area survives any sequence of crashes that
leaves individual files readable. For netmail in point software
and for inbound staging this trade-off remains the right one.

## References

- FTSC FTS-0001: A Basic FidoNet Technical Standard, §4 "Message
  format on disk". Randy Bush, 1986.
- FTSC FTS-0009: Message identification and reply linkage. Jim
  Nutt, 1990.
- FTSC FSC-0001 / FTS-4001: Zone and point routing kludges.
- FTSC FTS-0004 / FTS-4000 series: Echomail specification,
  defining `SEEN-BY`, `PATH`, and origin lines.
- FTSC FTS-0046: PID / TID kludges for software identification.
- FTSC FTS-5003: Character set kludge `CHRS`.
