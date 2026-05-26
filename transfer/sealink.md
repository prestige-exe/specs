# SEAlink – Sliding-Window XMODEM Variant

by System Enhancement Associates (SEA), Thom Henderson
Reference text: SEAlink protocol documentation distributed with the
SEAlink reference source SEALINK.C, circa 1986–1987, alongside the
SEAdog FidoNet mailer.

SEAlink is an extension of XMODEM-CRC designed by Thom Henderson at
System Enhancement Associates, the same company responsible for the
ARC archiver. Its purpose was to eliminate the per-block turnaround
penalty of XMODEM on links with non-trivial round-trip time, in
particular satellite hops and long-haul FidoNet calls. SEAlink keeps
the XMODEM frame on the wire byte-for-byte where possible, so a
SEAlink sender can fall back gracefully to plain XMODEM-CRC against a
peer that does not implement the sliding window. It was the native
file transfer protocol of the SEAdog mailer for several years and
served as one of the design references for ZMODEM's later streaming
window.

## Why XMODEM was slow on real links

XMODEM transmits a 128-byte block, then stops and waits for the
receiver to return ACK or NAK before transmitting the next block.
At 2400 bps the wire time of one 128-byte block is roughly 530 ms.
A geostationary satellite hop adds about 600 ms of round-trip
propagation delay before the ACK can return. The sender therefore
spends more time waiting than transmitting, and link utilisation
falls under 50%. SEAlink fixes this by letting the sender stream
ahead by up to six blocks before it must pause for an ACK.

## Block layout

The on-wire SEAlink block is identical to an XMODEM-CRC block:

```
<SOH> <blk#> <255-blk#> <128 data bytes> <CRC-high> <CRC-low>
```

| Field         | Size | Description |
| ---           | ---  | --- |
| SOH           | 1    | 0x01, start of block |
| Block number  | 1    | 8-bit sequence, wraps at 0xFF -> 0x00 |
| ~Block number | 1    | One's complement of block number |
| Data          | 128  | Fixed 128 data bytes |
| CRC           | 2    | CRC-16/XMODEM, high byte first |

The frame is bit-compatible with XMODEM-CRC. A SEAlink receiver
talking to an XMODEM sender sees ordinary XMODEM blocks; a SEAlink
sender talking to an XMODEM receiver sees ordinary ACKs and simply
never accumulates more than one unacknowledged block.

## Sliding window

The sender maintains a window of up to six unacknowledged blocks. It
may transmit the next block as soon as the previous one has cleared
the UART, without waiting for an ACK. The receiver returns one ACK
per successfully validated block, carrying the block number being
acknowledged:

```
<ACK> <blk#>
```

The single ACK byte alone is still accepted for compatibility with
plain XMODEM, but a SEAlink receiver appends the block number so the
sender can match it against the window. On NAK the receiver sends:

```
<NAK> <blk#>
```

where `blk#` is the block it expected next. The sender drops every
block in its window from that number onward and retransmits from there.
This is go-back-N, not selective retransmission. Six blocks of
ambiguity proved sufficient for the satellite links of the period
without requiring per-block tracking on the receiver.

## Overlapped I/O

The SEAlink reference implementation pairs the sliding window with
non-blocking disk I/O. The receiver writes the previous block to disk
while the next block is arriving on the serial port; the sender reads
the next block from disk while the previous block is still in the UART
FIFO. On systems with a slow disk, this overlap is responsible for as
much of the throughput gain as the window itself.

## Long block option

A later revision of SEAlink added an optional 1024-byte block, signalled
by STX (0x02) in place of SOH:

| Header byte | Data size |
| ---         | --- |
| SOH (0x01)  | 128 bytes  |
| STX (0x02)  | 1024 bytes |

The long block predated YMODEM's STX-keyed 1K block but uses the same
framing convention. Block numbers continue to increment by one per
block regardless of block size, and the two block sizes may be mixed
within a single transfer if both sides advertise support during the
batch header.

## Batch transfer (overdrive)

SEAlink optionally carries a "batch header" as block 0, sent before
the first data block. The batch header contains the file name, size,
and modification time, encoded similarly to the YMODEM block 0:

```
<SOH> 00 FF "filename\0 size mtime mode\0..." <CRC>
```

The receiver acknowledges block 0 with the usual ACK and the sender
proceeds to block 1 as the first data block. A second block 0 may
follow the EOT of the previous file to begin the next file in the
same batch; an empty block 0 (filename = "") signals the end of the
batch. This batch convention is essentially the same one that YMODEM
formalised later.

## Handshake

```
Sender                                  Receiver
                                <-----  'C' (XMODEM-CRC request)
SOH 00 FF <batch header> <CRC>  ----->
                                <-----  ACK 00
SOH 01 FE <data ...>            ----->
SOH 02 FD <data ...>            ----->
SOH 03 FC <data ...>            ----->
                                <-----  ACK 01
SOH 04 FB <data ...>            ----->
                                <-----  ACK 02
...
EOT                             ----->
                                <-----  ACK
                                <-----  'C'   (ready for next file)
SOH 00 FF <next file>           ----->
...
EOT                             ----->
                                <-----  ACK
SOH 00 FF "\0" <CRC>            ----->   (empty filename: end of batch)
                                <-----  ACK
```

The session begins with the same 'C' that initiates any XMODEM-CRC
transfer. The sender determines whether the receiver is SEAlink-capable
from the form of the returning ACK: a SEAlink receiver returns
`ACK <blk#>`, a plain XMODEM-CRC receiver returns `ACK` alone. The
sender shrinks its window to one block in the latter case.

## Errors and recovery

| Condition                         | Sender action |
| ---                               | --- |
| NAK <n>                           | Roll window back to block n, resume |
| CAN CAN                           | Abort transfer |
| No ACK within timeout             | Resend oldest unacknowledged block |
| Six consecutive NAKs on one block | Abort |
| Bad complement / bad CRC at receiver | Discard block, send NAK <expected> |

The character timeout within a block remains one second, the same as
XMODEM. The block timeout is extended to accommodate the window depth:
the sender waits up to ten seconds after the last byte of the window
goes out before assuming an ACK was lost.

## Place in the lineage

SEAlink occupies the niche between XMODEM-CRC and the streaming
protocols that followed. Its contributions to the lineage:

- Sliding window over an XMODEM-shaped frame demonstrated that
  streaming was practical without abandoning XMODEM compatibility.
- Overlapped disk and serial I/O established the receiver design that
  ZMODEM later required outright via the CANOVIO capability bit.
- The batch header convention was carried forward almost verbatim by
  YMODEM block 0.
- The 1024-byte block predated YMODEM's STX-keyed 1K block.

SEAdog used SEAlink as its default mail transfer protocol; Opus and
other FidoNet mailers adopted it for the same reasons. Once ZMODEM
became widely available with full-duplex streaming and crash recovery,
SEAlink fell out of use, but Janus, Hydra, and the bidirectional
protocols described elsewhere in this collection borrow directly from
its acknowledgement and window design.

## References

- Henderson, Thom. SEAlink protocol documentation, distributed with the
  SEAdog mailer source kit, System Enhancement Associates, circa 1986.
- The SEALINK.C reference source, widely re-distributed in FidoNet
  utility archives.
- Forsberg, Chuck. YMODEM and ZMODEM specifications, for the points of
  comparison cited above.
