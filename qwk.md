# QWK – Offline Mail Packet Format

by Mark "Sparky" Herring / Sparkware
First documented 1987, refined through the mid-1990s.

QWK is the offline-mail packet format used by callers who wanted to
read and reply to BBS conferences without staying connected. The user
logs in, the BBS hands them a `<bbsname>.QWK` file, the user logs off,
opens the file in a "QWK reader" (Blue Wave, OLX, Silver Xpress,
1stReader, MultiMail), reads and writes messages, then dials back in
to upload a `<bbsname>.REP` reply packet.

QWK was the dominant offline reader format on PCBoard, Wildcat, and
Renegade boards; FidoNet systems usually preferred BlueWave or OMEN,
which are conceptually identical but use different file layouts.

## Packet contents

A QWK packet is a ZIP/ARC/LZH archive (the BBS chooses) containing a
fixed set of files:

| File | Description |
| --- | --- |
| `MESSAGES.DAT` | The actual messages, in 128-byte fixed-record blocks |
| `CONTROL.DAT` | Conference list, user name, BBS info |
| `<bbsname>.NDX` | Optional per-conference message index for quick navigation |
| `001.NDX`, `002.NDX`, ... | Per-conference indexes, one file per conference |
| `WELCOME.ANS`, `WELCOME.ASC` | BBS welcome screens (optional) |
| `NEWS.ANS`, `NEWS.ASC` | Latest news (optional) |
| `GOODBYE.ANS`, `GOODBYE.ASC` | Goodbye screen (optional) |
| `DOOR.ID` | Identifies the QWK door package that produced the packet |
| `<bbsname>.BLT` | Bulletins |
| `SESSION.TXT` | Login session transcript (optional) |
| `NEWFILES.DAT` | List of new files (optional) |

`<bbsname>` is the 1-to-8-character BBS identifier from CONTROL.DAT.

## CONTROL.DAT

A line-oriented ASCII (CP437) text file. The first 11 lines are
fixed positional metadata; remaining lines are conference triplets.

```
Line  1: <BBS name>
Line  2: <BBS city / state>
Line  3: <BBS phone>
Line  4: <Sysop's name>
Line  5: <BBS ID>,<reserved>           (e.g. "001,PCBoard")
Line  6: <date and time of packet>     (MM-DD-YYYY,HH:MM:SS)
Line  7: <user's name>
Line  8: <reserved, usually empty>
Line  9: 0
Line 10: <total messages in packet>
Line 11: <total conferences − 1>       (zero-based)
Lines 12 onward repeat in pairs:
        <conference number>
        <conference name>
Line N+1: <welcome screen base filename>
Line N+2: <news screen base filename>
Line N+3: <goodbye screen base filename>
```

Conference number / name pairs continue until line `12 + 2 * (total
conferences)`. Conference 0 is the "main board" / fallback.

## MESSAGES.DAT

A single binary file of 128-byte records. The first record is a
header that simply repeats `"Produced by Qmail...Copyright (c) 1987..."`
padded to 128 bytes (the leading text is what most parsers use as a
sanity check).

Each message occupies an integer number of consecutive 128-byte
records: one record for the message header, followed by N records
holding the message body, where N is given in the header.

### Message header (128 bytes)

| Offset | Size | Field | Description |
| --- | --- | --- | --- |
| 0 | 1 | Status | Status flag, ASCII (see below) |
| 1 | 7 | MsgNum | Message number, ASCII decimal, space-padded |
| 8 | 8 | Date | "MM-DD-YY" |
| 16 | 5 | Time | "HH:MM" |
| 21 | 25 | To | Recipient name |
| 46 | 25 | From | Sender name |
| 71 | 25 | Subject | Subject |
| 96 | 12 | Password | Password (rarely used) |
| 108 | 8 | RefNum | Reference message number, ASCII |
| 116 | 6 | NumChunks | ASCII decimal: blocks of this message including the header |
| 122 | 1 | Status2 | Status flag 2 (active/deleted) |
| 123 | 2 | Conf | Conference number, little-endian u16 |
| 125 | 2 | MsgNum2 | Logical message number, little-endian u16 |
| 127 | 1 | Tagline | Tagline indicator (' ' or '*') |

### Status byte

ASCII character at offset 0:

| Char | Meaning |
| --- | --- |
| ` ` (space) | Public, unread |
| `-` | Public, read |
| `*` | Private, unread |
| `+` | Private, read |
| `~` | Comment to sysop, unread |
| ``` ` ``` | Comment to sysop, read |
| `%` | Sender-password-protected, unread |
| `^` | Sender-password-protected, read |
| `!` | Group-password-protected, unread |
| `#` | Group-password-protected, read |
| `$` | Password-protected and read |

### Message body

Records 2 through `NumChunks` of the message contain the body in
CP437. The body is padded with `0x20` (space) to fill the last record.

Hard line breaks within the body are encoded as `0xE3` (CP437 'π').
This is a quirk of the original Qmail door – Sparky used 0xE3 as a
delimiter to avoid conflicting with CR (0x0D), which BBSes use for
"end of paragraph" handling. QWK readers convert 0xE3 to CR on
display and back to 0xE3 on save.

## NDX index files

Per-conference index files are named `<conf-num>.NDX` with the
conference number zero-padded to three digits (`001.NDX`,
`002.NDX`, …, `999.NDX`). Each NDX is an array of 5-byte records:

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 4 | RecordNum | MS Binary floating-point record number into MESSAGES.DAT |
| 4 | 1 | ConfNum | Conference number byte |

The record number is encoded in Microsoft Binary Format (MBF) – a
4-byte floating-point format used by Microsoft BASIC. Modern parsers
need an MBF→IEEE 754 conversion. The integer value is the 1-based
record number of the message header within MESSAGES.DAT (record 1 is
the file's leading "Produced by..." block, so real messages start at
record 2).

This MBF encoding is the most fiddly part of QWK and the source of
many subtle reader bugs.

## DOOR.ID

A short ASCII file naming the QWK door that produced the packet.
Format: `Door = <doorname>` followed by version and revision lines.
Example:

```
Door = Qmail
Version = 4.0
System = PCBoard
Sysop = Joe Sysop
```

Readers use DOOR.ID to apply door-specific extensions (e.g. Qmail
4 KKK extensions, Mark-mail networking).

## REP – the reply packet

When the user replies, the reader produces `<bbsname>.REP` – also a
ZIP archive, containing:

| File | Description |
| --- | --- |
| `<bbsname>.MSG` | New messages from the user, same 128-byte format as MESSAGES.DAT |
| `HEADERS.DAT` | Optional KKK-extension headers (long subjects, long names) |

The user uploads the REP back to the BBS; the BBS unpacks it and
posts the messages.

## KKK (Kludge-Kludge-Kludge) extensions

To carry information beyond the 25-character To/From/Subject fields,
late-1990s readers like Blue Wave added extension lines in the
message body of the form:

```
^AKLUDGE: value
```

prefixed with `^A` (0x01) exactly as in FidoNet messages. Common
KKKs:

| Kludge | Meaning |
| --- | --- |
| `^AKMSGID:` | Long unique message identifier |
| `^AKREPLY:` | Long reply target |
| `^AKINTL:` | Inter-network address |
| `^AKLONGNAME:` | Real-name override for >25-char names |
| `^AKLONGSUBJ:` | Long subject override |

These were never standardised and reader compatibility varies.

## QWKE

QWKE ("QWK Extended") is a 1995 proposal by Patrick Y. Lee that
formalises the kludges into a separate `HEADERS.DAT` file with
proper UTF-8 support and unlimited field lengths. Synchronet,
Mystic, and other modern BBSes generate QWKE alongside classic QWK
when the door advertises support; readers fall back to plain QWK
otherwise.

## Why QWK matters

For users on metered long-distance calls or slow modems, QWK was the
practical way to participate in conferences with hundreds of new
messages a day. A 30-second QWK download replaced an hour of online
reading. Offline reading also encouraged thoughtful replies –
typing a thread of follow-ups offline cost no connect time.

The format outlived BBSes themselves: Synchronet still ships a QWK
door, and several Web-to-QWK gateways exist for users who want to
read modern forums in a 1990s offline reader. The MBF record number
in NDX is the only piece that has aged badly.
