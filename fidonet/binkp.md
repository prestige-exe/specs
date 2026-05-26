# BinkP – Binkley-compatible Protocol over reliable transports

by Dima Maloff, with later revisions by Nick Soveiko and Maxim Masiutin.
Originally published as FSP-1011 (July 2000). Adopted by the FidoNet
Technical Standards Committee as FTS-1026 (December 2005), with related
documents FTS-1027 (CRAM authentication), FTS-1028 (non-reliable mode),
and FTS-1029 (filename handling).

BinkP is the protocol two FidoNet systems use to exchange mail and
files over a reliable byte stream, typically TCP. It replaced the
modem-era FTS-0001 / EMSI handshake on the dial-up side once IP became
the dominant transport. Because TCP already provides ordering and
integrity, BinkP omits the error-control, flow-control, and call
management that the older handshake had to carry. The reference
implementation is binkd, Dima Maloff's mailer, which gave the protocol
its name and pinned every ambiguity in the spec by way of "whatever
binkd does".

BinkP runs on IANA-registered TCP port 24554.

## Frame format

Every byte on the wire belongs to one frame. A frame is a 2-octet
header followed by 0 to 32767 octets of data.

```
       MSB                                                       LSB
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
header | T | SIZE                                                      |
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
       | DATA (SIZE octets) ...
       +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
```

| Bit(s) | Field | Meaning |
| --- | --- | --- |
| 15 | T | 0 = data frame, 1 = command frame |
| 14..0 | SIZE | Length of the DATA section in octets (0..32767) |

The header is transmitted MSB first (network byte order); this is the
only multi-byte quantity in BinkP that is big-endian. A SIZE of 0 is
legal and means an empty frame, used as a keep-alive or to terminate
a command stream cleanly.

Data frames carry file content for the file most recently announced by
M_FILE. Command frames carry one BinkP command: the first octet of the
DATA area is the command ID (0..127), and the remaining SIZE-1 octets
are command arguments, an ASCII string with optional space-separated
parameters.

## Commands

| ID | Name | Stage | Argument syntax |
| --- | --- | --- | --- |
| 0 | M_NUL | any | `<keyword> <free text>` |
| 1 | M_ADR | setup | `<addr1> <addr2> ...` (5D addresses, space-separated) |
| 2 | M_PWD | setup | `<password>` or `-` for no password |
| 3 | M_FILE | transfer | `<name> <size> <unixtime> <offset>` |
| 4 | M_OK | setup | informational text (sent by answering side) |
| 5 | M_EOB | transfer | empty |
| 6 | M_GOT | transfer | `<name> <size> <unixtime>` |
| 7 | M_ERR | any | human-readable error message |
| 8 | M_BSY | any | human-readable busy message |
| 9 | M_GET | transfer | `<name> <size> <unixtime> <offset>` |
| 10 | M_SKIP | transfer | `<name> <size> <unixtime>` |

The size, unixtime, and offset fields in M_FILE / M_GOT / M_GET /
M_SKIP are decimal ASCII. `size` is the file's byte length, `unixtime`
is its modification time as a Unix timestamp, and `offset` is the byte
position at which to start (or resume) transmission.

### M_NUL keywords

M_NUL carries human-readable session information and protocol options.
The first whitespace-delimited token of the argument is a keyword
identifying what follows.

| Keyword | Meaning |
| --- | --- |
| `SYS` | System name |
| `ZYZ` | Sysop name |
| `LOC` | Location |
| `NDL` | Nodelist capabilities string |
| `TIME` | Current local date/time on sender |
| `VER` | Mailer name/version and supported BinkP version (e.g. `binkd/1.1a-115 binkp/1.1`) |
| `TRF` | Traffic prognosis: `<netmail_bytes> <arcmail_bytes>` |
| `OPT` | Protocol options (space-separated tokens) |
| `PHN` | Phone number |
| `OPM` | Operator message |

Defined `OPT` tokens include `NR` (non-reliable mode, FTS-1028),
`ND` (no-dupes mode), `MD5`, `CRAM-MD5-<challenge>` (FTS-1027), `MB`
(multiple batches), `EXTCMD` (extended command set), and `UTF8`.

## Session phases

A BinkP session has three phases. The originating side is the side
that opened the TCP connection; the answering side is the listener.

### 1. Setup

Both sides send M_NUL frames advertising their identity and `OPT`
capabilities, then M_ADR with their address list. The originating
side, after receiving the answering side's M_ADR, sends M_PWD with
the password. The answering side validates and replies with M_OK
(password accepted, optionally containing a human-readable comment)
or M_ERR (bad password, session terminated). If no password is
configured, the originator sends `M_PWD "-"`.

```
Originating                          Answering
-----------                          ---------
M_NUL "SYS Example BBS"
M_NUL "VER mymailer/1.0 binkp/1.1"
M_ADR "2:5020/1042@fidonet"  ----->
                             <----- M_NUL "SYS Other System"
                             <----- M_ADR "2:5020/9999@fidonet"
M_PWD "secret"               ----->
                             <----- M_OK ""
```

### 2. File transfer

Either side may transmit. The sender announces a file with M_FILE,
followed by zero or more data frames whose total length equals the
declared file size (minus `offset`). The receiver acknowledges each
fully received file with M_GOT echoing the file's name/size/time.

Receivers may interrupt the stream:

- `M_GET` requests retransmission of the current file from a different
  offset (used to resume a partial file from a previous session).
- `M_SKIP` defers the current file to a later session; the sender
  stops transmitting it and may move on.

Once a side has no more files to send, it transmits M_EOB. The session
ends when both sides have sent and acknowledged M_EOB.

### 3. Termination

Either side may end the session at any time:

- `M_ERR` signals a fatal condition. The connection is closed and the
  session counts as a failure.
- `M_BSY` signals a temporary, non-fatal condition (system busy,
  scheduled outage). The argument may begin with `RETRY <seconds>:` to
  suggest when to try again. The session counts as a soft failure;
  files queued but unsent should be retried.

## Authentication

Plain BinkP sends the password in cleartext inside M_PWD. FTS-1027
defines a challenge-response extension using CRAM (RFC 2195):

1. Answering side, during setup, sends `M_NUL "OPT CRAM-MD5-<hex>"`
   where `<hex>` is the challenge (typically a random nonce in hex).
2. Originating side replies with
   `M_PWD "CRAM-MD5-<hex-digest>"`, where `<hex-digest>` is the MD5
   HMAC of the challenge keyed with the shared password, lower-case
   hex.
3. Answering side recomputes the HMAC and accepts (M_OK) or rejects
   (M_ERR).

Both sides may advertise multiple supported algorithms. CRAM-SHA1 and
CRAM-SHA256 follow the same pattern; the algorithm names that both
sides list are intersected and the strongest is selected.

## Multiplexing and simultaneous transfer

BinkP is full-duplex. Each side can be transmitting one file while
receiving another. Because frames carry their own length and there is
no shared framing on the wire, the two directions never interfere. A
common implementation sends M_FILE / data / data / data while the
other side independently sends its own M_FILE / data / data, with
M_GOT frames interleaved as files complete in either direction.

There is no concept of windowing or per-frame acknowledgement; TCP
already provides reliable in-order delivery, so a data frame is
considered received the moment the receiver reads it.

## Non-reliable mode (FTS-1028)

The `NR` option, advertised in `OPT`, lets a receiver request that the
sender pause after each file and wait for M_GOT before starting the
next. This is intended for transports that are reliable in the
transport sense but expensive or interruption-prone (mobile data,
high-latency links), so that an interrupted session does not need to
re-send a file that the receiver has already committed to disk.

## Why binkd defines the spec

The BinkP RFC-style document is the formal definition, but in practice
the interoperability target is binkd. Two artifacts: the BinkP version
string in `VER` is conventionally `binkp/1.0` or `binkp/1.1` even from
non-binkd mailers, and several corner cases (handling of an empty
M_NUL, behavior when both sides send M_EOB simultaneously, address
matching against M_ADR when AKAs are present) are documented only by
binkd's source. Modern implementations such as Mystic's BinkIT,
Synchronet's binkit.js, and Argus all cross-test against binkd.

## References

- FSP-1011 Rev 3 / FTS-1026: Binkp - a protocol for transferring
  FidoNet mail over reliable connections. Dima Maloff, Nick Soveiko,
  Maxim Masiutin. ftsc.org.
- FTS-1027: BinkP/1.0 optional protocol extension CRAM.
- FTS-1028: BinkP/1.0 optional protocol extension Non-reliable Mode.
- FTS-1029: BinkP/1.0 protocol extension Crypt Mode and filename
  encoding.
- RFC 2195: IMAP/POP AUTHorize Extension for Simple Challenge/Response.
- binkd source distribution, https://github.com/pgul/binkd.
