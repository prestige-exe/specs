# XMODEM – Asynchronous File Transfer Protocol

Originally documented by Ward Christensen, 1 January 1982
("XMODEM Protocol Overview", revised from the original 1977 MODEM.ASM)

XMODEM (also called the "Christensen protocol" or "MODEM7 protocol") was
written by Ward Christensen in August 1977 for transferring files between
two CP/M microcomputers over an asynchronous serial link. It became the
de-facto file transfer protocol of the BBS era and is the parent of every
subsequent xMODEM/YMODEM/ZMODEM variant.

## Control characters

| Name | Value | Use |
| --- | --- | --- |
| SOH | 0x01 | Start of 128-byte block |
| EOT | 0x04 | End of transmission |
| ACK | 0x06 | Block received OK |
| NAK | 0x15 | Block bad / request retransmit / start request |
| CAN | 0x18 | Cancel transfer (two consecutive CANs end the session) |
| SUB | 0x1A | CP/M EOF, used as the pad byte inside the last block |

## Block layout (checksum variant)

```
<SOH> <blk#> <255-blk#> <128 data bytes> <checksum>
```

| Field | Size | Description |
| --- | --- | --- |
| SOH | 1 | 0x01 |
| Block number | 1 | Sequence number, starts at 01, wraps 00..FF |
| ~Block number | 1 | One's complement of block number (255 − blk#) |
| Data | 128 | Fixed 128 data bytes; last block padded with SUB (0x1A) |
| Checksum | 1 | 8-bit arithmetic sum of the 128 data bytes, modulo 256 |

The fixed 128-byte block size derives from the CP/M record size. The
protocol has no length field – the receiver always reads 128 bytes after
the header.

## Handshake and flow

```
Sender                                 Receiver
                                       (waits up to 10 s, sends NAK)
                                <----- NAK
SOH 01 FE <128 data> <csum>   ----->
                                <----- ACK
SOH 02 FD <128 data> <csum>   ----->
                                <----- ACK
...
EOT                            ----->
                                <----- ACK
```

1. The receiver initiates by sending NAK every 10 seconds for up to one
   minute (six NAKs) waiting for the first SOH from the sender.
2. The sender sends the first block when it sees the NAK.
3. After each block the receiver replies ACK on success or NAK on any
   error (bad SOH, wrong block number, bad complement, bad checksum,
   character timeout > 1 s within a block).
4. If the sender does not see ACK or NAK within 10 seconds it retransmits
   the last block. A block may be retransmitted up to 10 times before the
   sender aborts.
5. After the last data block the sender transmits EOT and waits for an
   ACK. If NAK arrives, EOT is sent again (up to 10 times).

## Block number handling

- Block numbers start at 01 and increment by one for every distinct
  block. After 0xFF the next block is 0x00.
- If the receiver gets a block whose number equals the previously
  acknowledged block, it is treated as a duplicate (line glitch on the
  ACK) and acknowledged silently without delivering the data again.
- Any other unexpected block number is a fatal error.

## Checksum

The classic checksum is the unsigned 8-bit sum of all 128 data bytes,
taken modulo 256. The header bytes (SOH, blk#, ~blk#) are not included.

```c
uint8_t cs = 0;
for (int i = 0; i < 128; i++) cs += data[i];
```

## Cancellation

Either side may abort the transfer by sending two CAN characters in a
row. Some implementations send up to eight CANs followed by eight
backspaces (0x08) to erase any "C" or NAK echoed by the peer.

## Limitations

- 128-byte blocks have ~3% overhead but a half-duplex turnaround for
  every block, limiting throughput on satellite or high-latency links.
- The 8-bit arithmetic sum will not detect about 0.4% of random errors,
  and many burst errors are completely transparent to it.
- Files must be transferred in 128-byte multiples; the final block is
  padded with 0x1A and the true end-of-file is the first 0x1A in that
  block. Binary files whose final 128 bytes legitimately contain 0x1A
  cannot be reliably round-tripped.
- The protocol carries no filename, size, date, or batch information.

## XMODEM-CRC (John Byrns, 1981)

A backward-compatible extension that replaces the 1-byte arithmetic sum
with a 16-bit CRC. Block layout becomes:

```
<SOH> <blk#> <255-blk#> <128 data bytes> <CRC-high> <CRC-low>
```

CRC parameters (the canonical "CRC-16/XMODEM" or "CRC-CCITT-FALSE-XMODEM"):

| Parameter | Value |
| --- | --- |
| Polynomial | 0x1021 (x¹⁶ + x¹² + x⁵ + 1) |
| Initial value | 0x0000 |
| Reflect input | No |
| Reflect output | No |
| XOR out | 0x0000 |
| Order | High byte transmitted first |

Handshake difference: the receiver initiates with the character 'C'
(0x43) instead of NAK to advertise CRC support. The sender, on seeing
the 'C', uses CRC blocks. If after three 'C's the sender still has not
responded, the receiver falls back to NAK and the checksum protocol.

```c
uint16_t crc = 0;
for (int i = 0; i < 128; i++) {
    crc ^= ((uint16_t)data[i]) << 8;
    for (int j = 0; j < 8; j++)
        crc = (crc & 0x8000) ? (crc << 1) ^ 0x1021 : (crc << 1);
}
```

## XMODEM-1K

A further extension (introduced with YMODEM but commonly bundled into
XMODEM implementations as "XMODEM-1K") that allows 1024-byte data
blocks. The header byte selects the block size:

| Header byte | Data size |
| --- | --- |
| SOH (0x01) | 128 bytes |
| STX (0x02) | 1024 bytes |

The CRC, sequence numbering, and handshake are otherwise identical to
XMODEM-CRC. Implementations may mix 128- and 1024-byte blocks within a
single transfer; the last block of a file is commonly the smaller size
to minimise padding waste.

## Notes for implementers

- The "1 second character timeout, 10 second block timeout" rule is
  important on real serial lines; on modern transports both may be made
  much shorter, but the relationship `block_timeout ≫ char_timeout`
  must be preserved or the receiver will give up mid-block.
- Always purge the input buffer before transmitting an ACK or NAK so
  that line noise does not become the next block number.
- A well-behaved sender debounces the initial NAK / 'C' from the
  receiver: noise on connect can produce phantom NAKs that, if
  immediately answered, will desynchronise the transfer.
