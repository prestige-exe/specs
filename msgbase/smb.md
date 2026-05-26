# SMB – Synchronet Message Base

by Rob Swindell, Digital Dynamics
First released with Synchronet 2.00 in 1996. Continuously
maintained as part of Synchronet through the current 3.20 series.
Documented in `smbdefs.h` and `smb.txt` in the Synchronet source
distribution at synchro.net.

SMB is a four-file, variable-length-record message base designed
around extensible typed header fields rather than a fixed C struct.
Synchronet uses it as the unified store for local messages,
echomail, netmail, internet email, NNTP newsgroups, and FidoNet
gated traffic. Every header field is typed, length-prefixed, and
optional, which makes SMB the most format-agnostic of the BBS-era
message bases.

## Files

A single SMB base consists of four files sharing a common base
name. Each conference (or "sub-board" in Synchronet parlance) has
its own set.

| Extension | Purpose |
| --- | --- |
| `.shd` | Header index – fixed-size records pointing into `.sdt` |
| `.sdt` | Data – variable-length header fields and message bodies |
| `.sda` | Allocation table for `.sdt` blocks |
| `.sid` | Sparse index by message number |

The split between `.shd` (fixed) and `.sdt` (variable) is the
defining design choice. `.shd` records are slot-addressable so a
reader can seek to message N in constant time. `.sdt` carries the
variable payload addressed by offset from the `.shd` record.

## File header

Both `.shd` and `.sdt` open with a fixed `smb_fileheader_t`
structure. The header carries the on-disk format version, base
configuration, the next-message counter, and the maximum-CRC
hash-table size used for dupe detection. The exact byte layout is
documented in `smbdefs.h`; the canonical fields are:

| Field | Description |
| --- | --- |
| `id` | "SMB\032" magic string |
| `version` | Format version, MSB.LSB |
| `length` | Length of this file header on disk |
| `total_msgs` | Active (non-deleted) message count |
| `last_msg` | Highest message number assigned |
| `header_offset` | Offset to first message header in `.shd` |
| `max_crcs` | Size of the CRC hash table for dupe detection |
| `max_msgs` | Optional cap on message count |
| `max_age` | Optional cap on message retention (days) |
| `attr` | Base-wide attribute flags |

Multi-byte integers are little-endian. The header is padded to a
fixed length to allow extension without breaking existing
readers.

## Message header

Each message has a fixed `smbmsg_t` "index" record in `.shd`
plus a variable-length set of typed header fields in `.sdt`. The
fixed record holds the values a reader needs without parsing
fields:

| Field | Description |
| --- | --- |
| `number` | This message's number |
| `offset` | Byte offset into `.sdt` of the variable header |
| `total_dfields` | Count of "data fields" (body pieces) |
| `hdr.type` | Message type (mail / post / file / etc.) |
| `hdr.version` | Header format version |
| `hdr.length` | Total bytes of variable header in `.sdt` |
| `hdr.attr` | 16-bit attribute bitmask |
| `hdr.auxattr` | 32-bit auxiliary attributes |
| `hdr.netattr` | 32-bit network attributes |
| `hdr.when_written` | `when_t` struct: time + UTC offset |
| `hdr.when_imported` | When the local system received it |
| `hdr.thread_back` | Message number of parent in thread |
| `hdr.thread_orig` | Original (root) message in thread |
| `hdr.thread_next` | Next sibling in thread |
| `hdr.thread_first` | First reply |

`when_t` is an 8-byte structure: a 32-bit Unix timestamp followed
by a signed 16-bit UTC offset in minutes and 2 bytes of padding.
This is the precision Synchronet uses internally for all message
timestamps.

The fixed record is followed in `.sdt`, at byte `offset`, by a
contiguous list of typed header fields and then by the body data
fields.

## Header fields (HFIELDs)

The variable-length header in `.sdt` is a stream of
`hfield_t` records. Each begins with a 4-byte header:

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 2 | length |
| 2 | 1 | type |
| 3 | 1 | reserved |

Immediately after the 4-byte header come `length` bytes of
field data. The field is then immediately followed by the next
`hfield_t`. A reader walks the chain until it has consumed
`hdr.length` bytes.

Field types are defined as constants in `smbdefs.h`. The common
ones:

| Type | Constant | Contents |
| --- | --- | --- |
| 0x00 | `SENDER` | Sender name |
| 0x01 | `SENDERAGENT` | Origin agent (mailer / NNTP server) |
| 0x02 | `SENDEREXT` | Sender extension / user number |
| 0x03 | `SENDERORG` | Sender organisation |
| 0x04 | `SENDERNETTYPE` | Network type code (FidoNet / Internet / QWK) |
| 0x05 | `SENDERNETADDR` | Network address in network's own format |
| 0x06 | `SENDERIPADDR` | Sender's IP address (string) |
| 0x07 | `SENDERHOSTNAME` | Sender's reverse DNS |
| 0x10 | `RECIPIENT` | Recipient name |
| 0x14 | `RECIPIENTNETTYPE` | Recipient network type |
| 0x15 | `RECIPIENTNETADDR` | Recipient network address |
| 0x20 | `SUBJECT` | Subject |
| 0x21 | `SMB_SUMMARY` | Summary / short description |
| 0x22 | `SMB_COMMENT` | Free-form comment |
| 0x40 | `RFC822HEADER` | Verbatim RFC 822 header line |
| 0x41 | `RFC822MSGID` | Internet Message-ID |
| 0x42 | `RFC822REPLYID` | In-Reply-To: value |
| 0x50 | `USENETPATH` | NNTP Path header |
| 0x51 | `USENETNEWSGROUPS` | Newsgroups header |
| 0x60 | `FIDOMSGID` | FidoNet MSGID kludge |
| 0x61 | `FIDOREPLYID` | FidoNet REPLY kludge |
| 0x62 | `FIDOPID` | FidoNet PID |
| 0x63 | `FIDOFLAGS` | FidoNet FLAGS kludge |
| 0x64 | `FIDOPATH` | FidoNet PATH line |
| 0x65 | `FIDOSEENBY` | FidoNet SEEN-BY line |
| 0x66 | `FIDOCTRL` | Other FidoNet control line (kludge) |
| 0x67 | `FIDOAREA` | Echomail AREA: tag |
| 0xE0 | `SMB_EDITOR` | Editor used to compose message |
| 0xF0 | `FILEATTACH` | Attached file name |

There are many more (the full list lives in `smbdefs.h`). Two
properties matter:

1. **Order is preserved.** A reader sees fields in the order the
   writer emitted them. Multiple instances of the same type are
   permitted and meaningful (multiple `RECIPIENT`, multiple
   `FIDOSEENBY`).
2. **Unknown types are forwarded.** A reader that does not know
   field type 0x?? must preserve it when rewriting. This is what
   lets Synchronet gateway FidoNet, NNTP, and SMTP through the
   same store without information loss.

## Data fields (DFIELDs)

After the header fields come `total_dfields` data-field
descriptors. Each is a `dfield_t`:

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 1 | type |
| 1 | 3 | reserved |
| 4 | 4 | offset |
| 8 | 4 | length |

`offset` and `length` describe a region of `.sdt` (not relative
to the message header but absolute within the data file) holding
that portion of the body. Types include:

| Type | Contents |
| --- | --- |
| 0x00 | Plain text body |
| 0x01 | "Tail" data (signature, tear line, origin) |
| 0x02 | Attached file pointer |

Splitting body and tail lets Synchronet hide the FidoNet tearline
and origin in echomail when displaying as Usenet, or vice versa,
without touching the original bytes.

## Allocation: .sda

`.sda` is the allocation table for `.sdt`. `.sdt` is partitioned
into fixed-size allocation units (`SDT_BLOCK_LEN`, typically 256
bytes). `.sda` is an array of 16-bit reference counts: one entry
per allocation unit. A non-zero entry means the unit is in use by
some message header or data field; zero means free.

This is the mechanism by which a single body can be referenced by
multiple message headers (cross-posts, twit-cage copies). Each
reference holds a count; the body is reclaimed only when every
reference is released.

When a new message is written:

1. The writer rounds each `.sdt` payload up to a multiple of
   `SDT_BLOCK_LEN`.
2. It scans `.sda` for a contiguous run of free blocks of
   sufficient size.
3. If found, the run is reused; allocation counts go from 0 to 1.
4. If not, the writer appends new blocks to `.sdt` and grows
   `.sda` in lockstep.

A periodic pack collapses free space; routine writes do not block
on packing.

## Index: .sid

`.sid` is a sparse index from message number to `.shd` slot.
Each entry:

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 4 | number |
| 4 | 4 | offset (into `.shd`) |
| 8 | 2 | attr |
| 10 | 2 | reserved |
| 12 | 4 | time |

`.sid` is what a reader scans when answering "give me the next
unread message after N". It is small (16 bytes per message) and
linear, so a sequential scan is fast even on a large base. The
`attr` and `time` are mirrored from the header for use during
the scan without touching `.shd` or `.sdt`.

When a message is deleted the `.sid` entry's `attr` gets the
`MSG_DELETE` bit set and remains in place; the corresponding
`.sda` blocks are decremented. The `.sid` entry itself is
removed only at pack time.

## Hash chains

SMB maintains a separate hash file (`.sch` or in-memory hash
table) used for fast duplicate detection. Each message contributes
one or more 32-bit hash values, typically over `RFC822MSGID`,
`FIDOMSGID`, the body CRC, and the (sender, subject, date)
tuple. Inbound tossers / gateways look up each candidate hash in
the chain before inserting; a hit means a duplicate and the
message is dropped.

The hash file is bounded by `max_crcs` in the file header. When
full, the oldest entries are evicted. This is what lets
Synchronet's NNTP and FidoNet imports run without ever consulting
the full `.shd` for dupes.

## Attribute flags

The 16-bit `hdr.attr` field uses values defined in `smbdefs.h`:

| Bit | Mnemonic | Meaning |
| --- | --- | --- |
| 0x0001 | `MSG_PRIVATE` | Private (netmail) |
| 0x0002 | `MSG_READ` | Recipient has read |
| 0x0004 | `MSG_PERMANENT` | Cannot be packed away |
| 0x0008 | `MSG_LOCKED` | Cannot be deleted |
| 0x0010 | `MSG_DELETE` | Marked deleted |
| 0x0020 | `MSG_ANONYMOUS` | Sender hidden |
| 0x0040 | `MSG_KILLREAD` | Delete after recipient reads |
| 0x0080 | `MSG_MODERATED` | Awaiting moderator approval |
| 0x0100 | `MSG_VALIDATED` | Approved by moderator |
| 0x0200 | `MSG_REPLIED` | Recipient has replied |
| 0x0400 | `MSG_NOREPLY` | No replies permitted |

`hdr.auxattr` and `hdr.netattr` are 32-bit and carry the
extended flags used for poll, expiry, file-attach handling, and
network-specific routing.

## Locking and concurrency

SMB is designed for multi-process operation. Synchronet ships with
file-locking helpers (`smb_locksmbhdr`, `smb_lockmsghdr`) that
take advisory locks on byte ranges of `.shd`. The model is:

- Reading a message: shared lock on its `.shd` slot.
- Writing or modifying a message header: exclusive lock on the
  slot.
- Allocating in `.sda` or appending to `.sdt`: exclusive lock on
  the file header of `.shd`.
- Walking the hash chain: exclusive lock on the hash file.

Lock timeouts are configurable (`smb.retry_time`). On contention,
operations retry rather than fail.

## Why SMB has lasted

- Field-level typing means a Synchronet base losslessly carries
  FidoNet kludges, Internet headers, NNTP path information, and
  QWK metadata simultaneously.
- Reference-counted `.sda` blocks make cross-posts cheap.
- The hash file decouples dupe detection from base size; bases
  with millions of messages remain practical.
- The base format has been forward-compatible since the 2.00
  release: new field types are added without breaking old
  readers.

The cost is implementation complexity. There is no second-party
SMB writer of consequence; in practice "SMB support" means linking
the Synchronet `smblib`. Readers in other languages (Go, Python,
Rust) exist but are read-only.

## References

- `smbdefs.h`, Synchronet source tree, Rob Swindell, Digital
  Dynamics. The canonical structure definitions.
- `smb.txt`, Synchronet Message Base specification document,
  distributed with the Synchronet source.
- `smblib`, the reference C library implementing read, write,
  pack, and lock operations.
- Synchronet developer documentation at `wiki.synchro.net`.
