# EMSI – Electronic Mail Standard Identifier

by Joaquim Homrighausen, Scott Dudley, et al.
FidoNet Standards Committee draft FSC-0056 (revised through FSC-0056.005,
August 1994). EMSI did not become a formal FTS, but every modern
FidoNet mailer implements it as the de-facto session-establishment
standard.

EMSI replaces the legacy FTS-0001 "Stone Age" handshake with a single
exchange that:

- Identifies both ends with a complete address list (multiple AKAs).
- Authenticates with a session password.
- Negotiates which file-transfer protocol to use.
- Optionally requests file requests, compression, or yoohoo handling.

The result is one round-trip from "carrier detect" to "ready to
transfer", instead of the cascade of SYN/ACK/NAK/YOOHOO bytes the
legacy handshake required.

## Wire format

EMSI exchanges consist of "packets" embedded in printable ASCII so
that they survive 7-bit links and screen-scraping intermediaries.
Every packet has the form:

```
**EMSI_<type><length><data><CRC-16>\r
```

| Field | Description |
| --- | --- |
| `**` | Two ASCII asterisks, packet introducer |
| `EMSI_` | Literal ASCII "EMSI_" |
| `<type>` | Three-letter type code (REQ, ACK, NAK, DAT, INQ, HBT, CLI, IIR, ICI, IRQ) |
| `<length>` | 4 ASCII hex digits, length of `<data>` |
| `<data>` | `<length>` bytes of packet payload |
| `<CRC-16>` | 4 ASCII hex digits, CRC-16/XMODEM over the entire packet up to but not including the CRC |
| `\r` | CR (0x0D) terminator |

The CRC uses the same polynomial 0x1021 / initial 0x0000 as
XMODEM-CRC ([[xmodem]]). All hex is uppercase.

## Packet types

| Type | From | Purpose |
| --- | --- | --- |
| REQ | calling side | "I want to start an EMSI session" |
| ACK | both | Positive acknowledgement |
| NAK | both | Negative acknowledgement |
| DAT | both | Session data (the meat of the handshake) |
| HBT | both | Heartbeat / keepalive |
| CLI | calling side | Client identifier |
| IIR | answering side | Interactive identifier response |
| ICI | – | Interactive client identifier |
| IRQ | – | Interactive request |
| INQ | – | Polling for EMSI capability |

## Handshake

After modem connect at any baud:

```
Calling                                  Answering
**EMSI_REQ00009B7E2\r                       (sent up to 6 times)
                                <-----  **EMSI_INQC0007...\r
**EMSI_DAT....\r              ----->
                                <-----  **EMSI_DAT....\r
                                <-----  **EMSI_ACK....\r
**EMSI_ACK....\r              ----->
                                                  (session begins)
```

1. The calling side transmits `**EMSI_REQ...` periodically. Length is
   0x0009 – nine bytes of header data which is conventionally just
   the literal string `"EMSI_REQ"` plus padding.
2. The answering side replies with `**EMSI_INQ...` (inquiry), which
   confirms it speaks EMSI.
3. Both sides exchange `**EMSI_DAT` packets containing the session
   data (addresses, password, capabilities, requested protocol).
4. Both sides ACK each other's DAT.
5. The negotiated transfer protocol (Janus, ZedZap, HYDRA, ...) takes
   over.

If the answering side does not respond to REQ within a few seconds
(or sends a non-EMSI banner), the caller falls back to FTS-0001
"Stone Age" or to YooHoo / 2U2 handshakes for older systems.

## DAT packet payload

The DAT packet's `<data>` carries a structured curly-brace list:

```
{EMSI}{<addresses>}{<password>}{<link-codes>}{<compat-codes>}{<mailer-info>}
```

| Field | Description |
| --- | --- |
| `{EMSI}` | Literal token |
| `{<addresses>}` | Space-separated 4D FidoNet addresses (AKAs) |
| `{<password>}` | Session password, or `-` if none |
| `{<link-codes>}` | Comma-separated capability list (see below) |
| `{<compat-codes>}` | Comma-separated compatibility list |
| `{<mailer-info>}` | Mailer product name and version |

Example:

```
{EMSI}{1:1/8 1:1/8.1@fidonet}{MyPassword}{8N1,PUA,NRQ,FNC,ZAP,RH1}{XX0000}{Argus,3.001,Argus}
```

### Link codes

| Code | Meaning |
| --- | --- |
| `8N1` | 8 data bits, no parity, 1 stop bit (essentially: 8-bit clean) |
| `PUA` | Pickup available |
| `PUP` | Pickup of files at end of session |
| `NPU` | No pickup |
| `NRQ` | No file request |
| `RQA` | Requests accepted |
| `RH1` | Receive hold = 1 |
| `RMA` | Remote mail attached |
| `XMA` | Extended mail attribute |
| `FNC` | File-name conversion needed (e.g. 8.3 on receiver side) |
| `FRQ` | File request from caller |
| `NCO` | No compression |
| `MD2` | Method 2 cookie |
| `ARC`, `ZIP`, `LZH`, `ZOO`, `ARJ`, `RAR` | Bundle formats supported |

### Compatibility codes (transfer protocols)

| Code | Protocol |
| --- | --- |
| `ZAP` | ZedZap (ZModem with FidoNet extensions, 8K blocks) |
| `ZMO` | Plain ZModem |
| `HYD` | HYDRA (full-duplex protocol by Arjen Lentz) |
| `JAN` | Janus (full-duplex protocol by Rick Huebner) |
| `KER` | Kermit |
| `DZA` | DirectZap |
| `NCP` | No compatible protocol – fallback |

Both ends pick the strongest protocol they have in common; HYDRA and
ZedZap are the practical winners in the 1994–2000 era.

## Inter-zone routing

The 4D addresses in the address list let each side announce all of
its AKAs in different zones. The peer chooses which AKA to use for
the actual mail exchange based on which zones it serves. This is what
makes a single mailer call deliver mail for multiple zones at once.

## Why EMSI matters

- Single round-trip handshake instead of three or four with the
  legacy protocol – significant latency savings on long-distance
  modem calls billed by the minute.
- 4D address negotiation (with point and zone) which the legacy
  protocol cannot carry cleanly.
- Capability advertisement, so a mailer that supports HYDRA can call
  one that supports only ZModem and the two will downgrade gracefully
  without operator intervention.
- The `{EMSI}` packet is a self-delimiting printable-ASCII envelope,
  so it survives line noise and intermediate equipment (PADs, fax
  switches) far better than the legacy SYN/ACK dance.

The modern FidoNet-over-Internet protocols (BinkP / FTS-1024) replace
the EMSI envelope with a TCP-friendly binary protocol but use the
same conceptual handshake: identify, authenticate, advertise, transfer.
