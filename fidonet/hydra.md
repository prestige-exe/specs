# HYDRA – Bidirectional file-transfer protocol

by Arjen G. Lentz (Lentz Software-Development) and Joaquim H. Homrighausen.
Specification revision 001, December 1, 1992, also published as EMSC-002
and later submitted to the FTSC as FSC-0072.

HYDRA is a bidirectional file-transfer protocol for full-duplex serial
links. It was designed to outperform ZMODEM (and its FidoNet variants
ZedZap, ZedZip, SEAlink) by sending and receiving files at the same
time over the same connection. On a 9600 bps full-duplex modem with
files going each way, HYDRA finishes in roughly the time ZMODEM takes
to send one direction. The reference implementation HydraCom, written
by Lentz and later joined by Adam Blake, defined the wire behaviour;
the spec itself acknowledges debts to Chuck Forsberg's ZMODEM and
Rick Huebner's Janus.

The protocol assumes the link can pass DLE (0x18, called `H_DLE` in
the spec) plus ASCII 32 through 126. Any other character may be
escaped or encoded as the link requires; HYDRA negotiates which
escapes are in effect.

## Terminology and encoding

| Term | Meaning |
| --- | --- |
| `BYTE` | 8-bit unsigned |
| `WORD` | 16-bit unsigned, little-endian on the wire |
| `DWORD` | 32-bit unsigned, little-endian |
| `LONG` | 32-bit signed, little-endian |
| `H_DLE` | ASCII 24 (Ctrl-X), the HYDRA link escape |
| Unix time | Local-time seconds since 1970-01-01 |

Numbers transmitted "in hex" are lower-case, left-padded with `0` to
their full byte width. A WORD with value 255 is `00ff`; a LONG is
`000000ff`.

## Packet structure

Every exchange is a framed packet. The unframed packet is:

```
+-------------------------------------------+
| zero or more bytes of packet payload      |
+-------------------------------------------+
| packet type byte (one ASCII character)    |
+-------------------------------------------+
| CRC-16 or CRC-32 over payload + type      |
+-------------------------------------------+
```

The CRC is one's-complemented and transmitted low-byte first. CRC-32
is negotiated in the INIT exchange via the `C32` capability flag;
without it, the link uses CRC-16 (the same polynomial as XMODEM-CRC).

The packet is then framed for transmission:

```
+-------+--------------------+
| H_DLE | packet format byte |
+-------+--------------------+
| encoded packet bytes       |
+-------+--------------------+
| H_DLE | 'a' (end-of-frame) |
+-------+--------------------+
```

| Format byte | ASCII | Encoding |
| --- | --- | --- |
| `a` | 97 | End of framed packet |
| `b` | 98 | BIN: raw 8-bit, H_DLE escaped as `H_DLE H_DLE` |
| `c` | 99 | HEX: each payload byte as two lowercase hex digits |
| `d` | 100 | ASC: shifted 7-bit (optional, negotiated) |
| `e` | 101 | UUE: UUencoded (optional, negotiated) |

Five consecutive `H_DLE` bytes are the cancel sequence. All packet
types other than `DATA` are followed by a CR (0x0D) to help survive
buffered terminal servers and to aid debugging.

## Packet types

| Type byte | Name | Direction | Purpose |
| --- | --- | --- | --- |
| `A` (65) | START | both | Wake-up / synchronisation |
| `B` (66) | INIT | both | Capability and option exchange |
| `C` (67) | INITACK | both | Acknowledge an INIT |
| `D` (68) | FINFO | sender | File information / start of file |
| `E` (69) | FINFOACK | receiver | Accept, reject, or resume a file |
| `F` (70) | DATA | sender | File data block |
| `G` (71) | DATAACK | receiver | Window-management ACK |
| `H` (72) | RPOS | receiver | Reposition / skip request |
| `I` (73) | EOF | sender | End of file |
| `J` (74) | EOFACK | receiver | Acknowledge EOF |
| `K` (75) | END | both | End of session |
| `L` (76) | IDLE | both | Keep-alive |
| `M` (77) | DEVDATA | both | Out-of-band data to a device (optional) |
| `N` (78) | DEVDACK | both | Acknowledge DEVDATA (optional) |

## Startup

Before the START packet, the sender transmits the literal ASCII string
`hydra\r` so a HYDRA-aware program at the far end can autostart even
from a shell prompt. Every five seconds the sender repeats the string
plus a START packet until it sees a START or INIT come back. Stale
packets from any earlier session received in this stage are ignored.

## INIT / INITACK negotiation

The INIT packet is in HEX format and carries:

```
+---------------------------------------+
| application ID string, NUL-terminated |
+---------------------------------------+
| supported options, NUL-terminated     |
+---------------------------------------+
| desired options, NUL-terminated       |
+---------------------------------------+
| desired transmitter window size (LONG)|
+---------------------------------------+
| desired receiver window size (LONG)   |
+---------------------------------------+
| general options, NUL-terminated       |
+---------------------------------------+
| packet prefix string, NUL-terminated  |
+---------------------------------------+
```

The application ID is `<RevDate>,<ProductName>,<ProductRevision>[,<Serial>]`
where `<RevDate>` is the Unix timestamp of the HYDRA spec the
implementation conforms to. Capability flags are three-letter
uppercase tokens separated by commas:

| Flag | Meaning |
| --- | --- |
| `XON` | Escape XON / XOFF |
| `TLN` | Escape the `<CR>@<CR>` Telenet sequence |
| `CTL` | Escape ASCII 0..31 and 127 |
| `HIC` | Escape the above three with the high bit set |
| `HI8` | Escape ASCII 128..255 and strip the high bit |
| `BRK` | Can transmit a break signal |
| `ASC` | Can handle ASC-format packets |
| `UUE` | Can handle UUE-format packets |
| `C32` | Can receive CRC-32 packets |
| `DEV` | Can receive DEVDATA packets |
| `FPT` | Can receive filenames containing paths |

The first five flags are mandatory. The "supported" list is everything
the sender can do; the "desired" list is what it wants enabled for
this session. An option becomes active only if both sides list it as
supported and at least one side desires it.

Window sizes are independent per direction. The smaller of the two
sides' requested sizes wins for each direction; any non-zero window
takes precedence over zero (full streaming). The packet prefix string
lets a side request that every outgoing packet from the remote be
preceded by a specific byte sequence (used to placate older modems and
terminal servers). Special bytes in the prefix: ASCII 221 = transmit
a break for 1 second, 222 = delay 1 second, 223 = transmit a NUL.

INITACK echoes the INIT and signals the remote that its options were
seen. Duplicate INITs (which happen if an INITACK is lost) are
re-acknowledged but do not reset the braindead timer.

## File transfer

After INIT/INITACK both sides enter the file-transfer loop. Each side
independently runs the sender state machine for its outbound files and
the receiver state machine for its inbound files, sharing the link.

### FINFO

The sender announces a file with FINFO containing:

```
LONG  timestamp        (Unix mtime, 0 if unknown)
LONG  filesize         (0 if unknown)
LONG  reserved (0)
LONG  transaction#     (0 unless a file-request batch)
LONG  filecount        (total on first file, ordinal on later files)
char[] short filename  (lowercase, 8.3, NUL-terminated)
char[] real filename   (optional, NUL-terminated; allows paths if FPT)
NUL
```

End of batch is signalled by an empty FINFO (just a NUL).

The receiver replies with FINFOACK carrying a LONG that is either a
file offset (>= 0, possibly non-zero to resume), `-1` ("already have
this file, skip"), or `-2` ("don't send it now, queue for a later
batch").

### DATA / DATAACK / RPOS

DATA packets are:

```
LONG offset
0..2048 bytes of file data
```

The receiver maintains an expected offset and compares it against the
offset in each incoming DATA packet. A match advances the offset and
the data is written to disk. A mismatch (lost or corrupted packet) is
not "NAK'd"; instead the receiver sends an RPOS:

```
LONG offset             (desired resume point)
WORD block size         (64..2048)
LONG packet ID          (non-zero, unique per file)
```

The sender seeks to `offset` and resumes from there using the
requested block size. The packet ID lets the sender ignore duplicates
of the same RPOS.

DATAACK is used only when the receiver has requested a window. It
contains the LONG of the receiver's current file offset and lets the
sender stay no more than `window` bytes ahead of the last
acknowledged offset.

To skip an in-progress file, the receiver sends RPOS with offset `-2`
and waits for an EOF with the matching `-2` offset, ACKs it, and
moves on.

### EOF / EOFACK

When the sender reaches the declared file size, it sends EOF (a LONG
offset, which is the final position). The receiver verifies the
offset matches its own, then sends EOFACK. EOF for a skipped file
carries offset `-2`; EOFACK confirms.

The sender then either sends the next FINFO or signals end-of-batch
with an empty FINFO. Both sides exchange END to terminate the
session.

### IDLE

Either side may send IDLE to declare "I'm alive but nothing to send".
Receipt of IDLE resets the braindead timer. This is used when a one-
sided imbalance (only one side has files) leaves the link quiet in
one direction.

## CRC selection

HEX-formatted packets always use CRC-16 because the encoding doubles
the size already and the wire was historically slow. BIN/ASC/UUE
packets use CRC-32 if both sides advertised `C32`; otherwise CRC-16.
The CRC is computed over the payload and the packet type byte, then
one's-complemented, then transmitted low-byte first.

## Where HYDRA fit

HYDRA was the de-facto bidirectional protocol in FidoNet for the
remainder of the modem era. It is negotiated through the EMSI session
setup (see [[emsi]]) as the `HYDR` flag. ZedZap remained more common
for one-way file requests because most mailers had it; HYDRA paid off
on netmail polls where both sides routinely had traffic queued.

## References

- HYDRA.DOC rev 001, December 1, 1992. Joaquim H. Homrighausen and
  Arjen G. Lentz. Also EMSC-002.
- FSC-0072.001, FTSC submission of the HYDRA specification.
- HydraCom 1.00 reference implementation, Lentz Software-Development.
- HydraCom 1.08, Adam Blake (Wandoo Valley Software) and Arjen Lentz.
- ZMODEM (Chuck Forsberg) and Janus (Rick Huebner), the two protocols
  HYDRA combined.
