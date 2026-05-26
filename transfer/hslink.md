# HSLink – Bidirectional File Transfer Protocol

by Samuel H. Smith / Buffalo Creek Software
Reference text: HSLINK.DOC, distributed with the shareware HSLINK.EXE
reference implementation, first widely circulated in 1991.

HSLink is a full-duplex, bidirectional, streaming file transfer protocol
written for MS-DOS by Samuel H. Smith. It uses both directions of a
full-duplex modem link simultaneously, so a sender and receiver can
swap files in opposite directions in a single session without the
half-session turnaround that ZMODEM imposes. On error-corrected modems
(MNP, V.42) over a clean line it routinely outperformed ZMODEM, in part
because acknowledgement traffic and reverse-direction file payload
share the back-channel at no additional cost. HSLink was distributed
as shareware; a registration fee was required for continued use beyond
an evaluation period, and the source code was not published.

## Design features

- Bidirectional: each side acts as both sender and receiver, with two
  independent file queues sharing one serial link.
- Streaming: data flows continuously without per-block ACK turnaround.
  Acknowledgements piggyback on the reverse data channel.
- Sliding window with selective retransmission: only damaged blocks
  are retransmitted, not everything since the error.
- Variable block size, negotiated at handshake.
- CRC-16 and CRC-32 block integrity, negotiated.
- 8-bit transparent with optional DLE escaping for links that strip
  the high bit or filter control characters.
- Crash recovery comparable to ZMODEM: a partially received file is
  resumed from the receiver's existing byte count.
- Batch transfer with file name, size, and modification time per file.
- Optional inline data compression for files not already compressed.

## Wire layout

HSLink frames everything in DLE-quoted packets. A packet carries either
a control message or a chunk of file data, and is keyed by a one-byte
type code following an SOH-like sentinel.

```
<SYN> <SYN> <type> <length-lo> <length-hi> <data ...> <crc-lo> <crc-hi> [<crc-mid> <crc-high>]
```

| Field   | Bytes      | Description |
| ---     | ---        | --- |
| SYN     | 2          | Frame lead-in. Two synchronisation bytes mark a packet boundary. |
| Type    | 1          | Packet type code. |
| Length  | 2          | Payload length, little-endian. |
| Data    | 0..N       | Payload bytes, DLE-escaped on the wire. |
| CRC     | 2 or 4     | CRC-16/XMODEM or CRC-32 over type + length + data, negotiated. |

DLE escaping replaces the bytes 0x10, 0x11, 0x13, the SYN value, and
any byte the peer requested via the option mask in the init exchange.
Each escaped byte becomes `DLE <byte XOR 0x40>`, mirroring the
ZMODEM ZDLE convention. The CRC is computed over the unescaped bytes
plus the unescaped type and length fields.

## Packet types

| Code | Mnemonic | Direction       | Meaning |
| ---  | ---      | ---             | --- |
| 'A'  | INIT     | both            | Capability advertisement and option mask |
| 'B'  | INITACK  | both            | Reply to INIT, agreed options |
| 'C'  | FILE     | sender          | File header: name, size, mtime, mode |
| 'D'  | DATA     | sender          | File data chunk at implicit offset |
| 'E'  | EOF      | sender          | End of current file, final byte count |
| 'F'  | FILEACK  | receiver        | Accept, skip, or resume offset for FILE |
| 'G'  | DATAACK  | receiver        | Cumulative byte count acknowledged |
| 'H'  | NAK      | receiver        | Damaged region, retransmit from offset |
| 'I'  | DONE     | both            | No more files to send in this direction |
| 'J'  | FIN      | both            | End session |
| 'K'  | ABORT    | either          | Abort current file or batch |

The mnemonics above match the conventional names in the reference
implementation; the protocol document numbers the codes but the
high-level semantics are as listed.

## Bidirectional model

HSLink treats the link as two independent simplex streams that share
the same DLE-escaped framing layer. Each side maintains:

- A send queue of local files queued for upload.
- A receive context for the inbound file currently being written.
- A retransmit window: the byte range that has been transmitted but
  not yet acknowledged by the peer's DATAACK.

When both ends have files queued, the link saturates in both
directions concurrently. When only one side has files, the reverse
channel still carries DATAACK and NAK messages, but the asymmetric
case still beats a unidirectional ZMODEM session because the back
channel is no longer wasted on idle ACKs.

A side that finishes its send queue transmits DONE. When both sides
have sent DONE and all data has been acknowledged, either side may
send FIN to close the session.

## Initialisation handshake

```
Side A                                  Side B
INIT(version, options, blocksize, crc, window) ---->
                              <----   INIT(version, options, blocksize, crc, window)
INITACK(agreed)                        ---->
                              <----   INITACK(agreed)
FILE("foo.zip", 12345, mtime)          ---->
                              <----   FILE("bar.zip", 67890, mtime)
                              <----   FILEACK(accept, offset=0)
FILEACK(accept, offset=0)              ---->
DATA(...)                              ---->
                              <----   DATA(...)
DATA(...) DATAACK(N)                   ---->
                              <----   DATAACK(M) DATA(...)
...
EOF(12345)                             ---->
                              <----   EOF(67890)
DONE                                   ---->
                              <----   DONE
FIN                                    ---->
                              <----   FIN
```

Both sides drive their own send pipeline; interleaving of DATA, ACK,
NAK, and the next FILE header is unconstrained beyond the packet
boundary. Order within each direction is preserved by the underlying
serial framing.

## Options negotiated at INIT

| Option            | Description |
| ---               | --- |
| Block size        | Maximum DATA payload, typically 256 to 4096 bytes. |
| CRC width         | CRC-16 or CRC-32 over each packet. |
| Window size       | Maximum unacknowledged bytes outstanding per direction. |
| DLE escape mask   | Additional bytes (beyond DLE, XON, XOFF) to escape. |
| Compression       | Inline LZ-style compression of DATA payloads, on or off. |
| Resume            | Whether to honour partial-file resume on FILE. |
| Overlap I/O       | Receiver may overlap disk writes with serial reads. |

The lower of the two advertised values wins for numeric options; the
intersection wins for capability masks.

## Acknowledgement and retransmission

DATAACK carries a cumulative byte offset: "I have received every byte
up to and including offset N for the current file." The sender advances
its retransmit window to N and may discard the buffered data behind it.

NAK carries a (file, offset, length) tuple identifying a damaged
region. The sender retransmits exactly that region, then resumes
streaming from where it had been before the NAK. Because the window is
selective rather than go-back-N, a single corrupted block on a long
file does not force retransmission of everything after it.

A CRC mismatch on a DATA packet causes the receiver to discard the
packet and queue a NAK. Successive NAKs for the same region trigger
an exponential backoff in block size and may downgrade the link to
a smaller block.

## Crash recovery

When a FILE packet arrives, the receiver checks for an existing file
of the same name. If one exists and resume is enabled, the receiver
replies FILEACK with offset = current size; the sender seeks to that
offset and begins streaming from there. The receiver may instead
request a partial CRC over the existing bytes to confirm they match
the sender's copy, refusing the resume on mismatch and restarting at
offset 0. This mirrors the ZMODEM ZCRC mechanism.

## Comparison with ZMODEM

| Property                    | HSLink         | ZMODEM |
| ---                         | ---            | --- |
| Direction                   | Bidirectional  | Unidirectional per session |
| ACK style                   | Cumulative + selective NAK | Implicit, occasional ZACK |
| Block size                  | Negotiated     | Negotiated, up to 1024 |
| CRC                         | CRC-16 / CRC-32 | CRC-16 / CRC-32 |
| Resume                      | Yes            | Yes (ZRPOS / ZCRC) |
| Auto-download trigger       | No standard trigger string | ZRQINIT hex header |
| Source available            | No (shareware) | Yes (rzsz) |
| Best case throughput        | Both directions saturated | One direction saturated |

On a clean error-corrected link with files queued in both directions,
HSLink can deliver roughly twice the aggregate throughput of a pair of
sequential ZMODEM transfers because both directions of the modem
carrier are used at once. On a unidirectional transfer over a clean
link the two protocols are close, with HSLink's selective NAK giving
it a small edge on lossy lines and ZMODEM's larger installed base
giving it the practical advantage on most BBS systems.

## Licensing

HSLink was distributed as shareware. The HSLINK.EXE binary could be
freely copied and evaluated; continued use required a registration
fee paid to the author. The protocol itself was documented in
HSLINK.DOC included with the distribution, but no source reference
implementation was published, which is one reason the protocol did
not achieve the broad third-party support that ZMODEM did.

## References

- Smith, Samuel H. HSLINK.DOC. Buffalo Creek Software, circa 1991.
- The HSLINK.EXE distribution archive, typically named HSLINK.ZIP
  or HSLINKnn.ZIP where nn is a version number.
- Forsberg, Chuck. "The ZMODEM Inter Application File Transfer
  Protocol", Omen Technology, 1988, for the comparison points
  referenced above.
