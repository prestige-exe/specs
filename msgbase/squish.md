# Squish Message Base Format

by Scott J. Dudley, Lanius Corporation
Documented in `SQUISH.DOC` distributed with Squish 1.0+ (1992) and the
Maximus BBS package; the spec was also published as part of the SQUISH
SDK in 1994.

Squish is the message base behind the Maximus BBS and was the dominant
tosser / message-base format on FidoNet during the 1992–1995 window
before JAM took over. Many JAM design decisions are direct improvements
on Squish, but Squish remains supported by every modern FTN tosser.

## Files

A Squish base is **two files**:

| Extension | Purpose |
| --- | --- |
| `.SQD` | Squish Data – file header, frame index, messages |
| `.SQI` | Squish Index – per-message offset pointers, frame indexes |

(An optional `.SQL` lastread file exists, formatted identically to
JAM's `.JLR`.)

## File header (SQD, 256 bytes)

| Offset | Size | Field | Description |
| --- | --- | --- | --- |
| 0 | 2 | Length | Header length (256) |
| 2 | 2 | RsvdWord1 | Reserved – 0 |
| 4 | 4 | NumMsg | Number of active messages |
| 8 | 4 | HighMsg | Highest message number used |
| 12 | 4 | SkipMsg | Number of "skipped" (deleted) frames |
| 16 | 4 | HighWater | High water mark (highest read by anyone) |
| 20 | 4 | Uid | Next available message UID |
| 24 | 80 | BaseName | Full path of base, NUL-terminated |
| 104 | 4 | BeginFrame | First frame offset (offset into SQD) |
| 108 | 4 | LastFrame | Last frame offset |
| 112 | 4 | FreeFrame | First free frame offset |
| 116 | 4 | LastFreeFrame | Last free frame offset |
| 120 | 4 | EndFrame | First byte after last frame (file size) |
| 124 | 4 | MaxMsg | Maximum messages (0 = unlimited) |
| 128 | 2 | KeepDays | Days to keep messages (purge older) |
| 130 | 2 | SzSqHdr | Size of an SQHDR frame header (64) |
| 132 | 124 | Reserved | Reserved – zero |

The file is "linked list of frames" – BeginFrame points to the first
frame, FreeFrame points to the head of the free-list. Insertion and
deletion are O(1) at the head/tail of either list.

## Frame layout

Every frame begins with a 32-byte SQHDR:

| Offset | Size | Field | Description |
| --- | --- | --- | --- |
| 0 | 4 | SqId | "0AFA0AFA" (0xAFAE4AFA) – frame magic |
| 4 | 4 | NextFrame | Offset of next frame (0 if last) |
| 8 | 4 | PrevFrame | Offset of previous frame |
| 12 | 4 | FrameLen | Length of variable data following this header |
| 16 | 4 | MsgLen | Length of message text portion |
| 20 | 4 | ControlLen | Length of control info |
| 24 | 2 | FrameType | 0 = normal, 1 = free, 2 = LRA |
| 26 | 6 | Reserved | Zero |

Following the SQHDR for a normal message:

```
+--------------------+
| XMSG header (238)  |
+--------------------+
| Control info       |  (ControlLen bytes)
+--------------------+
| Message text       |  (MsgLen − ControlLen − 238 bytes)
+--------------------+
```

The "control info" is the message header's NUL-terminated metadata
the same way FTS-0001 message records carry: it contains kludge
lines (`^AMSGID:`, `^APATH:`, `^ASEEN-BY:`, `^AFLAGS`) that travelled
with the message in the inbound packet.

## XMSG record (238 bytes)

| Offset | Size | Field | Description |
| --- | --- | --- | --- |
| 0 | 4 | Attr | Attribute flags (same bitmask as JAM/FTS-0001) |
| 4 | 36 | From | Sender's name |
| 40 | 36 | To | Recipient's name |
| 76 | 72 | Subject | Subject |
| 148 | 8 | OrigZNNP | Origin: zone, net, node, point (4 × u16) |
| 156 | 8 | DestZNNP | Destination address |
| 164 | 4 | DateWritten | Unix time |
| 168 | 4 | DateArrived | Unix time |
| 172 | 2 | Utc | UTC offset in minutes |
| 174 | 4 | Replyto | Message number this is a reply to |
| 178 | 36 | Replies | Array of 9 u32 reply message numbers |
| 214 | 4 | UMsgId | Unique message ID (matches Uid in file header) |
| 218 | 20 | __ftsc_date | Original FTSC date string |

Multi-byte fields are little-endian. The 9-deep `Replies` array
embeds the reply tree directly in the message header.

## Index file (SQI)

The SQI is an array of 4-byte little-endian offsets, one per message
slot. Slot N holds the offset of message UID `N + 1` within the SQD.
Slot 0 is therefore the offset of UID 1, etc. A slot whose value is
0xFFFFFFFF is a deleted message.

## Free-list management

Deleted frames are not physically removed. Instead:

1. The SQHDR `FrameType` is changed from 0 to 1 (free).
2. The frame is linked into the file header's FreeFrame list.
3. The corresponding SQI slot is set to 0xFFFFFFFF.

When a new message arrives Squish tries to reuse a free frame of
sufficient size; failing that, it appends a new frame and updates
EndFrame. Periodic packing walks the free-list and collapses adjacent
free frames; an explicit "squish" command can also rewrite the file
to remove the holes entirely.

## Attribute flags

The Attr field uses the FidoNet message attribute bitmask plus a few
Squish-specific extensions:

| Bit | Meaning |
| --- | --- |
| 0x00000001 | Private |
| 0x00000002 | Crash |
| 0x00000004 | Received |
| 0x00000008 | Sent |
| 0x00000010 | File attached |
| 0x00000020 | In transit |
| 0x00000040 | Orphan |
| 0x00000080 | Kill after send |
| 0x00000100 | Local origin |
| 0x00000200 | Hold |
| 0x00000800 | File request |
| 0x00001000 | Return receipt request |
| 0x00002000 | Return receipt |
| 0x00004000 | Audit request |
| 0x00008000 | File update request |
| 0x00010000 | Scanned |
| 0x00020000 | Echomail |
| 0x00040000 | Locked |
| 0x00080000 | Squish-specific extension |

## Comparison with JAM

| Aspect | Squish | JAM |
| --- | --- | --- |
| File count | 2 (SQD, SQI) | 4 (JHR, JDT, JDX, JLR) |
| Header size | Fixed 238 bytes per msg | Fixed 76 bytes + sub-fields |
| Variable fields | Embedded in `ControlInfo` | First-class sub-fields |
| Reply tree | 9-deep array in header | Doubly-linked via Reply1st/ReplyNext |
| Extensibility | Limited (control info parsing) | Sub-field IDs are reservable |
| Lookup by recipient | Linear scan of XMSG.To | CRC-32 hash in JDX |
| Body length limit | 64K (16-bit field) | 4G (32-bit field) |

Squish's main weakness vs JAM is the 16-bit body length cap and the
embedded fixed-width From/To fields, which truncate long names. Its
strengths are the simpler two-file storage and the proven Maximus
codebase.

## Tooling

The canonical implementations:

- **Squish 1.10** – Scott Dudley, MS-DOS, the original.
- **Maximus 3.01** – the BBS that ships Squish natively.
- **HPT / Husky Project** – cross-platform open-source tosser, reads
  and writes Squish bases on Linux / *BSD / Win32.
- **MsgEdit** family of message editors – every FTS-aware editor
  speaks SQD/SQI alongside MSG and JAM.
