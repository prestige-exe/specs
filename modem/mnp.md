# MNP – Microcom Networking Protocol

by Microcom, Inc. (Norwood, Massachusetts)
Originally developed in the mid-1980s by Microcom as a proprietary
modem-to-modem error-correction and compression protocol. Classes 1
through 4 were placed in the public domain in 1988 and later folded
into ITU-T recommendation V.42 as an alternative framing for LAPM
detection. Classes 5 and above remained Microcom intellectual property
but were licensed broadly enough that virtually every dial-up modem
shipped between 1989 and 1998 implemented at least MNP 1 through 5.

MNP defines a numbered hierarchy of "service classes". Each class is a
strict superset of the one below it. A modem advertises the highest
class it supports; the link negotiates down to the highest class both
ends share. The negotiation happens in-band immediately after carrier
is established, before any user data is exchanged.

The point of MNP is twofold: detect and retransmit bit errors caused
by line noise so the DTE never sees a corrupt byte, and (from class 5
onward) compress data on the fly so throughput exceeds the raw DCE
rate. A 2400 bps modem with MNP 5 can sustain roughly 4000 bps of
plain-text payload.

## Class hierarchy

### Class 1 – asynchronous, byte-oriented

The base class. Half-duplex, asynchronous, byte-by-byte ARQ. Each user
byte is wrapped in an MNP frame with a checksum; the receiver ACKs or
NAKs each frame. Throughput is roughly 70 percent of the raw line rate
because every byte carries framing overhead and the link reverses
direction for every ACK. Class 1 was a stopgap for modems with very
little processing power and was effectively obsolete by the time MNP
became widespread.

### Class 2 – asynchronous, full-duplex

Same byte-oriented framing as class 1, but full-duplex. The receiver
can ACK frames in the reverse channel without halting the forward
data stream. Sliding-window ACK is not yet present. Throughput is
roughly 84 percent of the raw line rate.

### Class 3 – synchronous framing on an asynchronous channel

The DTE-to-DCE link remains asynchronous (start and stop bits), but
the DCE-to-DCE link drops the start and stop bits and transmits the
payload synchronously. This saves the 20 percent overhead of async
framing on the wire. Quoted at about 108 percent of the raw async line
rate, or roughly an 8 percent throughput gain over class 2. Class 3
introduces HDLC-style flag-delimited frames.

### Class 4 – sliding window, adaptive packet sizing

Adds two important features:

- Adaptive packet assembly. The frame size grows on clean lines (up to
  256 bytes of payload) and shrinks on noisy lines (down to 64 bytes
  or less). Small frames retransmit faster but carry more overhead;
  large frames carry less overhead but cost more to retransmit. The
  adaptive logic tracks the error rate and picks a size.
- Data Phase Optimization. Repetitive header fields are elided after
  the first frame of a connection.

Throughput on a clean line is roughly 120 percent of the raw line
rate, a gain of about 20 percent over class 3. Class 4 is the floor
that ITU-T V.42 adopted as the fallback when LAPM negotiation fails.

### Class 5 – Huffman compression on top of class 4

Adds adaptive run-length and Huffman compression to the class 4 frame
payload. The Huffman tree is updated on the fly based on observed byte
frequencies. On English text and other low-entropy data the gain
approaches 2:1, giving roughly 200 percent of the raw line rate.

Class 5 is the source of the "%C0 / %C1" controversy. Compressing data
that is already compressed (a ZIP, an LHA, a JPEG) makes the output
slightly larger than the input because the Huffman codes for
already-near-random data are no better than 8 bits per byte and the
adaptive tree wastes bytes thrashing. A ZModem transfer of pre-zipped
files over an MNP 5 link is measurably slower than the same transfer
with compression disabled. Every Hayes-extended modem grew an `AT%C0`
command (disable compression) to handle this case, and BBS terminal
programs grew per-file-extension logic to issue `+++AT%C0` before
sending an archive.

Class 5 is incompatible with V.42bis in the sense that a link cannot
run both at once. Modems that support both negotiate one or the other
during connection setup, with V.42bis preferred.

### Class 6 – Universal Link Negotiation

Adds Universal Link Negotiation, allowing two modems to start at a
common low speed and then probe upward to the highest mutually
supported modulation. Also incorporates statistical duplexing for
V.29 (a half-duplex 9600 bps modulation) so it appears full-duplex to
the DTE. Class 6 was a transitional class on the path to V.32.

### Class 7 – enhanced data compression

Replaces the class 5 Huffman compressor with a higher-order Markov
predictor. Reported gains on text are around 3:1, slightly better than
V.42bis on the same data. Class 7 was Microcom-proprietary and
licensed less widely than class 5. It is rare in the field.

### Class 8 – reserved

Not publicly released. Some Microcom documentation references class 8
in passing; there is no corresponding deployed product.

### Class 9 – extended frame format and piggyback ACK

Extends the frame header to support larger sequence numbers (more
outstanding frames in flight) and piggyback ACKs (the ACK for a
received frame rides along with an outbound data frame). Combined
with class 7 compression. Class 9 was a Microcom-only feature in
their own Microcom-brand modems.

### Class 10 – Adverse Channel Enhancements

Designed for poor-quality circuits, originally cellular. Class 10
adds:

- Negotiated speed upshifts and downshifts mid-call without dropping
  the connection.
- Aggressive packet-size adaptation in response to bursty errors.
- Multiple transmit attempts during link setup before declaring the
  call failed.

Class 10 was the basis for what became MNP 10EC ("Enhanced Cellular")
used in the Motorola cellular modems and various PCMCIA cellular
cards of the mid-1990s.

## Frame format (class 3 onward)

```
+------+----------+--------+----------+-----+------+
| FLAG | HDR LEN  | HDR    | PAYLOAD  | FCS | FLAG |
| 7E   | 1 byte   | n bytes| variable | 2/4 | 7E   |
+------+----------+--------+----------+-----+------+
```

The framing is HDLC-derived. The header carries a frame-type byte (LR
for link request, LD for link disconnect, LT for link transfer, LA for
link acknowledge, LN for link attention, LNA for link attention
acknowledge) and type-specific parameters. The Frame Check Sequence is
CRC-16 (CCITT polynomial 0x1021) by default; class 9 can negotiate
CRC-32.

## Negotiation

When two MNP modems connect, the answering modem stays silent for a
brief window (typically S register S46 = 138 selects "extended
buffer", S48 = 7 selects "auto-detect"). The originating modem sends
a Link Request (LR) frame announcing its highest class. The answering
side replies with its own LR. Both ends settle on min(originator,
answerer) and enter LT (link transfer) mode.

If the answering modem hears no LR within the window, it falls back
to "normal" (unprotected) mode and the link runs without error
correction. The user-visible result code distinguishes the two:
`CONNECT 2400/REL` or `CONNECT 2400/MNP` for a reliable link,
`CONNECT 2400` alone for normal.

## Interaction with V.42

ITU-T V.42 was published in 1988 as the official error-correction
standard, with LAPM as its primary protocol. V.42 specifies an MNP 4
fallback so that a V.42 modem talking to an MNP-only modem can still
establish a reliable link. The "Alternative Procedure" annex of V.42
is essentially MNP classes 2 through 4 placed in the public domain.

A V.42 modem in `\N3` auto-reliable mode will:

1. Attempt LAPM negotiation (V.42 detection phase, ODP/ADP exchange).
2. On failure, attempt MNP 4 negotiation.
3. On failure, fall through to a normal (non-reliable) connection.

Compression layers stack on top: V.42bis runs over LAPM, MNP 5 runs
over MNP 4. You cannot run V.42bis over MNP 4 or MNP 5 over LAPM.

## AT commands controlling MNP

Standard across most Hayes-extended modems:

| Command | Meaning |
| --- | --- |
| `\N0` | Normal (no error correction) |
| `\N1` | Direct (no buffering, locked DTE rate) |
| `\N2` | Reliable: require MNP, hang up if unavailable |
| `\N3` | Auto-reliable: prefer LAPM, fall back to MNP 4, then normal |
| `\N4` | LAPM only: require V.42 |
| `\N5` | MNP only: require MNP 4 |
| `%C0` | Disable compression |
| `%C1` | MNP 5 compression only |
| `%C2` | V.42bis compression only |
| `%C3` | Either compression |

USR Couriers use a different syntax (`&M`, `&K`) covered in the
Courier reference.

## Practical notes

- MNP 5 compresses by frame, not by file. There is no decompression
  cost on the receive side beyond running the same adaptive Huffman
  decoder, but the encoder state is reset every time the modems drop
  carrier.
- MNP 5 imposes about 1 ms of additional latency per frame for
  Huffman tree updates. On interactive sessions this is unnoticeable.
- The `%C0` workaround for pre-compressed transfers became automatic
  in better terminal programs (Telix, Qmodem) which would inspect the
  file extension before issuing the upload command.
- A `CONNECT 2400/MNP5` result code indicates class 5 compression is
  active on a class 4 reliable link.

## References

- Microcom, Inc., "Microcom Networking Protocol: Class 5 Data
  Compression", 1985.
- ITU-T Recommendation V.42 (1988, revised 1993, 1996, 2002), "Error-
  correcting procedures for DCEs using asynchronous-to-synchronous
  conversion". Annex A describes the MNP-derived Alternative
  Procedure.
- Microcom MNP 10 specification, 1991.
- AT&T Paradyne, USR, Hayes, and Rockwell modem reference manuals,
  1989 through 1996, which document the `\N` and `%C` AT commands
  in identical or near-identical form.
