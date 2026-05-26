# YMODEM – Batch File Transfer Protocol

by Chuck Forsberg / Omen Technology, Inc.
Originally distributed as "YMODEM.DOC" with the Professional-YAM
reference implementation, June 1985.

YMODEM is an extension of XMODEM-1K that adds batch transfers, file
names, file sizes, modification times, and streaming. It is the direct
ancestor of ZMODEM and the protocol most often labelled "YMODEM" in BBS
terminal programs is actually what Forsberg called "YMODEM Batch with
1024 byte blocks and CRC".

## Vocabulary

YMODEM uses the XMODEM control characters and adds none of its own:

| Name | Value | Use |
| --- | --- | --- |
| SOH | 0x01 | Start of 128-byte block |
| STX | 0x02 | Start of 1024-byte block |
| EOT | 0x04 | End of file |
| ACK | 0x06 | Block received OK |
| NAK | 0x15 | Block bad / start (checksum) |
| CAN | 0x18 | Cancel (two consecutive) |
| 'C' | 0x43 | CRC mode start, also "ready for next file" |
| 'G' | 0x47 | YMODEM-G streaming mode start |

## Block format

Same as XMODEM-CRC / XMODEM-1K:

```
<SOH|STX> <blk#> <255-blk#> <data> <CRC-high> <CRC-low>
```

The CRC is CRC-16/XMODEM (polynomial 0x1021, initial 0x0000, big-endian
on the wire).

## Block 0 – header block

Each file transfer begins with a block numbered 0 containing the file
metadata as a null-terminated ASCII string:

```
filename\0length[space]modtime[space]mode[space]serial\0<pad>
```

| Field | Format | Description |
| --- | --- | --- |
| filename | ASCII, lower case preferred | Path-stripped name; no spaces |
| length | Decimal | File length in bytes |
| modtime | Octal | Modification time as Unix `time_t` |
| mode | Octal | Unix-style file mode bits (0 if unknown) |
| serial | Octal | Sender serial number, "0" if unused |

Only the filename is required; remaining fields are optional and
separated by single spaces. Unused fields and the unused tail of the
block are zero-filled (not space-filled – this is the one place YMODEM
differs visibly from XMODEM padding). The header block is typically
sent as a 128-byte block (SOH); a 1024-byte block (STX) is permitted
when needed.

Example header block (decoded):

```
README.TXT\013137 04567123456 100644 0\0\0\0\0...
```

## Handshake

```
Sender                                  Receiver
                                <-----  'C'                (CRC mode)
SOH 00 FF <header> <CRC>        ----->
                                <-----  ACK
                                <-----  'C'
STX 01 FE <1024 data> <CRC>     ----->
                                <-----  ACK
STX 02 FD <1024 data> <CRC>     ----->
                                <-----  ACK
...
EOT                              ----->
                                <-----  NAK
EOT                              ----->
                                <-----  ACK
                                <-----  'C'  (ready for next file)
SOH 00 FF <header for next>     ----->
...
SOH 00 FF <empty header> <CRC>  ----->  (filename = "\0", terminates batch)
                                <-----  ACK
```

Notes on the dance:

1. Receiver opens the session with 'C' (CRC mode). The sender responds
   with the block-0 header.
2. Receiver ACKs the header and immediately sends another 'C' to start
   data reception in CRC mode.
3. Data blocks follow, numbered from 01 and wrapping through 00 after
   FF.
4. EOT is acknowledged with a NAK first (this is a feature, not an
   error – it lets the receiver flush) then a second EOT is sent and
   ACKed.
5. The receiver sends 'C' again and the sender responds with the header
   for the next file, or with a block-0 whose filename field is empty
   (a single 0x00 byte) to signal end-of-batch.

## YMODEM-G

A streaming variant for use over error-correcting modems (MNP, V.42).
The receiver opens with 'G' (0x47) instead of 'C'. The sender then
transmits every block back-to-back without waiting for ACKs. If the
receiver detects any error it aborts the transfer with CAN CAN – there
is no retransmission, and YMODEM-G is therefore unsuitable for
unprotected lines.

YMODEM-G typically achieves >95% of raw link throughput because the
turnaround latency between every block is eliminated.

## File length handling

If the length field in block 0 is populated, the receiver MUST truncate
the output file to that length after the last data block. Without this,
files that do not end on a 128- or 1024-byte boundary contain trailing
pad bytes (usually 0x1A) that change the file's contents and checksums.

A length of "0" or an absent length field signals "unknown" – the
receiver should treat the file as terminated at EOT and accept whatever
pad bytes are present, exactly as in XMODEM.

## Mixed block sizes

A sender may freely switch between 128-byte (SOH) and 1024-byte (STX)
blocks within a single file. The conventional choice:

- All 1024-byte blocks for bulk data, to maximise throughput.
- A 128-byte block for the final block of a file to minimise padding
  waste when the remaining data is < 1024 bytes.

The block-0 header is conventionally 128 bytes.

## Cancellation and error handling

- Two CAN characters in a row abort the session; many implementations
  send eight CANs followed by eight backspaces for robustness against
  losing one or two CANs to noise.
- A block error (bad CRC, bad complement, character timeout) causes the
  receiver to NAK and the sender to retransmit. Up to 10 retries per
  block are permitted before the sender gives up.
- An unexpected block number that is neither the previous nor the
  current expected block is a fatal error.

## Compatibility footnote

What most BBS-era documentation calls "YMODEM" is the batch + 1024-byte
+ CRC combination described above. "True YMODEM" in Forsberg's original
documents required all of:

1. CRC-16 always (no fallback to checksum).
2. 1024-byte blocks (128-byte blocks are tolerated but discouraged).
3. Batch transfer with block-0 headers including filename and length.

Several terminal programs of the era shipped a "YMODEM" that was really
just XMODEM-1K-CRC (no batch support, no length-aware truncation).
Forsberg referred to those as "true XMODEM-1K, not YMODEM" and asked
that authors not call them YMODEM. The distinction matters: only true
YMODEM produces byte-exact files without trailing padding.
