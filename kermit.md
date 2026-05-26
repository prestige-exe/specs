# Kermit – File Transfer Protocol

by Frank da Cruz, Bill Catchings, et al.
Columbia University Center for Computing Activities, New York
Original public release: April 1981. Reference text: "Kermit – A File
Transfer Protocol", Frank da Cruz, Digital Press, 1987 (ISBN
0-932376-88-6).

Kermit was designed at Columbia University to move files between IBM
mainframes, DECSYSTEM-20s, and the new wave of microcomputers over the
limited 7-bit, half-duplex, flow-controlled serial connections those
machines actually had in common. Where XMODEM assumes a clean 8-bit
binary link, Kermit assumes the worst: 7-bit ASCII, control characters
swallowed by intervening hardware, line speeds throttled by XON/XOFF,
and lines that may turn into a remote shell prompt the moment you stop
talking.

## Design constraints addressed

- 7-bit clean data path only (mainframe terminals).
- Control characters (anything < 0x20 except CR) may be intercepted by
  intermediate devices and never reach the peer.
- Half-duplex links with line-turnaround delays.
- Lines whose framing is sensitive to XON/XOFF (DC1/DC3) and to BREAK.
- File systems with wildly different record structures (record-based
  VMS files vs. byte streams vs. EBCDIC-encoded mainframe files).

The protocol survives all of the above by encoding *every* packet as
printable ASCII and by negotiating capabilities up front.

## Packet format

```
MARK LEN SEQ TYPE [DATA...] CHECK [EOL]
```

| Field | Bytes | Description |
| --- | --- | --- |
| MARK | 1 | Start-of-packet, default SOH (0x01). Negotiable. |
| LEN | 1 | Packet length encoded as `char(len + 32)`, where `len` counts SEQ + TYPE + DATA + CHECK. |
| SEQ | 1 | Packet number 0..63, encoded as `char(n + 32)`. |
| TYPE | 1 | ASCII letter selecting packet type. |
| DATA | 0..n | Encoded payload (see encodings below). |
| CHECK | 1, 2, or 3 | Block check (negotiated). |
| EOL | 0..1 | Optional packet terminator; default CR (0x0D), or absent if MARK and length suffice. |

Numeric fields are turned into a printable character by adding 32
("tochar"), and decoded by subtracting 32 ("unchar"). This keeps every
byte of a packet in the range 0x20–0x7E, with the exception of MARK
itself.

### Maximum packet size

The classic length field is 1 byte and stores `len + 32`, so 0x20–0x7E
yields a length of 0–94. Maximum data per packet in basic mode is
therefore 94 minus the 3 bytes of overhead (SEQ + TYPE + CHECK) = 91
bytes.

Long-packet extension: if a Send-Init exchange announces a max packet
size > 94, packets with `LEN = 0` (encoded as space, 0x20) carry the
real length in two additional fields after TYPE, raising the cap to
9024 bytes. Sliding-window packets and even longer packet extensions
exist in modern "Super Kermit" implementations.

## Packet types

| Char | Type | Direction | Purpose |
| --- | --- | --- | --- |
| S | Send-Init | sender→receiver | Negotiate parameters |
| Y | ACK | both | Positive acknowledgement |
| N | NAK | both | Negative acknowledgement |
| F | File-Header | sender→receiver | Name of the file about to be sent |
| D | Data | sender→receiver | File data |
| Z | EOF | sender→receiver | End of current file |
| B | Break / End-of-Transmission | sender→receiver | End of batch |
| E | Error | both | Fatal error, packet data is human-readable |
| A | Attribute | sender→receiver | File metadata (length, dates, encoding) |
| I | Init-Info | both | Out-of-band negotiation, like S but no transfer follows |
| R | Receive-Initiate | receiver→sender | "Server, send me file X" |
| G | Generic command | client→server | Server-mode commands (DIR, CWD, …) |
| C | Host command | client→server | Pass-through OS command |
| T | Reserved | – | Tag (used in some extensions) |
| Q | Reserved | – | Internal |
| X | Text-Header | sender→receiver | Like F but for screen display |

## Send-Init negotiation

The first packet of any session is the Send-Init (S) packet. Its data
field is a fixed sequence of `tochar`-encoded bytes; each side picks
the minimum of its own and the peer's capability:

| Pos | Field | Meaning |
| --- | --- | --- |
| 1 | MAXL | Maximum length of packet this sender accepts (94 default) |
| 2 | TIME | Number of seconds peer should wait before timing out |
| 3 | NPAD | Number of pad characters this sender wants prefixed to incoming packets |
| 4 | PADC | Control character to use as padding (XOR 0x40) |
| 5 | EOL | Packet terminator character (encoded `+32`) |
| 6 | QCTL | Control-character prefix, default '#' (0x23) |
| 7 | QBIN | 8-bit prefix character (`Y`/`N`/`&`), or `N` if 8-bit not supported |
| 8 | CHKT | Block-check type: '1', '2', or '3' |
| 9 | REPT | Repeat-count prefix (`~`), or space if RLE not supported |
| 10 | CAPAS | Capability bitmap (long packets, sliding windows, attribute packets, …) |
| 11+ | Extension | Long-packet sizes, window size, etc. |

After the receiver ACKs the S packet with its own parameters, both
sides are committed for the rest of the session.

## Data encoding

To traverse hostile links, Kermit transforms arbitrary file bytes into
printable ASCII before transmission:

### Control-character quoting (QCTL)

Bytes in the range 0x00–0x1F or 0x7F are transmitted as two characters:
the QCTL prefix (default `#`) followed by the byte XORed with 0x40.
0x00 becomes `#@`, 0x01 becomes `#A`, 0x7F becomes `#?`. A literal
QCTL byte is sent as `##`.

### 8-bit quoting (QBIN)

If the link is 7-bit, every byte with the high bit set is prefixed
with QBIN (default `&`) and the low 7 bits transmitted. The receiver
restores the high bit. Both sides must agree to use 8-bit prefixing
during Send-Init.

If the link is 8-bit clean, both sides set QBIN to `N` and high-bit
bytes pass through untouched (still subject to control-character
quoting if the value happens to lie in 0x00–0x1F | 0x7F.

### Repeat-count compression (REPT)

If both sides advertise REPT in Send-Init, runs of 3 or more identical
bytes are encoded as `~<count> <byte>` where `<count>` is `tochar(n)`.
A literal `~` is sent as `~~`. This single-byte run-length encoding is
particularly effective on text files and source code.

### Quoting precedence

Encoding rules are applied in this order: REPT first, then QBIN, then
QCTL. The receiver reverses them in opposite order: QCTL first, then
QBIN, then REPT.

## Block check (CHKT)

Three block-check types are defined; the strongest both sides support
is selected during Send-Init:

| CHKT | Bits | Algorithm |
| --- | --- | --- |
| '1' | 6 | Single character, arithmetic sum of bytes mod 64 + 32. Default; always supported. |
| '2' | 12 | Two characters, arithmetic sum of bytes mod 4096, split into two 6-bit `tochar` values. |
| '3' | 16 | Three characters, CRC-16/CCITT (poly 0x1021, init 0), split into three 6-bit `tochar` values. |

The block check covers SEQ, TYPE, and DATA.

## Window management

Classic Kermit is stop-and-wait: every packet is ACKed before the next
is sent. "Sliding-windows Kermit" (Super Kermit) negotiates a window
size of up to 31 unacknowledged packets, raising throughput on
high-latency links toward the full link speed.

Combined with long packets (up to 9 KB), sliding windows allow Kermit
to match or exceed ZMODEM on the connections it was designed for.

## Server mode

Unlike XMODEM/YMODEM/ZMODEM, Kermit defines a peer-to-peer command
protocol. After a "REMOTE SERVER" command, the far-end Kermit waits for
G (generic) and C (host) packets:

| Generic | Mnemonic | Purpose |
| --- | --- | --- |
| I | Login | Authenticate |
| C | Change-dir | Set working directory |
| L | Logout | End session |
| D | Directory | Return directory listing |
| U | Disk-usage | Return free space |
| E | Erase | Delete a file |
| T | Type | Display a file |
| R | Rename | Rename a file |
| K | Kermit-version | Identify peer |
| H | Help | Help text |
| F | Finish | Exit server mode |
| Q | Query | Ask a question |
| V | Variable | Get/set a variable |
| X | Xtended | Other |

This is the ancestor of FTP-style command/response semantics over
serial links and lets a Kermit client drive a remote Kermit as a file
server.

## Attribute (A) packets

File metadata is transmitted in an attribute packet before the data
stream. Each attribute is a 3-character group: 1-byte tag, 1-byte
length (`tochar`-encoded), and length bytes of value. Tags include:

| Tag | Meaning |
| --- | --- |
| `!` | File length in K |
| `"` | File length in bytes |
| `#` | Encoding type (ASCII, image, binary, …) |
| `$` | Creation date (YYYYMMDD HH:MM:SS) |
| `%` | Last-modified date |
| `*` | Encoding character set (e.g. "A" for ASCII, "I" for image/binary) |
| `+` | Disposition (e.g. mail, print) |
| `,` | File access protection |
| `-` | File character set |
| `.` | Record format (fixed, variable) |
| `/` | Record size |
| `0` | System ID (origin OS) |
| `1` | System-info text |

These were essential for moving record-structured VMS or MVS files to
stream-oriented Unix targets without corrupting them.

## Why Kermit lasted

- Runs over *anything* that can move printable characters – BBS lines,
  3270 emulators, TELNET sessions through 5 layers of gateway,
  X.25 PAD, even modems with software handshaking turned all the way
  up. Many sites that could not move a single byte with XMODEM moved
  gigabytes with Kermit.
- Open documentation and a permissive licence let it be ported to
  hundreds of platforms; the Columbia University project maintained a
  reference list of ports well into the 2010s.
- Defined a complete client/server file-management protocol, not just a
  byte mover, anticipating FTP-style usage two decades before SSH
  killed both.

The Kermit Project at Columbia University was officially closed in
2011, but Kermit 95 and C-Kermit have been released as open source and
remain maintained by the community.
