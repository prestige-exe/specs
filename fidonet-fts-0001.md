# FTS-0001 – A Basic FidoNet Technical Standard

by Randy Bush, FidoNet Technical Standards Committee
Original: FSC-0001, June 1986. Promoted to FTS-0001 after the
FidoNet Technical Standards Committee was formed. The version most
implementers worked from is FTS-0001.016 (December 1995).

FTS-0001 is the protocol that two FidoNet mailers use to exchange
mail and files. It defines the handshake, packet structure, and
file-transfer envelope on top of which all other FTN services
(echomail, file-bone, netmail routing) were built.

## Scope

FTS-0001 covers:

- The "type 2" mail packet structure (the .PKT file).
- The message format inside a packet.
- The basic session protocol over an 8-bit clean asynchronous link
  (typically a modem call).
- The file-transfer envelope used to move packets and other files
  between systems.

It does **not** cover echomail (defined in FTS-0004 and FTS-0005),
routing decisions, or address translation; those are layered above
FTS-0001.

## Addresses

A FidoNet address is `zone:net/node.point@domain`. Within FTS-0001
only `zone:net/node.point` is used; the domain is part of later
extensions. Typical addresses:

```
1:138/152         (zone 1, net 138, node 152, no point)
2:250/107.3       (zone 2, net 250, node 107, point 3)
```

A "point" is a personal system that polls a node for its mail. Points
were how individual hobbyists participated in FidoNet without running
a full BBS.

## Packet (.PKT) – type 2

A `.PKT` file is the unit of exchange. It contains a 58-byte header
followed by zero or more messages, terminated by two null bytes.

### Packet header (58 bytes)

| Offset | Size | Field | Description |
| --- | --- | --- | --- |
| 0 | 2 | OrigNode | Originating node, little-endian |
| 2 | 2 | DestNode | Destination node, little-endian |
| 4 | 2 | Year | Year of packet creation (full 4-digit, e.g. 1995) |
| 6 | 2 | Month | 0..11 |
| 8 | 2 | Day | 1..31 |
| 10 | 2 | Hour | 0..23 |
| 12 | 2 | Minute | 0..59 |
| 14 | 2 | Second | 0..59 |
| 16 | 2 | Baud | Maximum baud rate of originator (0 if unknown) |
| 18 | 2 | PktType | Packet type, must be 2 |
| 20 | 2 | OrigNet | Originating net |
| 22 | 2 | DestNet | Destination net |
| 24 | 1 | ProdCodeLow | Low byte of software product code |
| 25 | 1 | RevisionMajor | Major revision number |
| 26 | 8 | Password | Optional packet password (CP437, space-padded, NUL-terminated if shorter) |
| 34 | 2 | OrigZone | Originating zone (added in 2.2; 0 in plain 2.0) |
| 36 | 2 | DestZone | Destination zone |
| 38 | 2 | AuxNet | Auxiliary net (for points) |
| 40 | 2 | CapWord | Capability word (byte-swapped copy in lo/hi forms) |
| 42 | 1 | ProdCodeHigh | High byte of product code |
| 43 | 1 | RevisionMinor | Minor revision number |
| 44 | 2 | CapValid | Capability validation (copy of CapWord; must match) |
| 46 | 2 | OrigZone2 | Originating zone, duplicate field |
| 48 | 2 | DestZone2 | Destination zone, duplicate field |
| 50 | 2 | OrigPoint | Originating point (0 if not a point) |
| 52 | 2 | DestPoint | Destination point |
| 54 | 4 | ProdData | Product-specific data |

Multi-byte fields are little-endian. The bizarre duplicate
zone/capability fields are an artifact of the 2.0 → 2.2 evolution:
older 2.0 systems treated those offsets as a "fill" region, and 2.2
implementations co-opted them while leaving them at zero for backward
compatibility.

### Message records

Each message immediately follows the packet header and consists of a
fixed-length header plus four NUL-terminated strings:

```
Offset  Size   Field
0       2      Type, must be 2 (little-endian)
2       2      OrigNode
4       2      DestNode
6       2      OrigNet
8       2      DestNet
10      2      Attribute
12      2      Cost
14      20     DateTime, ASCII " DD Mmm YY  HH:MM:SS" + NUL
34      36     ToUserName, NUL-terminated
70      36     FromUserName, NUL-terminated
106     72     Subject, NUL-terminated
178     ...    MessageBody, NUL-terminated
```

After the body, the next message follows (starting with another
Type=2 word). The packet is terminated by two NUL bytes – effectively
a "next message" whose Type field is 0.

### Message body conventions

The body is plain CP437 text (see [[cp437]]). Two control conventions
are universal:

- **0x8D (`<SOH>` written as 0xCD in some sources – actually 0x01)
  control lines**: "kludge lines" of the form `<SOH>FIELDNAME: value`,
  used for routing, echomail control, and EMSI extensions. These
  lines begin with `^A` (0x01) and end with `^M` (0x0D).
- **`--- ` (three hyphens and a space) "tear line"**: marks the end of
  the body and the start of the originating editor's tagline. The
  next line is conventionally an `* Origin:` line identifying the
  origin BBS and address.

### Common kludge lines (FTS-0001 and its echomail successors)

| Kludge | Defined in | Meaning |
| --- | --- | --- |
| `^AMSGID: <orig> <id>` | FTS-0009 | Unique message identifier |
| `^AREPLY: <orig> <id>` | FTS-0009 | This message is a reply to MSGID `<orig> <id>` |
| `^APID: <product>` | FSC-0046 | Product identifier of writer |
| `^ATID: <tosser>` | FSC-0046 | Identifier of the echomail tosser |
| `^APATH: <addr> <addr> ...` | FTS-0004 | Echomail path |
| `^ASEEN-BY: <node> <node> ...` | FTS-0004 | Echomail seen-by list |
| `^AINTL <dest-zone:net/node> <orig-zone:net/node>` | FSC-0006 | Inter-zone routing |
| `^AFMPT <point>` | – | Originating point number |
| `^ATOPT <point>` | – | Destination point number |
| `^ACHRS: <charset> <level>` | FSC-0054 | Character set declaration |
| `^AFLAGS <flags>` | FTS-0001 | Netmail flags |

## Attribute word

The 16-bit attribute word in the message header carries flags:

| Bit | Value | Flag | Meaning |
| --- | --- | --- | --- |
| 0 | 0x0001 | Private | Private mail (netmail), not echomail |
| 1 | 0x0002 | Crash | Send immediately, regardless of schedule |
| 2 | 0x0004 | Recd | Received |
| 3 | 0x0008 | Sent | Sent |
| 4 | 0x0010 | FileAttached | File(s) listed in subject are attached |
| 5 | 0x0020 | InTransit | Currently in transit |
| 6 | 0x0040 | Orphan | Could not be delivered |
| 7 | 0x0080 | KillSent | Delete after sending |
| 8 | 0x0100 | Local | Created locally |
| 9 | 0x0200 | Hold | Hold for pickup |
| 10 | 0x0400 | – | Reserved |
| 11 | 0x0800 | FileRequest | Subject lists files being requested |
| 12 | 0x1000 | ReturnReceiptRequest | Receipt requested |
| 13 | 0x2000 | IsReturnReceipt | This is a return receipt |
| 14 | 0x4000 | AuditRequest | Audit trail requested |
| 15 | 0x8000 | FileUpdateReq | File update request |

## Session protocol

### Connection establishment

After a modem connection at 1200 bps or faster:

1. The answering side waits for the calling side to send a packet
   beginning with the FTS-0001 banner. (The banner is `**EMSI_REQ`
   for EMSI sessions or a series of SYN / NAK bytes for the legacy
   handshake.)
2. The two sides exchange addresses and passwords.
3. The link drops into one of two modes:
   - **Type 2 / "Stone Age" / "1-Type" protocol**: send each file with
     a built-in framing, terminating with EOT.
   - **One of the negotiated higher-level protocols**: Janus,
     ZedZap (ZModem variant), HYDRA. These are out of scope of
     FTS-0001 itself but advertised during the EMSI handshake.

The legacy protocol described in FTS-0001 §6 is the one of last
resort; almost all real FidoNet traffic in the 1990s used Janus, EMSI
+ ZedZap, or HYDRA.

### File transfer envelope (legacy)

The sender transmits, for each file:

```
<TSYNC> <ACK> <SYN> <ACK> <NAK> <ACK> <YOOHOO>
... ZModem-style framed file ...
<EOT>
```

with the exact byte values defined in §6 of FTS-0001. The receiver
ACKs each frame; an unrecoverable error is signalled with three CANs.

### File naming convention

Mail packets exchanged between two systems are named after the
destination:

```
<MMDDHHmm>.PKT   typical outbound .PKT
<NNNNNNNN>.MO    Monday outbound bundle (Tu = TU, etc.)
<NNNNNNNN>.HUT   "hold until" – tossed locally
```

where `<NNNNNNNN>` is an 8-character hex form of the destination
zone/net/node. The day-of-week extensions (`.MO`, `.TU`, `.WE`,
`.TH`, `.FR`, `.SA`, `.SU`) became the "Binkley-Style Outbound" (BSO)
folder layout that almost every modern FidoNet mailer uses.

## Why type 2 stuck

FTS-0001 type 2 packets remain the FidoNet packet format three
decades after their introduction. Type 2+ (Mike Ratledge, FSC-0048),
type 3 (FSC-0039), and type 4 (rare) all proposed cleaner headers
and were technically superior, but type 2 was good enough and every
mailer / tosser already spoke it. The 4D address support that type 2+
adds is the only piece that became universal.

## What FTS-0001 leaves to other standards

- **Echomail topology and propagation** – FTS-0004 (the architecture)
  and FTS-0005 (the file format atop FTS-0001 packets).
- **Mailer session negotiation** – FSC-0056 / EMSI ([[emsi]]).
- **Local message base storage** – JAM ([[jam]]), Squish
  ([[squish]]), MSG ("Fido Star" .MSG files in a numbered directory).
- **Nodelist** – FTS-5001 (the weekly nodelist file format).
- **File request and update protocols** – FTS-0006 / FSC-0086.
