# Janus – Bidirectional FidoNet Mail Transfer Protocol

by Rick Huebner / Opus-CBCS
Reference text: JANUS.DOC, distributed with the Opus-CBCS source kit
and FSC-0028 / FTS proposal drafts in the FidoNet Technical Standards
series, circa 1988–1989.

Janus is a full-duplex, bidirectional file transfer protocol written by
Rick Huebner for the Opus-CBCS bulletin board and mailer. Like its
contemporary HSLink and its successor Hydra, Janus carries files in
both directions of a full-duplex modem session at the same time, which
roughly doubles aggregate throughput on a session that has mail
queued in both directions, as a FidoNet mail exchange typically does.
Janus was the first widely deployed bidirectional protocol in the
FidoNet world and shaped the design of every protocol that followed it
in that lineage, most directly Hydra (Joaquim Homrighausen and Arjen
Lentz) and, more indirectly, ZedZap, a 1990s ZMODEM-with-8K-blocks
variant carried inside the same EMSI session framework.

## Design goals

- Use both directions of a full-duplex modem simultaneously.
- Operate over the standard FidoNet session framework: an initial
  handshake (later EMSI) negotiates capabilities, then the agreed
  transfer protocol takes over.
- Resume partially received files.
- Be tolerant of error-correcting modems (MNP, V.42) and of the older
  uncorrected links Janus was originally designed against.
- Carry FidoNet mail's typical workload of many small packet files and
  occasional large archives without per-file turnaround cost.

## Frame layout

Janus frames each unit of work as a packet keyed by a single-byte type
code. Packets are DLE-quoted so that DLE, XON, XOFF, and a small set of
other control bytes never appear unescaped in the data stream.

```
<DLE> <SOH> <type> <length-lo> <length-hi> <data ...> <crc ...>
```

| Field   | Bytes      | Description |
| ---     | ---        | --- |
| DLE SOH | 2          | Packet lead-in, 0x10 0x01. |
| Type    | 1          | Packet type code. |
| Length  | 2          | Payload length, little-endian. |
| Data    | 0..N       | Payload bytes, DLE-escaped on the wire. |
| CRC     | 2 or 4     | CRC-16/XMODEM or CRC-32, selected at handshake. |

DLE escaping replaces 0x10, 0x11, 0x13, 0x90, 0x91, 0x93, and the lead
SOH byte with `DLE <byte XOR 0x40>` on the wire. CRC is computed over
the unescaped type, length, and data fields. Janus negotiates between
CRC-16/XMODEM (polynomial 0x1021) and CRC-32 (the PKZIP / Ethernet
CRC-32, polynomial 0xEDB88320 reflected) during the init exchange.

## Packet types

The reference implementation defines packet codes for both control and
data traffic. The set below covers the operationally important codes;
specific numeric values are in JANUS.DOC and the Opus source.

| Mnemonic | Direction | Meaning |
| ---      | ---       | --- |
| INIT     | both      | Capability advertisement and option flags |
| INITACK  | both      | Reply to INIT, agreed options |
| FNAME    | sender    | File header: name, size, mtime, mode |
| FINFO    | receiver  | Reply: accept, skip, or resume from offset |
| DATA     | sender    | Data chunk at implicit running offset |
| ACK      | receiver  | Cumulative byte offset acknowledged |
| NAK      | receiver  | Damaged region, retransmit from offset |
| EOF      | sender    | End of current file |
| EOB      | both      | End of batch in this direction |
| END      | both      | End of session |
| BRK      | either    | Abort current file |

## Bidirectional model

A Janus session has two independent send pipelines, one per direction,
multiplexed onto the same framed serial link. Each side maintains:

- A queue of outbound files (typically the FidoNet outbound packet
  directory plus any attached files).
- An inbound write context for the file currently being received.
- A retransmit buffer covering the bytes sent but not yet acknowledged.

Either side may interleave DATA, ACK, NAK, FNAME, and EOF packets in
any order on its outbound channel. Order within one direction is
preserved by the serial link; order between the two directions is not
meaningful.

When a side has nothing more to send, it transmits EOB and continues to
service the inbound channel and to emit ACK / NAK as needed. When both
sides have sent EOB and all outstanding bytes have been acknowledged,
either side may send END to close the session.

## Handshake

In Opus and early Opus-derived mailers, Janus was negotiated by a
proprietary "Yoohoo / Yoohoo2u2" exchange that advertised the calling
system's address and the protocols it supported. With the standardisation
of EMSI (Electronic Mail Standard Identification, FSC-0056 / FTS-0006)
the protocol selection moved into the EMSI_DAT packet. An EMSI session
that selects Janus arrives at the protocol layer with the link already
in 8-bit binary mode and with both sides knowing each other's FidoNet
address and password.

```
Caller                                  Answerer
<EMSI handshake; both sides agree on "JAN" capability>
INIT(version, options, crc, blocksize) ---->
                              <----   INIT(version, options, crc, blocksize)
INITACK(agreed)                        ---->
                              <----   INITACK(agreed)
FNAME("net.pkt", 4096, mtime)          ---->
                              <----   FNAME("reply.pkt", 1024, mtime)
                              <----   FINFO(accept, offset=0)
FINFO(accept, offset=0)                ---->
DATA(...)                              ---->
                              <----   DATA(...) ACK(N)
DATA(...) ACK(M)                       ---->
                              <----   DATA(...)
...
EOF                                    ---->
                              <----   EOF
EOB                                    ---->
                              <----   EOB
END                                    ---->
                              <----   END
```

After END, control returns to the caller's mailer for any
post-transfer processing (toss inbound, mark outbound sent, etc.).

## Options negotiated at INIT

| Option         | Description |
| ---            | --- |
| CRC width      | CRC-16/XMODEM or CRC-32. CRC-32 is preferred on clean links. |
| Block size     | DATA payload size, typically 256 to 2048 bytes. |
| Window size    | Maximum bytes outstanding per direction before the sender must pause. |
| Resume         | Whether to honour offset-based resume on FINFO. |
| Compression    | Whether DATA payloads may be compressed inline. |

The lower of the two advertised values wins for numeric options; the
intersection wins for capability bits.

## Acknowledgement, NAK, and resume

ACK carries a cumulative byte offset for the current file: "I have
received every byte up to and including offset N." The sender advances
its retransmit window past N and may discard the buffered data.

NAK names a (file, offset) at which the receiver wants the sender to
resume. NAK is generated on CRC mismatch, on a missing or out-of-order
DATA, or after an inbound timeout. Janus uses go-back-N from the NAK
offset; selective retransmission of individual blocks is not part of
the protocol.

Resume of an interrupted file uses the same mechanism as ZMODEM's
ZRPOS: the receiver, on FNAME, finds an existing partial file and
replies FINFO with offset = size of the partial. The sender seeks and
streams from there. An option in the INIT exchange may instead request
a partial CRC check before the resume is honoured.

## Comparison with Hydra and ZedZap

| Property            | Janus              | Hydra              | ZedZap |
| ---                 | ---                | ---                | --- |
| Direction           | Bidirectional      | Bidirectional      | Unidirectional |
| Frame layer         | DLE-quoted         | DLE-quoted, slightly larger | ZMODEM (ZDLE) |
| Max block           | ~2 KB              | ~8 KB              | 8 KB |
| CRC                 | CRC-16 / CRC-32    | CRC-16 / CRC-32    | CRC-16 / CRC-32 |
| Resume              | Yes                | Yes                | Yes (ZRPOS) |
| EMSI integration    | Yes (EMSI "JAN")   | Yes (EMSI "HYD")   | Yes (EMSI "ZAP") |
| Streaming model     | DATA + cumulative ACK | DATA + cumulative ACK | ZDATA + subpackets |

Hydra is essentially a refined Janus with a larger block size, tighter
CRC choice, and improved behaviour over satellite links; it superseded
Janus in most FidoNet mailers by the mid-1990s. ZedZap is not a
bidirectional protocol; it is ZMODEM with the maximum subpacket raised
to 8192 bytes and the session embedded inside EMSI. The three are
commonly all available in the same mailer, with EMSI selecting whichever
protocol is supported by both ends.

## Place in the lineage

Janus was the first bidirectional file transfer protocol to see wide
deployment in FidoNet and was the proof that two-way streaming was a
practical improvement over ZMODEM for mailer-to-mailer sessions. Its
direct descendant Hydra absorbed the operating experience of several
years of Janus deployment, increased the block size to match the
buffer sizes of error-correcting modems, and is the protocol most
modern FidoNet mailers still default to for sessions where both sides
support it.

## References

- Huebner, Rick. JANUS.DOC, distributed with the Opus-CBCS source kit,
  circa 1988–1989.
- FidoNet Technical Standards Committee, FSC-0028 and related drafts
  describing Janus framing.
- FSC-0056 / FTS-0006, the EMSI handshake specification that carries
  Janus capability advertisement on modern mailer links.
- Homrighausen, Joaquim, and Lentz, Arjen. Hydra-spec.txt, the
  successor protocol's reference, for the comparison points above.
