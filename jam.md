# JAM – Joaquim-Andrew-Mats Message Base Format

by Joaquim Homrighausen, Andrew Milner, Mats Birch, and Mats Wallin
First release: July 1993. Named for the four authors' initials.

JAM was designed as a unified replacement for Squish, Hudson, *.MSG
and FidoNet-MSG message bases. It is the format that virtually every
FidoNet-aware editor and tosser written after 1995 supports natively
and is the canonical message base for modern Synchronet, Mystic,
Argus, Husky, jamNNTPd, and most other working FTN systems.

## Files

A JAM message base is **four files** sharing a common base name:

| Extension | Purpose |
| --- | --- |
| `.JHR` | Header file – fixed-size headers and variable-length sub-fields |
| `.JDT` | Data file – message body text |
| `.JDX` | Index file – per-message offset + CRC of "To" name |
| `.JLR` | Lastread file – one record per user tracking their pointer |

Variable-size data (sub-fields, body text) lives in JHR and JDT.
Fixed-size pointers and indexes live in JDX and JLR for fast random
access.

## File header (JHR, 1024 bytes)

The JHR file begins with a 1024-byte file header:

| Offset | Size | Field | Description |
| --- | --- | --- | --- |
| 0 | 4 | Signature | "JAM\0" (4A 41 4D 00) |
| 4 | 4 | DateCreated | Time the file was created, Unix time |
| 8 | 4 | ModCounter | Updated every time the base is changed |
| 12 | 4 | ActiveMsgs | Active messages (not deleted) |
| 16 | 4 | PasswordCRC | CRC-32 of the optional area password |
| 20 | 4 | BaseMsgNum | Lowest message number in JDX |
| 24 | 1000 | Reserved | Zero-filled |

Multi-byte integers are little-endian throughout JAM.

## Message header (JHR, variable)

Each message has a fixed 76-byte header in JHR followed by 0..N
sub-fields:

| Offset | Size | Field | Description |
| --- | --- | --- | --- |
| 0 | 4 | Signature | "JAM\0" |
| 4 | 2 | Revision | 1 |
| 6 | 2 | ReservedWord | 0 |
| 8 | 4 | SubfieldLen | Total bytes of sub-fields following |
| 12 | 4 | TimesRead | Number of times this message has been read |
| 16 | 4 | MSGIDcrc | CRC-32 of MSGID kludge (for fast dupe lookup) |
| 20 | 4 | REPLYcrc | CRC-32 of REPLY kludge |
| 24 | 4 | ReplyTo | Message number this is a reply to |
| 28 | 4 | Reply1st | First reply message number |
| 32 | 4 | ReplyNext | Next reply at the same level |
| 36 | 4 | DateWritten | When message was written (Unix time) |
| 40 | 4 | DateReceived | When message was received |
| 44 | 4 | DateProcessed | When tosser handled it |
| 48 | 4 | MessageNumber | This message's number (matches JDX index) |
| 52 | 4 | Attribute | Message attributes (see below) |
| 56 | 4 | Attribute2 | Reserved – 0 |
| 60 | 4 | Offset | Offset into JDT of message body |
| 64 | 4 | TxtLen | Length of message body |
| 68 | 4 | PasswordCRC | CRC-32 of the message password |
| 72 | 4 | Cost | Cost of message (FidoNet metric) |

Sub-fields immediately follow the header, totalling `SubfieldLen`
bytes. Each sub-field has its own 8-byte header:

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 2 | LoID | Low half of field identifier |
| 2 | 2 | HiID | High half (reserved, 0) |
| 4 | 4 | DataLen | Length of the variable data that follows |
| 8 | DataLen | Variable text data |

### Sub-field IDs (LoID)

| ID | Name | Contents |
| --- | --- | --- |
| 0 | OAddress | Origin address (`zone:net/node[.point]`) |
| 1 | DAddress | Destination address |
| 2 | SenderName | Sender's full name |
| 3 | ReceiverName | Recipient's full name |
| 4 | MsgID | MSGID kludge value |
| 5 | ReplyID | REPLY kludge value |
| 6 | Subject | Message subject |
| 7 | PID | PID kludge value |
| 8 | TraceTo | "Trace to" routing |
| 9 | EnclFile | Enclosed-file name |
| 10 | EnclFwAlias | Enclosed file with alias |
| 11 | EnclFreq | File request |
| 12 | EnclFileWc | Enclosed file, wildcard |
| 13 | EnclIndFile | Enclosed indirect file |
| 14 | EmbInDat | Embedded data |
| 15 | FTSKludge | Other FTS-style kludge |
| 16 | SEEN-BY2D | SEEN-BY (echomail) |
| 17 | PATH2D | PATH (echomail) |
| 18 | FLAGS | FLAGS kludge |
| 19 | TZUTCInfo | Time zone info |
| 1000–1999 | (vendor) | Reserved for vendor extensions |
| 2000 | UCFLAGS | User-conference flags |

The header stays a fixed 76 bytes; everything that varies in length
becomes a sub-field. This is what makes JAM trivial to extend.

## Attribute flags

The Attribute field (offset 52 in the message header) is a 32-bit
bitmask:

| Bit | Mnemonic | Meaning |
| --- | --- | --- |
| 0x00000001 | LOCAL | Posted locally |
| 0x00000002 | INTRANSIT | Currently in transit |
| 0x00000004 | PRIVATE | Private mail |
| 0x00000008 | READ | Recipient has read |
| 0x00000010 | SENT | Sent OK |
| 0x00000020 | KILLSENT | Delete after sending |
| 0x00000040 | ARCHIVESENT | Archive after sending |
| 0x00000080 | HOLD | Hold for pickup |
| 0x00000100 | CRASH | Send immediately |
| 0x00000200 | IMMEDIATE | Send via crash |
| 0x00000400 | DIRECT | Send direct |
| 0x00000800 | GATE | Gate to other network |
| 0x00001000 | FILEREQUEST | File request |
| 0x00002000 | FILEATTACH | File attached |
| 0x00004000 | TRUNCFILE | Truncate file after send |
| 0x00008000 | KILLFILE | Kill file after send |
| 0x00010000 | RECEIPTREQ | Return receipt requested |
| 0x00020000 | CONFIRMREQ | Confirmation requested |
| 0x00040000 | ORPHAN | Orphan |
| 0x00080000 | ENCRYPT | Body is encrypted |
| 0x00100000 | COMPRESS | Body is compressed |
| 0x00200000 | ESCAPED | Body has escaped 0x7E |
| 0x00400000 | FPU | Force pickup |
| 0x00800000 | TYPELOCAL | Message valid for local use |
| 0x01000000 | TYPEECHO | Message valid for echomail |
| 0x02000000 | TYPENET | Message valid for netmail |
| 0x10000000 | NODISP | Not displayed in editor |
| 0x20000000 | LOCKED | Locked – do not delete |
| 0x40000000 | DELETED | Deleted (will be removed at pack) |

## Index file (JDX, 8 bytes per record)

The JDX file contains a fixed 8-byte record per message:

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 4 | UserCRC | CRC-32 of the recipient's name, lowercased |
| 4 | 4 | HdrOffset | Byte offset of header in JHR |

The UserCRC enables fast "show me my mail" lookups: hash the user's
name, scan the JDX, jump straight to the headers. Deleted messages
have UserCRC = 0xFFFFFFFF and HdrOffset = 0xFFFFFFFF.

The record at JDX offset 0 corresponds to message `BaseMsgNum` (from
the file header); offset 8 is `BaseMsgNum + 1`; etc.

## Last-read file (JLR, 16 bytes per record)

One record per user:

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 4 | UserCRC | CRC-32 of user name, lowercased |
| 4 | 4 | UserID | Numeric user ID |
| 8 | 4 | LastRead | Last message number read |
| 12 | 4 | HighRead | Highest message read |

The JLR is appended to as new users read the conference; entries are
not removed when a user is deleted, just marked.

## CRC-32

JAM uses the same CRC-32 as PKZIP / Ethernet:

| Parameter | Value |
| --- | --- |
| Polynomial | 0xEDB88320 (reflected 0x04C11DB7) |
| Initial value | 0xFFFFFFFF |
| Reflect input | yes |
| Reflect output | yes |
| XOR out | 0xFFFFFFFF |

For name-based CRCs (UserCRC, MSGID, REPLY), names are first
lowercased (ASCII only – CP437 high characters are not folded).

## Packing

Deleted messages are not physically removed at delete time – the
DELETED bit is set in the attribute word and the JDX entry's
HdrOffset is set to 0xFFFFFFFF. A periodic "pack" operation:

1. Reads every message whose DELETED bit is clear.
2. Writes them to new .JHR / .JDT files in order.
3. Rebuilds the .JDX.
4. Atomically swaps the new files in.

The .JLR is rewritten with translated message numbers if BaseMsgNum
changes.

## Concurrency

JAM is designed for multi-process access (sysop's editor, ringing
mailer, message reader, web gateway). The spec defines `.LCK` files
and OS-level file locking on JHR / JDX. Implementations differ in
strictness; in practice a SHARE-aware DOS implementation or POSIX
fcntl locking on Unix is sufficient.

## Why JAM won

- 4D addresses are first-class (sub-field 0 / 1) – no header-byte
  expansion contortions like FTS-0001 type 2's duplicate zone fields.
- The sub-field model means adding new metadata (NNTP message IDs,
  PGP signatures, BBS-specific flags) does not break old readers.
- Fast lookup by recipient name via JDX UserCRC.
- Trivially gateways to NNTP, since the sub-fields map cleanly to RFC
  822 headers.
- Multi-process safety is part of the spec.

A modern Synchronet or Mystic system uses JAM as its message base
even for completely non-FidoNet conferences for these reasons.
