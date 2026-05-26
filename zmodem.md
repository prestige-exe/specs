# ZMODEM – Streaming File Transfer Protocol with Crash Recovery

by Chuck Forsberg / Omen Technology, Inc.
"The ZMODEM Inter Application File Transfer Protocol", revision of
10-14-88, distributed as ZMODEM.DOC with the Professional-YAM
reference implementation.

ZMODEM was developed under a contract from Telenet (now Sprint) in 1986
to overcome the half-duplex turnaround and reliability limitations of
XMODEM and YMODEM over packet-switched networks. It is the high-water
mark of pre-Internet asynchronous file transfer protocols, and the
default protocol of essentially every shareware terminal program from
1988 onward.

## Key design features

- **Full-duplex streaming**: data flows continuously from sender to
  receiver; acknowledgements are not required for every packet.
- **Variable-length subpackets**: between 32 and 1024 bytes.
- **CRC-32 option**: 32-bit CRC for data integrity in addition to the
  CRC-16/XMODEM fallback.
- **Crash recovery**: a partially received file can be resumed from the
  point at which the previous attempt failed.
- **Auto-download**: a "ZRQINIT" header sent by the sender triggers
  auto-receive in a properly configured terminal.
- **8-bit transparent**: control characters in the data stream are
  escaped, so ZMODEM works over links that strip the 8th bit (with
  reduced throughput) and over links that swallow XON/XOFF.

## Vocabulary

| Name | Value | Use |
| --- | --- | --- |
| ZPAD | 0x2A | '*'; one or two ZPAD bytes lead every header |
| ZDLE | 0x18 | Escape character (also CAN) |
| ZDLEE | 0x58 | Escaped ZDLE in data: ZDLE 0x58 → ZDLE |
| ZBIN | 0x41 | 'A'; binary header with CRC-16 follows |
| ZHEX | 0x42 | 'B'; hex-encoded header follows |
| ZBIN32 | 0x43 | 'C'; binary header with CRC-32 follows |
| XON | 0x11 | Used to release any held output after a header |
| CR/LF | 0x0D/0x0A | Trailers on hex headers, for readability |

## Frame structure

Every ZMODEM frame begins with a *header*. A header is one of three
encodings:

- **Hex** (`ZHEX`) – ASCII printable; used for low-bandwidth control
  messages and to survive 7-bit links.
- **Binary CRC-16** (`ZBIN`) – compact, 16-bit CRC; used for data.
- **Binary CRC-32** (`ZBIN32`) – compact, 32-bit CRC; default for data
  when both sides advertise CRC-32 in the init exchange.

After a header that introduces file data (`ZFILE`, `ZDATA`), one or
more *subpackets* follow, each terminated by its own CRC. A subpacket
trailer byte selects the next action.

### Header layout (binary)

```
ZPAD ZDLE ZBIN  <type> <flags 0..3>  <CRC-16>
ZPAD ZDLE ZBIN32 <type> <flags 0..3> <CRC-32>
```

| Field | Bytes | Description |
| --- | --- | --- |
| ZPAD | 1 | 0x2A |
| ZDLE | 1 | 0x18 |
| Format | 1 | ZBIN (0x41) or ZBIN32 (0x43) |
| Type | 1 | Frame type (see table below) |
| F3..F0 | 4 | Four flag bytes; meaning depends on type |
| CRC | 2 or 4 | CRC-16/XMODEM or CRC-32, computed over type + flags |

Binary headers, including the type, flag, and CRC bytes, are subject to
ZDLE escaping: bytes 0x18, 0x10, 0x11, 0x13, 0x90, 0x91, 0x93, and any
other byte the receiver requested escaped during ZRINIT are replaced
with `ZDLE <byte XOR 0x40>` on the wire.

### Header layout (hex)

```
ZPAD ZPAD ZDLE ZHEX  <hh hh hh hh hh hh hh hh hh hh>  CR LF [XON]
```

Ten ASCII hex digit pairs encode the type, four flag bytes, and a
CRC-16. The terminating XON (only sent if XON/XOFF flow control is in
use) releases any held output on the peer.

## Frame types

| Type | Value | Meaning |
| --- | --- | --- |
| ZRQINIT | 0 | Sender asks receiver to send ZRINIT |
| ZRINIT | 1 | Receiver advertises capabilities |
| ZSINIT | 2 | Sender sends control byte attention string + flags |
| ZACK | 3 | ACK to a header that required one |
| ZFILE | 4 | File name / metadata follow as a subpacket |
| ZSKIP | 5 | Skip this file, send the next |
| ZNAK | 6 | Last header was garbled; retransmit |
| ZABORT | 7 | Abort batch transfer |
| ZFIN | 8 | Finish session |
| ZRPOS | 9 | Resume data transmission at this byte offset |
| ZDATA | 10 | Data subpackets follow at the given file offset |
| ZEOF | 11 | End-of-file; file length follows in flags |
| ZFERR | 12 | Fatal read or write error |
| ZCRC | 13 | Request file CRC and response |
| ZCHALLENGE | 14 | Receiver challenge for sender to echo |
| ZCOMPL | 15 | Request is complete |
| ZCAN | 16 | Pseudo-frame produced by 5 consecutive CANs |
| ZFREECNT | 17 | Request for free bytes on receiver's file system |
| ZCOMMAND | 18 | Command from sender to receiver shell |
| ZSTDERR | 19 | Output to receiver's stderr |

## Subpacket framing

After a `ZDATA` or `ZFILE` header, subpackets stream until the next
header. A subpacket consists of:

```
<0..1024 data bytes> ZDLE <frame-end> <CRC-16 or CRC-32>
```

The frame-end byte chosen by the sender controls what happens after the
subpacket:

| Byte | Mnemonic | Meaning |
| --- | --- | --- |
| ZCRCE (0x68 'h') | End | Last subpacket of the frame; no more data, next thing is a header |
| ZCRCG (0x69 'i') | Go | More subpackets follow; no ACK needed |
| ZCRCQ (0x6A 'j') | Query | More subpackets follow; receiver sends ZACK |
| ZCRCW (0x6B 'k') | Wait | Last subpacket; sender waits for receiver ZACK |

The streaming form is `ZCRCG` for every subpacket except the last; this
yields full-duplex throughput with no per-packet turnaround.

Data bytes within a subpacket are ZDLE-escaped just like header bytes.
The CRC is computed over the unescaped data bytes plus the unescaped
frame-end byte.

## Initialisation handshake

```
Sender                                  Receiver
ZRQINIT  ----->
                                <-----  ZRINIT (flags: CANFC32, ESCCTL, ...)
ZSINIT (optional, attn string, escape map) ----->
                                <-----  ZACK
ZFILE  + subpacket("name\0length modtime mode serial files left bytes left\0")
       ----->
                                <-----  ZRPOS  offset = 0       (or > 0 for resume)
ZDATA (offset)
  <data subpkt ZCRCG>...
  <data subpkt ZCRCW>
       ----->
                                <-----  ZACK
ZEOF (length) ----->
                                <-----  ZRINIT          (ready for next file)
ZFILE  + subpacket (next file)
...
ZFIN ----->
                                <-----  ZFIN
OO ----->                                                (over and out)
```

The two ASCII 'O' characters at the end ("over and out") flush the
receiver's read buffer and conventionally end a session.

### ZRINIT flag bits (F0)

| Bit | Mnemonic | Meaning |
| --- | --- | --- |
| 0x01 | CANFDX | Receiver can send and receive simultaneously (full duplex) |
| 0x02 | CANOVIO | Receiver can overlap disk I/O with serial I/O |
| 0x04 | CANBRK | Receiver can send a BREAK signal |
| 0x08 | CANCRY | Receiver can decrypt |
| 0x10 | CANLZW | Receiver can uncompress (LZW) |
| 0x20 | CANFC32 | Receiver can use CRC-32 |
| 0x40 | ESCCTL | Escape all control characters in data |
| 0x80 | ESC8 | Escape characters with bit 7 set |

### ZFILE conversion / management flags

ZFILE flags select file naming and overwrite policy:

| Flag byte | Value | Meaning |
| --- | --- | --- |
| F0: ZCBIN | 1 | Binary transfer, no translation |
| F0: ZCNL | 2 | Convert LF/CRLF to local newline |
| F0: ZCRESUM | 3 | Resume interrupted file – sender starts at receiver's existing file size |
| F1: ZMNEWL | 1 | Transfer if newer or different length |
| F1: ZMCRC | 2 | Transfer if CRC differs |
| F1: ZMAPND | 3 | Append to existing file |
| F1: ZMCLOB | 4 | Replace existing file unconditionally |
| F1: ZMNEW | 5 | Transfer if newer |
| F1: ZMDIFF | 6 | Transfer if dates or lengths differ |
| F1: ZMPROT | 7 | Protect: skip if file already exists |
| F1: ZMCHNG | 8 | Change filename if file exists |

## Crash recovery

When the receiver answers a `ZFILE` with `ZRPOS offset = N`, the sender
seeks to byte N in the source file and resumes there. The receiver
chooses N as the current size of the partial file it has on disk. This
is the canonical "Z-modem resume" feature.

For safety against silent corruption mid-resume, the receiver may
instead send `ZCRC offset = N` requesting the sender's CRC-32 of the
first N bytes; if the CRCs disagree, the resume is abandoned and the
file is restarted at offset 0.

## ZDLE escaping rules

Every byte transmitted *after* the header lead-in is checked:

| Byte | Action |
| --- | --- |
| 0x18 (ZDLE/CAN) | Always escaped |
| 0x10 (DLE), 0x11 (XON), 0x13 (XOFF), 0x90, 0x91, 0x93 | Always escaped |
| 0x0D (CR) followed by '@' | Escape CR (to defeat Telenet escape sequence) |
| Any byte the peer requested via ESCCTL / ESC8 | Escaped |

Escape encoding: `ZDLE <byte XOR 0x40>`. The receiver reverses by XORing
0x40 unless the byte is one of the special two-byte sequences
(ZCRCE/G/Q/W, ZRUB0/1, etc.) listed in the reference.

## CRC

- **CRC-16/XMODEM**: polynomial 0x1021, initial 0x0000, big-endian on
  the wire, identical to XMODEM-CRC.
- **CRC-32**: polynomial 0xEDB88320 (reflected 0x04C11DB7), initial
  0xFFFFFFFF, reflected input/output, XOR-out 0xFFFFFFFF –
  *bit-identical* to the PKZIP / Ethernet CRC-32. Little-endian on the
  wire.

CRC selection is per-frame:
- A `ZBIN` header carries a CRC-16.
- A `ZBIN32` header carries a CRC-32.
- Subpackets after a `ZBIN`/`ZBIN32` header inherit that CRC width.

## Auto-download

A `ZRQINIT` header sent in printable hex form (ZHEX) doubles as an
"attention string": a terminal program watching the data stream sees
`**` `ZDLE` `B` `00` (the first four bytes of a hex ZRQINIT) and
automatically launches its ZMODEM receiver. This is why most BBS
terminals "just know" when a download starts.

## Cancellation

Five consecutive CAN characters (0x18) form the pseudo-frame ZCAN and
abort the session. To survive minor noise, eight CANs followed by eight
backspaces is the conventional belt-and-braces form.

## Why ZMODEM won

- Streams full-duplex – no turnaround per block, so error-free links
  approach 100% efficiency.
- Resume is a first-class operation, not a bolt-on, so flaky links no
  longer mean re-transferring multi-megabyte archives from scratch.
- Auto-download makes the protocol invisible to the user.
- The receiver advertises capabilities, so a 1986 sender talks to a
  1995 receiver and both sides negotiate the best mode they share.
- The reference implementation (Forsberg's `rzsz`) was widely ported
  and free for non-commercial use.

## Reference implementation

`rzsz` (later `lrzsz` for Linux) is the canonical implementation and
remains the de-facto reference for any new ZMODEM implementer. The
source contains a great many empirical timeouts and retry heuristics
that the spec does not require but every interoperating implementation
ends up matching.
